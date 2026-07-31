# AGENTS.md

本仓库是 **Songloft 的 Home Assistant 加载项仓库**，作为主仓库
[songloft-org/songloft](https://github.com/songloft-org/songloft) 的子模块挂在
`home-assistant-addon/` 路径下。

**唯一真实来源是 [README.md](README.md)** —— 设计决策、踩坑（尤其是「为什么
`repository.yaml` 必须在根目录」、「为什么音乐目录走 `MUSIC_DIR` env 而不是 `-music` flag」）、
CI 版本同步机制、本地验证步骤全在那里。**动本仓库任何文件之前先把 README 读完**，
这套东西踩坑密度高、又极少被碰，凭直觉改几乎必踩。

## 绝对不要做的事

- **不要把 `repository.yaml` 移进子目录**。Supervisor 只在仓库根找它，移走等于让所有用户
  加不了这个仓库（这就是 songloft-org/songloft#340 的成因，本仓库正是为此而拆分）
- **不要新增任何非点号目录下的 `config.yaml` / `config.yml` / `config.json`**。Supervisor 用
  递归 glob `**/config.*` 发现加载项（只跳过以 `.` 开头的路径段和 `rootfs`），多一个就多一个
  幽灵加载项。`.github/` 下的安全
- **不要把 `run.sh` 的 `MUSIC_DIR` 环境变量改回 `-music` flag**。理由见 README 踩坑 2
  （未知 env 被旧后端静默忽略 → 优雅降级；未知 flag 会让后端直接崩溃）
- **不要给 `config.yaml` 加 `image:` 字段**。加了就无法把 HA 选项页填的账号密码注入进去
- **不要删 `arch` 里的 `armv7`**。Supervisor 会打 deprecation 告警，但仅告警不失效；
  删掉会直接砍掉 32 位 ARM 用户

## Git 约定（与主仓库一致）

- 直接提交到 `main`，不新建分支、不走 PR
- Conventional Commits：`type(scope): description`
- 提交信息**禁止** `Co-Authored-By` 尾部标记
- **引用主仓库 issue 必须写完整路径** `songloft-org/songloft#123`。短写 `#123` 会被 GitHub
  解析成本仓库自己的 issue

## 改完文档要回主仓库 bump 指针

文档站 <https://songloft.hanxi.cc/addon/> 由主仓库的 `scripts/sync-docs.mjs` 从
**主仓库记录的子模块指针**读取本仓库的 `README.md` / `songloft/DOCS.md`。改了这两个文件后
不回主仓库更新指针，文档站会一直显示旧内容。命令见 README 末节。

`sync-version.yml` 产生的版本号 commit **不需要**这么做（`config.yaml` / `build.yaml` 不进文档站）。
