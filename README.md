# 陋居 Web 构建

把陋居的每个版本导出成独立的 Godot Web 页面，用于并排对比不同表现方案和不同世界快照。

## 公网地址

同一份内容发布在两个平台，任何人可直接访问：

- GitHub Pages：<https://wenh09.github.io/burrow-web-builds/>
- itch.io：<https://wenh09.itch.io/burrow-web-builds>

GitHub Pages 从仓库 `wenh09/burrow-web-builds` 的 `main` 分支根目录发布。itch.io 是同名项目，
类型为 HTML、定价 Free、嵌入视口 1152×648。

索引页的版本链接写成显式的 `<dir>/index.html`。itch 的 CDN 不提供目录索引，
写成 `<dir>/` 会 404；GitHub Pages 两种都能解析，所以显式写法同时适配两边。

## 本地打开

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

## 重新发布到公网

```bash
python3 tools/godot/publish_burrow_pages.py    # 生成去重后的 artifacts/web-site/

# GitHub Pages
cd artifacts/web-site && git add -A && git commit -m "..." && git push

# itch.io（butler 只上传变化的部分）
export BUTLER_API_KEY=<你的 key>               # itch.io/user/settings/api-keys
butler push artifacts/web-site wenh09/burrow-web-builds:html
```

Pages 会在约 40 秒内自动重新构建。itch 侧推送后需等 build 处理完成，
用 `butler status wenh09/burrow-web-builds:html` 查看。

`tools/godot/package_burrow_itch.py` 可另外打出一份手动上传用的 zip
（`artifacts/burrow-itch-io.zip`，144MB），在没有 butler 时用网页拖拽上传。

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

## 公网发布的额外处理

**引擎二进制去重。** 八份构建各带一份完全相同的 37.7 MB `index.wasm` 和相同的 `index.js`，直接上传是 438 MB，其中约 300 MB 是纯重复。Godot 加载器从 `executable` 解析引擎路径（`${loadPath}.wasm` / `.js`），但数据包由 `mainPack` 独立指定，所以可以让所有版本共用一份 `engine/`、各自只带自己的 pck。发布目录因此降到 172 MB，最大单文件 58.7 MB——在 Pages 的 1 GB 站点上限与 100 MB 单文件上限之内（58.7 MB 会触发 GitHub 超过 50 MB 的建议值 warning，不影响推送）。

`publish_burrow_pages.py` 在 hoist 之前会逐个比对 md5，任一文件在各构建间不一致就直接中止，不会盲目共享。

**公网侧验证。** `engine/index.wasm` 返回 `content-type: application/wasm`，八个页面与其 pck 均为 200。Pages 上的实际渲染通过 canvas 像素读回确认：canvas 按 devicePixelRatio 放大到 1316×1622、加载层已移除、WebGL2 可用，中心区域采样到房屋墙面的米黄色 (234,220,197)。

**itch.io 侧的两个差异。** itch 把整份 zip 放在 `html-classic.itch.zone` 上用 iframe 承载，
CDN 不提供目录索引，所以索引页链接必须写成 `<dir>/index.html`（第一次发布时写成 `<dir>/`，
八个版本在 itch 上全部 404，Pages 上却正常）。itch 会自动配好 Godot Web 需要的跨源头，
不需要像 Pages 那样依赖关掉线程支持。

itch 侧同样验证到实际渲染：`03-true3d-heroic` 页面标题为 `hero_shooter_light`，
canvas 2560×1920、加载层已移除，画面中部采样到木质棕 (104,71,48)、下部蓝灰 (44,61,75)，
多次采样稳定。

**itch 项目的必填项。** 项目类型必须设为 HTML 并配置嵌入尺寸，否则页面报
"haven't configured how your project is embedded"。AI 内容披露为强制项，此处按实际情况
勾选了 Graphics 与 Code。封面图为必填但尚未上传，需在项目编辑页手动上传
（`/tmp/burrow-cover-final.png` 是从 02 版本实际画面导出的 1260×1000 图，可直接用）。
