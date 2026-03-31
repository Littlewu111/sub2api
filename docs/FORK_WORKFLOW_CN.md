# Fork 维护与发布流程

本文档记录当前仓库的推荐维护方式，目标是同时满足两件事：

- 保留你自己的定制修改
- 持续同步上游 `Wei-Shaw/sub2api` 的新改动

## 当前仓库结构

当前远端约定：

- `origin`: 你的 fork `https://github.com/Littlewu111/sub2api.git`
- `upstream`: 上游仓库 `https://github.com/Wei-Shaw/sub2api.git`

当前分支约定：

- `main`: 尽量保持和上游同步，不直接写业务定制
- `custom/prod`: 你的长期定制分支，生产环境从这里部署
- `feat/*`: 临时功能分支，开发完成后合并回 `custom/prod`

## 日常开发流程

不要直接在 `main` 上改代码。

建议每次都从 `custom/prod` 切一个功能分支：

```bash
git checkout custom/prod
git pull --rebase origin custom/prod
git checkout -b feat/your-change
```

开发完成后提交：

```bash
git add .
git commit -m "feat: your change"
```

合并回 `custom/prod`：

```bash
git checkout custom/prod
git merge --no-ff feat/your-change
git push origin custom/prod
```

如果这个功能分支后续不再使用，可以删除：

```bash
git branch -d feat/your-change
```

## 同步上游更新

当原项目有新提交时，按下面顺序同步：

```bash
git fetch upstream

git checkout main
git merge --ff-only upstream/main
git push origin main

git checkout custom/prod
git merge main
git push origin custom/prod
```

说明：

- `main` 只做“跟上游对齐”
- `custom/prod` 通过合并 `main` 获取上游更新
- 如果有冲突，只在 `custom/prod` 解决，不要在 `main` 上做定制修改

## 前端样式修改的本地测试

如果只是改前端页面，比如 `/home` 页面样式，最快的本地预览方式是：

1. 先保证后端容器在本地可访问
2. 单独运行前端开发服务器

启动前端开发服务器：

```bash
cd frontend
pnpm install
VITE_DEV_PROXY_TARGET=http://127.0.0.1:18080 pnpm dev
```

然后访问：

```text
http://127.0.0.1:3000/home
```

说明：

- 前端代码改动后会自动热更新
- `/home` 对应页面文件是 `frontend/src/views/HomeView.vue`
- 如果后台设置里的 `home_content` 不为空，`/home` 会优先显示后台 HTML/iframe，而不是默认 Vue 页面

## 打包态验证

开发态看起来没问题后，建议再验证一次“真正部署后的效果”：

```bash
cd deploy
docker compose -f docker-compose.dev.yml up --build -d
```

这套配置会基于本地源码重新构建镜像，而不是拉官方远程镜像。

如果端口冲突，可以修改 `deploy/.env` 中的端口，例如：

```env
SERVER_PORT=18081
```

然后访问：

```text
http://127.0.0.1:18081/home
```

## 生产发布建议

生产环境推荐从 `custom/prod` 部署，不要直接部署 `main`。

推荐方式：

1. 本地在 `custom/prod` 完成开发和验证
2. 推送到 `origin/custom/prod`
3. 生产环境使用你的 fork 和 `custom/prod` 分支部署

如果生产环境也是源码构建，建议明确拉取你的 fork：

```bash
git clone -b custom/prod https://github.com/Littlewu111/sub2api.git
```

## 常用检查命令

查看当前分支状态：

```bash
git status -sb
git branch -vv
git remote -v
```

查看最近提交关系：

```bash
git log --oneline --decorate --graph -n 20 --all
```

检查本地功能提交是否已被远端等价吸收：

```bash
git cherry -v origin/custom/prod custom/prod
```

## 冲突处理建议

如果 `git merge main` 时有冲突：

- 优先保留你在 `custom/prod` 上的业务定制
- 对于上游新增功能，尽量以最小改动接入
- 不要修改已经存在的旧 migration
- 数据库结构变更统一新增 migration 文件

## 不建议做的事

- 不要直接在 `main` 上做定制开发
- 不要把运行时数据、测试账号、数据库 volume 提交进 Git
- 不要修改已经执行过的 migration 文件
- 不要长期在 `custom/prod` 上堆很多未拆分的大改动

## 推荐节奏

比较稳的维护节奏是：

1. 平时在 `custom/prod` 基础上切 `feat/*` 开发
2. 每隔一段时间同步一次 `upstream/main`
3. 先把 `main` 跟上游对齐，再把 `main` 合并进 `custom/prod`
4. 每次上线前做一次本地打包验证

按这套方式维护，后续同步上游时的冲突会明显更少。
