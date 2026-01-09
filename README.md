<p align="center">
  <img src="src/images/logo.webp" alt="Dockhand" width="300">
</p>

<p align="center">
  <strong>现代化 Docker 管理界面</strong>
</p>

<p align="center">
  <a href="https://dockhand.pro">官网</a> •
  <a href="https://dockhand.pro/manual">文档</a> •
  <a href="#license">许可证</a>
</p>

---

## 关于

Dockhand 是一款现代、高效的 Docker 管理应用，提供容器实时管理、Compose 栈编排与多环境支持，同时保持轻量、安全与隐私友好。

> 说明：本 README 基于上游 Dockhand 项目的英文内容翻译与整理，版权与署名归原作者所有。原始信息与链接请以官方来源为准。

### 功能特性

- **容器管理**：实时启动、停止、重启与监控容器
- **Compose 栈**：Docker Compose 的可视化编辑与部署
- **Git 集成**：通过 Webhook 与自动同步从 Git 仓库部署
- **多环境**：管理本地与远程 Docker 主机
- **终端与日志**：交互式 Shell 与实时日志流
- **文件浏览**：浏览、上传与下载容器内文件
- **认证**：OIDC SSO、本地用户与可选 RBAC（企业版）

## 技术栈

- **基础层**：通过 apko 使用 <a href="https://github.com/wolfi-dev/os">Wolfi packages</a> 从零构建 OS 层，Dockerfile 中显式声明每个包
- **前端**：SvelteKit 2、Svelte 5、shadcn-svelte、TailwindCSS
- **后端**：Bun 运行时 + SvelteKit API 路由
- **数据库**：SQLite 或 PostgreSQL（Drizzle ORM）
- **Docker**：直接调用 Docker API

## 许可证

Dockhand 使用 [Business Source License 1.1](LICENSE.txt)（BSL 1.1）授权。

### 许可要点

- **免费使用**：个人使用、企业内部使用、非营利组织、教育与评估
- **不允许**：以商业 SaaS/托管服务形式提供 Dockhand
- **自动转为 Apache 2.0**：2029 年 1 月 1 日

完整条款见 [LICENSE.txt](LICENSE.txt)。


<a href="https://buymeacoffee.com/dockhand" target="_blank">
  <img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png"
       alt="Buy Me A Coffee"
       height="40">
</a>


## 链接

- **官网**：https://dockhand.pro
- **文档**：https://dockhand.pro/manual

---

© 2025-2026 Finsys / Jarek Krochmalski
