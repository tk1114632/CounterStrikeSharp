# TG 生产维护手册（fork 专用，不合入上游）

> 最后更新：2026-07-17。维护者：Henry + Claude。
> 背景：2026-07 调查确认，上游自 v1.0.369 升级 .NET 10 后，`libcoreclr` 开始从外部导入
> `operator new/delete`，在 CS2 进程内被 `libtier0`（jemalloc）抢先绑定，导致跨分配器
> 分配/释放错配 → 生产服凌晨批量段错误（崩溃点 `je_calloc`，9 份 core 实锤）。
> 生产解法：**保持上游最新代码 + 回退 .NET 8 runtime**（net8 分支）。上线首日全类型实例零崩溃。

## 分支结构

| 分支 | 用途 | 状态 |
|---|---|---|
| `main` | 跟随上游 roflmuffin/main + fork 专属文件（本文档、`.github/workflows/fork-release.yml`） | 持续同步 |
| `fix/net8-runtime-downgrade` | **生产主线**。相对上游 main 仅 2 文件差异（见下） | 生产运行中 |
| `fix/dotnet-tier0-allocator-mismatch` | RTLD_DEEPBIND 实验修复 | ⚠️ 挂起：启动即崩（`free(): invalid pointer`，CSS native memoverride↔runtime 边界反向错配 + 当时部署环境混入双版本 runtime）。仅作研究，勿部署 |

`fix/net8-runtime-downgrade` 的全部差异（合并冲突只可能出现在这两处）：

1. `managed/CounterStrikeSharp.API/CounterStrikeSharp.API.csproj`
   - `<TargetFramework>net8.0</TargetFramework>`（上游为 net10.0；上游若加新 PackageReference，保留但把 `10.x` 版本改回对应 `8.x`）
2. `src/scripting/dotnet_host.cpp`
   - hostfxr 路径 `dotnet/host/fxr/8.0.3/`（上游为 `10.0.3`）

## 上游更新 → 生产发布 SOP

```bash
# 1) 同步 fork main（保留 fork 专属文件；GitHub 冲突时用网页 Sync fork 或手动 merge）
gh repo sync tk1114632/CounterStrikeSharp --source roflmuffin/CounterStrikeSharp
cd ~/Documents/CS2Dev/CounterStrikeSharp
git fetch origin   # origin = roflmuffin（上游）
git checkout main && git merge origin/main && git push fork main

# 2) 变基生产分支（冲突面见上表，通常 1 分钟内解决）
git checkout fix/net8-runtime-downgrade
git rebase main
git push -f fork fix/net8-runtime-downgrade

# 3) 触发 fork CI 构建（native 用官方同款 steamrt sniper 容器）
gh workflow run "Fork Release Build" --repo tk1114632/CounterStrikeSharp \
  -f ref=fix/net8-runtime-downgrade -f version=1.0.<上游版本号>-net8 -f runtime=8.0.3

# 4) 下载产物（也可在 Actions 页面下载）
gh run list --repo tk1114632/CounterStrikeSharp --workflow "Fork Release Build" --limit 1
gh run download <run-id> --repo tk1114632/CounterStrikeSharp

# 5) 发布前校验（三板斧）
#   a. 解包出 counterstrikesharp.so： strings 应含 "fxr/8.0.3"，不含 "DEEPBIND"
#   b. GLIBCXX 需求 ≤ 3.4.30（机群最老 Ubuntu 22.04 的上限；steamrt 构建天然满足）
#   c. api/CounterStrikeSharp.API.runtimeconfig.json 的 tfm 为 net8.0
```

## 部署 SOP（每台宿主机每个游戏用户）

```bash
cd ~/serverfiles/game/csgo/addons
cp -r counterstrikesharp counterstrikesharp.bak-$(date +%m%d)   # 备份
# ⚠️ 关键：先删旧 runtime 与 api，禁止覆盖解压（07-17 deepbind 启动崩溃的直接教训：
#    dotnet/ 内残留双版本 runtime 会被同时加载）
rm -rf counterstrikesharp/dotnet counterstrikesharp/api counterstrikesharp/bin
unzip -o /tmp/counterstrikesharp-with-runtime-linux-<ver>.zip -d ~/serverfiles/game/csgo/
# configs/ gamedata/ plugins/ 不受影响（zip 内 gamedata 会覆盖为上游最新，属预期）
```

- 生效：等 06:00 例行 quit + monitor 重启，或手动重启单实例先验证
- 验证日志行：console 出现 `Loading hostfxr from .../dotnet/host/fxr/8.0.3/libhostfxr.so` 且插件正常加载
- 回滚：`rm -rf counterstrikesharp && mv counterstrikesharp.bak-<date> counterstrikesharp`

## 硬性约束与风险雷达

1. **所有插件 TargetFramework 必须 ≤ net8.0**（net8 宿主加载不了 net10 目标程序集）。
   新装/重编插件前检查其 `*.deps.json` 的 `.NETCoreApp,Version=`。
2. **上游引入 net10-only API 或 C# 13+ 语法时**，net8 分支变基可能编译失败——CI 的
   build_managed 任务会立刻暴露，届时逐点改写或 `#if NET10_0_OR_GREATER` 处理。
3. **.NET 8 LTS 支持期至 2026-11-10**。到期前需要重新评估：либо上游已正确修复
   分配器错配（跟踪上游 issue/PR），либо把 runtime 换到当时的新 LTS 并重验。
4. 上游相关线索：PR #1363（.NET 10 时代另一独立崩溃：GC 回收委托，SIGABRT+明确报错，
   与本错配无关但同期出现）。我们的根因分析建议以 issue 形式提交上游。
5. 构建产物的 runtime 固定 aspnetcore-runtime-**8.0.3**（与 7 月前长期稳定组合一致）。
   如需升 8.0.x 补丁版，改 workflow 入参 `runtime` 并同步改 `dotnet_host.cpp` 的 fxr 路径。
