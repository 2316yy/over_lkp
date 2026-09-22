# 向卡米尔许愿 · 项目交接文档（HANDOFF）

> 给新对话 / 新协作者看的上下文文档。
> 建议新对话第一句直接发：
>
> `项目在 /Volumes/outer_unlimited/vibetool/for_camil/wish_camille。先读 HANDOFF.md，再继续开发。`

---

## 一、项目一句话

基于源游戏《向克苏鲁许愿》（`game2.5`）的求签玩法，替换角色资产为**卡米尔微调版**、把全部 64 签换成新版签文（12 种心情标签）后得到的《向卡米尔许愿 · 集曜 2.5》。

- 本地目录：`/Volumes/outer_unlimited/vibetool/for_camil/wish_camille`
- Pages：**https://2316yy.github.io/over_lkp/**
- 本地端口：**7260**
- 技术形态：纯静态 HTML/CSS/JS + three.js（无构建步骤，直接由 `server.js` 静态服务）

---

## 二、运行与测试

```bash
cd /Volumes/outer_unlimited/vibetool/for_camil/wish_camille
/opt/homebrew/bin/node server.js
# 打开 http://127.0.0.1:7260/
```

- Node 在 `/opt/homebrew/bin/node`（当前 shell 的 PATH 里可能没有，注意用绝对路径或先 export）。
- `node_modules/three` 已随工程提交，clone 后可以直接跑，不需要 npm install。
- 浏览器自动化：本机有 Playwright 包 `/Volumes/outer_unlimited/cargame/node_modules/playwright-core`，浏览器缓存在 `~/Library/Caches/ms-playwright`。
- 本地调试可用的全局钩子：
  - `window.__ready`
  - `window.__ritual`（3D 仪式：`awaken / energy / flash / aside / calm / toGarden`）
  - `window.__controls`（OrbitControls）
  - `window.__cubes`（积木台）
  - `window.__dex`（图鉴数据层）
  - `window.__device`（移动端 / 触摸探测）

---

## 三、目录与文件职责

```text
wish_camille/
├── index.html      页面结构 + 全部 CSS + importmap
├── main.js         3D 场景：祭坛、粒子、相机、卡米尔模型加载、idle 动画
├── game.js         占卜流程：心情系统、摇签、抽签池、结果卡、复制文案
├── lots.js         64 签数据（自动生成后同步过来的，含 moods 心情标签）
├── dex.js          集曜 / 签谱 / 曜录：localStorage + 浮层 UI
├── cubes.js        3D 积木：拖拽、吸附、堆叠、重力、点击翻阅
├── audio.js        Web Audio：摇签、钟磬、揭签、宝石音效
├── server.js       极简静态服务器（默认 7260）
├── package.json    只有 three 依赖；npm run dev = node server.js
├── assets/
│   ├── kamier.glb  卡米尔微调版角色（6.07 MB，17 个动画）
│   └── cubes/      七曜积木 + 星宝石（8 个 GLB）
├── README.md       面向使用者的简要说明
└── HANDOFF.md      本文件
```

---

## 四、关键背景：角色模型

- 最终使用：`assets/kamier.glb`
- 来源链路：
  - 原始微调模型：`/Volumes/outer_unlimited/kimi/threegame/overcook/assets/卡米尔微调版/卡米尔微调版.pmx`
  - 已烘焙 GLB：`/Volumes/outer_unlimited/kimi/threegame/overcook/public/models/kamier2.glb`
  - 判断依据：`overcook/assets/rebake_kamier2.log` 明确写的是从 `assets/卡米尔微调版/卡米尔微调版.pmx` 导入烘焙
- 不要用 `kamier.glb` 原版：同机位、逐眼顶点射线检测下：
  - 原版眼睛可见率约 **11%**，大面积被帽檐遮挡
  - 微调版约 **28%**，能正常看到眼睛
- GLB 含 17 个动画：`idle / walk / run / carry / walkcarry / chop / wave / hit / death / pose-*`
- `main.js` 的 `placeModel(model, animations)` 里创建 `AnimationMixer`，优先播 `idle`；没有 idle 时才退到 `pose-*`。
- 换模型时注意：
  - 保持文件名 `assets/kamier.glb`，或同步修改 `index.html` preload 和 `main.js` 加载路径。
  - 优先保留名为 `idle` 的动画；否则要改 `main.js` 里的选 clip 逻辑。
  - 模型面朝 +Z：眼睛 z 坐标大于头部中心 z 才说明正脸朝向相机（原烘焙约定如此）。

---

## 五、关键背景：64 签与 12 种心情

### 1. 数据形态

`lots.js` 中每签：

```js
{ n: 1, name: '乾为天', grade: '上上', moods: ['lost', 'courage', 'hope'],
  poem: ['星汉西流夜未央，', ...四句] }
```

- 每签 **2~3 个 moods**（新版规则）
- `VERDICTS`：六档签级判词
- `SAN_RANGE`：签级 → 理智消耗数值

### 2. 12 种心情

阴翳六：`lost 迷茫 / tired 疲惫 / lonely 孤独 / fear 恐惧 / angry 愤怒 / hollow 空洞`

微光六：`joy 欢喜 / calm 平和 / hope 期待 / gratitude 感念 / courage 勇气 / relief 释然`

> 旧版只有 `grace` 一个感念 key；新版改为 `gratitude`，并新增 `courage / relief`。
> `dex.js` 里保留了旧档兼容：`grace → gratitude`。

### 3. 抽签池逻辑

