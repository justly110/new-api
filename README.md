# new-api (Patched Build)

一个用于**自动跟随上游 [QuantumNous/new-api](https://github.com/QuantumNous/new-api) 官方 Release、打补丁并编译发布**的自动化仓库。

本仓库**不存放 new-api 的源码**，只包含一个 GitHub Actions 工作流：
[`.github/workflows/auto-sync-patch-release.yml`](.github/workflows/auto-sync-patch-release.yml)

---

## 补丁说明

上游 `main.go` 默认监听 `:<port>`（即所有网卡，`0.0.0.0`）。本仓库在构建前会把监听地址改为：

```go
Addr: "127.0.0.1:" + port
```

即**只监听本机回环地址**，服务不再对外直接暴露，必须由 Nginx / Caddy 等反向代理转发访问。

这样做的用途：把一个公网可达的服务限制在内网，避免端口直接暴露在公网上。

---

## 工作原理

每 2 小时（`cron: '0 */2 * * *'`）自动执行一次，流程如下：

1. **检查上游最新 Release** — 通过 `gh release view --repo QuantumNous/new-api` 获取最新 tag，例如 `v1.0.0-rc.49`。
2. **判断是否需要构建** — 若 `<上游tag>-patched` 的 Release 已存在则跳过；否则继续。
3. **检出上游代码** — 直接 checkout 上游对应 tag 的源码（`actions/checkout` + `repository`/`ref`），**不使用**本仓库中的代码。
4. **打补丁** — 用 `sed` 将 `main.go` 的 `Addr: ":" + port` 替换为 `Addr: "127.0.0.1:" + port`，并校验替换是否成功（失败则构建报错退出）。
5. **构建前端** — 使用 Bun `1.4.4` 在 `web/` 下执行 `bun install --frozen-lockfile` + `bun run build`（前端产物是 `go:embed` 的依赖，必须先于后端构建）。
6. **编译后端** — 使用 `go.mod` 指定的 Go 版本，`CGO_ENABLED=0 GOOS=linux GOARCH=amd64` 静态编译，版本号写入 `common.Version`。
7. **发布 Release** — 打包为 `new-api-linux-amd64-<tag>.tar.gz`，通过 GitHub API 创建 tag 为 `<上游tag>-patched` 的 Release 并上传附件。
8. **清理旧版本** — 只保留最近 `KEEP_RELEASES` 个（默认 `2` 个）`-patched` Release，更早的连同 tag 一起删除。

---

## 使用方法

### 自动

无需任何操作。上游发布新版本后，最多 2 小时内会自动产出对应的 `-patched` 构建。

### 手动触发

在 **Actions → Auto Sync Upstream Release, Patch and Build → Run workflow** 中运行，可选填写：

| 输入项 | 说明 |
| --- | --- |
| `force_tag` | 指定上游 tag 强制构建，例如 `v1.0.0-rc.49`；留空则自动检测最新 Release |

若指定的版本对应的 Release 已存在，会被删除后重新构建。

### 下载构建产物

在 [Releases](https://github.com/justly110/new-api/releases) 页面下载：

```
new-api-linux-amd64-<上游tag>.tar.gz
```

解压后得到 `new-api-linux-amd64` 可执行文件：

```bash
tar -xzvf new-api-linux-amd64-v1.0.0-rc.49.tar.gz
chmod +x new-api-linux-amd64
./new-api-linux-amd64 --port 3000
```

此时服务只监听 `127.0.0.1:3000`，从其他机器访问会失败 —— 这是预期行为，请在前面配置反向代理。

---

## 注意事项

- **仅构建 Linux amd64** 单一平台产物。需要其他架构（arm64 等）请自行修改 `GOARCH`，或在 workflow 中加入矩阵构建。
- **本仓库代码不参与构建**：构建的源码来自上游 tag。若你想在本仓库维护自己的代码改动，需要把 `actions/checkout` 的 `repository`/`ref` 参数去掉并改为自行合并上游。
- **补丁依赖上游代码特征**：若上游修改了 `main.go` 中 `Addr: ":" + port` 的写法，`sed` 会匹配失败，工作流会显式报错退出（不会静默产出未打补丁的版本），需要更新正则。
- **权限**：工作流使用 `secrets.GITHUB_TOKEN` 并声明 `permissions: contents: write`，无需额外配置 Personal Access Token。
- **并发控制**：通过 `concurrency: auto-sync-build` 保证同一时间只有一个构建在跑，避免重复发布。
- **上游项目协议与文档**：请参考 [QuantumNous/new-api](https://github.com/QuantumNous/new-api)。

---

## 相关链接

- 上游项目：<https://github.com/QuantumNous/new-api>
- 本仓库 Releases：<https://github.com/justly110/new-api/releases>
