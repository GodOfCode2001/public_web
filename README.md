# 如何让本地部署上云，告别 localhost —— 无需公网IP

> 让外网能访问你电脑上的网页，不再只有你自己能看到

## 写在前面

你在本地写了一个网页、跑了一个服务，输入 `http://localhost:3000` 能打开，但想让朋友点开链接就能访问——关键是你家宽带**没有公网IP**，运营商给你的是内网地址。

本文介绍两种纯免费方案，都不需要公网IP，也不需要路由器开端口：

| 方案 | 原理 | 适合场景 |
|------|------|----------|
| **Cloudflare Workers + Pages** | 代码部署到 Cloudflare 边缘节点 | 静态网站、API代理、纯前端项目 |
| **dynv6 + Cloudflare Tunnel** | 电脑和 Cloudflare 之间建立隧道 | 有后端服务、数据库、不想改代码的已有项目 |


---

# 方案一：Cloudflare Workers + Pages（纯云端部署，不用自己电脑）

这个方案的本质是：**代码不跑在你电脑上，而是部署到 Cloudflare 全球边缘节点**。别人访问时直接请求 Cloudflare 的服务器，跟你本地电脑没有任何关系。

### 适用场景

- 个人博客、作品集网站
- 纯前端项目（HTML/CSS/JS）
- 单页应用（React/Vue/Angular 打包后的静态文件）
- 轻量级 API 代理

## 子方案 A：Cloudflare Pages（托管静态网站）

Pages 是 Cloudflare 的静态网站托管服务，类似 Vercel/Netlify，但免费额度更大。

### 第一步：准备你的网站文件

把你的前端项目准备好，比如一个文件夹里包含：

```
my-website/
├── index.html
├── style.css
├── script.js
└── 图片等资源
```

如果用的是 React/Vue 框架，先运行 `npm run build` 生成 `dist` 或 `build` 文件夹。

### 第二步：上传到 Cloudflare Pages

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 左侧菜单点击 **Workers & Pages** → 点击 **创建应用** → 选择 **Pages**
3. 有三种方式上传：
   - **连接 Git 仓库**（推荐）：关联 GitHub/GitLab，每次 push 自动部署
   - **直接上传文件夹**：把整个网站文件夹拖进去
   - **Wrangler CLI**：用命令行工具部署
4. 点击 **部署**，等待几秒钟
5. 部署完成后，Pages 会给你一个 `.pages.dev` 的二级域名，比如 `my-project.pages.dev`
6. 点开就能访问，全球 CDN 加速

### 第三步（可选）：绑定你自己的域名

1. 在 Pages 项目设置里找到 **自定义域**
2. 点击 **设置自定义域**，输入你的域名（比如 `guoyu.v6.rocks`）
3. Cloudflare 会自动帮你配置 DNS，等待生效即可

**免费额度**：Pages 免费版支持无限请求数，每月 500 次构建次数，个人项目完全够用。

## 子方案 B：Cloudflare Workers（部署 JavaScript 服务）

Workers 是 Cloudflare 的 Serverless 函数平台，可以跑 JavaScript/TypeScript 代码，用来做 API、代理、重定向等。

### 第一步：创建 Worker

1. Cloudflare Dashboard → **Workers & Pages** → **创建应用** → **Worker**
2. 选择模板或空白模板
3. 在线编辑器中写代码，比如一个简单的 API：

```javascript
export default {
  async fetch(request) {
    return new Response('Hello from Cloudflare Workers!', {
      headers: { 'Content-Type': 'text/plain' }
    });
  }
}
```

4. 点击 **部署**，就上线了

### 第二步：获取访问地址

部署后会得到一个 `xxx.workers.dev` 的域名，打开即可访问。

### 常用场景举例

**1. 代理跨域请求（解决前端本地开发跨域问题）**

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const targetUrl = url.searchParams.get('url');
    if (!targetUrl) return new Response('Missing url param', { status: 400 });
    
    const response = await fetch(targetUrl);
    const data = await response.text();
    return new Response(data, {
      headers: { 'Content-Type': 'application/json' }
    });
  }
}
```

**2. 重定向短链接**

```javascript
const redirectMap = {
  'gh': 'https://github.com/yourname',
  'blog': 'https://yourblog.com'
};

