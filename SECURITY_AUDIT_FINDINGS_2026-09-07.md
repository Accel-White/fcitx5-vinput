# fcitx5-vinput 安全审计发现（2026-09-07）

审查基线：`a090b807d57ce6d0bea108b22f3249c0002c1228`

目标：记录本轮静态审查中已确认、值得独立跟踪的安全/隐私问题。当前 fork 的 GitHub Issues 功能关闭，因此先在独立分支保存，后续可拆分为独立 Issue。

---

## 1. [高] `context_lines=0` 仍持续持久化跨应用输入历史

### 概要

当所有 scene 都保持默认 `context_lines = 0` 时，Vinput addon 仍会监听 Fcitx `InputContextCommitString` 并把文本写入：

```text
${XDG_CACHE_HOME:-$HOME/.cache}/vinput/context.jsonl
```

`context_lines=0` 只阻止后处理阶段把历史注入 LLM，没有阻止收集和落盘；并且所有 scene 都为 0 时，历史截断逻辑也不会运行。

### 代码路径

- `src/common/scene/postprocess_scene.h`：`kDefaultContextLines = 0`
- `src/addon/core/vinput.cpp`：`onCommitString()` 对非空提交直接进入 `accumulateContextBuffer()`
- `src/addon/core/vinput.cpp`：`appendContextEntry()` 无条件追加；仅 `keep_lines > 0 && ++commit_write_count_ >= 100` 才截断
- `src/daemon/postprocess/post_processor.cpp`：`BuildContextPrefix(0)` 只在读取/注入端返回空

固定版本：

- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/addon/core/vinput.cpp
- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/daemon/postprocess/post_processor.cpp

### 已验证行为

函数级隔离测试使用合成标记观察到：

```text
context_lines=0: 101 synthetic commits persisted; truncate counter=0
LLM context helper: disabled => empty; later context=1 => old marker present
umask=022: cache directory=755, file=644
context_lines=3: 100 writes trimmed to 3 lines; chmod 0600 became 0644 after rotation
```

这不是密码框/真实桌面的端到端测试；结论限定为已经进入 Fcitx `CommitString` 事件的文本。

### 影响

- 用户未启用上下文功能时仍可能留下私人/工作输入历史。
- 全 0 配置下文件可能长期增长。
- 后续启用 `context_lines > 0` 时，之前禁用期间采集的内容仍可能进入 LLM 请求。
- 文件创建和轮转没有强制保持私有权限。

### 修复建议

1. 在收集、flush、落盘入口统一执行明确的 opt-in 开关。
2. 从启用切到关闭时停止 timer、清空待写 buffer，并明确旧缓存清理策略。
3. 敏感输入上下文应尽可能排除。
4. 缓存使用私有权限和安全临时文件。
5. 加入全 0 不落盘、关闭后不补写、禁用期间文本不被未来注入等回归测试。

---

## 2. [高] Provider / Adapter registry 脚本缺少完整性校验，且默认使用第三方代理回退源

### 概要

默认配置把以下地址作为 registry base URL：

```text
https://raw.githubusercontent.com/xifan2333/vinput-registry/main
https://gh-proxy.com/https://raw.githubusercontent.com/xifan2333/vinput-registry/main
https://ghfast.top/https://raw.githubusercontent.com/xifan2333/vinput-registry/main
```

实际 `vinput-registry` 的 `script_urls` 也为 provider/adapter 配置相同的第三方代理回退地址。

但是脚本 registry 结构没有 SHA256 / 签名字段，`DownloadScript()` 下载后直接赋予 executable bit，没有执行内容完整性验证。

### 代码路径

- `data/default-config.json`
- `src/common/registry/registry_scripts.h`：`RegistryEntry` 只有 `script_urls`，没有哈希/签名字段
- `src/common/registry/registry_scripts.cpp`：`DownloadScript()` 直接 `DownloadFile()` + `EnsureExecutable()`
- `src/cli/config/asr_actions.cpp` / `llm_actions.cpp`、GUI Resource page 都直接使用该下载路径

固定版本：

- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/data/default-config.json
- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/common/registry/registry_scripts.cpp
- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/common/registry/registry_scripts.h

### 风险链

1. 官方 GitHub Raw 路径不可达或失败。
2. 下载逻辑回退到第三方代理。
3. 代理若被攻陷、DNS/网络链路被劫持或返回恶意内容，客户端没有独立完整性校验可发现变化。
4. 返回内容被保存并标记为可执行脚本。
5. 用户启用 ASR provider 或启动 LLM adapter 后，以当前用户身份执行该脚本。

Provider/Adapter 还可能拿到 API key、音频、文本等敏感运行时数据，因此影响不只是任意本地代码执行，还可能伴随凭证/数据泄露。

### 修复建议

1. `RegistryEntry` 增加强制 `sha256`（至少对官方可执行脚本）。
2. 下载完成后必须在落盘/启用前校验哈希。
3. 更优方案：registry 元数据做签名，客户端内置可信公钥验证。
4. 第三方代理不应作为默认可信可执行代码源；如保留，必须由独立哈希/签名兜底。
5. 记录并展示最终 resolved URL，代理回退时给用户明确警告。
6. 增加“代理返回内容与官方内容不同必须拒绝安装”的回归测试。

---

## 3. [高] `VINPUT_DEBUG=1` 会把 LLM API Key 和请求正文原样写入日志

