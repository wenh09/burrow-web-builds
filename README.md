# 陋居 Web 构建

把陋居的每个版本导出成独立的本地 Godot Web 页面，用于并排对比不同表现方案和不同世界快照。

## 打开方式

```bash
python3 tools/godot/serve_burrow_web_builds.py --port 8010
# 然后访问 http://127.0.0.1:8010/
```

必须经 HTTP 打开。直接双击 `index.html` 会因为 `file://` 下无法拉取 `.wasm` / `.pck` 而卡在加载条。

## 重新构建

```bash
python3 tools/godot/build_burrow_web_builds.py              # 全部
python3 tools/godot/build_burrow_web_builds.py --only 02-true3d-retro
```

依赖 Godot 4.7.1 在 PATH 中，以及已安装的 4.7.1 Web 导出模板。

## 版本清单

| 目录 | 场景 | 说明 |
| --- | --- | --- |
| `01-paper-theatre` | `paper_theatre` | 10 张卡片 + 1 个可进入房间，最早的表现试验 |
| `02-true3d-retro` | `true_3d_probe` | 1,057 mesh Blender GLB，12 房间 119 碰撞体，`clean_retro_3d` |
| `03-true3d-heroic` | `true_3d_probe` | 同一 GLB，启动参数切到 `hero_shooter_light` |
| `04-world-proxy-r6` | `main` | `world.json` 当前 revision 6，77 实体 |
| `05-snapshot-r0001` | `main` | checkpoint r0001，50 实体，`burrow_grounds` 首次出现 |
| `06-snapshot-r0002` | `main` | checkpoint r0002，66 实体 |
| `07-snapshot-r0003` | `main` | checkpoint r0003，66 实体 |
| `08-snapshot-r0004` | `main` | checkpoint r0004，72 实体，checkpoint 链末端 |

04-08 走同一份 `world_runtime.gd` 代理渲染，只有原始盒体和颜色，没有 authored mesh。真实几何看 02 / 03。

r5 和 r6 没有 checkpoint（V0 原型期绕过 World Patch 直接同步修改，见 `.world_agent/revision_migrations.json`），所以快照序列止于 r0004，无法回滚到 r5/r6。

## 构建脚本的三个必要处理

**按版本隔离 staging。** 直接对项目根目录导出会把 `assets/vendor/quaternius`（489 MB，运行时未引用）和无关的 Klein-bottle 切片一起打包，试导出时 pck 达 168 MB。脚本为每个版本用 APFS `cp -Rc` 克隆出只含该版本运行时依赖的最小工程，纸片剧场因此降到 16.4 MB。

**用户参数需要 `--` 分隔符。** 场景通过 `OS.get_cmdline_user_args()` 读取 `--style-heroic` 等风格开关，只能看到裸 `--` 之后的参数。写进 `GODOT_CONFIG.args` 的裸参数会被引擎当作未知主参数吃掉，风格静默不生效——03 第一次构建就复现了这个问题（HUD 仍显示 Clean Retro 3D）。脚本注入时会自动补上分隔符。

**HUD 需要中文字形。** Godot 内置字体不含 CJK，Web 导出里 HUD 中文会渲染成方框。脚本从 `scripts/*.gd` 和纸片剧场 presentation 里扫出实际用到的字符，用 fontTools 从系统 Hiragino Sans GB 子集出一个约 69 KB 的字体，注册为项目级 `theme/custom_font`。字符集必须同时包含可打印 ASCII 和 Latin-1 标点，HUD 会把两种文字混在一个 label 里（`WASD 移动 · 鼠标观察`），只取 CJK 会让 WASD 和 `·` 变成方框。字体只作用于 Web 构建，桌面运行不受影响。

## 已验证

八个版本均在 Chrome 中实际加载并截图确认出画面，非仅构建成功。02 / 03 的画面差异可见（材质、光照、地面色调），历史上出现过的"数值门禁通过但外墙缺失"未复现。

`godot --headless` 的 smoke 只证明契约数值成立，不能替代浏览器里的画面确认。
