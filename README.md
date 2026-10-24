# 都市摩天楼 · 小米快应用（Vela 手表）

Tower Bloxx 的手表版：楼层在塔顶左右摆动，点一下把它放下 —— 叠正了塔就长高一层，叠偏了方块擦着塔边掉下去，掉三次游戏结束。

- 平台：小米快应用 / Vela（`deviceTypeList: ["watch"]`）
- 设计宽度：`192`（方屏），包名 `com.chenxuecong.citybloxx`
- 构建产物：`dist/com.chenxuecong.citybloxx.debug.1.0.0.rpk`（约 101 KB）

## 玩法

| 规则 | 数值 |
|---|---|
| 摆动 | 楼层从塔顶左右摆动：`sx = 48 + sin(phase) × 46`，`phase += 0.035` 每 `16 ms`，一个来回约 **2.9 秒** |
| 判定 | 与中心偏移 `> 24px` 判为叠偏（方块宽 96，即重叠不足 75%） |
| 叠正 | 塔高 +1，并回 1 条命（上限 3 条） |
| 叠偏 | 命 −1，方块播放「擦边掉落」动画，这一层不计入 |
| 游戏结束 | 命掉到 0，0.5 秒后跳结算页 |
| 放下冷却 | 每次成功放下后 `0.5 秒` 内不能再点 |
| 视野滚动 | 塔超过 3 层后整体每层下移 `20px`，高楼不会顶出屏幕 |
| 最高分 | 存 `system.storage` 的 `highscore` 键，破纪录时结算页显示「新纪录！」 |

## 页面

| 路由 | 文件 | 职责 |
|---|---|---|
| `pages/index` | `src/pages/index/index.ux` | 主菜单：快速游戏 / 最高分 / 退出 |
| `pages/tower` | `src/pages/tower/tower.ux` | 游戏本体：摆动、判定、楼层渲染、生命与楼层 HUD |
| `pages/gameover` | `src/pages/gameover/gameover.ux` | 结算：本次层数、是否破纪录，并把 `highscore` 写入存储 |
| `pages/highscore` | `src/pages/highscore/highscore.ux` | 读取并展示最高分 |

结算页的「返回」用 `router.replace('/pages/index')`，避免把已经结束的塔留在页面栈里；其余页面的「返回」用 `router.back()`。

## 快速上手

### 环境要求

- Node.js（`package.json` 声明 `>=8.10`）
- `aiot-toolkit` 2.0.5（`npm install` 自动装）

### 开发与构建

```bash
npm install
npm start      # aiot start --watch，改动热更新
npm run build  # 构建到 dist/
npm run release
```

> ⚠️ `npm run release` 会报 `there is a problem with the certification path`：生产模式不认 node_modules 里自带的调试证书，需要自己从小米快应用平台申请正式证书（放到 `sign/`，已在 `.gitignore` 里排除）。日常构建用 `npm run build` 即可。

## 目录结构

```
src/
├── app.ux                    应用生命周期（空壳）
├── manifest.json             包名、权限、路由、designWidth=192
├── common/
│   ├── logo.png              应用图标
│   └── assets/
│       ├── block.png         楼层方块
│       ├── block-rope.png    正在下落的方块（带绳子感）
│       └── hook.png          塔顶的吊钩
└── pages/
    ├── index/index.ux
    ├── tower/tower.ux
    ├── gameover/gameover.ux
    └── highscore/highscore.ux
```

## 实现要点

- **只用定时器，不用 `requestAnimationFrame`**：摆动由 `setInterval(16ms)` 推进相位，掉落动画用另一个 `setInterval`，都不依赖帧回调，手表上更省心。
- **定时器跟着页面生命周期走**：`_timer`（摆动）在 `gameOver` 和 `onDestroy` 里都会清，`_fallTimer`（掉落动画）在 `onDestroy` 里清 —— 否则玩到一半点「返回」，16ms 的循环会在已销毁的页面上继续空转。
- **死亡瞬间就锁状态**：命归零时先把 `gameState` 置为 `over`、停掉摆动，再等 0.5 秒跳结算页。只靠那 0.5 秒的 `setTimeout` 的话，玩家在这段动画里再点一下会**重复调度一次** `router.push`，叠出第二个结算页。
- **楼层用 `for` 动态渲染**：模板里 `block for="{{fl}}"` 按 `_locs` 逐层画，层数不设上限；每层间距 `20px`（`bottom: 7 + idx*20`），超过 3 层后整座塔再按 `(count-3)*20` 整体下移（`bottomShift`），实现视野滚动。
- **单位都是设计单位**：`designWidth: 192`，`px` 由框架按屏宽缩放；布局常量 `CX = (192-96)/2 = 48`、`HX = (192-28)/2 = 82`。
- **本地存档只有一个键**：`highscore`，读写都走 `system.storage`，读写失败时降级为 0 / 直接写入本次成绩。

## 体积

当前 rpk **101.3 KB**（103,705 字节），其中 `logo.png` 一张图就占 73 KB。

早期版本是 **1,323 KB**，因为 `src/common/assets/` 里躺着 **1,283.3 KB 零引用资源**：`bgm.ogg`+`bgm.mp3`（1,138.5 KB）、`game-over` / `rotate` / `drop` / `drop-perfect` 各 mp3+ogg 双份、以及 `main-index-store.png`、`main-index-start.png`。代码里从来没有 import 过 `@system.audio`，`manifest.json` 也没声明音频能力，这些音效从来没接上过。删掉后包体降了 92%。

## 已知限制

- **没有音效与背景音乐**：原素材已在清理中删除；要加回来需要引入 `@system.audio`、在 `manifest.json` 声明 `system.audio`，并把音频文件放回 `assets/`。
- **只按 192 设计宽调过**：方屏之外的圆屏/长条屏没有专门适配。
- **没有暂停**：游戏中途只能点「返回」直接退出，塔的进度不保留。
- **`npm run lint` 没有**：仓库没有 ESLint 配置，且 `.ux` 需要自定义 parser/processor 才能解析，所以没加这个脚本。

## 许可证

尚未指定开源许可证。要放开复用建议 MIT 或 Apache-2.0 —— 确定后补一个 `LICENSE` 文件即可。
