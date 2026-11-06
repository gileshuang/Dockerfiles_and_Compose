# ReMe 记忆服务 · podman compose 部署

外部记忆服务 + DSH 插件分离模式：ReMe 独立运行并通过 HTTP 提供记忆能力，
DSH 用 `@agentscope-ai/reme-dsh-plugin` 连接；记忆以 Markdown 落盘，多个 agent 共用同一工作区。

## 文件

| 文件 | 用途 |
|---|---|
| `compose.yaml` | **正式编排**，用于 root podman（或具备 subuid 的 rootless podman） |
| `.env.example` | 环境变量样例，复制为 `.env` 后修改 |
| `compose.verify.yaml` | 本次在 bwrap 沙箱内做验证时实际使用的编排（见「沙箱差异」） |
| `data/` | 记忆工作区（Markdown + 可重建索引） |

## 部署（root podman）

```sh
mkdir -p /srv/reme/data && chown 1000:1000 /srv/reme/data
cd /srv/reme
cp /path/to/compose.yaml .
cp /path/to/.env.example .env      # 按需填写 LLM / EMBEDDING
$EDITOR .env                       # REME_DATA_DIR 改成 /srv/reme/data
```

```sh
podman compose up -d
podman compose ps
podman compose logs -f --tail 50
```

验证：

```sh
curl -s -X POST http://127.0.0.1:2333/health_check -H 'Content-Type: application/json' -d '{}'
# → {"answer":"✅ ReMe v0.4.2.1 - healthy","success":true,...}
```

浏览器打开 `http://127.0.0.1:2333/` 是 ReMe Workspace（Studio）界面。

停止 / 清理：

```sh
podman compose down          # 保留 ./data
podman compose down -v       # 连容器一起清掉（bind mount 数据仍在）
```

## 工作区布局（全部是普通文件）

```
data/
├── daily/YYYY-MM-DD.md      # 每日记忆：事实、会话摘要、资料阅读
├── daily/YYYY-MM-DD/*.md    # 主题卡片
├── digest/                  # 长期记忆：personal / 经验 / 知识节点
├── resource/                # 外部原始资料
├── session/                 # 会话原始记录（jsonl）
└── metadata/                # 索引、图、目录、缓存 —— 可重建，删了不丢数据
```

`daily/` 与 `digest/` 下的 `.md` 是**唯一真相来源**；服务用 watcher 监听这两个目录，
文件一变就重新索引。手工编辑、`git` 管理、其他 agent 直接写文件都能被感知。

## DSH 侧接入

```sh
dsh plugin --profile web add @agentscope-ai/reme-dsh-plugin
```

插件配置页 → ReMe bundle → `reme-memory` → Configure：

| 配置 | 值 |
|---|---|
| `endpoint` | `http://127.0.0.1:2333`（或服务实际地址） |
| `autoMemoryEnabled` | 需要自动记录就保持 `true` |
| `autoDreamEnabled` | 未配 LLM 时设为 `false`，否则后台整理会失败 |
| `dreamCron` | 默认 `0 23 * * *`，多客户端共用时应只留一个调度方 |

其他 agent（Claude Code / Codex / OpenClaw 等）各自装对应插件并指向同一个 `endpoint`，
即可共用同一份 Markdown 记忆。</br>

## 本次验证结果

环境：bwrap 容器内 rootless podman 5.x + podman-compose，镜像 `ghcr.io/agentscope-ai/reme:latest`（1.69 GB）。

| 项 | 结果 |
|---|---|
| 镜像拉取 | ✅ 从 `ghcr.io` 成功 |
| 容器启动 | ✅ uvicorn 监听 `0.0.0.0:2333` |
| `/health_check` | ✅ `ReMe v0.4.2.1 - healthy` |
| Studio 界面 | ✅ `GET /` 返回 ReMe Workspace |
| Markdown 落盘 | ✅ 写入 `data/daily/2026-10-28.md` |
| watcher 自动索引 | ✅ 索引体积增长，无需重启 |
| 检索召回 | ✅ `/search` 返回 `daily/2026-10-28.md:4-13`，score 1.7261 |
| agent 写入通道 | ✅ `/write` 写入 `daily/2026-10-28/api-probe.md` |

检索为纯关键词（BM25）路径即可命中，未配置 embedding 也能工作。

## 沙箱差异（仅验证环境需要，正式部署可忽略）

本次验证在 dsh 的 bwrap 容器内进行，与常规主机有以下差异，`compose.verify.yaml` 为此做了三处改动：

1. **无 `/etc/subuid`**：rootless 只有单 UID 映射（容器 uid 0 = 宿主 uid 1000），
   镜像里多 UID/GID 的文件解包会 `chown` 失败。解法是 `--storage-driver vfs --storage-opt ignore_chown_errors=true`。
2. **`/run/user/1000` 只读**：`--root` / `--runroot` / `XDG_RUNTIME_DIR` 全部重定向到可写目录。
3. **缺 `/dev/net/tun`**：rootless 端口发布不可用，改用 `network_mode: host` 且 `user: "0:0"`。

正式部署使用 `compose.yaml` 即可，以上都不需要。

## 注意事项

- **ReMe HTTP 无鉴权**：默认只绑 `127.0.0.1`。跨机访问必须走 SSH 隧道、内网 ACL 或前置反向代理加认证。
- **`user:` 与数据目录归属必须匹配**：不匹配会导致 Markdown 文件写不进去或属主错乱。
- **`auto_dream` 会调用 LLM**：需要 `LLM_API_KEY`/`LLM_BASE_URL`；多客户端共用一个服务时只应有一个调度器。
- **升级**：`podman compose pull && podman compose up -d`；`metadata/` 会自动重建。
