# `ghcr-push` — GHCR 健壮推送 composite action

组织级 self-hosted runner（内网 local 机，上行 ~1MB/s）专用。取代各仓库 `publish` job 里
裸奔的 `docker push ghcr.io/...`，根治三类老问题：

- **推送超时红**：链路抖动 / 大层超时 → 整个 job 红，需手动 `gh run rerun <id> --failed`。
- **多仓库同时推「双红」**：N 个 publish 并发把 1MB/s 上行切成 N 份，每份都慢到超时。
- **重推浪费**：rerun 从头再传。

## 三层机制

1. **宿主级 flock 串行**：10 个 runner 共享 `ci-locks` 卷上的同一把锁，push 前 `flock` 抢锁。
   同一时刻**只有一个 GHCR push 独占整条上行**，最快传完、最不易超时；其余排队而非互相拖垮。
   退避 sleep 期间释放锁，把上行让给别的 job。
2. **指数退避重试**：单个 tag push 失败自动重试（默认 6 次，退避 2→4→…→上限 60s）。
3. **断点续传（层级）**：`docker push` 对已存在的层做 HEAD 检查后跳过（`Layer already exists`），
   所以每次重试**只补传中断的那一层**，而非从头再来。

> 锁卷缺失时自动降级为「仅重试」（打 warning），不会因此失败。

## 用法

在 `publish` job 里，用本 action 取代原来的「登录 GHCR + retag + docker push」若干步：

```yaml
  publish:
    runs-on: [self-hosted, local]
    if: github.event_name == 'push'
    needs: [checks, image]
    permissions:
      contents: read
      packages: write
    steps:
      - name: 发布 GHCR（健壮推送）
        uses: birditch-dev/.github/.github/actions/ghcr-push@main
        with:
          image: ghcr.io/${{ github.repository_owner }}/<你的镜像名>
          src: 127.0.0.1:10000/<你的镜像名>:ci-${{ github.sha }}
          token: ${{ secrets.GITHUB_TOKEN }}
          # 以下均有默认值，通常无需填：
          # extra-tags: latest      # 逗号分隔；短 SHA 另由 short-sha 控制
          # short-sha: true         # 额外推 7 位短 SHA
          # retries: 6
          # lock-file: /ci-locks/ghcr.lock
```

默认会推 `latest` + 7 位短 SHA 两个 tag（与既有惯例、`ghcr-cleanup.yml` 保留策略一致），
并写入 `$GITHUB_STEP_SUMMARY`。action 内部自带 `docker login`，无需单独登录步骤。

## 输入

| 输入 | 必填 | 默认 | 说明 |
|------|------|------|------|
| `image` | 是 | — | GHCR 目标镜像（不含 tag） |
| `src` | 是 | — | 本地 registry 的 ci-<sha> 源镜像 |
| `token` | 是 | — | 传 `secrets.GITHUB_TOKEN` |
| `extra-tags` | 否 | `latest` | 逗号分隔额外 tag；空串则不加 |
| `short-sha` | 否 | `true` | 是否额外推 7 位短 SHA |
| `username` | 否 | `github.actor` | 登录用户名 |
| `registry` | 否 | `ghcr.io` | 目标 registry |
| `retries` | 否 | `6` | 单 tag 最大尝试次数 |
| `max-backoff` | 否 | `60` | 退避单次上限（秒） |
| `lock-file` | 否 | `/ci-locks/ghcr.lock` | flock 互斥文件（共享卷） |
| `lock-timeout` | 否 | `3600` | 抢锁等待上限（秒） |

## 依赖：`ci-locks` 共享卷

宿主级串行需要 10 个 runner 都挂载同一个卷。`github-runner/docker-compose.yml` 里每个 runner 的
`volumes:` 追加 `- ci-locks:/ci-locks`，并在顶层 `volumes:` 声明 `ci-locks:`，然后 `docker compose up -d`。
未挂载时 action 仍可用（自动降级为仅重试）。