### 概要

`src/daemon/postprocess/post_processor.cpp` 的 `LogLlmRequest()` 会遍历并打印全部 HTTP headers。代码注释明确说明 API key 会以明文形式包含在日志中。

当 `VINPUT_DEBUG=1` 时，日志会进入 stdout / journald；官方文档也建议用户在排障时开启该环境变量。

### 代码路径

`LogLlmRequest()`：

```cpp
// Debug-only: dump headers and body verbatim. The API key is included in
// plain text ...
```

随后打印：

```text
headers=[Authorization: Bearer ...]
LLM request body: ...
```

固定版本：

- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/daemon/postprocess/post_processor.cpp
- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/data/man/en/vinput-daemon.1.md

### 影响

排障时用户经常会把 `journalctl` 输出贴到 GitHub Issue、论坛、聊天或发给其他人。此时日志可能直接泄露：

- LLM `Authorization: Bearer ...` API key
- ASR/用户文本
- context history
- prompt/request body 中的选中文本

### 修复建议

1. 无论 debug 是否开启，都必须对 `Authorization`、API key、token 等 header 做不可逆脱敏。
2. 默认不要输出完整 request body；提供单独的更高风险 opt-in，并明确告警。
3. 对 body 中可能存在的 secret 字段增加 redaction。
4. 增加测试确保 debug 日志中不存在配置中的真实 key/token。

---

## 4. [中高] Remote Input 监听全部网卡并通过明文 HTTP/WS 发送长期 API Key

### 概要

`RemoteTextService` 使用：

```cpp
addr.sin_addr.s_addr = htonl(INADDR_ANY);
```

因此服务默认监听所有 IPv4 网卡，而不是仅 localhost。

内置页面根据页面协议决定 WebSocket：

```js
const proto = location.protocol === 'https:' ? 'wss:' : 'ws:'
ws = new WebSocket(proto + '//' + location.host + '/ws')
```

服务本身没有 TLS 终止，因此典型 LAN 使用路径是 HTTP + `ws://`。客户端随后直接发送：

```js
send({ type: 'auth', api_key: apiKey })
```

### 代码路径

- `src/daemon/remote/remote_text_service.cpp`

固定版本：

- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/daemon/remote/remote_text_service.cpp

### 影响

在酒店 Wi-Fi、公司共享网络、公共 LAN 等不可信二层网络中，能观察明文流量的攻击者可能截获 Remote API Key；随后可冒充远程输入端连接服务，并在存在活动 VInput session 时发送/修改文本。

### 修复建议

1. 默认只绑定 `127.0.0.1`，LAN 模式必须明确 opt-in。
2. LAN 模式优先使用 TLS/WSS；可提供证书指纹/二维码配对。
3. 不使用长期静态 API key 作为浏览器端明文认证值；改用一次性或短期 token。
4. 对认证失败、暴力尝试加入速率限制。
5. UI 明确显示当前监听地址及“明文 LAN”安全警告。

---

## 5. [中] Adapter PID 文件未验证进程身份，stale PID 复用时可能误杀无关进程

### 概要

Adapter 运行状态只持久化一个数字 PID：

```text
$XDG_RUNTIME_DIR/vinput/adapters/<adapter>.pid
```

`ProcessExists()` 仅通过 `kill(pid, 0)` 判断对应 PID 是否存在，没有验证该进程是否仍是原来的 adapter。

`Stop()` 随后直接对该 PID 发送 `SIGTERM`，等待后仍存在就发送 `SIGKILL`。

### 代码路径

- `src/common/llm/adapter_manager.cpp`

固定版本：

- https://github.com/Accel-White/fcitx5-vinput/blob/a090b807d57ce6d0bea108b22f3249c0002c1228/src/common/llm/adapter_manager.cpp

### 风险场景

1. daemon / adapter 异常退出，PID 文件残留。
2. 系统之后复用了该 PID 给另一个同用户进程。
3. `vinput adapter stop <id>` 读取旧 PID。
4. `ProcessExists()` 返回 true。
5. Vinput 对无关进程发 `SIGTERM` / `SIGKILL`。

可能造成编辑器、构建任务、数据库等无关进程被终止并导致数据丢失。

### 修复建议

1. PID 文件同时记录进程启动时间（如 `/proc/<pid>/stat` starttime）或随机 instance token。
2. Stop 前验证 PID + starttime + executable/command identity。
3. 最好采用 pidfd（Linux）或由长期 supervisor 持有真实 child handle，而不是只信任磁盘 PID。
4. stale PID 不匹配时只清理 PID 文件，不发送信号。
5. 增加 PID reuse / stale pid 回归测试。

---

## 优先级建议

### P0 / P1

1. Registry 可执行脚本无完整性校验 + 默认第三方代理回退
2. `context_lines=0` 仍持续持久化输入历史

### P1

3. `VINPUT_DEBUG` 明文记录 API Key / request body
4. Remote Input 明文 WS 发送 API Key

### P2

5. Adapter stale PID / PID reuse 误杀无关进程

---

## 备注

- 以上问题在本 fork `main` 与上游同一审查提交上确认。
- 已对 fork 的现有 Issue 做关键词排重；当前 fork Issues 功能关闭，因此没有可创建的 Issue 记录。
- 该文档用于安全修复跟踪，不代表已完成完整攻防测试。