export default {
  async fetch(request) {
    const url = new URL(request.url);
    const path = url.pathname.slice(1);
    if (redirectMap[path]) {
      return Response.redirect(redirectMap[path], 301);
    }
    return new Response('Not found', { status: 404 });
  }
}
```

### 免费额度

| 资源 | 免费额度 |
|------|----------|
| 每日请求数 | 10 万次 |
| CPU 时间 | 每天 10 毫秒 × 100 万次 |
| 脚本大小 | 最多 1 MB |
| 存储（KV） | 1 GB |

个人项目几乎不会超限。


---

# 方案二：dynv6 + Cloudflare Tunnel（本地电脑当服务器）

这个方案适合：**你有一个现成的项目跑在本地，不想改代码，也不想重写部署**，就想让外网直接访问 `localhost:3000`。

## 原理

```
用户访问 your-domain.com
        ↓
    Cloudflare 边缘节点
        ↓
  Cloudflare Tunnel（加密隧道）
        ↓
   你的电脑 localhost:3000
```

关键点：隧道是**你的电脑主动发起的出站连接**，所以不需要公网IP，也不需要路由器开端口。

## 为什么还需要 dynv6？

因为 Cloudflare Tunnel 需要绑定一个域名。你可以：
- **直接用 Cloudflare 购买的域名**（最佳实践）
- 用 dynv6 提供的免费域名（比如 `xxx.v6.rocks`）作为入口

下面以 `guoyu.v6.rocks` 为例，手把手教你配置。

## 第一步：注册 dynv6 获取免费域名

1. 访问 [dynv6.com](https://dynv6.com) 注册账号
2. 登录后点击 **"Create Zone"**
3. 输入你想要的前缀（比如 `guoyu`），选择后缀 `v6.rocks`
   - dynv6 提供多个免费后缀：`v6.rocks`、`v6.army`、`v6.navy` 等
4. 点击创建，域名就归你了
5. 进入域名详情页，**复制你的 Token**（后面要用，但我们的场景里用不到它，因为不是用 DDNS 更新 IP，而是把域名托管给 Cloudflare）

## 第二步：把域名托管到 Cloudflare

因为 Cloudflare Tunnel 必须通过 Cloudflare 的 DNS 来绑定域名，所以需要把 `guoyu.v6.rocks` 的 DNS 解析权交给 Cloudflare。

1. 注册 [Cloudflare](https://dash.cloudflare.com/) 账号（免费）
2. 点击 **添加站点**，输入 `guoyu.v6.rocks`
3. Cloudflare 会扫描现有 DNS 记录，扫描完点击继续
4. 选择 **免费套餐**
5. Cloudflare 会给你两个 Nameserver 地址（类似 `abby.ns.cloudflare.com`）
6. 回到 dynv6.com，点击你的域名进入详情页
7. 找到 **"Nameserver"** 设置，改成 Cloudflare 给你的那两个地址
8. 等待 DNS 生效（通常 10-30 分钟，最长 24 小时）

> **注意**：这一步之后，`guoyu.v6.rocks` 就由 Cloudflare 管理了，dynv6 的 DDNS 功能不再使用，它现在只负责"免费提供域名"这个角色。

## 第三步：安装 cloudflared

**Windows（用管理员权限打开 PowerShell）：**

```powershell
winget install --id Cloudflare.cloudflared
```

**macOS：**

```bash
brew install cloudflared
```

**Linux（Ubuntu/Debian）：**

```bash
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
```

## 第四步：启动隧道，绑定域名

假设你的本地网页服务跑在 `http://localhost:3000`：

```bash
cloudflared tunnel --hostname guoyu.v6.rocks --url http://localhost:3000
```

运行成功后，终端会显示：

```
INF  Your tunnel has started!
INF  You have successfully connected to Cloudflare's network
```

