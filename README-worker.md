# worker_速度端点.js 使用说明

这份文件只讲一件事：**怎么把这个 JS 片段加到你的 edgetunnel Worker 里，让 CFIP 能测「真实路径」的速度。**

不需要懂编程，照着粘贴就行。全程大概三分钟。

---

## 一、它是干什么的

CFIP 默认测速是打 Cloudflare 自己的静态大文件（jsdelivr / unpkg 上的 wasm）。这条路径只经过「你 → CF 边缘」这一段，不会跑到外网去。

而你真正翻墙时的路径长这样：

```
你的电脑  ──①──>  CF 边缘节点（你选的优选 IP）  ──②──>  你的 Worker  ──③──>  YouTube / GitHub
```

- **①** 是你唯一能优化的，选哪个 IP 就是优化这一段
- **②** CF 内部网络，固定
- **③** CF 回源到外网，**这一段常常才是瓶颈**

只测 ① 得到的数字会虚高：你在 CFIP 里看到 200Mbps，刷 YouTube 却卡，就是因为 ③ 那段压根没被测到。

加上这个片段之后，CFIP 会把测速请求打到你自己的 Worker 上的 `/__speed` 端点，让 Worker 去外网真拉一段数据再流式返回给你。这样测出来的数字就同时包含了 ① 和 ③ —— **才是你实际能用的速度**。

---

## 二、开始之前：fetch handler 是什么

打开你的 `_worker.js`，开头十几行大概长这样：

```js
export default {
  async fetch(request, env, ctx) {
    let 请求URL文本 = request.url;
    let url = new URL(请求URL文本);
    ...
```

`async fetch(request, env, ctx) {` 这一行就是**入口函数**（行话叫 fetch handler）。Cloudflare 每收到一个请求，都会调用它一次，把请求当成 `request` 参数传进来。

它下面那一长串 `if ... else if ... else if ...` 是在按路径分发：

```js
if (访问路径 === 'version') {          // 访问 /version 时返回版本信息
} else if (admin密码 && upgradeHeader === 'websocket') {   // WebSocket 代理
} else if (admin密码 && ... POST) {    // gRPC / XHTTP 代理
} else {                               // 其它，返回首页 / 登录页 / 管理页
}
```

我们要做的，就是在**这一串 if 的最前面**插入一段自己的判断：如果访问的是 `/__speed`，就走测速逻辑，不再往下走。

---

## 三、第一步：找到位置

用任何文本编辑器打开 `_worker.js`，搜索这个字符串：

```
async fetch(request, env, ctx) {
```

edgetunnel 官方版本里，这一行大约在**第 19 行**（不同版本会差几行，以搜索结果为准）。

---

## 四、第二步：腾出一行空位

把光标放到 `async fetch(request, env, ctx) {` 这一行的**行尾**，按一下回车。

这样就在它下面多出一个空行，原来的第 20 行被挤到了第 21 行。

> **为什么要按这一下？** 因为片段本身以 `{` 开头、以 `}` 结尾，它需要一个「干净的开头位置」。如果直接插在已有代码的行中间，很容易把原有结构弄断。

---

## 五、第三步：整块复制粘贴

打开 `worker_速度端点.js`，**全选、复制全部内容**，粘贴到你刚腾出来的那个空行上。

粘贴完应该长这样：

```js
export default {
  async fetch(request, env, ctx) {

// ====================== 测速端点开始 ======================
{
  const 测速URL = new URL(request.url);
  if (测速URL.pathname.toLowerCase() === '/__speed') {
    ...
  }
}
// ====================== 测速端点结束 ======================
    let 请求URL文本 = request.url;     ← 原来第 20 行的内容，现在在片段下面
    ...
```

### 三个注意事项

1. **连最外层的 `{` 和 `}` 一起复制。** 不要只复制里面的 `if` 部分。外层那对大括号是把片段里的变量（`测速URL` 之类）关在一个局部作用域里，避免和你文件后面几万行里的同名变量打架。

2. **首尾那两行注释也一起复制。** 它们标出了片段的边界，以后想删掉或者更新版本时，一眼就能找到范围。

3. **原有代码一个字都不要改。** 这个片段是「独立 if 块」形态，不是接在原来的 `else if` 链后面，所以不需要动任何一个 `}`。这也是为什么它插在函数最前面最安全。

