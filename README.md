# HomeVault (家庭数字资产管理系统)

HomeVault 是一个面向个人与家庭的私有化数字文件管理项目；高可用、台账和容灾能力仍属于后续目标。

---

## 快速启动 (Quick Start)

### 1. 检查环境变量配置
根目录下已预置初始 `.env` 文件（或从 `.env.example` 复制）：
```bash
cp .env.example .env
```

默认管理员账号：
- **用户名**: `admin`
- **初始密码**: `HomeVaultAdmin2026!`

### 2. 启动 Docker 容器集群
```bash
docker compose up -d --build
```

系统将自动启动 7 个核心容器服务：
1. **homevault-caddy**: 单一入口反向代理网关 (端口 `8080`)
2. **homevault-backend**: Python 3.12 + FastAPI 核心业务服务
3. **homevault-worker**: ARQ 异步后台处理与定时任务
4. **homevault-frontend**: Next.js 15 PWA 统一前端
5. **homevault-postgres**: PostgreSQL 16 + pgvector 关系型与向量数据库
6. **homevault-redis**: Redis 7 缓存与任务队列
7. **homevault-meilisearch**: 已随栈启动；文件搜索目前使用 PostgreSQL 查询，Meilisearch 索引尚未接入

### 3. 访问系统
- **Web 前端入口**: [http://localhost:8080](http://localhost:8080)
- **API 文档 (Swagger UI)**: [http://localhost:8080/api/docs](http://localhost:8080/api/docs)
- **API Redoc**: [http://localhost:8080/api/redoc](http://localhost:8080/api/redoc)
- **健康检查**: [http://localhost:8080/api/v1/health](http://localhost:8080/api/v1/health)

---

## 当前开发进展

截至 2026-11-25，基础架构、账号/空间接口和文件管理 MVP 可运行。当前已实现：

- [x] 登录后查看有权限访问的家庭共享空间和个人空间
- [x] 上传单个或多个文件，使用 SHA-256 去重；当前唯一范围为同一空间、同一所有者
- [x] 上传本地文件夹并保留相对目录层级；所选顶层文件夹会建在当前 HomeVault 路径下
- [x] 上传照片时解析 EXIF GPS 坐标，也可手动填写地点说明、纬度和经度；文件列表可打开地图坐标
- [x] 上传时选择用途分类、私人/家庭公开范围、备注和标签
- [x] 每条文件记录固定归属一个用户；可在私人空间与家庭共享空间之间切换可见范围
- [x] 按用途分类建立可嵌套的虚拟视图：文件可加入多个子视图，父视图汇总后代视图内容，不复制或改变实际目录
- [x] 虚拟视图按文件权限显示：只返回用户可访问文件所在的视图路径；列表仍逐文件过滤权限，不会泄露同一视图中的私人文件
- [x] 按文件名、备注、标签和已提取 OCR 文本搜索（PostgreSQL 查询，尚非 Meilisearch/Cmd+K 全局搜索）
- [x] 文件入库后编辑用途分类、备注、标签及多个虚拟视图归属（仅所有者）
- [x] 登记外部文件位置，不上传文件内容
- [x] 文件夹树，支持嵌套子文件夹、批量上传到当前目录和移动已有文件
- [x] 图片与 PDF 在线预览，其他文件下载
- [x] 软删除、回收站与恢复（恢复时检查同空间重复文件）
- [x] 文件访问沿用空间读写权限；保险箱上传在加密能力完成前会被拒绝
- [x] 图片/PDF 后台文本提取与 OCR，并纳入搜索（本地运行，不向外部 OCR 服务发送文件）
- [ ] 热文件夹监听、缩略图与 EXIF 提取
- [ ] Meilisearch 索引同步、Cmd/Ctrl+K 搜索与照片时间轴
- [ ] 台账管理、到期提醒、二维码、动态水印和限时外链
- [ ] 保险箱落盘加密与访问保护（现有 PIN 接口不代表保险箱已可用；保险箱文件上传会被拒绝）
- [ ] 多 NAS 备份、恢复演练、完整性巡检、紧急接管与本地 AI
- [ ] 家庭成员和权限管理界面（已有部分用户/空间 API）

### 文件夹与虚拟视图

物理文件夹支持嵌套，单个文件保存在一个实际文件夹中。每个用途分类下面可以建立多层子虚拟视图，例如“说明书 / 家电 / 清洁家电”。文件可以加入多个视图；父视图自动汇总后代视图中的文件。比如扫地机器人说明书可以同时放在物理“扫地机器人”文件夹和“说明书 / 家电 / 清洁家电”视图中，而文件只存一份。虚拟视图不单独授予文件权限：用户可见的文件会让其所属视图路径可见；视图列表只包含当前用户可访问的文件。当前尚无设备/台账关联界面。

文件 API（均需 Bearer 登录令牌）：

- `POST /api/v1/assets/upload?space_id={uuid}`：multipart 字段 `file`
- 上传接口可选 multipart 字段 `latitude`、`longitude`、`location_label`；照片在未手动指定坐标时读取 EXIF GPS
- `POST /api/v1/assets/register-location?space_id={uuid}`：登记外部地址，不上传文件
- `GET /api/v1/folders?space_id={uuid}` / `POST /api/v1/folders?space_id={uuid}`：读取文件夹树 / 在当前空间创建文件夹（提交 `parent_id` 可创建嵌套文件夹）
- `GET /api/v1/virtual-views?space_id={uuid}`：列出当前用户可见的嵌套视图树及其可见文件数量
- `POST /api/v1/virtual-views?space_id={uuid}&business_type={type}`：在用途分类根节点或指定父视图下创建子视图；请求体 `{"name":"清洁家电","parent_id":"..."}`，根下创建时 `parent_id` 传 `null`
- `GET /api/v1/assets?space_id={uuid}&folder_id={uuid}&virtual_view_id={uuid}&q={text}&media_type={type}&business_type={type}`：普通目录浏览使用 `folder_id`；用途分类是跨物理文件夹的根视图，指定 `virtual_view_id` 后显示该节点及后代视图中用户有权访问的文件
- `PATCH /api/v1/assets/{id}/folder`：移动文件；请求体 `{"folder_id":"..."}`，传 `null` 移至根目录
- `PATCH /api/v1/assets/{id}/metadata`：所有者更新用途分类、备注、标签和 `virtual_view_ids`；不改变文件归属、可见范围或实际目录
- `PATCH /api/v1/assets/{id}/visibility?visibility=private|public`：切换私人/家庭共享；公开仅限授权的家庭共享空间
- `GET /api/v1/assets/trash?space_id={uuid}`：回收站
- `GET /api/v1/assets/{id}/file`：下载；`?inline=true` 用于图片/PDF 预览
- `DELETE /api/v1/assets/{id}`：移入回收站
- `POST /api/v1/assets/{id}/restore`：恢复文件

### 暂缓与未完成

当前开发聚焦于文件归档、权限边界和跨目录虚拟视图。更完整的家庭台账、提醒、分享、水印、二维码、保险箱、备份、数字遗产与 AI 能力保留在需求/spec 作为后续路线；除文件管理闭环外，尚未完成的清单暂不纳入当前迭代。

用途分类：`identity`（证件资料）、`receipt`（票据凭证）、`insurance`（保险保单）、`home_appliance`（家电资料）、`manual`（说明书）、`family_photo`（家庭照片）、`family_video`（家庭视频）、`audio`（音频）、`other`（其他）。

## Sprint 1 基础架构
- [x] `docker-compose.yml` 完整编排定义
- [x] `caddy/Caddyfile` 单端口 (8080) 路由配置
- [x] 后端基础骨架 (FastAPI + SQLAlchemy 2.0 Async + asyncpg)
- [x] 完整数据库 ORM 模型与 Alembic 迁移脚本 (`0001_initial_schema`)
- [x] 管理员与默认共享空间自动初始化 (`seed_service`)
- [x] 用户鉴权与 JWT 登录接口 (`/api/v1/auth/login`, `/api/v1/auth/me`)
- [x] 高敏保险箱 PIN 码设定与短期令牌换发 (`/api/v1/auth/vault/pin`, `/api/v1/auth/vault/unlock`)
- [x] 用户管理接口 (`/api/v1/users`)
- [x] 空间与成员权限管理接口 (`/api/v1/spaces`)
- [x] 前端 Next.js 容器与 Dockerfile 基础骨架
