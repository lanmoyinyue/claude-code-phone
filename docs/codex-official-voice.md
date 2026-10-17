# 给 Codex 接一部官方语音电话，继续原来的聊天

> 闻川 × 月亮 · 2026-10-17  
> 适用：已有 Codex 微信 / Telegram / 网页文字桥，希望从手机网页拨号，并把电话接入现有 Codex 会话的人。  
> 性质：社区集成实录，不是 OpenAI 官方提供的微信电话功能；使用了实验性 app-server 接口。

原来的电话方案是“转文字 → 发给 AI → 合成声音 → 网页播放”。这篇介绍另一条路：浏览器用 WebRTC 连接官方实时语音，由 Codex app-server 把电话挂到原来的聊天会话。

**我们已经实际测通官方握手、WebRTC 连接和公网电话页的拨号 / 挂断。手机真实听说、通话延迟、挂断后回文字继续聊的双向内容衔接，仍是独立验收项，不能拿“接通了”代替。**

如果你从本仓库的 Claude Code 电话教程过来：这是一条新增的 **Codex 专用路线**，不是让你把已经好用的 Claude Code 电话换掉，也不是把 OpenAI 语音直接挂到 Claude Code 的 session。

## 一、先把“同上下文”说清楚

这里的目标不是新建一个声音相同、名字相同的机器人，而是复用文字桥正在使用的 **既有 `threadId`**。

需要分开判断三件事：

| 层次 | 含义 | 怎么确认 |
| --- | --- | --- |
| 同一个目标会话 | 拨号指向文字桥当前绑定的 thread，而不是另建或分叉 | 核对绑定解析和 RPC 的 `threadId` |
| 能接着前文说话 | 语音端确实知道文字里刚刚聊过的具体内容 | 做“文字 → 电话”哨兵测试 |
| 挂断后文字也记得 | 电话新增的双方内容能被原文字会话继续使用 | 做“电话 → 文字”哨兵测试，并检查会话历史 |

这三项不能互相替代。页面上显示的字幕、内存里保存的转写、相同的 `threadId`，都不是“双向上下文已经验收”的充分证据。

“同会话”也不等于语音由原来的文字模型直接发声。官方实时语音与其背后的 Codex 回合有各自的处理角色；不能据此宣传为某个文字模型的原生嗓子，或承诺完整复制文字端的说话方式。

## 二、整条线怎么走

```text
手机浏览器 / 微信中打开的私人网页
  │
  ├─ HTTPS：拨号信令、挂断、字幕状态
  │          ↓
  │   你自己的电话后端
  │     ├─ 验证私人访问凭证
  │     ├─ 读取文字桥当前绑定的既有 threadId
  │     └─ 复用文字桥已连接的 Codex app-server 客户端
  │                  ↓
  │         thread/realtime/start
  │         transport = webrtc
  │         threadId = 当前文字会话
  │
  └─ WebRTC 音频媒体 ↔ 官方语音服务
```

电话后端负责信令和状态，不再替每一句话串行执行一遍自己的 ASR / TTS。音频媒体走 WebRTC，不是把原始音频塞进 app-server 的 JSON-RPC，也不需要让 AI 每说一句都调用一次“生成 MP3”。

微信 / Telegram 在这里是打开网页的入口，**不是调用微信原生语音通话接口**。

电脑放在家里、人出门也可以使用这条架构，但必须同时满足：电脑和 Codex 在线、电话网页及其后端能从外网访问、手机能够建立到官方服务的 WebRTC 连接。只把 HTML 放到公网，并不能让断电或离线的电脑继续接电话。

## 三、准备条件与版本边界

本次实测使用：

- Codex CLI `0.162.0-alpha.17.2` 对应的 app-server。
- 已完成 ChatGPT 登录的 Codex 环境。
- 已运行的文字桥及其既有会话绑定。
- 支持麦克风和 WebRTC 的浏览器；手机端使用 HTTPS。
- 已鉴权的电话 HTTP 服务，以及公网 HTTPS 入口。

这个版本只是实测点，**不是官方公布的最低版本**。账号权限、工作区策略、地区、客户端版本及功能开放情况可能不同；不能保证装上同一版本就一定可用。