> ### 为什么不是写成 `} else if (访问路径 === '__speed') {`
>
> 那种写法的片段末尾自带一个 `}`，粘到原有 `} else if (...)` 前面就会变成 `}\n} else if` —— 多出来一个右括号，后面那个 `else` 成了孤儿，部署时直接报 `SyntaxError: Unexpected token 'else'`。
>
> 现在这个版本刻意避开了这个坑：它自己带完整的大括号，插在函数第一个语句的位置，前后都不碰原有结构。

### 关于缩进

片段里的缩进可能和你文件的缩进风格不一致（有的是 2 空格，有的是 Tab）。**这不影响运行，不用管。** JS 不在乎缩进。

---

## 六、第四步：保存并部署

1. 保存 `_worker.js`
2. 打开 Cloudflare Dashboard → Workers & Pages → 你的 Worker → 编辑代码
3. 把改好的 `_worker.js` 全文粘进去（或者用 `wrangler deploy` 部署）
4. 保存 / Deploy

如果编辑器报语法错误，重点检查两件事：

| 报错 | 原因 |
|---|---|
| `Unexpected token 'else'` | 说明你粘的是旧版 `else if` 形态的片段，换成现在这版独立 if 块 |
| `Unexpected token '{'` | 片段最外层那对 `{ }` 没复制全，或复制了两遍 |
| `Unexpected end of input` | 片段结尾的 `}` 漏了 |

---

## 七、第五步：验证端点是否生效

部署完等十几秒，然后在命令行里跑：

```bat
curl.exe -s -o NUL -D - --max-time 30 -r 0-1023 "https://你的域名/__speed?mb=1"
```

把 `你的域名` 换成你自己的域名（Worker 绑定的那个）。

**成功的输出**长这样，重点看三行：

```
HTTP/1.1 200 OK
X-Speed-Bytes: 1048576
X-Upstream-Mode: gvt1-video-cdn
X-Upstream-Landed: rr1---sn-oj5hn5-5t.gvt1.com
```

| 头 | 含义 |
|---|---|
| `X-Speed-Bytes` | 这次要返回多少字节（`mb=1` 就是 1048576） |
| `X-Upstream-Mode` | 走的是哪条上游。`gvt1-video-cdn` = YouTube 视频 CDN（默认），`dl.google.com` = Google 静态下载 CDN（加了 `&u=dl` 时） |
| `X-Upstream-Landed` | 实际落到了哪台服务器。正常情况下应该是 `rr*.gvt1.com` 或 `*.googlevideo.com` |

### 排错表

| 现象 | 原因 | 怎么办 |
|---|---|---|
| `HTTP/1.1 404` 或返回 613 字节的 nginx 欢迎页 | 片段没生效 | 检查是不是粘到了 `fetch` 函数外面，或者没点 Deploy |
| `code=000`（curl 直接超时） | 端点不存在或域名没绑好 | 先用 `curl https://你的域名/` 确认域名本身能访问 |
| `HTTP/1.1 502` + 正文「上游全部失败」 | Worker 所在机房连不上 Google | 在 CFIP 里换一条回源线路（下拉选线路 2 或 3） |
| `X-Upstream-Landed` 是 `dl.google.com` 但你没加 `u=dl` | 视频 CDN 三个候选全失败了，回退到了静态 CDN | 正常现象，能测但数值偏低，可以换线路重试 |

---

## 八、第六步：在 CFIP 里启用

### 图形界面

1. **域名(SNI/Host)** 填你自己的域名（就是刚才验证用的那个）
2. **测速方式** 下拉选 `真实路径：走 Worker 回源（准）`（v1.6 默认就是它）
3. **回源线路** 保持默认 `YouTube 视频CDN 1（ndk 781MB）`；如果大量短读或 502 就换下一个
4. **测速Host / 测速路径** 两栏留空（程序会自动用 `/__speed?mb=50`）
5. **超时ms** 改成 `5000`（回源路径首字节慢，1000ms 会把好 IP 全判死）
6. 点「开始测试」

### 命令行

```bat
cfip_gui.exe --cli --domain 你的域名.com --iplib 5 --ipcount 20 ^
  --realpath --threads 1 --streams 2 --repeat 5 --timelimit 20 --timeout 5000
```