这时候打开浏览器访问 `https://guoyu.v6.rocks`，就能看到你本地的网页了。

**HTTPS 自动搞定**：Cloudflare Tunnel 默认提供 HTTPS，你不需要自己配证书。

## 第五步：后台长期运行（开机自启）

如果不想每次手动启动，可以做成系统服务：

**Linux（以 systemd 为例）：**

```bash
# 安装为服务
sudo cloudflared service install --hostname guoyu.v6.rocks --url http://localhost:3000

# 启动服务
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

**Windows（用任务计划程序）：**

1. 新建一个 `.bat` 文件，内容：`cloudflared tunnel --hostname guoyu.v6.rocks --url http://localhost:3000`
2. 打开任务计划程序 → 创建任务 → 触发器选"开机时" → 操作选启动这个 .bat 文件


---

# 两种方案对比总结

| 对比维度 | Workers + Pages | dynv6 + Tunnel |
|---------|-----------------|----------------|
| 代码跑在哪里 | Cloudflare 边缘节点 | 你自己的电脑 |
| 需要本地电脑开机 | ❌ 不需要 | ✅ 需要一直开机 |
| 支持后端/数据库 | ⚠️ 有限（Worker可写逻辑） | ✅ 完全支持 |
| 部署方式 | Git push / 拖拽上传 | 命令行启动隧道 |
| 适合场景 | 新项目、纯前端、轻量API | 已有项目、有后端、不想改动 |
| 域名来源 | `.pages.dev` 或自带域名 | dynv6免费域名 + Cloudflare托管 |
| 费用 | 完全免费 | 完全免费 |

## 快速决策

**选择 Workers + Pages，如果：**
- 你的项目是纯前端（HTML/JS/CSS）或静态网站
- 你愿意把代码部署到云端
- 不想一直开着电脑

**选择 dynv6 + Cloudflare Tunnel，如果：**
- 你的项目有后端服务（Python/Node/Java/Go 等）
- 需要连接本地数据库
- 项目已经跑在 localhost 上，不想改任何代码
- 你的电脑可以 24 小时开机


---

# 附：两种方案组合使用（高级玩法）

其实两个方案可以**同时用**，各取所长：

- 用 **Pages** 托管你的前端界面（速度快、不耗本地资源）
- 用 **Tunnel** 把本地后端 API 暴露出去（比如 `/api` 路径指向本地）
- 前端代码里请求 `/api/xxx`，实际会走到你本地的后端服务

这样前端享受 CDN 加速，后端保留本地灵活性，完美结合。


# 常见问题

**Q：dynv6 的域名免费多久？**
A：只要 30 天内有更新活动（DNS变更或访问），就永久免费。把域名托管给 Cloudflare 后，Cloudflare 会自动管理 DNS，dynv6 那边不会过期。

**Q：Cloudflare Tunnel 安全吗？**
A：隧道是加密的（TLS 1.3），而且 Cloudflare 默认开启 DDoS 防护和 WAF，比直接暴露公网IP更安全。

**Q：免费版 Tunnel 有什么限制？**
A：免费版支持无限隧道，每个隧道不限流量（但视频等大流量场景可能受限），普通网页服务完全够用。

**Q：域名必须用 dynv6 的吗？**
A：不一定。如果你有自己的域名（比如在 Namesilo、GoDaddy 买的），直接托管到 Cloudflare 就行，不需要 dynv6。dynv6 只是提供一个免费域名的来源。

**Q：我没有公网IP，dynv6 的 DDNS 功能用不了怎么办？**
A：在这个方案里，dynv6 只负责提供免费域名，DNS 解析权交给 Cloudflare，DDNS 更新 IP 的功能根本不需要用。所以没有公网IP完全不影响。


# 参考链接

- [Cloudflare Tunnel 官方文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare Pages 官方文档](https://developers.cloudflare.com/pages/)
- [Cloudflare Workers 官方文档](https://developers.cloudflare.com/workers/)
- [dynv6 官方网站](https://dynv6.com/)