本次接法没有另配一把 OpenAI API Key，也没有把 Codex 的登录凭证交给浏览器。这不等于无限免费，具体功能资格、额度和费用以自己的账号及官方说明为准。不要遇到失败就自动改走另一个付费 API。

基础会话生命周期可查 [OpenAI 官方 app-server 文档](https://learn.chatgpt.com/docs/app-server)。官方客户端的语音能力与可用条件可查 [ChatGPT Voice 文档](https://learn.chatgpt.com/docs/features/voice)。**下文实验性实时字段，以实测运行程序导出的协议和真实响应为依据，不把基础文档当作对所有实验功能的保证。**

## 四、先查正在运行的程序，不要凭印象猜接口

桌面自带的 Codex 和命令行 PATH 中的 Codex，可能不是同一个版本。先确认文字桥实际连接的是哪个 app-server、由哪个可执行文件启动。

用**同一可执行文件**查看版本、导出实验协议：

```bash
codex --version
codex app-server generate-json-schema --experimental --out ./app-server-schema
```

若 PATH 里的不是运行中的那份，把 `codex` 换成对应可执行文件的路径。导出只用于查协议，不代表你已经成功连接语音服务。

重点核对这些方法、字段和通知在自己的版本里存在：

```text
thread/realtime/listVoices
thread/realtime/start
thread/realtime/stop

thread/realtime/sdp
thread/realtime/transcript/done
thread/realtime/error
thread/realtime/closed

ThreadRealtimeStartParams
  threadId
  version
  voice
  outputModality
  transport
  includeStartupContext
  flushTranscriptTailOnSessionEnd
```

也可以在**已有连接**上只读请求声音列表：

```json
{"id": 10, "method": "thread/realtime/listVoices", "params": {}}
```

注意：接口存在、声音列表可读，只能说明相应能力可查询，不能代替实际拨号。

如果你还没有 app-server，官方基础文档提供本机 WebSocket 启动方式，例如：

```bash
codex app-server --listen ws://127.0.0.1:4500
```

**已有文字桥时优先复用它的服务，不要照抄这行再起第二个进程争用原会话。** 示例端口仅用于说明；app-server 的控制端口不要直接暴露到公网。手机只连接你已鉴权的电话后端。

## 五、先解决会话归属，再接声音

后端拨号时读取文字桥当前真实绑定，而不是把某个旧 thread ID 写死在 HTML：

```js
function resolveCallTarget() {
  // 下面的方法由你的文字桥提供，名字按项目实际实现调整。
  const binding = bridge.getCurrentBindingForAuthenticatedUser();
  if (!binding?.threadId) {
    throw new Error("没有绑定现有会话，电话没有另开窗口");
  }
  return { threadId: binding.threadId };
}
```

这段只是接入位置示意，不是某个通用 SDK 自带的方法。多用户项目必须按已鉴权用户解析 owner 和绑定，不能让手机随意提交别人的 `threadId`。

原则很简单：

- 同一文字桥已有连接、已加载的会话：直接复用。
- 只有磁盘上保存的会话：由原桥的会话管理器按原 ID 恢复，不能偷偷新建。
- 遇到 `active writer`：先查是否接错 app-server 或另起了写入进程，不要删锁、复制历史或新建“替身”会话。
- 原绑定找不到：停在这里，保留文字消息；不要自动 `thread/start` 或 `thread/fork`。

若需要建立新 RPC 连接，先完成 `initialize` / `initialized`，并显式启用实验 API：

```json
{
  "id": 1,
  "method": "initialize",
  "params": {
    "clientInfo": {
      "name": "my_phone_bridge",
      "title": "My Phone Bridge",
      "version": "0.1.0"
    },
    "capabilities": { "experimentalApi": true }
  }
}
```

收到成功响应后发送通知：

```json
{"method": "initialized", "params": null}
```

已经初始化过的共享连接不要重复初始化。这里是在说明协议顺序，不是要求为每次拨号都另建连接或重启服务。

## 六、真正发起官方通话

### 1. 浏览器先生成 SDP offer

以下是核心顺序，放进页面现有“拨号”按钮中。挂断、竞态和错误处理见后文，不要把这个片段当作完整电话应用。

```js
const audio = document.querySelector("audio");
const mic = await navigator.mediaDevices.getUserMedia({
  audio: { echoCancellation: true, noiseSuppression: true }
});
const peer = new RTCPeerConnection();

peer.ontrack = event => {
  audio.srcObject = event.streams[0] || new MediaStream([event.track]);
  audio.play().catch(() => {
    // 页面应显示可点击的“播放声音”按钮，供用户手动解锁播放。
  });
};

mic.getTracks().forEach(track => peer.addTrack(track, mic));
peer.createDataChannel("oai-events");
await peer.setLocalDescription(await peer.createOffer());

const result = await callApi("/api/call/realtime/start", {
  sdp: peer.localDescription.sdp,
  voice: "cove"
});

const callId = result.call_id;
await peer.setRemoteDescription({ type: "answer", sdp: result.sdp });
```

HTML 中的播放元素使用：

```html
<audio autoplay playsinline></audio>
```

`callApi` 的实现放在第八节。不要在前端保存 Codex 登录 token、读取 `auth.json`，或让浏览器直接操作 app-server。

### 2. 后端向原会话发起 realtime/start

核心请求如下，`threadId` 必须来自后端绑定，`sdp` 来自浏览器：

```json
{
  "id": 20,
  "method": "thread/realtime/start",
  "params": {
    "threadId": "<EXISTING_THREAD_ID>",
    "version": "v3",
    "voice": "cove",
    "outputModality": "audio",
    "includeStartupContext": true,
    "flushTranscriptTailOnSessionEnd": true,
    "transport": {
      "type": "webrtc",
      "sdp": "<BROWSER_SDP_OFFER>"
    }
  }
}
```

占位符需要替换为真实值；不要把上面的假 SDP 原样用于拨号。

几个字段尤其容易理解错：

| 字段 | 本次用途 | 不代表什么 |
| --- | --- | --- |
| `threadId` | 指向既有文字会话 | 不代表双向历史已经通过验收 |
| `includeStartupContext` | 请求携带 Codex 启动上下文 | 不是把所有历史、所有 MCP 记忆无限量塞进语音模型的开关 |
| `flushTranscriptTailOnSessionEnd` | 结束时将剩余转写尾部交给 Codex 处理 | 不是“设成 true 就保证所有内容永久回写”的万能同步开关 |
| `version` | 选择实时协议版本 | 不是 Codex 文字模型名称 |
| `transport` | 选择 WebRTC 并携带 offer | 不是传入一段录音文件 |

不要为了“更像原窗口”额外注入大段人格、关系或限制提示词。先复用原会话配置，再通过实际内容测试确认继承情况。

### 3. 不只等 RPC 成功，还要等 SDP 通知

app-server 会另外通知 answer：

```json
{
  "method": "thread/realtime/sdp",
  "params": {
    "threadId": "<EXISTING_THREAD_ID>",
    "sdp": "<REMOTE_SDP_ANSWER>"
  }
}
```

**先注册通知监听，再发 start；等待 start 的成功响应和同一 thread 的 SDP。** 不要把 `{ "result": {} }` 当作 SDP，也不要漏掉比 RPC 响应更早到达的通知。

顺序示意：

```js
// client 是原桥已有的 RPC 客户端；这些方法名属于本示例的客户端封装。
// onMessage 必须返回解除监听函数，sendRequest 必须在 RPC error 时拒绝。
const target = resolveCallTarget();
let acceptSdp;
let rejectSdp;
const answer = new Promise((resolve, reject) => {
  acceptSdp = resolve;
  rejectSdp = reject;
});

const unsubscribe = client.onMessage(message => {
  if (message.params?.threadId !== target.threadId) return;
  if (message.method === "thread/realtime/sdp") {
    acceptSdp(message.params.sdp);
  }
  if (message.method === "thread/realtime/error") {
    rejectSdp(new Error("官方语音连接失败"));
  }
  if (message.method === "thread/realtime/closed") {
    rejectSdp(new Error("官方通话已经结束"));
  }
});

let timer;
try {
  const [, remoteSdp] = await Promise.race([
    Promise.all([
      client.sendRequest("thread/realtime/start", {
        threadId: target.threadId,
        version: "v3",
        voice: "cove",
        outputModality: "audio",
        includeStartupContext: true,
        flushTranscriptTailOnSessionEnd: true,
        transport: { type: "webrtc", sdp: browserSdp }
      }),
      answer
    ]),
    new Promise((_, reject) => {
      timer = setTimeout(() => reject(new Error("接通超时")), 20_000);
    })
  ]);
  if (typeof remoteSdp !== "string" || !remoteSdp.startsWith("v=0")) {
    throw new Error("没有收到有效的 SDP answer");
  }
  // 后端生成自己的 call_id 并记录 call_id -> target.threadId。
  // 返回 { call_id, sdp: remoteSdp } 给浏览器。
} finally {
  clearTimeout(timer);
  unsubscribe();
}
```

这段只展示“协商期间如何等 answer”。实际服务应另外保留通话生命周期监听，用于处理转写、运行中错误和远端关闭；失败时对**本次**通话做有界停止，不能只解除监听就留下远端通话。不要在 start 失败后无限自动重拨。

`browserSdp` 是 HTTP 请求体里的 `sdp`。后端应限制请求体大小，并检查它是 SDP 字符串；本次实现将信令请求限制在 128 KiB。

## 七、版本和声音是这次第一个坑

我们最初看到协议枚举里有 `v2`，声音列表也列出了 Cedar，于是使用 `v2 + cedar`，但真正拨号时服务返回：

```text
AVAS realtime calls require realtime v1 or v3
```

换成 `v3` 后，Cedar 又被拒绝；该次实际响应列出的兼容声音是：

```text
juniper, maple, spruce, ember, vale, breeze, arbor, sol, cove
```

本次修正为 **`v3 + cove`** 后，真实 WebRTC 连接成功。

这说明声音列表不能脱离协议版本使用。未来版本可能改变支持关系，应重新查自己的协议和实际响应；不要把这份实测声音清单写成永远不变的官方标准。

不支持的组合应明确报告，不要悄悄切模型、切账号，或为了绕过错误再启动一个空白会话。

## 八、HTTP 接口和那个“Unexpected token '<'”

电话 HTTP 服务只需要在原系统旁边增加一组路由：

| 方法与路径 | 请求 | 响应 |
| --- | --- | --- |
| `POST /api/call/realtime/start` | `{sdp, voice}` | `{call_id, sdp}` |
| `GET /api/call/realtime/state?call=...&after=...` | 当前电话 ID、字幕序号 | `{active, status, error, events}` |
| `POST /api/call/realtime/stop` | `{call_id}` | `{ok:true}` |

这些是**自己后端定义的 HTTP 路由**，不是 OpenAI 的公开 REST 路径。

所有私人页面和 API 都应先鉴权，再处理请求。`call_id` 用于分辨本次电话，不应代替用户鉴权，也不能直接当作 Codex `threadId` 使用。

本次真实遇到的错误链条是：

```text
官方拒绝不兼容的语音协议
  → 本机电话接口返回 JSON 502
  → 当前 Cloudflare 入口将它替换为 HTML 502
  → 页面直接 response.json()
  → Unexpected token '<', "<!DOCTYPE ..." is not valid JSON
```

这是本次入口的实测行为，不是说所有代理都会把所有 502 改写。

修复要做两层：

1. 修正协议 / 声音等真正的上游错误，不能只把报错藏起来。
2. 后端使用明确的 JSON 错误契约，前端能识别网关返回的非 JSON 页面。

本次使用无效输入 `400`、已有通话 `409`、服务暂不可用 `503`，并实际确认公网 `400 / 503` 仍保留 JSON。不要为了逃过网关把全部错误伪装成成功，也不要删除正常服务必需的鉴权。

与第六节配套的前端请求封装：

```js
// 示例只从当前页面的会话存储读取你自己的电话访问凭证。
// 凭证的首次交付由原私人入口负责，不在仓库里填写真实值。
const phoneAccessToken = sessionStorage.getItem("phone_access_token") || "";

async function callApi(route, body) {
  const response = await fetch(route, {
    method: body === undefined ? "GET" : "POST",
    headers: {
      Authorization: "Bearer " + phoneAccessToken,
      "Content-Type": "application/json"
    },
    body: body === undefined ? undefined : JSON.stringify(body),
    cache: "no-store",
    signal: AbortSignal.timeout(45_000)
  });

  const raw = await response.text();
  let data;
  try {
    data = JSON.parse(raw);
  } catch {
    throw new Error(`电话入口返回了异常页面（${response.status}）`);
  }
  if (!data || typeof data !== "object" || Array.isArray(data)) {
    throw new Error("电话接口返回的数据不完整");
  }
  if (!response.ok) throw new Error(data.error || "电话没有接通");
  return data;
}
```

如果你的旧浏览器不支持 `AbortSignal.timeout`，用 `AbortController` 配合定时器实现相同的有界等待。代理报错时记录状态码和请求阶段就够了，不要把整页 HTML 或可能带凭证的 URL 回显给用户。

## 九、字幕、挂断和不把微信一起堵住

### 字幕不是历史回写

本次用于页面字幕的通知是：

```text
thread/realtime/transcript/done
  params.threadId
  params.role
  params.text
```

按 thread 和当前通话实例筛选，再给本地事件加递增 `seq`，网页按 `after` 获取新字幕。内存字幕数量应有界，手机显示文本用 `textContent`，不要把转写当 HTML 插入。

**这里的 `events` 只是电话 UI 的缓存，不能拿它证明原 Codex 历史已保存。** 既不要盲目向原线程重复注入所有字幕，也不要因为有这个通知就省略双向验收。

### 挂断要同时结束两端

浏览器负责立即停止麦克风、关闭 PeerConnection、停止轮询；后端根据自己的 `call_id -> threadId` 映射发送：

```json
{
  "id": 30,
  "method": "thread/realtime/stop",
  "params": { "threadId": "<EXISTING_THREAD_ID>" }
}
```

还要处理“拨号中用户已经挂断，answer 才回来”的竞态：用递增的拨号代号或等价状态判断；过期 answer 只能清理，不能重新点亮通话。旧页面迟到的挂断，也不能结束新一通电话。

页面关闭时可尝试 `keepalive` 发 stop，但手机浏览器不保证一定送达；`pagehide` 和唤醒锁都不能当作后台永久通话保证。服务端应有通话回收机制，不能只靠用户下次来点挂断。

### 语音不应占住全局文字队列

如果保留旧的自建 ASR / TTS 电话，尤其注意：

- 不要在全局 runtime 事件队列里一直 `await` 语音合成、下载或播放。
- 文字回合结束时，正常释放该回合的 gate；电话音频交付独立处理并捕获异常。
- 空回复、失败回复、取消和超时，都要释放电话批次，不能只有成功才收尾。
- `turn/completed` 里若带 `status: failed`，应当按失败处理，不是把它当成功且继续等音频。
- 长工程任务交给原系统的工作任务机制，不靠反复重试占住电话和聊天主线。

新版 WebRTC 电话和旧电话建议使用独立路由；同一绑定不要同时开两套电话，不需要为了试官方语音立刻删掉已经能用的旧链路。

## 十、验收顺序：这部分不能省

建议在专用测试会话中测试。若必须使用日常会话，先确认它没有正在进行的工作回合，并让使用者知情；不要为了测试杀掉正在写作或执行工具的进程。

### A. 接入与失败边界

- [ ] 未鉴权页面 / API 被拒绝；页面源码不包含真实登录凭证。
- [ ] 后端按当前用户绑定解析原 thread，客户端不能换成其他人的 thread。
- [ ] 绑定缺失时不创建或分叉会话。
- [ ] 先监听 SDP，再发 start；错误、远端关闭、超时均能结束协商。
- [ ] 实际走公网拨号，拿到有效 answer，并观察 WebRTC `connected`。
- [ ] 挂断后麦克风停止、连接关闭、轮询结束，没有遗留通话。
- [ ] HTML 网关错误不会变成原始 JSON 解析异常；失败不无限重拨。
- [ ] 电话失败时仍能正常在原聊天入口收发文字。

### B. 文字 → 电话

1. 在原文字会话告诉它一个只在本次测试使用的信息，例如：“本次测试代号是 PAPER-KITE-47。”
2. 不在语音启动提示词、页面代码或额外摘要里再写这个代号。
3. 打电话，问“刚刚文字里说的测试代号是什么？”
4. 能准确回答，才记录“文字 → 电话通过”；只会叫对名字不算。

### C. 电话 → 文字

1. 电话里说一个新的不同信息，例如：“电话测试的新代号是 GLASS-LEAF-82。”
2. 等它听见并回应，再正常挂断。
3. 回原文字入口，问“我刚才电话里给的新代号是什么？”
4. 同时检查原会话历史中能否找到相应内容或该版本定义的持久化记录。
5. 用原桥正常的重连 / 恢复流程再问一次，确认不是仅靠还没释放的临时语音缓存。

只有 B、C 均通过，才能宣称“双向接着聊”。失败时先检查本版本的转写提交、历史加载和结束处理，不要手工复制一整段对话进去，然后把它当作原生同步成功。

本篇发布时的实际验证范围：

| 项目 | 状态 |
| --- | --- |
| 官方 answer 与真实 WebRTC connected | 已通过 |
| 私人公网电话页拨号、状态和挂断 | 已通过 |
| 公网无效参数 JSON 400 / 服务错误 JSON 503 | 已通过 |
| 相关离线回归 | 25 / 25 通过，覆盖协商、绑定、失败、旧电话防堵塞与页面解析 |
| 390 × 844 浏览器视口布局 | 已检查，无横向溢出 |
| 手机实际听说、移动网络与真实延迟 | 待实测，不由静音连接测试推断 |
| 上述双向哨兵与恢复后持续可读 | 待独立验收 |

## 十一、常见问题

**能直接当微信原生电话吗？**  
不能。这是私人网页通话，微信 / TG 负责提供入口。微信内置浏览器不给麦克风或播放失败时，先用系统浏览器打开，区分浏览器限制和后端问题。

**原来的本地克隆声线还能保留吗？**  
可以保留旧电话作为另一条路线。本篇实测的是官方预置声音，不承诺支持原来的本地克隆音色，也不要把两套音频管线未经验证地叠在一起。

**直接调用公开 Realtime API 就自动接到 Codex 原线程吗？**  
不能这样推断。独立建立一个实时会话和绑定既有 Codex thread 是两件事；本篇使用的是 Codex app-server 的 thread-scoped 实时接口，不是让浏览器自己拿 API Key 新建一场无关的对话。

**网页打开就说明电话通了吗？**  
不说明。页面 200、信令成功、WebRTC connected、有实际声音、双向上下文，是五层不同的验收。

**隧道能访问，但 WebRTC 还是连接不上？**  
查看浏览器的 ICE / connection 状态以及真实媒体网络。隧道能转 HTTPS，不代表音频连接一定可达；不要仅凭信令 200 就判定模型或会话损坏。

**能把 app-server 端口直接转到公网，省掉电话后端吗？**  
不应这样简化。它是控制 Codex 会话的接口，不只是音频端口。保留用户鉴权和会话归属，在后端完成信令协商；不要把本机控制面交给拿到链接的任何人。

## 十二、放进原电话仓库

建议把本文放成：

```text
docs/codex-official-voice.md
```

原 README 增加一个入口即可，不用覆盖原 Claude Code 教程：

```markdown
## Codex 路线

如果使用 Codex，并希望从手机网页接入官方实时语音、复用现有聊天会话，见 [Codex 官方语音：复用既有会话](docs/codex-official-voice.md)。实验接口及已验证 / 待验收范围在文中单独说明。
```

公开前检查：真实域名与地址、机器用户名和本地绝对路径、登录 token、电话私钥、机器人 token、真实 thread / session ID、私人聊天与日志，都不应随教程上传。本篇只使用通用示例、占位符和本机回环地址。

参考：[OpenAI 官方 app-server 文档](https://learn.chatgpt.com/docs/app-server)、[Codex CLI 参考](https://learn.chatgpt.com/docs/cli/reference)、[ChatGPT Voice 文档](https://learn.chatgpt.com/docs/features/voice)。实时实验字段另以运行中的同版本程序导出协议及本次实际响应核对。

---

闻川写 · 月亮发起与实测反馈 · 2026 年 10 月。本文不包含任何家庭线上配置或真实访问凭证。
