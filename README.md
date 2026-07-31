# Songloft — Home Assistant 加载项

本仓库是 Songloft 的 **Home Assistant OS（HAOS）加载项仓库**。用户在 HA「加载项商店 → 仓库」添加本仓库地址即可一键安装：

```
https://github.com/songloft-org/home-assistant-addon
```

> **主仓库**：[songloft-org/songloft](https://github.com/songloft-org/songloft) —— Songloft 的后端与
> 客户端源码、CHANGELOG、issue 都在那里。本仓库只放 Home Assistant 加载项的打包定义，并作为
> 主仓库的子模块挂在 `home-assistant-addon/` 路径下（文档站 `/addon/` 页面即由此同步生成）。
> **请到主仓库提 issue。**

> **本文件面向维护者 / AI**，讲清楚这套加载项怎么设计、为什么这么设计、怎么本地验证、怎么发版。
> **终端用户文档**（安装步骤、选项说明）在 [`songloft/DOCS.md`](songloft/DOCS.md)（双语，HA 详情页展示）；
> 主仓库 README 也有「Home Assistant 加载项」小节。

## 目录结构

```
repository.yaml                # 加载项仓库清单（必须在仓库根，见踩坑 0）
LICENSE                        # Apache-2.0，与主仓库同一份
songloft/                      # 加载项本体（Supervisor 递归 glob 发现）
├── config.yaml                # 加载项核心元数据（arch/ports/options/schema/map/webui）
├── build.yaml                 # 各架构 base image = songloft/songloft:<tag>
├── Dockerfile                 # FROM base image + run.sh 薄层
├── run.sh                     # 读 /data/options.json → 转 env → 启动后端
├── DOCS.md                    # 用户向文档（双语，HA 详情页）
├── icon.png / logo.png        # 商店展示图（由主仓库 docs/public/logo.png 手工生成）
└── translations/{en,zh}.yaml  # 选项字段的本地化标签
.github/workflows/sync-version.yml  # 定时从主仓库最新正式版同步版本号（见下）
```

## 设计决策与踩坑（重要）

### 0. 为什么有这个独立仓库：`repository.yaml` 必须在**仓库根目录**（songloft-org/songloft#340）
Supervisor 校验自定义仓库时**只看仓库根目录**：`supervisor/store/repository.py` 的
`RepositoryGit.validate()` 逐个尝试 `Path(self._git.path / f"repository{filetype}")`
（`filetype ∈ [.yaml, .yml, .json]`），一个都不存在就直接 `return False` →
`StoreInvalidAppRepo: ... is not a valid app repository`，并把刚 clone 的仓库删掉。
**它不会递归查找。**

加载项原先寄居在主仓库的 `addon/` 目录，清单也放在 `addon/repository.yaml`，于是对用户表现为
「加载项商店添加仓库必失败」——从加载项上线之日起就是坏的，**从来没有人成功添加过**。
本仓库把清单放在根目录，这就是那个 bug 的终态修复。

反过来，**加载项本体不必在根目录**：`supervisor/store/data.py` 的 `_find_app_configs` 用的是
`path.glob("**/config.*")`（递归，仅跳过以 `.` 开头的路径段和 `rootfs`），所以 `songloft/config.yaml`
能被正常发现。

推论与注意：
- 递归 glob 会扫**整个仓库**。仓库里任何**非**点号目录下的 `config.yaml` / `config.yml` /
  `config.json` 都会被当成加载项候选去校验。目前仅 `songloft/config.yaml` 命中
  （`.github/**` 因 `.github` 以 `.` 开头被跳过）。**新增此类文件前先确认不会被误当加载项**，
  回归测试见下文「本地验证」。
- 新版 Supervisor 已把 add-on 改称 app（源码目录 `supervisor/apps/`），日志里出现的是
  "app repository" 字样，但配置文件名仍是 `repository.yaml` / `config.yaml`，向后兼容。
- **别把加载项塞回主仓库，也别在主仓库放一份 `repository.yaml` 做「兼容」。** Supervisor clone
  自定义仓库时带 `recursive=True` + `depth=1` + `shallow-submodules`，会把主仓库全部 13 个子模块
  一起拉下来（约 60 MiB，用户实测 2 分钟）。而且主仓库把本仓库挂成了子模块 ——
  子模块已 checkout 时那个递归 glob **确实会命中** `home-assistant-addon/songloft/config.yaml`，
  于是拆仓收益全部还回去，还多出「两个来源同一个 slug `songloft`」的冲突。
  本仓库无任何子模块，clone 是百 KiB 级。

### 1. 薄层复用已发布镜像，不重新编译 Go
`Dockerfile` 只做 `FROM songloft/songloft:<tag>` + 叠一个 `run.sh`，**不重新编译 Go**。base image 由 `build.yaml` 指定，复用 CI 已推送到 Docker Hub 的多架构 manifest。构建极轻（只加一层脚本）。

### 2. 音乐目录走 `MUSIC_DIR` 环境变量，不用 `-music` flag
`run.sh` 用 `MUSIC_DIR` env 传音乐目录（默认 `/media`），**不用 `-music` flag**。原因：
- `-music` flag 曾长期未进入发布版，用未知 flag 启动会让后端**直接崩溃**（`flag provided but not defined`）。
- 基础镜像声明了 `VOLUME /app/music`，容器内该路径是挂载点，`rm`/`ln -s` 会 `Resource busy`，**无法用 symlink 把 `/app/music` 重定向到 `/media`**。
- **env 的优势：未知环境变量会被旧后端静默忽略，不崩溃** → 优雅降级。不支持 `MUSIC_DIR` 的镜像仍能正常启动（音乐目录回落默认），用户可在 Web UI 手动设 `/media`。

后端侧入口在主仓库 `internal/app/app.go` 的 `ParseConfig`（`MUSIC_DIR` 与 `LISTEN_PORT`/`BASE_PATH`/`DB_PATH` 同款 env fallback，优先级低于 `-music` flag）。

### 3. 不设 `image:` 字段 → 本地自构建
`config.yaml` **故意不设 `image:`**，从而走 `build.yaml` 本地自构建。若用纯 `image:` 引用预构建镜像，就**无法把 HA 选项页填的账号密码/音乐路径注入进去**（HA 只把选项写到 `/data/options.json`，得靠 `run.sh` 读取转 env）。

### 4. 数据持久化到 `/data`，音乐映射 `/media` `/share`
`run.sh` 设 `DB_PATH=/data/songloft.db`，DB/缓存/插件数据落到 HA 持久化目录 `/data`，卸载重装不丢。音乐来源目录 `map: [media:rw, share:rw]`。

### 5. `ENTRYPOINT []` 重置基础镜像入口
基础镜像的 `docker-entrypoint.sh` 含二进制热替换逻辑（Docker 场景用），HA 场景不需要（HA 靠重建镜像升级）。`Dockerfile` 用 `ENTRYPOINT []` 重置，改由 `run.sh`（`CMD`）直接启动。

### 6. 架构对应
`config.yaml` 的 `arch: [amd64, aarch64, armv7]` 对应镜像 manifest 的 `linux/amd64`、`linux/arm64`、`linux/arm/v7`（已由 CI `--platform` 确认齐全）。

Supervisor 会对 `armv7` 打一条 deprecation 告警（`App 'Songloft' uses deprecated 'arch' values: ['armv7']`），
但**仅告警、不失效**。**刻意保留**：我们的 Docker manifest 确实发了 `linux/arm/v7`，删掉会直接砍掉 32 位 ARM 用户。

## CI 版本同步（`.github/workflows/sync-version.yml`）

由**本仓库自己定时拉取**主仓库的最新正式版版本号，改写两个文件后 commit：
- `songloft/config.yaml` 的 `version:` → 决定 HA「有可用更新」提示
- `songloft/build.yaml` 的三处 `songloft/songloft:<tag>` → 让自构建拉该版本 base image，而非漂移的 `:latest`

触发：`schedule: '17 */6 * * *'`（每 6 小时；分钟刻意避开 0/30，那两个刻度上全球 Actions 排队最凶）
\+ `workflow_dispatch`（发版后想立刻生效就手动跑一次）。

**为什么是本仓库「拉」而不是主仓库「推」**：跨仓库 push 需要 PAT 并长期维护它的过期，而本仓库
反向拉取只用自己的 `GITHUB_TOKEN`，**零密钥**。代价是最长 6 小时延迟 —— HA 侧 store 刷新本来
也不实时，可接受。

**三道守卫，缺一个都有真实故障模式**：
1. 形态守卫 `^[0-9]+\.[0-9]+\.[0-9]+$` —— 万一拿到 `dev` 之类异常 tag，宁可整个 job 失败，
   也绝不能让 `build.yaml` 指向不存在的 base image tag（那会让**所有** HA 用户装不上）
2. `sort -V` 只允许版本前进 —— 主仓库重跑同一个 tag 时会先 `gh release delete` 再 create，
   那个窗口里 `releases/latest` 会短暂回落到上一个正式版；没这道守卫就会来回写两次无意义 commit
3. `docker manifest inspect` —— 兜住「release 已建但镜像推送失败」的极端情况

**两个已核实的前提**：`releases/latest` 的语义是「最新的非 draft、非 prerelease release」，
而主仓库 dev 构建建 release 时带 `--prerelease`、正式发布带 `--latest`，双重保证拿到的是正式版；
主仓库 `create-release` job 的 `needs` 含 `docker-build-push`，所以 release 出现时多架构
manifest 已推送完毕。

> GitHub 会停用「60 天无活动的公开仓库」的 schedule。本仓库活动很少，真被停了就手动
> `workflow_dispatch` 跑一次恢复即可，不必为此加额外机制。

## 本地验证

不装 HA 也能验证核心逻辑：

```sh
# BUILD_FROM 用 build.yaml 里写的确切版本 tag，顺带验证那个 tag 真的可拉
V=$(grep -m1 '^version:' songloft/config.yaml | sed -e 's/^version:[[:space:]]*//' -e 's/"//g')
docker build -t songloft-addon --build-arg "BUILD_FROM=songloft/songloft:${V}" songloft

mkdir -p /tmp/sl-data /tmp/sl-media
# 刻意用非默认账号，这样才能证明选项真的被注入（用 admin/admin 测等于没测）
echo '{"admin_username":"haadmin","admin_password":"test123","music_path":"/media","base_path":""}' \
  > /tmp/sl-data/options.json
# 宿主端口挑一个空闲的：58091 常被开发中的服务占着，端口冲突时容器会起不来，
# 而 curl 会打到那个【别的】服务上，给出一切正常的假象
docker run -d --name sl-addon-test -p 58291:58091 \
  -v /tmp/sl-data:/data -v /tmp/sl-media:/media songloft-addon
sleep 8
docker ps --filter name=sl-addon-test --format '{{.Status}}'   # 先确认容器真的在跑
curl -s http://localhost:58291/api/v1/version                 # 应返回 base image 的版本
```

三条**有效**的断言（下面这些才真正证明 `run.sh` 的选项转换生效）：

```sh
# 1) ADMIN_USERNAME / ADMIN_PASSWORD —— 用 options.json 的账号能登录，且默认 admin/admin 被拒
curl -s -X POST http://localhost:58291/api/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"username":"haadmin","password":"test123"}' | head -c 60   # 应返回 access_token
curl -s -X POST http://localhost:58291/api/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"admin"}'                    # 应返回 401 凭据错误

# 2) DB_PATH —— DB 落在挂载出来的 /data
ls /tmp/sl-data/songloft.db

# 3) MUSIC_DIR —— 启动日志里的音乐目录应为 /media
docker logs sl-addon-test 2>&1 | grep '音乐目录'

docker rm -f sl-addon-test
```

> **不要用 `docker exec ... printenv MUSIC_DIR ADMIN_USERNAME DB_PATH` 来验证。**
> `docker exec` 起的是新进程，只继承**镜像的 `ENV`**，拿不到 `run.sh` 里 `export` 的变量
> （那些只传给它 `exec` 出来的后端进程）。实测它会显示基础镜像预设的 `admin`/`admin`，
> 与 `options.json` 里填的值完全无关 —— 一个稳定给出错误结论的「验证」。
>
> **若第 3 条显示 `path=music` 而不是 `/media`**，先看 base image 版本：`MUSIC_DIR` 支持是
> 2026-07-09 随加载项一起加进后端的，早于此的镜像（如 v2.9.6）不认这个变量。
> 此时后端**静默忽略并回落默认**、不崩溃 —— 这正是踩坑 2 所说的优雅降级，不是缺陷。
> 用 `:latest` 或确切版本 tag 重测即可。

**仓库级校验**（验证 HA「添加仓库」这一步会不会被拒，即踩坑 0 的回归测试）。在**仓库根目录**跑，
复刻 `RepositoryGit.validate()` + `_find_app_configs()` 两处判定，零额外依赖：

```sh
python3 - <<'PY'
from pathlib import Path
root = Path(".")
SUFFIX = (".yaml", ".yml", ".json")
# 1) Supervisor 只在【仓库根目录】找 repository.{yaml,yml,json}，找不到就拒收整个仓库
assert any((root / f"repository{s}").exists() for s in SUFFIX), "根目录缺 repository.yaml"
# 2) 加载项用递归 glob 发现；这里必须【只有】songloft/config.yaml 一项
apps = sorted(str(p) for p in root.glob("**/config.*")
              if p.suffix in SUFFIX
              and not [x for x in p.parts if x.startswith(".") or x == "rootfs"])
assert apps == ["songloft/config.yaml"], f"加载项候选异常: {apps}"
print("OK")
PY
```

**版本一致性**（`config.yaml`、`build.yaml`、主仓库 release、Docker Hub 四者必须对齐）：

```sh
V=$(grep -m1 '^version:' songloft/config.yaml | sed -e 's/^version:[[:space:]]*//' -e 's/"//g')
[ "$(grep -c "songloft/songloft:${V}" songloft/build.yaml)" = 3 ] && echo "build.yaml 三处一致"
curl -s https://api.github.com/repos/songloft-org/songloft/releases/latest \
  | grep -o '"tag_name": *"[^"]*"'                     # 应为 v$V
docker manifest inspect "songloft/songloft:${V}" \
  | grep -oE '"architecture": *"[^"]*"|"variant": *"[^"]*"' | sort -u
# 应覆盖 amd64 / arm64 / arm+v7，与 config.yaml 的 arch: [amd64, aarch64, armv7] 对应
```

要连 `config.yaml` / `build.yaml` 的字段一起按**真实** schema 校验，就 clone
`home-assistant/supervisor`，在 python 3.12 容器里 `pip install voluptuous awesomeversion pyyaml
aiohttp attrs ciso8601 pycares sentry-sdk aiodns atomicwrites-homeassistant orjson pyudev securetar
GitPython` 后 import `supervisor.store.validate.SCHEMA_REPOSITORY_CONFIG` /
`supervisor.apps.validate.{SCHEMA_APP_CONFIG,SCHEMA_BUILD_CONFIG,SCHEMA_APP_TRANSLATIONS}` 逐个喂进去。
依赖较重且 upstream `main` 偶尔带语法错误（跑不通时先 grep `except A, B:` 之类），
所以只在改动 `config.yaml` 字段时才值得走这条路。

HA 端真机：加载项商店 → 添加仓库 `https://github.com/songloft-org/home-assistant-addon` → 安装 →
配置页填选项 → 启动 → 「打开 Web UI」。添加仓库应在**数秒内**完成（对比寄居主仓库时的 2 分钟）。

## 生效前提

音乐目录在 HA 端全自动，需 base image 是**包含 `MUSIC_DIR` 支持的发布版**。旧镜像上加载项仍可安装运行，仅音乐目录需在 Web UI 手动设 `/media`（优雅降级，见上文第 2 点）。

## 改完本仓库之后

本仓库是主仓库的子模块。文档站 <https://songloft.hanxi.cc/addon/> 由主仓库的
`scripts/sync-docs.mjs` 从**主仓库记录的子模块指针**读取本仓库的 `README.md` 与
`songloft/DOCS.md`。改了这两个文件后若不回主仓库 bump 指针，文档站会一直显示旧内容：

```sh
cd <songloft 主仓库>
git submodule update --remote home-assistant-addon
git add home-assistant-addon
git commit -m "chore(addon): bump home-assistant-addon submodule pointer"
git push origin main
```

`sync-version.yml` 产生的版本号 commit **不需要**这么做（`config.yaml` / `build.yaml` 不进文档站）。

> 主仓库 `static.yml` 的 `paths:` 里登记的是**裸路径** `home-assistant-addon`（不是
> `home-assistant-addon/**`）—— 子模块变更在 push event 里的路径就是 gitlink 本身、不带斜杠，
> 带 `/**` 的模式永远匹配不上，指针 bump 就不会触发文档站部署。

本仓库 commit **引用主仓库 issue 必须写完整路径** `songloft-org/songloft#123`，
短写 `#123` 会被 GitHub 解析成本仓库自己的 issue。
