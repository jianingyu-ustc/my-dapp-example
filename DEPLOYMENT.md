# 部署指南

本指南将帮助你将投票 DApp 部署到 Vercel 或 Cloudflare Pages。

## ⚡ 快速开始（Vercel - 推荐）

如果你只想快速部署，按照以下步骤：

1. **确保合约已部署**：
   ```bash
   cd packages/hardhat
   __RUNTIME_DEPLOYER_PRIVATE_KEY=你的私钥 yarn deploy --network monadTestnet
   ```

2. **登录 Vercel**：
   ```bash
   yarn vercel:login
   ```

3. **部署**：
   ```bash
   yarn vercel
   ```

4. **完成！** 按照提示操作，Vercel 会自动处理 monorepo 配置。

---

## 📋 部署前准备

### 1. 确保智能合约已部署

在部署前端之前，确保你的智能合约已经部署到 Monad Testnet：

```bash
cd packages/hardhat
__RUNTIME_DEPLOYER_PRIVATE_KEY=你的私钥 yarn deploy --network monadTestnet
```

部署成功后，合约地址会自动更新到 `packages/nextjs/contracts/deployedContracts.ts`。

### 2. 检查部署的合约地址

查看 `packages/nextjs/contracts/deployedContracts.ts`，确认合约地址正确。

---

## 🚀 方法一：Vercel 部署（推荐）

Vercel 是 Next.js 的官方平台，部署最简单。

### 步骤 1：安装 Vercel CLI（如果还没有）

```bash
npm i -g vercel
```

### 步骤 2：登录 Vercel

```bash
yarn vercel:login
```

### 步骤 3：配置项目

项目根目录已包含 `vercel.json` 配置文件，会自动处理 monorepo 构建。

如果需要自定义，可以修改根目录的 `vercel.json`：

```json
{
  "buildCommand": "cd packages/nextjs && yarn build",
  "outputDirectory": "packages/nextjs/.next",
  "installCommand": "yarn install",
  "framework": "nextjs",
  "rootDirectory": "packages/nextjs"
}
```

### 步骤 4：部署

#### 方式 A：使用命令行（推荐）

```bash
# 从项目根目录运行
yarn vercel
```

或者直接进入 nextjs 目录：

```bash
cd packages/nextjs
yarn vercel
```

按照提示：
- 选择项目（或创建新项目）
- 确认设置
- 等待部署完成

**注意**：如果使用命令行部署，Vercel 会自动检测到 `vercel.json` 配置。

#### 方式 B：通过 Vercel 网站

