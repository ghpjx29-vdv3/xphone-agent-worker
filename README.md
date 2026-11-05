# XPhone Agent Worker

> **中文** | [English](#english)

一部住着 AI 角色的手机，由你自己的个人 AI 代理（Muse、GPT 等）驱动。

XPhone is a phone inhabited by AI characters, powered by your own personal AI agent (Muse, GPT, and more).

---

## 中文

### 这是什么

**XPhone** 是一部虚拟手机：里面有能聊天的 AI 角色、故事世界、地图、社交动态、备忘录。你的个人 AI 代理就是这部手机背后的"操作系统"——它替角色回消息、推进剧情、发社交动态。

**Agent Worker** 是跑在 Cloudflare Workers 上的 MCP 服务器，负责三件事：

1. **认证**：验证你的个人代理身份，发放配对凭证；
2. **转发**：在手机客户端和你的个人代理之间传递加密事件与动作；
3. **隔离**：Worker 只转发密文，读不到任何剧情内容。

```
手机客户端 (GIGI / phone.xenoflex.xyz)
        ↕ 加密事件 / 加密动作
XPhone Agent Worker (Cloudflare Workers, MCP)
        ↕ WebSocket / MCP
你的个人代理网关 (xphone-gateway, 跑在你自己的机器上)
        ↕
你的个人 AI 代理 (Muse / GPT)
```

这个公开仓库只放**公开的工具契约**（`xphone-tool-contract.md`），即个人代理在 XPhone 里能调什么、怎么调。Worker 的完整实现是私有的，部署在 Cloudflare 上，不在这里。

相关仓库：网关（跑在你机器上的那一半）→ [xphone-gateway](https://github.com/ghpjx29-vdv3/xphone-gateway)。

### 快速上手：配对你的个人代理

整个过程只需要做两件事：**部署网关，输入配对码**。其余步骤由你的个人代理自动完成。

#### 1. 部署网关

对你的个人代理（Muse）说一句话：

> 帮我部署 github.com/ghpjx29-vdv3/xphone-gateway 并连接 xphone

它会拉取网关代码、装好依赖、装好常驻服务。网关需要一台长期在线的 Linux 机器（Muse 用户直接用自己的 Muse VM 即可，不需要另购 VPS）。

#### 2. 配对（8 位配对码）

网关装好后，在网关机器上运行：

```bash
.venv/bin/python pair.py
```

终端会输出一个 **8 位配对码（例如 `3EY64LSN`），有效期 5 分钟**。在 XPhone / GIGI 手机前端打开「个人代理配对」页，输入配对码并确认授权。

确认后网关自动完成兑换，凭证落盘到 `~/.config/xphone/`（权限 0600）：

| 文件 | 用途 |
|---|---|
| `content.key` | 内容解密密钥，网关启动时读进内存，永不离开本机 |
| `account.token` | 账号令牌，网关与 MCP 的身份凭证 |

全程不需要手工复制粘贴任何长密钥。配对码过期了就重新运行 `pair.py` 再取一个。

#### 3. 验证：冒泡测试

网关启动后会发一条测试消息。去 XPhone 前端「个人代理」的对话里看，应该能看到：

> 🫧 冒泡测试：网关链路正常

**看到了 = 全链路打通**（认证 → 取票 → WebSocket → 加密提交 → 客户端 applied → 渲染）。

### 在其他平台连接（MCP 直连）

不想跑网关、只想让平台上的个人代理直连 Worker 的 MCP，也可以：

- **MCP 地址**：`https://xphone-agent-worker.xphone-maps-gateway.workers.dev/mcp`
- **认证**：API Key，放在 `Authorization: Bearer <accountToken>`。这个 key 在 GIGI 前端「个人代理」设置页点「复制 MCP API Key」获得（每个存档独立，切换存档不会撤销另一存档的配对）。
- **GPT**：在 GPT 里创建自定义 MCP，选择 OAuth2，连接时会跳转到一个显示 8 位配对码的页面，把配对码填进 GIGI 前端即完成连接。

> 注意：目前 Muse 平台的 OAuth2 连接流程存在平台侧问题（DCR 方式报"无法连接这个应用"），因此配对码 + 复制 API Key 是当前主推路径，OAuth2 作为后续增强。

### 登录与开始游玩

1. 打开 XPhone 前端（`phone.xenoflex.xyz`），登录你的账号。
2. 配对成功后进入**空白手机的个人代理模式**：手机是空的，等你来玩。
3. 不知道从哪开始？先做这三件：
   - **进一个故事世界**：选一个故事（例如《圣埃斯佩兰萨的棠》），直接进入有剧情、有人、有事发生的世界；
   - **创建第一个联系人**：捏一个 AI 朋友，建完立刻能发消息；
   - **逛逛地图**：地图是故事世界的总览，点一个地点看详情、看角色在哪里。
4. 每个 App 第一次打开都会有一句提示，告诉你它能干嘛。随时可以在设置里找回新手引导。

手机里能玩的东西：与角色私聊（DM）、社交动态（发帖/评论）、地图（地点、角色位置、偶遇）、相册、备忘录、日志。角色会主动找你聊天，也会在社交动态里发帖。

### 试用与邀请

- 新用户有 **7 天试用期**。
- **邀请码**：把你的邀请码分享给好友，好友在设置页填入后，你的试用期**延长 30 天**。邀请码可多次使用，每次有效推荐都叠加 30 天。
- 试用期间，在你正玩得高兴的时候（比如地图上刚偶遇到新朋友），会出现一张轻量的邀请卡片，不打扰，关了就不再追着推。

### 安全要点

- **内容密钥永不离开网关**：所有剧情内容用 AES-256-GCM 加密，Worker 只转发密文，读不到任何原文。
- **密钥不进聊天、不进日志、不进代码**：配对时密钥由 ECDH 加密传输，网关用本地私钥解密后落盘（0600）。
- **一个存档同时只允许一个 Agent 在线**：如果看到 409 冲突，先停掉旧网关再连。
- **accepted ≠ applied**：服务端说"已接收"只代表入队，只有客户端回执 `applied` 才代表真的落地显示了。

### 排错

| 现象 | 可能原因 / 做法 |
|---|---|
| 配对码失效 | 5 分钟过期，重新运行 `pair.py` 再取一个 |
| 网关连不上 Worker | 检查出站网络与代理；Python 请求必须带 `User-Agent: XPhone-Gateway/1.0`，默认 UA 会被 Cloudflare 拦截 |
| 409 冲突 | 另一台网关还在线，先停掉它 |
| 发了消息前端看不到 | 用 `xphone.get_delivery` 复核 batch 是否到 `applied`；只到 `accepted` 说明还在路上或被拒了 |
| VM 替换后网关没了 | 正常现象，watchdog 每 10 分钟检查并从模板重装恢复；`systemctl status xphone-gateway` 看状态，日志在 `~/xphone/gateway/gateway.log` |

### 给开发者

- **工具契约**：`xphone-tool-contract.md` 是个人代理在 XPhone Agent 模式下的工具契约（故事读取、事件收发、动作提交、图片加密传输），以它为准。
- **动作协议**：动作用 v2 协议提交——语义动作数组写文件，只传任务文档路径给 Worker，网关负责拼 header 并加密。模型不填 batchId、step 等路由字段。
- **网关仓库**：[xphone-gateway](https://github.com/ghpjx29-vdv3/xphone-gateway)（含配对、WS 客户端、看门狗、联调指南）。

---

<a id="english"></a>

## English

### What is this

**XPhone** is a virtual phone inhabited by AI characters, with story worlds, a map, social feeds, and notes. Your personal AI agent is the "operating system" behind the phone — it answers messages as the characters, advances the plot, and posts to social feeds.

**Agent Worker** is an MCP server running on Cloudflare Workers. It does three things:

1. **Authentication**: verifies your personal agent and issues pairing credentials;
2. **Relay**: passes encrypted events and actions between the phone client and your personal agent;
3. **Isolation**: the Worker only relays ciphertext and cannot read any story content.

```
Phone client (GIGI / phone.xenoflex.xyz)
        ↕ encrypted events / encrypted actions
XPhone Agent Worker (Cloudflare Workers, MCP)
        ↕ WebSocket / MCP
Your personal agent gateway (xphone-gateway, runs on your own machine)
        ↕
Your personal AI agent (Muse / GPT)
```

This public repo holds only the **public tool contract** (`xphone-tool-contract.md`): what a personal agent can call inside XPhone and how. The full Worker implementation is private and deployed on Cloudflare; it is not here.

Companion repo — the gateway (the half that runs on your machine): [xphone-gateway](https://github.com/ghpjx29-vdv3/xphone-gateway).

### Quick start: pair your personal agent

There are only two things for you to do: **deploy the gateway, enter a pairing code**. Your personal agent handles the rest.

#### 1. Deploy the gateway

Say one sentence to your personal agent (Muse):

> Help me deploy github.com/ghpjx29-vdv3/xphone-gateway and connect XPhone

It will fetch the gateway code, install dependencies, and set up the resident service. The gateway needs an always-on Linux machine (Muse users can just use their own Muse VM — no extra VPS needed).

#### 2. Pair (8-character pairing code)

On the gateway machine, run:

```bash
.venv/bin/python pair.py
```

The terminal prints an **8-character pairing code (e.g. `3EY64LSN`), valid for 5 minutes**. Open the "Personal Agent Pairing" page in the XPhone / GIGI phone frontend, enter the code, and confirm.

After confirmation the gateway redeems the code automatically and stores credentials under `~/.config/xphone/` (mode 0600):

| File | Purpose |
|---|---|
| `content.key` | Content decryption key, loaded into gateway memory at startup, never leaves the machine |
| `account.token` | Account token, the gateway's and MCP's identity credential |

No manual copy-pasting of long secrets at any point. If the code expires, just run `pair.py` again.

#### 3. Verify: bubble test

Once the gateway is up it sends a test message. Look in the "Personal Agent" conversation in the XPhone frontend — you should see:

> 🫧 冒泡测试：网关链路正常

**Seeing it means the whole chain works** (auth → ticket → WebSocket → encrypted submit → client `applied` → rendered).

### Connecting from other platforms (direct MCP)

If you don't want to run a gateway and just want a platform-hosted personal agent to connect to the Worker's MCP directly:

- **MCP URL**: `https://xphone-agent-worker.xphone-maps-gateway.workers.dev/mcp`
- **Auth**: API key in `Authorization: Bearer <accountToken>`. Get the key from the "Copy MCP API Key" button on the Personal Agent settings page in the GIGI frontend (one key per archive; switching archives does not revoke another archive's pairing).
- **GPT**: create a custom MCP in GPT, choose OAuth2, and on connect you will be redirected to a page showing an 8-digit pairing code. Enter that code in the GIGI frontend to finish connecting.

> Note: the Muse platform's OAuth2 connection flow currently has a platform-side issue (DCR fails with "unable to connect to this app"), so pairing code + pasted API key is the recommended path for now; OAuth2 remains a future enhancement.

### Sign in and start playing

1. Open the XPhone frontend (`phone.xenoflex.xyz`) and sign in.
2. After pairing you land in the **blank phone's personal-agent mode**: an empty phone, waiting for you to play.
3. Not sure where to start? Do these three things first:
   - **Enter a story world**: pick a story (e.g. *The Tang of Santa Esperanza*) and jump straight into a world with plot, people, and things happening;
   - **Create your first contact**: make up an AI friend — you can message them the moment they're created;
   - **Explore the map**: the map is an overview of the story world. Tap a place to see details and where the characters are.
4. Every app shows a one-line hint the first time you open it. You can always revisit the beginner's guide from Settings.

What you can play with: direct messages with characters, social feeds (posts and comments), the map (places, character positions, chance encounters), photos, notes, and the journal. Characters will message you first, and post to social feeds on their own.

### Trial and referrals

- New users get a **7-day free trial**.
- **Referral code**: share your code with friends. When a friend enters it in Settings, your trial is **extended by 30 days**. Codes can be used multiple times, and every successful referral stacks another 30 days.
- During the trial, when you're clearly having fun (e.g. you just met a new friend on the map), a lightweight invite card may appear. It never blocks what you're doing, and once dismissed it won't chase you.

### Security notes

- **The content key never leaves the gateway**: all story content is encrypted with AES-256-GCM; the Worker only relays ciphertext and cannot read plaintext.
- **Keys never enter chats, logs, or code**: during pairing the key is delivered encrypted with ECDH; the gateway decrypts it with its local private key and stores it (0600).
- **One agent online per archive at a time**: if you see a 409 conflict, stop the old gateway first.
- **accepted ≠ applied**: "accepted" from the server only means queued. Only a client receipt of `applied` means it actually landed on screen.

### Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Pairing code expired | Codes live 5 minutes; run `pair.py` again for a fresh one |
| Gateway can't reach the Worker | Check outbound network and proxy; Python requests must send `User-Agent: XPhone-Gateway/1.0` — the default UA is blocked by Cloudflare |
| 409 conflict | Another gateway is still online; stop it first |
| Sent a message but nothing shows in the frontend | Check the batch with `xphone.get_delivery` until it reaches `applied`; `accepted` alone means it is still in flight or was rejected |
| Gateway gone after a VM replacement | Expected; the watchdog checks every 10 minutes and reinstalls from the template. Check `systemctl status xphone-gateway`; logs are in `~/xphone/gateway/gateway.log` |

### For developers

- **Tool contract**: `xphone-tool-contract.md` is the source of truth for what a personal agent can do in XPhone Agent mode (story reads, event delivery, action submission, encrypted image transfer).
- **Action protocol**: actions use the v2 protocol — the agent writes a semantic action array to a file and only passes the task document path to the Worker; the gateway builds the header and encrypts the envelope. Models never fill in batchId, step, or routing fields.
- **Gateway repo**: [xphone-gateway](https://github.com/ghpjx29-vdv3/xphone-gateway) (pairing, WebSocket client, watchdog, integration guide).
