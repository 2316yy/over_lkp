# 向卡米尔许愿 · 集曜 2.5

玩法照搬 `game2.5`《向克苏鲁许愿》，角色资产换成 **卡米尔微调版**。

## 运行

```bash
cd wish_camille
npm run dev
# → http://127.0.0.1:7260/
```

不装 npm 依赖也可以直接跑，`node_modules/three` 已经随工程带上：

```bash
cd wish_camille
node server.js
```

## 资产说明

- `assets/kamier.glb`
  - 由 `/Volumes/outer_unlimited/kimi/threegame/overcook/assets/卡米尔微调版/卡米尔微调版.pmx` 烘焙出的 `kamier2.glb` 复制而来（对应 `rebake_kamier2.log`）。
  - 不是 `卡米尔` 原版：原版在同机位测试下眼睛可见率约 11%，大量被帽檐遮挡；微调版约 28%，达到“能看见眼睛”的效果。
  - GLB 自带 17 个动画，游戏内循环播放 `idle`。
- `assets/cubes/*.glb`：七曜积木与星宝石，沿用原游戏。

## 本次迁移改动

- 标题、引导、求签提示、签卡标签、图鉴文案里的克苏鲁/深渊指向改为卡米尔/星海主题。
- 签文库同步为 64 签新版：每签带 2~3 个心情标签，共 12 种心情（阴翳六 + 微光六）；求签时按当前心情筛选签池，签卡 / 签谱 / 曜录都会显示共鸣标签。
- 签级判词里的旧深渊文案已改为卡米尔 / 群星 / 星轨主题。
- 模型加载路径改为 `assets/kamier.glb`，并接入 `AnimationMixer` 播放 idle。
- `localStorage` 命名空间改为 `camil_*`，与源游戏 `cth_*` 隔离。
- 默认端口改为 `7260`，可与源游戏同时运行。
- 修复首访引导：在模型加载完成前点“进入星台”不会又被重新弹出。
