# rosz-ui

> **已并入 rosz 仓库**: 自 0.4.0 起本包在 [erban-robotics/rosz](https://github.com/erban-robotics/rosz) 的 [`packages/rosz-ui`](https://github.com/erban-robotics/rosz/tree/main/packages/rosz-ui) (提交历史一并搬入), 版本号与 rosz 统一, tgz 挂在 rosz 的 Release: `"@erban-robotics/rosz-ui": "https://github.com/erban-robotics/rosz/releases/download/v0.4.0/erban-robotics-rosz-ui-0.4.0.tgz"`. 开发期 alias 改指 `~/projects/rosz/packages/rosz-ui/src`. 本仓库不再更新, 0.3.0 及以前的 Release 保留; issue 已转到 rosz.

rosz 的共享可视化 UI 包: 数据源状态, 播放时钟与回放 / 实时查看面板, 供 ADP, erban-caring webapp 与 rosz 查看器共用. 从 ADP 回放 (`frontend/src/replay`, `views`) 抽取, ADP 牵头开发.

- 接缝在包内的前端状态: 适配器 (ADP, webapp, 查看器) 写通道目录与样本, 订阅状态发出的需求窗口; 面板只读状态与播放时钟. 状态, 时钟为纯 TS 对象, React 侧只有 Provider, 钩子与面板
- 首批组件: 视图组装 (可拖拽的布局系统) 与播放控件, 图像面板 (彩色, 视频, 深度伪彩色, 2D 叠加), 序列泳道 (uplot), 事件轨, 3D 场景, 消息树, 通道列表
- 解码在渲染时做: JPEG / PNG 由浏览器解 (支持 WebCodecs `ImageDecoder` 时用它, 否则 `createImageBitmap`), 深度 (`16UC1; zstd`, `16UC1; compressedDepth`) 与 CDR 用 rosz 核心库 WASM (`@erban-robotics/rosz-wasm`), 深度经 WebGL2 R16UI 纹理上 turbo 伪彩色 (固定量程 200-2000 mm, 逐帧归一化会闪烁), 悬停读毫米值. 未显示的面板不解码

## 安装

产物挂在 [GitHub Release](https://github.com/erban-robotics/rosz-ui/releases), 不发公共平台, 依赖写下载地址精确钉版本:

```json
"@erban-robotics/rosz-ui": "https://github.com/erban-robotics/rosz-ui/releases/download/v0.1.0/erban-robotics-rosz-ui-0.1.0.tgz"
```

npm 12 起默认拒装远程 tgz, 项目 `.npmrc` 写 `allow-remote=all`. 本包依赖的 rosz-wasm 也是 rosz Release 上的 tgz, 随之装上.

peer 依赖: `react` `react-dom` 19, `three` 0.185, `@react-three/fiber` 9.

## 接入应用

应用栈为 React + Vite + Tailwind v4.

1. 样式: 在 Tailwind 入口里引入组件样式; 应用没有 ADP 的颜色 token (`surface-*` `foreground-*` `line*` `status-*` `series-*` `primary` 等) 时一并引入默认主题

   ```css
   @import "tailwindcss";
   @import "@erban-robotics/rosz-ui/theme.css"; /* 已有这些 token 的应用 (如 ADP) 不引 */
   @import "@erban-robotics/rosz-ui/rosz-ui.css"; /* 内含 @source, Tailwind 会扫描本包的类名 */
   ```

2. Vite: rosz-wasm 的胶水代码按 `import.meta.url` 找 `.wasm`, 不要让 Vite 预打包它

   ```ts
   optimizeDeps: { exclude: ['@erban-robotics/rosz-wasm'] }
   ```

   vitest 测到本包面板时, 本包以 `?url` 引 `.wasm` (Vite 的写法), 要交给 Vite 转换, 不让 node 原样加载:

   ```ts
   test: { server: { deps: { inline: ['@erban-robotics/rosz-ui'] } } }
   ```

3. 应用骨架挂一个 `<HintProvider>` (面板里的悬停提示须在其内)

### 开发期指向本地源码

ADP 牵头改 UI 时, 用环境变量让 vite alias 指向本机 `~/projects/rosz-ui` 的源码, 边改边看, 改完发版再升依赖. 包内一律相对路径 import, 指向源码即可用:

```ts
// vite.config.ts
import { readFileSync } from 'fs'
const roszUi = process.env.ROSZ_UI_SRC // 如 ~/projects/rosz-ui/src
// 源码目录不在应用仓库下, 源码里的裸import往上找不到应用的node_modules: 本包的依赖 (随发版包装进应用) 一律从应用解析
const roszUiDependencies = roszUi
  ? Object.keys(JSON.parse(readFileSync(`${roszUi}/../package.json`, 'utf8')).dependencies ?? {})
  : []
export default defineConfig({
  resolve: {
    alias: roszUi
      ? [
          { find: /^@erban-robotics\/rosz-ui$/, replacement: `${roszUi}/index.ts` },
          { find: /^@erban-robotics\/rosz-ui\/(.*\.css)$/, replacement: `${roszUi}/$1` },
        ]
      : [],
    // peer依赖须与应用用同一份, 否则React出两份, 钩子失效
    dedupe: ['react', 'react-dom', 'three', '@react-three/fiber', ...roszUiDependencies],
  },
  server: { fs: { allow: roszUi ? ['.', roszUi] : ['.'] } },
})
```

本地源码下的 `rosz-ui.css` 同样带 `@source "./"`, Tailwind 扫的是源码目录. 源码目录不必 `npm install`, 本包依赖由应用里装的发版包带进; 源码新加依赖时应用要先另装上.

## 主题

组件只引用颜色等设计 token (CSS 变量, 经 Tailwind v4 的 `@theme` 成为工具类), 不写死色值; 外观由使用方提供的 token 决定.

- `rosz-ui.css`: 组件样式, 必须引入. 内含 `@source`, 使用方的 Tailwind 会扫描本包的类名
- `theme.css`: 默认主题, 可选. 暗色, 取自 ADP 视觉系统, 定义组件用到的全部 token: 底色与层级 (`surface-*` `background` `gutter`), 文字 (`foreground-*` `muted-foreground`), 线 (`line` `line-strong`), 状态色 (`status-*`), 主色 (`primary*`), 曲线类别色 (`series-1` 到 `series-8`), 圆角, 字体, 小字字号与字重 (`--text-xs` `--text-sm` `--small-text-weight`)

使用方按自身情况三选一:

| 情况 | 做法 | 例 |
|---|---|---|
| 已定义同一套 token | 只引 `rosz-ui.css` | ADP |
| 没有自己的视觉系统 | 引 `theme.css` + `rosz-ui.css` | rosz 查看器 |
| 有自己的配色 | 只引 `rosz-ui.css`, 在自己的样式里把上述 token 映射到本站配色 | erban-caring webapp |

自行映射时注意:

- 曲线类别色 `series-*` 建议照搬默认主题: 这组颜色做过色觉缺陷区分度校验, 自定义时要重新校验
- 小字字重经 `--small-text-weight` 给 12/13px 小字加重一档, 不映射则面板小字比 ADP 略细
- 漏映射的 token 不报错, 对应元素会失去颜色; 换配色后对照 playground 逐个面板看一遍

## 数据源接口

```ts
import { createDataSourceState, ViewComposition, usePlayback, useLivePlayback } from '@erban-robotics/rosz-ui'

const state = createDataSourceState({ mode: 'offline' }) // 或 'live'
const dataSource = { state, zero } // zero: 会话零点 (绝对纳秒 BigInt), 显示时刻换算到它的秒数

// 回放
const playback = usePlayback()
<ViewComposition dataSource={dataSource} playback={playback} scope="data-kind/3" layout={defaultLayout} />
// 实时: 播放头为已收到的最大接收时刻, 无播放控件
const playback = useLivePlayback(dataSource)
```

适配器的职责:

- 通道目录 (`ChannelInfo`, 见 `src/data-source/types.ts`): 从数据源解析后 `setCatalog` (离线一次给全) 或 `upsertChannels` / `removeChannels` (实时话题上下线). 六种通道 `image` `series` `events` `spatial` `overlay` `message`; 离线图像给帧时刻表 `frames`, 离线通道给首末时刻 `range` (时间轴总长与步进按它们算); 实时 transient_local 话题标 `isStatic`, 解不了的类型给 `undecodable` 原因. 频率由状态按样本统计 (`stats`), 适配器不报
- 样本 (`Sample`: 接收时刻, 可选采样时刻, 内容): 离线按段 `write` (段内样本到齐, 此后该段算已覆盖) 或一次写全 `writeStatic` (不淘汰); 实时 `append`. 内容按通道种类: 图像为 `ImagePayload` (压缩字节 + `format`), 结构化消息为 `MessagePayload` (原始 CDR), 序列为 `{ times, columns }`, 事件为 `QualityCheckEvent` / `TaskEvent`, 叠加与空间为各自的一帧 (见 `image-overlay.ts`, `spatial-layers.ts`)
- 视频图像 (`format`为`h264` / `av1`, 即`foxglove_msgs/CompressedVideo.format`的写法): 一条样本一帧, 不重复写同一帧 (同步帧表里一条消息占多个帧号时也只写一条). H.264为Annex B, 每个关键帧前带SPS/PPS, 无B帧; AV1为一个时间单元的OBU序列, 关键帧含Sequence Header. 每条给`isKeyframe`; 离线通道的`frames.keyframeIndices`给关键帧帧号 (升序, 如ADP由回放MCAP文件尾`video_keyframes` Metadata的log_time换算). 图像面板据此把需求窗口起点前移到播放头前最近的关键帧, 适配器照常取窗口内样本加起点前一条, 不必特殊处理. 关键帧到播放头之间须连续覆盖才解, 否则显示加载中; 解码用WebCodecs `VideoDecoder`, 每路一个, 顺序播放增量喂, 跳转与不连续时从关键帧重解
- 空间通道 (3D 场景, 见 `src/scene3d/scene-layers.ts`): `type` 为细分种类 (`transform` `pose` `point_cloud` `keypoints_3d`) 时内容为 `spatial-layers.ts` 的各形状, 坐标系给在目录 `properties.frame` (坐标变换另给 `childFrame`); 也可用两种自带坐标系的通用形状: `TransformsSample` (一条多个坐标变换, 如 ROS 的 TFMessage) 与 `PackedPointCloudSample` (点为 Float32Array, 可带颜色与 `frame`). 实时话题可直接写原始 CDR (`MessagePayload`), `type` 为 ROS 类型 `tf2_msgs/msg/TFMessage` / `sensor_msgs/msg/PointCloud2` (`rosSpatialKind` 判断), 由场景渲染时用 rosz-wasm 解, 与结构化消息同一约定. 内容不带 `time` 时取样本在当前时间轴上的时刻. 大消息 (点云) 在目录标 `keepLatest`, 实时只留最新一条
- 需求窗口: `onDemandChange` 订阅, `demandWindows()` 读 `{ 通道集合, 时间范围, 分辨率提示 }`, 把缺的取来写入 (含窗口起点前最近一条); 取失败 `reportError`. 实时适配器据此决定订阅哪些话题
- 不连续: 实时重连或丢帧时 `markDiscontinuity()`; 回放的跳转, 步进, 循环回绕与切轴由包内记

内存与淘汰由状态统一管: 离线丢掉远离各需求窗口的段, 实时保留播放头前 10 秒 (`keepLatest` 的通道只留最新一条, 频率照算); 每通道至少留最新一条, 静态通道不淘汰.

### 3D 场景

视图组装里的 `scene3d` 视图与单独用的 `Scene3dView` (须在 `DataSourceContext` 内, 如查看器的 3D 面板) 是同一个组件:

- TF 树由目录里全部坐标变换通道合成 (不必绑进视图, 只画绑定的), 每个子坐标系取播放时刻当时或之前最近的一条含它的变换, 不插值. 位姿, 点云, 3D 关键点与末端轨迹按其坐标系经 TF 树换算到基座标系, 不在同一棵树上的不画并标出
- 基座标系: 视图的 `frame` (或 `Scene3dView` 的 `frame`), 没给取人在下拉里选的, 再没有取坐标系最多的那棵树的根 (同样多取先声明的坐标系所在的树); 可为树里任一坐标系
- 取景只在换了图层或基座标系后内容首次非空时做一次, 数据更新不复位视角, 右上角 "重新取景" 手动再取; 包围盒抽样计算 (每图层至多 64 条样本, 每帧至多 2000 点)
- 实时数据源下视图为所画通道与全部坐标变换通道设需求窗口 (适配器据此订阅), 面板藏起时清掉; 离线不设, 由适配器照常写入
- 每条内容只换算一次 (按内容对象缓存), 点云直接用 Float32Array 上屏; 拿不到 WebGL 或渲染出错只在视图内提示

布局描述 (视图表 + 布局树) 的 schema 见 `src/views/catalog.ts`, 与 ADP 原有的兼容: 绑定的 `kind` 由通道种类得出 (叠加与空间按 `type` 细分, 见 `bindingKindOf`), `role` `derivative` `name` 分别对通道的 `role` `source` `name`. 本地布局调整存 localStorage 的 `rosz-ui.local-layouts`.

### 从 ADP 就地版迁移

ADP `frontend/src/replay` 与 `views` 中搬进本包的模块改从本包导入; 留在 ADP 的是适配器一侧: `use-replay-model`, `replay-queries`, `data-source/kernel-adapter`, `views/use-replay-data-source`, `TakeSidebar`, `TakeViewComposition`. 适配器要改的对应关系:

| ADP 就地版 | rosz-ui |
|---|---|
| `ViewComposition` 吃 `model: ReplayModel` | 吃 `dataSource` (状态 + 零点), 加载中与失败经 `isLoading` / `error` 给 |
| 面板按 `model.layers` (内核 `Layer`) 解析绑定 | 按状态的通道目录: 适配器把图层写成 `ChannelInfo` (`id` 即状态通道名, `name` `role` `source` 对应图层的 `name` `role` `derivative_name`, 图层属性放 `properties`) |
| `staticChannel(kind, layerKey)` 等命名约定找通道 | 面板只按目录找, 通道名由适配器定 |
| 序列样本为 `SeriesTrack` (带 `layer`) | 样本为 `{ times, columns }`, 列序与 `side` `unit` `trajectory` `frame` 放通道 `properties` |
| 事件按通道名分质检与任务 | 事件通道 `type` 为 `quality` 或 `task`; 事件的 `layerKey` 改名 `channel` |
| 叠加 / 空间的说明在 `model.overlays` / `model.spatialLayers` | 在通道目录: `type` 为细分种类, `properties` 给 `stream` `width` `height` / `frame` `childFrame`, 无法识别的给 `issue`; 内容 JSON 用 `parseOverlayContent` / `parseSpatialContent` 解析 |
| `model.duration` `stepTimes` 与 `axisTimeline(frameGroups, ...)` | 由目录的 `frames` (同步帧表给 `group` 与 `isSynchronized`) 与 `range` 算 |
| 图像样本 `{ format: 'jpeg', data: Blob }` | 同, `data` 也可为 `Uint8Array`; 支持深度格式与视频 (`h264` `av1`, 带`isKeyframe`) |
| `useUiStore` 的 `localLayouts` | `useLayoutStore` (存储键不同, 本地调整需重做一次) |
| `usePlaybackStore` 在 `ui-store.ts` | 从本包导入; 测试里复位用 `usePlaybackStore.setState(usePlaybackStore.getInitialState())` |
| `createDataSourceState(axis)` | `createDataSourceState({ mode, axis })` |
| `FailureAlert` 认内核 `KernelFailure` | 认错误上的字符串 `code` |

图像 tab 未显示时不设需求窗口也不解码 (原先隐藏的图像 tab 仍预取).

## 开发

```bash
npm install
npm run check        # tsc + eslint + vitest (状态接口的测试)
npm run build        # dist/: ESM + 类型声明 + css
npm run dev          # playground, http://localhost:5173
```

playground (`playground/`, 不进包) 回放 `capture.py` 录下的真实话题: `ROSZ_ROUTER=tcp/<orin>:7447 python playground/capture.py playground/captures/head 6` (需装 rosz wheel 与 zstandard; 录制不入库). 查询参数见 `playground/main.tsx` 开头: `mode=offline` 走离线回放, `copies=3` 把彩色复制成多路测 CPU, `depth=zstd|png` 只放一种深度. 页面上 `await probeDepth()` 解各深度首帧, 与录制时 Python `depth()` 的毫米值对照. 视频: `node playground/encode-video.mjs`用ffmpeg (libx264, libsvtav1) 造H.264与AV1各一路到`playground/captures/video`, 打开`?capture=video&mode=offline`, 页面上`await compareVideo('/video/h264', 帧号)`把画面与ffmpeg解出的同帧PNG逐像素比 (同帧RGB平均差约0.5, 相邻帧6以上); 播放头经`playbackStore.getState().seek(秒)`跳.

## 发版

改 `package.json` 的 `version`, 合进 main 后手动推 `vX.Y.Z` tag, GitHub 托管 runner (`.github/workflows/release.yml`) 检查, 构建, `npm pack` 并把 tgz 挂到 Release.