---

## 九、参数说明

端点支持四个查询参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `mb` | 50 | 要返回多少 MB，范围 1~200。测速时按你的「下载(s)」和带宽估个值，太大浪费额度，太小测不准 |
| `u` | 空 | 写 `u=dl` 切换到 Google 静态下载 CDN（`dl.google.com`），用于对比 |
| `n` | 0 | 选择视频 CDN 的第几个候选（0/1/2 循环）。默认 0 最稳 |
| `token` | 空 | 如果你在 Worker 里设了环境变量 `SPEED_TOKEN`，必须带上 `&token=你的值`，否则 403 |

例：

```
/__speed?mb=20              20MB，走视频 CDN 第 1 候选
/__speed?mb=20&u=dl         20MB，走 Google 静态 CDN
/__speed?mb=20&n=1          20MB，走视频 CDN 第 2 候选
```

---

## 十、为什么这么设计（技术细节）

### 为什么用 `globalThis.fetch` 而不是 `fetch`

Worker 里直接写 `fetch(...)` 有可能被解析成「调用自己的 fetch handler」，形成递归。加 `globalThis.` 前缀就明确指向全局的 fetch，绕开自己的 handler。

### 为什么要手动截断

`redirector.gvt1.com` 会 302 跳到 `*.googlevideo.com`，而**跳转过程中 Range 头会丢失**，上游可能一口气开始传整个 781MB 文件。

所以片段里挂了一个 `TransformStream`，边传边数：一旦发够你要的字节数就停止转发。这样既拿到了真实吞吐，又不会把 200MB 额度瞬间烧光。

### 为什么响应是 chunked 而不是 Content-Length

CF 在 `TransformStream` 后面会强制改成 `Transfer-Encoding: chunked`，即使你设了 `Content-Length` 也会被丢掉。所以 CFIP 判断「这一段收完了」不靠长度，而是靠 chunked 的结束标记 `0\r\n\r\n`。

### 为什么三个视频 CDN 候选要依次试

`redirector.gvt1.com` 背后的实际服务器由 Google 调度决定，某个机房可能连不上其中某一台。片段会依次尝试三个不同的大文件源，第一个成功就用，全失败才回退到静态 CDN。

### 默认走视频 CDN 的原因

看视频、刷网页，走的大多是 `*.googlevideo.com` 那张网。测这张网的速度，比测 `dl.google.com`（软件下载 CDN）更接近你刷 YouTube 的体感。

---

## 十一、额度消耗

每次 `/__speed` 请求算 **1 次** Worker 请求（不是按字节算）。测 20 个 IP、每个重复 5 次、2 条流，一共 200 次请求，占免费额度（10 万次/天）的 0.2%。日常用完全不用担心。

CPU 时间也不用担心：这个端点本质是「拉流 + 转发」，几乎不烧 CPU（免费版限 10ms/请求）。

---

## 十二、常见问题

**Q：加了这段会不会影响我正常翻墙？**

不会。只有访问 `/__speed` 这一个路径时才走测速逻辑，其它所有请求原样走你原来的代理逻辑。片段插在最前面，判断完不匹配就直接往下走。

**Q：以后更新 edgetunnel 怎么办？**

新版 `_worker.js` 里没有这段，需要重新粘一次。位置还是 `async fetch(request, env, ctx) {` 下面。

**Q：能不能用别人的域名测？**

不行。`/__speed` 必须跑在**你自己的** Worker 上，因为它要调用你的 Worker 去回源。用别人的域名只能测到 ① 段（静态源模式）。

**Q：测出来的数字比静态源低很多，是不是坏了？**

正常。静态源（jsdelivr / unpkg）是 CF 自家 CDN，几乎零距离；而回源要跨太平洋去 Google，还要受免费 Worker 出口限速。0.5~2Mbps 是常见值，**这个低数字才是你真能用到的速度**。

**Q：想测 YouTube 真实视频流（videoplayback）行不行？**

不行。YouTube 的 `videoplayback` 直链需要签名参数 + PO Token，程序拿不到。现在用 `redirector.gvt1.com` 落地到 `*.googlevideo.com`，已经是无凭证情况下最接近的路径 —— 落地的服务器和 YouTube 视频分片是同一批。