`game.js`：

```js
function pickLotForMood(moodKey) {
  const fit = LOTS.filter((l) => Array.isArray(l.moods) && l.moods.indexOf(moodKey) >= 0);
  const base = fit.length ? fit : LOTS;
  const pool = (currentLot && base.length > 1)
    ? base.filter((l) => l.n !== currentLot.n)
    : base;
  return pick(pool.length ? pool : base);
}
```

- 你选哪个心情，就只从带该标签的签里抽。
- 同心情池里尽量不连续抽到同一支签。
- 结果卡、签谱详情、曜录回看都会显示「签 心 相 应」标签，当前心情高亮。
- 相关样式在 `index.html` 的 `.lot-tags / .lt-cap / .ltag`。

---

## 六、签文更新流程（重要）

当前仓库里的 `lots.js` 是**同步自源工程生成结果**，项目里没有放构建工具。以后大改签文，推荐：

1. 编辑源工程文案：
   - 人类可读稿（推荐）：`/Volumes/outer_unlimited/vibetool/for_xiaoke/game2.5/LOTS_COPY.md`
   - 另一份新稿：`/Volumes/outer_unlimited/vibetool/for_xiaoke/game2.5/LOST_COPY2.md`
   - 注意：构建脚本实际读取的是 `LOTS_COPY.md`。
2. 在源工程里生成：
   ```bash
   cd /Volumes/outer_unlimited/vibetool/for_xiaoke/game2.5
   sh build-lots.sh
   ```
   校验：`sh build-lots.sh --check`
3. 把生成的 `game2.5/lots.js` 同步到本项目：
   - 复制后必须把标题 `向克苏鲁许愿 · 六十四签` 改成 `向卡米尔许愿 · 六十四签`。
   - 检查 `VERDICTS` 里有没有残留的 `深渊 / 旧日 / 克苏鲁`，本项目已改为卡米尔 / 群星 / 星轨主题。
4. 如果以后频繁改文案，建议把 `LOTS_COPY.md` 和 `build-lots.sh + tools/lots_build.cjs` 一起纳入本仓库，避免手工同步。

### 特别提醒

`LOST_COPY2.md` 与 `LOTS_COPY.md` 内容基本一致，但前者个别地方缺少「签诗：」标题行；优先以构建脚本读取的 `LOTS_COPY.md` 为准。

---

## 七、本项目的卡米尔化改动

- 界面文案：克苏鲁 / 深渊 / 神像 → 卡米尔 / 星台 / 星海主题
- `index.html`：标题、preload、按钮、引导文案、星台相关文字
- `game.js`：心情语料里的克苏鲁句子已改；结果卡、复制分享文案、曜方提示已改
- `dex.js`：图鉴文案已改
- `lots.js`：签级判词里的深渊 / 旧日残留已适配
- 存储命名空间：
  - `camil_dex_v1`
  - `camil_coach`
  - `camil_muted`
- 首访引导修复：在模型加载完成前关闭，不会被轮询重新弹出

---

## 八、Git / 部署状态

- 远程：`https://github.com/2316yy/over_lkp.git`
- 分支：`main`
- 关键提交：
  - `d4bb4dd Initial commit: wish-to-camille game2.5`
  - `d621550 feat: new 64-lot corpus with 12 mood tags`
- GitHub Pages：从 `main` 分支根目录发布（legacy Pages），必须有 `.nojekyll`
- 更新发布流程：
  ```bash
  cd /Volumes/outer_unlimited/vibetool/for_camil/wish_camille
  git add -A
  git commit -m "feat: ..."
  git push origin main
  ```
  push 后 Pages 会自动重建，一般 1~3 分钟。
- macOS 已配置 `credential.helper=osxkeychain`，GitHub Token 存在本机钥匙串，**文档里不要记录 Token 明文**。

### 仓库可见性提醒

之前 API 查询 `2316yy/over_lkp` 显示 `"private": false`，但 Pages 链接可以公开访问。GitHub 免费版的私有仓库通常不能再发布 Pages；如果之后一定要“私有仓库 + 公开测试链接”，需要升级 GitHub Pro 或改用 Cloudflare Pages / Netlify 等外部托管。

---

## 九、已知注意事项 / 后续可做

- **版权**：卡米尔模型来自《凹凸世界》MMD 模型，使用规约禁止商用；现在 Pages 链接是公开的，若要长期公开分享，请先确认模型授权。
- **Token 安全**：GitHub Token 曾在对话里明文出现过；如果之后换新 Token，重新 push 一次让钥匙串自动更新即可。
- **移动端**：12 个心情按钮比旧版更多，若在小屏上显得拥挤，可改 `index.html` 里 `#stage-home #moods` 的布局/滚动。
- **签文源文件**：当前项目能跑，但若要大幅改签文，建议按第六节把源稿和构建脚本带进本仓库。
- **测试建议**：改完用本地 `7260` + Playwright 跑一遍：心情按钮数量 12、`pickLotForMood` 抽签、结果卡标签、签谱 / 曜录回看、`camil_*` 存档。

---

## 十、给新对话的推荐开场白

```text
项目在 /Volumes/outer_unlimited/vibetool/for_camil/wish_camille。
请先读 README.md 和 HANDOFF.md，了解项目结构和当前状态。
这是一个纯静态 three.js 项目《向卡米尔许愿》，跑在 7260 端口，远程仓库 2316yy/over_lkp，Pages 是 https://2316yy.github.io/over_lkp/。
本次我想开发：<在这里写你的需求>
```