1. 访问 [vercel.com](https://vercel.com)
2. 点击 "Add New Project"
3. 导入你的 Git 仓库
4. 配置项目设置：
   - **Framework Preset**: Next.js
   - **Root Directory**: `packages/nextjs`（重要！）
   - **Build Command**: `yarn build`（会自动从根目录安装依赖）
   - **Output Directory**: `.next` (默认)
   - **Install Command**: `cd ../.. && yarn install`（从根目录安装 monorepo 依赖）

**重要提示**：由于是 monorepo，Vercel 需要：
- 在根目录运行 `yarn install` 来安装所有 workspace 依赖
- 在 `packages/nextjs` 目录运行构建命令

### 步骤 5：配置环境变量（重要）

在 Vercel 项目设置中添加环境变量（如果需要）：

通常 Next.js 应用不需要额外的环境变量，因为合约地址已经硬编码在 `deployedContracts.ts` 中。

但如果你的应用需要环境变量，可以在 Vercel Dashboard 中设置：
1. 进入项目设置
2. 点击 "Environment Variables"
3. 添加所需变量

### 步骤 6：重新部署

配置完成后，Vercel 会自动重新部署，或者你可以手动触发部署。

---

## ☁️ 方法二：Cloudflare Pages 部署

### 步骤 1：安装 Wrangler CLI

```bash
npm i -g wrangler
```

### 步骤 2：登录 Cloudflare

```bash
wrangler login
```

### 步骤 3：配置项目

在项目根目录创建 `wrangler.toml`：

```toml
name = "my-voting-dapp"
compatibility_date = "2024-01-01"
pages_build_output_dir = "packages/nextjs/.next"

[build]
command = "yarn install && yarn next:build"
cwd = "."

[[build.environment]]
NODE_VERSION = "20"
```

### 步骤 4：修改 Next.js 配置

由于 Cloudflare Pages 需要静态导出，需要修改 `packages/nextjs/next.config.ts`：

```typescript
const nextConfig: NextConfig = {
  reactStrictMode: true,
  devIndicators: false,
  output: 'export', // 添加这行，用于静态导出
  trailingSlash: true, // 添加这行
  images: {
    unoptimized: true, // 添加这行
  },
  typescript: {
    ignoreBuildErrors: process.env.NEXT_PUBLIC_IGNORE_BUILD_ERROR === "true",
  },
  eslint: {
    ignoreDuringBuilds: process.env.NEXT_PUBLIC_IGNORE_BUILD_ERROR === "true",
  },
  webpack: config => {
    config.resolve.fallback = { fs: false, net: false, tls: false };
    config.externals.push("pino-pretty", "lokijs", "encoding");
    return config;
  },
};
```

**注意**：静态导出会限制一些 Next.js 功能（如 API Routes、服务端渲染等）。如果你的应用需要这些功能，建议使用 Vercel。

### 步骤 5：通过 Git 部署（推荐）

1. 访问 [dash.cloudflare.com](https://dash.cloudflare.com)
2. 进入 "Workers & Pages"
3. 点击 "Create Application" > "Pages" > "Connect to Git"
4. 选择你的 Git 仓库
5. 配置构建设置：
   - **Framework preset**: Next.js (Static HTML Export)
   - **Build command**: `yarn install && yarn next:build`
   - **Build output directory**: `packages/nextjs/out`
   - **Root directory**: `/` (项目根目录)

### 步骤 6：通过 Wrangler 部署

```bash
# 先构建
yarn next:build

# 部署
wrangler pages deploy packages/nextjs/out --project-name=my-voting-dapp
```

---

## 🔧 常见问题

### 1. Monorepo 构建问题

如果遇到构建错误，确保：
- 在项目根目录运行 `yarn install`
- 构建命令正确指向 `packages/nextjs`

### 2. 合约地址未更新

确保部署合约后，`packages/nextjs/contracts/deployedContracts.ts` 已更新。

### 3. 网络配置

确保 `packages/nextjs/scaffold.config.ts` 中的 `targetNetworks` 包含 Monad Testnet。

### 4. 环境变量

如果应用需要环境变量，确保在部署平台正确配置。

---

## 📝 部署检查清单

- [ ] 智能合约已部署到 Monad Testnet
- [ ] `deployedContracts.ts` 包含正确的合约地址
- [ ] `scaffold.config.ts` 配置了正确的网络
- [ ] 本地构建成功：`yarn next:build`
- [ ] 本地测试通过：`yarn start`
- [ ] 环境变量已配置（如需要）
- [ ] Git 仓库已推送最新代码

---

## 🎯 推荐方案

**推荐使用 Vercel**，因为：
1. Next.js 官方平台，兼容性最好
2. 支持服务端渲染和 API Routes
3. 自动 HTTPS 和 CDN
4. 部署流程最简单
5. 免费额度充足

**使用 Cloudflare Pages** 如果：
1. 需要静态站点（更低的成本）
2. 已经在使用 Cloudflare 服务
3. 需要更快的全球 CDN

---

## 🔗 部署后

部署成功后，你会获得一个公网 URL，例如：
- Vercel: `https://your-project.vercel.app`
- Cloudflare: `https://your-project.pages.dev`

分享这个链接，用户就可以访问你的投票 DApp 了！

---

## 📚 相关资源

- [Vercel 文档](https://vercel.com/docs)
- [Cloudflare Pages 文档](https://developers.cloudflare.com/pages/)
- [Next.js 部署文档](https://nextjs.org/docs/deployment)

