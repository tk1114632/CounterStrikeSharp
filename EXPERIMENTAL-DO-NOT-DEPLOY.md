# ⚠️ 实验分支 — 禁止部署生产

2026-07-17 生产验证：本分支（RTLD_DEEPBIND 预载）启动即崩
（hostfxr 初始化阶段 `free(): invalid pointer`）。

原因：DEEPBIND 使运行时侧 new/delete 转向 glibc 后，与 CSS 自身
native 层（hl2sdk memoverride → tier0 jemalloc）在 CSS↔hostfxr
边界产生反向跨分配器错配。修复思路需要重设计（如仅对 coreclr 生效、
或配合 CSS 侧边界隔离）。

生产请使用 `fix/net8-runtime-downgrade` 分支，见 main 分支 PRODUCTION.md。
