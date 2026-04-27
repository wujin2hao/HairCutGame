# 技术实现文档 · 头发与剪刀

> 配套实现：
> - [`Scene/hair-strand-demo-v5.html`](../Scene/hair-strand-demo-v5.html) — Step 1 剪刀手感
> - [`Scene/hair-strand-demo-v6.html`](../Scene/hair-strand-demo-v6.html) — Step 2 头发切割
> - [`Scene/hair-strand-demo-v7.html`](../Scene/hair-strand-demo-v7.html) — Step 4+5 状态机 + NPC 比划 + 设定虚线
>
> 关联设计：[`Docs/GDD-剪刘海.md`](GDD-剪刘海.md)

本文档说明 demo 中的核心技术点：**头发几何与渲染**、**剪刀图像旋转与开合动画**、**剪刀物理积分**、**头发切割（v6）**、**状态机与游戏流程（v7）**。评分等后续阶段不展开。

---

## 一、坐标系与画布

```
画布：600 × 520 px
  ┌─────────────────────────┐
  │                         │  y=0
  │      头顶 CY0=155       │
  │       ↓                 │
  │   ◯ 头部圆 R=116        │
  │       ↓                 │
  │   ─── CYB=271（弧终点） │
  │       ↓ EXTRA=62        │
  │   ─── CYBT=333（发梢）  │
  │                         │
  │   脸 / 脖子 / 肩         │
  └─────────────────────────┘
```

| 常量 | 值 | 含义 |
|------|----|------|
| `CX` | 300 | 头部水平中心 |
| `CY0` | 155 | 头顶（发根所在） |
| `R` | 116 | 头形半径 |
| `CYB` | `CY0 + R = 271` | 圆弧终止 Y（发丝沿头形结束的位置） |
| `EXTRA` | 62 | 圆弧之后的直线下垂段长度 |
| `CYBT` | `CYB + EXTRA = 333` | 发梢最终 Y |
| `N` | 58 | 发丝总根数 |

> **关键设计**：每根发丝由两段构成 —— **沿头形包裹的弧段** + **顺势下垂的直线段**。这是后续做长度评估的基础（直线段是被剪的对象）。

---

## 二、发丝几何：弧段的三次 Bezier 近似

### 2.1 输入参数
`makeStrand(pos, jitter)` 接收一个归一化的横向位置 `pos ∈ [-0.5, +0.5]`，左半为负、右半为正、中分为 0。

### 2.2 角度映射
```javascript
theta = sign(pos) · arcsin( min(0.9999, |pos| · 2) )
```
- `pos = 0`  → `theta = 0`（正中分发，垂直向下）
- `pos = ±0.5` → `theta = ±π/2`（贴头侧到最宽处）
- `arcsin` 的非线性映射让发丝在头顶密集、两侧稀疏，符合真实头发分布。

### 2.3 三次 Bezier 控制点（弧段）
弧的两个端点：
- 起点 `p0 = (CX, CY0)`（头顶）
- 终点 `arcEnd = (CX + sin(theta)·R, CYB)`（沿头形圆弧）

控制点距离公式（圆弧的标准三次 Bezier 近似）：
```
cpLen = (4/3) · R · tan(theta / 4)
```
这是计算机图形学中将圆弧近似为三次 Bezier 的经典公式，**当 |theta| < π/2 时误差 < 0.03%**，肉眼无法察觉。

控制点几何：
- `p1 = p0 + (sin(theta), cos(theta)) · cpLen` —— 沿发根切线方向（向下偏向 theta 一侧）
- `p2 = arcEnd - (0, cpLen)` —— 沿弧终点的切线方向（向上）

### 2.4 直线下垂段
```
fullEnd = (arcEnd.x, CYBT)
```
弧结束后直接垂直向下到 `CYBT`。这一段是后续切割逻辑要操作的"可剪区域"。

### 2.5 抖动（jitter）
为打破完美对称，每根发丝在 `p0`、`arcEnd`、`fullEnd` 处加入小幅随机偏移（±2~3px）。在生成头发底色时关闭 jitter 以保证轮廓规整。

---

## 三、头发渲染管线

绘制顺序至关重要，决定层次感。每帧按如下顺序：

### 3.1 头部底层（drawHead）
椭圆脸 + 矩形脖颈 + 椭圆耳朵 + 五官。固定形状，不参与动态。

### 3.2 头发底色（drawBase）

**v5**：取 `pos = -0.5` 与 `pos = +0.5` 两根**无抖动**的"边界发丝"作为左右轮廓，加 `CYBT` 横线封底，构成块状区域填充暗紫渐变：
```
0%   #3e2153
35%  #200f30
100% #09060f
```
这层是"头发体积"的视觉基础，避免发丝缝隙露出脸部底色。

**v6 改动（路线 A）**：`drawBase()` 函数保留但**不再调用**。原因：底色块的下边界永远在 `CYBT`，发丝被切短后底色仍在，肉眼看不到切割效果。改为**纯发丝渲染**，靠下面 §3.6 的边缘填充发丝补齐侧边密度。

### 3.3 单根发丝（drawStrand）
每根发丝绘制三件事：
1. **路径**：从 `p0` → 通过 `bezierCurveTo(p1, p2, arcEnd)` → `lineTo(fullEnd)`
2. **颜色**：每根独立的 HSL 渐变（hue 抖动 ±8°，亮度顶端 ~28% / 末梢 ~6%）
3. **描边**：`lineWidth = 5~7.5`、`lineCap = 'round'`

### 3.4 排序技巧（z-order）
```javascript
strands.sort((a, b) => abs(b.theta) - abs(a.theta))
```
**绝对角度大的先画**（即两侧的发丝在底层），中分的发丝在最上层。这样头发中心区域不会被两侧的发丝压住，看起来更自然。

### 3.5 裁剪
绘制头发前 `ctx.rect(0, 0, W, CYBT + 4); ctx.clip()`，避免发丝因 jitter 溢出到肩膀以下。

### 3.6 边缘填充发丝（v6 新增）

去掉 `drawBase` 后，主体 N=58 根发丝按 `pos = (i/(N-1)) - 0.5` 均匀排布，会在两侧出现**曲率断层**导致的视觉缝隙。

**断层成因**：`theta = arcsin(2·|pos|)` 这个映射让 X 在 pos 上线性（发梢横向均匀），但 theta 在 pos 上非线性 —— `d(theta)/d(pos)` 在 |pos|→0.5 时趋向无穷。具体表现：

| pos | theta | cpLen | arcEnd.x |
|-----|-------|-------|----------|
| 0.5    | 90°   | ≈64 | 416 |
| 0.4825 | 75°   | ≈50 | 412 |
| 0.4649 | 68.4° | ≈45 | 408 |

主体最外两根（i=0 / i=57）X 只差 4px，但 cpLen 从 64 掉到 50，**最外那根弯得贴轮廓，紧邻的内侧那根曲率明显变浅**，看起来"突然变直"。

**解法**：在 `theta = 78° / 82° / 86° / 90°` 各补一根**无 jitter** 发丝（每侧 4 根，总 8 根），bridge 主体最外的 75° 到轮廓的 90°，每档曲率差 ~3.5°，肉眼平滑。反推 pos：

```javascript
const EDGE_THETAS_DEG = [78, 82, 86, 90];
EDGE_THETAS_DEG.forEach(deg => {
  const theta = deg * Math.PI / 180;
  [-1, +1].forEach(sign => {
    const pos = sign * Math.sin(theta) / 2;   // pos = sin(theta)/2
    const s = makeStrand(pos, false);          // 无 jitter
    strands.push(_buildStrandState(s, { w: 6.5, hue: 265, ltop: 28, lbot: 6 }));
  });
});
```

边缘发丝宽度 6.5px（略粗），固定 hue 265 不抖动，z-order 靠 `sort by |theta|` 自动落在最底层（|theta|=π/2 最大），主体发丝盖在上面，看起来像主体的延伸而非新元素。

---

## 四、剪刀渲染：旋转 + Clip 开合

### 4.1 资源
- 图源：`Assets/jiandao.png`，130×173 RGBA
- 原图朝向：握把在顶（rows 0~60），刀尖在底（rows 110~173）

### 4.2 旋转策略
Demo 中剪刀从右向左推进，刀尖必须朝左。对原图执行 `ctx.rotate(+π/2)`（顺时针 90°）：
- 原图顶（握把）→ 屏幕右侧
- 原图底（刀尖）→ 屏幕左侧 ✓

旋转后的显示尺寸：
```
SC_LEN   = 110         // 旋转后的水平长度（握把→刀尖）
SC_THICK = 130/173 · SC_LEN ≈ 83  // 旋转后的垂直厚度（保持原图比例）
```

### 4.3 旋转坐标系约定
执行 `ctx.rotate(+π/2)` 后，旋转坐标系 `(rx, ry)` 与世界坐标 `(X, Y)` 的关系：

| 旋转系轴 | 世界方向 |
|---------|---------|
| `+rx`   | 世界 +Y（向下） |
| `-rx`   | 世界 -Y（向上） |
| `+ry`   | 世界 -X（向左） |
| `-ry`   | 世界 +X（向右） |

绘制时：
- `drawImage` 的宽度参数 `DW = SC_THICK` → rx 方向（世界上下）
- `drawImage` 的高度参数 `DH = SC_LEN` → ry 方向（世界左右）

### 4.4 开合动画 —— 双 Clip 偏移法
合拢与张开走两条不同的渲染路径。

**合拢（openAmt < 0.5%）**：
```javascript
ctx.drawImage(scissorImg, -DW/2, -DH/2, DW, DH);
```
直接整张绘制。

**张开**：将图片在旋转系内**沿 rx 中线一分为二**，左半（rx<0）当作上叶片，右半（rx>0）当作下叶片，向各自外侧偏移 `openPx = openAmt · MAX_OPEN_PX`（`MAX_OPEN_PX = 14`）。

```javascript
// 上叶片：clip 限定在左半区，整图向 -rx 方向位移
ctx.save();
ctx.beginPath();
ctx.rect(-DW/2 - 2, -DH/2 - 4, DW/2 + 4, DH + 8);  // 左半 clip 框
ctx.clip();
ctx.drawImage(scissorImg, -DW/2 - openPx, -DH/2, DW, DH);
ctx.restore();

// 下叶片：clip 限定在右半区，整图向 +rx 方向位移
ctx.save();
ctx.beginPath();
ctx.rect(-2, -DH/2 - 4, DW/2 + 4, DH + 8);  // 右半 clip 框
ctx.clip();
ctx.drawImage(scissorImg, -DW/2 + openPx, -DH/2, DW, DH);
ctx.restore();
```

**为什么 clip 框比 DW/2 多 2~4 像素**：避免抗锯齿在分界线产生缝隙。

**世界方向核对**：上叶片在旋转系向 `-rx` 偏移 → 世界 `-Y` → 屏幕向上 ✓；下叶片向 `+rx` → 世界 `+Y` → 向下 ✓。

### 4.5 开合插值
按键时 `sc.open` 在 `false ↔ true` 间切换；每帧 `openAmt` 向目标值线性逼近：
```javascript
sc.openAmt += (target - sc.openAmt) · min(1, dt · 20)
```
速率 `20/s` 让一次开合在约 50ms 内完成，比击键反馈节奏快一档。

### 4.6 向量降级
图片加载失败时调用 `_drawFallback`，用两个梯形 + 椭圆握把 + 中心螺丝绘制。逻辑只在旋转系内做 `±ang` 旋转（最大 0.30 rad ≈ 17°），保证图片缺失也能继续测试物理手感。

---

## 五、物理积分

### 五个力 / 项，按固定顺序施加
每帧 `updatePhysics(dt)`：

```javascript
// 1. 重力（每帧累加）
sc.vy += CONFIG.gravity * dt;            // gravity = 1100

// 2. 抬升推力（按住空格时）
if (keys.space) sc.vy -= CONFIG.thrustForce * dt;  // thrust = 2400

// 3. 线性阻尼（指数衰减）
sc.vy *= Math.exp(-CONFIG.linearDrag * dt);        // drag = 2.0

// 4. 水平推进（恒定向左）
sc.x -= CONFIG.autoAdvanceSpeed * dt;              // 75 px/s

// 5. 位置积分
sc.y += sc.vy * dt;
```

### 5.1 为什么阻尼用 `exp(-k·dt)` 而不是 `vy *= 0.98`
- **帧率无关**：`exp(-k·dt)` 在不同 dt 下衰减一致，hi-fps 与 lo-fps 手感相同。
- **物理直观**：等效于连续时间下 $\dot v = -kv$ 的精确解。
- **参数语义清晰**：`k = 2.0` 即"约 0.35s 速度减半"（`ln 2 / 2 ≈ 0.347`）。

### 5.2 重力 / 推力 / 阻尼的设计配比
- `thrustForce / gravity = 2400 / 1100 ≈ 2.18`：按住时净加速度向上 1300 px/s²。
- 阻尼系数 2.0：单次 100ms 点按产生的 vy 增量在约 0.35s 内衰减一半 → 玩家无法靠"长按"飞起，只能"啵啵啵"维持悬停。
- **这一组比例是手感的核心**，调整任意一个都会显著改变难度。GDD 第九章给出的起手值与本 demo 实测略有差异，以 demo 为准。

### 5.3 边界处理
```javascript
if (sc.y < CONFIG.minY) { sc.y = minY; sc.vy = max(0, sc.vy); }
if (sc.y > CONFIG.maxY) { sc.y = maxY; sc.vy = min(0, sc.vy); }
```
**关键**：撞墙时**只清掉与墙同向的速度分量**，保留反向速度。这避免了"贴墙后无法离开"的 bug。

### 5.4 飞出左侧自动重置
```javascript
if (sc.x < -SC_LEN) resetScissor();
```
demo 阶段没有切割结束逻辑，剪刀飞出画面后回到右侧重新开始，方便反复测试手感。

### 5.5 dt 上限
```javascript
const dt = Math.min((ts - lastTime) / 1000, 0.05);
```
单帧最长 50ms。窗口失焦再切回时，dt 会很大，不限会导致剪刀瞬间穿地或飞出画面。

---

## 六、输入

| 输入 | 事件 | v5 行为 | v6 行为 |
|------|------|---------|---------|
| Space 按下 | `keydown` | `keys.space = true` | 同 v5 |
| Space 抬起 | `keyup`   | `keys.space = false` | 同 v5 |
| J 按下     | `keydown` | toggle `sc.open` | toggle `sc.open` **+ 调用 `performCut()`** |
| 鼠标左键   | `mousedown` | toggle `sc.open` | toggle `sc.open` **+ 调用 `performCut()`**（debug） |

> v6 的 J `keydown` 监听加了 `!e.repeat` 守卫，长按只触发一次切割，避免连发。

**v7 改动**：所有事件先到全局 dispatcher，再 route 到 `states[currentState].handleInput(e)`。AIMING 之外的状态会忽略 Space/J/canvas 鼠标点击。详见 §8.5。

---

## 七、头发切割（v6 新增）

v6 在 v5 物理基础上加入"剪刀闭合瞬间切断刀刃覆盖范围内发丝"的机制。无切割动画，瞬切。

### 7.1 刀刃范围

每根发丝被命中的判定基于刀刃区间，取自剪刀的几何：

| 量 | 取值 | 含义 |
|---|---|---|
| `bladeY`  | `sc.y` | 剪刀中心 Y（旋转后刀刃恰在中线） |
| `bladeX0` | `sc.x - SC_LEN/2` | 刀尖（剪刀左端） |
| `bladeX1` | `sc.x` | 中心螺丝（左半段为刀刃，右半为握把不切） |

### 7.2 数据结构改动（`makeStrand` + `initStrands`）

每根发丝在生成时**预采样**弧段成 21 个点（`t = 0/20 ~ 20/20`），运行时切割只查表，不再算 Bezier。

```javascript
// makeStrand 返回值新增字段：
samples: [{ x, y, t }; 21]   // 弧段的均匀 t 采样

// initStrands 每根 strand 新增切割状态：
endY:     fullEnd.y,   // 直线段切：发梢 Y 缩到这里
cutInArc: false,
cutP1:    null,        // 弧段切后新 bezier 的 cp1
cutP2:    null,        // 弧段切后新 bezier 的 cp2
cutEnd:   null,        // 弧段切后曲线上的实际切点
```

> 保留原始 `p0/p1/p2/arcEnd/fullEnd` 不动，"切"只在派生状态上发生，便于 reset 复原与潜在的二次切割比较。

### 7.3 切割判定（`performCut`）

对每根发丝按以下顺序判：

1. **X 范围**：`arcEnd.x` 不在 `[bladeX0, bladeX1]` → 跳过
2. **已比刀刃高**：当前发梢 Y（`cutInArc ? cutEnd.y : endY`）≤ `bladeY` → 跳过
3. **直线段切**（`bladeY ≥ arcEnd.y`）：直接 `endY = bladeY`
4. **弧段切**（`bladeY < arcEnd.y`）：走 §7.4

### 7.4 弧段切：20 采样找 t + De Casteljau 子分

> 用户约束：用数值采样近似找参数，不解析求交；但渲染必须保持 bezier 平滑。

**Step 1 · 采样找参数 t**

扫 `samples` 找首个跨过 `bladeY` 的相邻对 `(a, b)`，线性插值：

```javascript
u    = (bladeY - a.y) / (b.y - a.y)
tCut = a.t + (b.t - a.t) · u
```

**Step 2 · De Casteljau 在 tCut 处子分**

对原四点 `(p0, p1, p2, arcEnd)` 做一次 De Casteljau，截出"左半段"的新三次 Bezier：

```javascript
Q0 = lerp(p0,    p1,     t)
Q1 = lerp(p1,    p2,     t)
Q2 = lerp(p2,    arcEnd, t)
R0 = lerp(Q0,    Q1,     t)
R1 = lerp(Q1,    Q2,     t)
S  = lerp(R0,    R1,     t)   ← 曲线上的实际切点

cutP1 = Q0,  cutP2 = R0,  cutEnd = S
```

新的 `(p0, Q0, R0, S)` 仍是合法的三次 Bezier，绘制时保持平滑。

### 7.5 渲染分支（`drawStrand`）

```javascript
ctx.moveTo(p0);
if (cutInArc) {
  // 截短 bezier，无直线段
  bezierCurveTo(cutP1, cutP2, cutEnd);
} else {
  // 完整 bezier + 缩短的直线段
  bezierCurveTo(p1, p2, arcEnd);
  lineTo(arcEnd.x, endY);
}
```

渐变终点用实际终点（`cutEnd` 或 `(arcEnd.x, endY)`），保证根→梢明暗过渡正确。

### 7.6 重复切割（再切更短）

每次 `performCut` 已用 "新发梢 Y 必须更小" 作前置判断；进入弧段分支后，若已 `cutInArc`，再判 `bladeY < cutEnd.y` 才生效。注意：弧段切后无直线段，所以"已 cutInArc → 直线段重新切"这条路径不可能发生（bladeY 只会更小）。

### 7.7 调试可视化

调试模式（`btnDbg`）下额外显示：

- 每根发丝的 21 个采样点（蓝色 1.6px 圆点）
- 已发生弧段切的发丝在 `cutEnd` 处显示红色高亮
- 剪刀刀刃区间在 `(bladeX0, bladeY) ~ (bladeX1, bladeY)` 显示绿色虚线
- HUD 右上角添加 `cut = 已切数 / N`

---

## 八、状态机与游戏流程（v7 新增）

v7 引入显式状态机，把 v5/v6 的"剪刀世界"包成 `AIMING` 状态，前面接两个新状态：

```
GESTURING（NPC 比划，玩家观看）
   ↓ 自动衔接（手退场 + 0.5s 暂停）
SETTING_GUIDE（玩家拖动虚线）
   ↓ 点击「确认」按钮
AIMING（玩家操作剪刀切割，沿用 v6）
```

### 8.1 状态机骨架

每个状态是一个对象，含可选的 `enter / update / exit / handleInput`：

```javascript
const STATE = { GESTURING, SETTING_GUIDE, AIMING };
const states = { [STATE.GESTURING]: stateGesturing, ... };

function transitionTo(next) {
  if (currentState && states[currentState].exit) states[currentState].exit();
  currentState = next;
  if (states[next].enter) states[next].enter();
}
```

主循环只调当前状态的 `update(dt)`，事件分发只调 `handleInput(e)`，这样物理积分 / 输入响应自动按状态隔离。

### 8.2 GESTURING：NPC 比划

**生成阶段**（`buildGestureSequence`，在 `enter` 时调用一次）：

| 段 | 数量 | 停留时长 | 标记 |
|----|------|----------|------|
| 干扰停留 | 1~2 个 | 0.3~0.8s | `isReal=false` |
| **真目标** | 1 个 | **1.0~1.5s** | `isReal=true` |
| 干扰停留 | 0~1 个 | 0.3~0.8s | `isReal=false` |

每段以 `{ fromY, toY, holdDuration, isReal }` 记录，所有 Y 都从 `[HAIR_TARGET_Y_MIN, HAIR_TARGET_Y_MAX] = [240, 310]` 范围随机抽。生成时把 `realY` 写入 `game.targetY`（Step 6 评分会用）。

**运行阶段**（`update(dt)`）— 状态机内的子状态机：

| `gesture.phase` | 行为 |
|-----------------|------|
| `transition` | `handY = lerp(fromY, toY, easeInOut(t))`，`t = phaseTime / HAND_TRANSITION_TIME (0.4s)` |
| `hold`       | 保持 `toY`；若 `isReal`，叠加 `sin(2π·NOD_FREQ·t) · NOD_AMPLITUDE` 做点头 |
| `exiting`    | 序列走完，从最后位置回退到 `H+80`（屏幕外） |
| `paused`     | `HAND_PAUSE_AFTER (0.5s)` 后自动 `transitionTo(SETTING_GUIDE)` |

手图：`Assets/finger.png`，按 `HAND_BASE_X = 165` 居中绘制（脸的左侧，方便手调）。

### 8.3 SETTING_GUIDE：拖动虚线 + 确认按钮

**虚线状态**：
```javascript
const guide = { y: GUIDE_DEFAULT_Y, dragging: false };
```

**拖动判定**（`mousedown`）：
- 命中按钮（`CONFIRM_BTN` 矩形）→ `game.playerGuideY = guide.y; transitionTo(AIMING)`
- 命中虚线：`|y - guide.y| ≤ GUIDE_HIT_VERT (12)` **且** `x ∈ [HAIR_LEFT_X-20, HAIR_RIGHT_X+20]` → 进入拖动

**拖动**（`mousemove` 时若 `guide.dragging`）：
- `guide.y = clamp(mouseY, CY0+20, CYBT+20)` —— 限制在刘海可见范围

**渲染**：
- 红虚线（`#d04545`，`setLineDash([8, 6])`），横跨 `[HAIR_LEFT_X-10, HAIR_RIGHT_X+10]`
- 右端一个实心圆点把手暗示可拖
- 确认按钮 canvas 内绘制：白底 + 黑边 + 阴影 + "✓ 确认"，位置 `(W/2-60, 470, 120×36)`

### 8.4 AIMING：沿用 v5 物理 + v6 切割

`enter()` 调 `resetScissor()` 并将 `scissorFadeIn` 置 0；`update` 推进 `scissorFadeIn` 在 0.5s 内涨到 1，作为 `drawScissor` 的 `globalAlpha`。其余等同 v6。

进入 AIMING 后 `guide.y` 转存为 `game.playerGuideY`，虚线以 30% alpha 继续显示作为视觉参考。

剪刀飞出左侧（`sc.x < -SC_LEN`）触发 `resetGame()`：重置发丝、剪刀、`game.targetY/playerGuideY/guide.y`，回到 `GESTURING` 重新走流程。

### 8.5 输入屏蔽

| 状态 | Space | J | 鼠标 down/move/up |
|------|-------|---|-------------------|
| GESTURING     | 无效 | 无效 | 无效 |
| SETTING_GUIDE | 无效 | 无效 | 仅响应虚线拖动 + 按钮点击 |
| AIMING        | 抬升 | 切割 | 切割（debug） |

实现方式：`stateGesturing.handleInput` 是空函数；`stateSettingGuide.handleInput` 只匹配 `mousedown/move/up`；`stateAiming.handleInput` 处理键盘 + 鼠标的切割。

### 8.6 调试跳过

- **K 键**（在 `GESTURING` 或 `SETTING_GUIDE` 时）：直接跳到 `AIMING`，自动补未生成的 `targetY` / `playerGuideY`
- **`⏭ 直接进切割` 按钮**：同 K 键
- **🔵 调试**：HUD 右上角额外显示 `targetY / playerGuideY / guide.y`，GESTURING 阶段画出 `targetY` 横线（橙色虚线）

### 8.7 状态相关全局变量

```javascript
const game = {
  targetY:      null,   // NPC 真目标 Y（Step 6 评分用）
  playerGuideY: null,   // 玩家确认的虚线 Y（Step 6 评分用）
};
```

未来 Step 6 评分模块直接读这两个值，不需要再去状态机内部翻数据。

---

## 九、待实现部分（v7 之后）

GDD 第十章开发优先级里，已完成 1（剪刀手感）/ 2（头发切割）/ 5（NPC 比划）/ 6（虚线设定）。剩下：

| 项 | 关键技术点 | 预计涉及 |
|---|----------|---------|
| 评分 | 长度分（指数衰减 + 不对称惩罚）、直线分（max-min Y），用 `game.targetY` / `game.playerGuideY` / 切割轨迹计算 | 新增 RESULT 状态 + 评分函数 |
| 顾客反应 | 切割完成后切 RESULT 状态，根据星级显示 `Assets/01~04.png` | 新增 RESULT 状态渲染 |
| 掉落动画 | 切下来的发丝段加 vy + gravity，独立 update，到地落消失 | 新增 `fallingPieces[]` |
| 关卡 / 多顾客 | 每位顾客比划长度不同，重置后进新一轮 | 顾客队列 + 进度 |

---

## 附录 A · 关键常量速查

```javascript
// 头发
R=116, EXTRA=62, CX=300, CY0=155, CYB=271, CYBT=333, N=58

// 物理
gravity=1100, thrust=2400, drag=2.0, autoAdvance=75
startX=560, startY=230, minY=90, maxY=470

// 剪刀图
SC_LEN=110, SC_THICK≈83, MAX_OPEN_PX=14

// 开合插值速率
openInterpRate = 20  // 1/s

// v6 切割
ARC_SAMPLES = 20         // 弧段采样段数（生成 21 个 {x,y,t} 点）
bladeY      = sc.y       // 刀刃 Y
bladeX0     = sc.x - SC_LEN/2   // 刀尖
bladeX1     = sc.x              // 螺丝（刀刃右端）

// v6 边缘填充发丝
EDGE_THETAS_DEG = [78, 82, 86, 90]   // 每侧 4 根，无 jitter，宽 6.5px

// v7 NPC 比划
HAND_BASE_X            = 165         // 手的水平位置（脸左侧）
HAND_SIZE              = 72
HAND_TRANSITION_TIME   = 0.4         // 位置间过渡（秒）
HAND_PAUSE_AFTER       = 0.5         // 比划结束 → 进 SETTING_GUIDE 的暂停
FAKE_STOP_RANGE        = [0.3, 0.8]
REAL_STOP_RANGE        = [1.0, 1.5]
HAIR_TARGET_Y_MIN/MAX  = 240 / 310   // 真目标 Y 抽样区间
NOD_AMPLITUDE          = 4           // 真目标点头幅度（px）
NOD_FREQUENCY          = 4           // Hz

// v7 虚线 + 按钮
GUIDE_HIT_VERT     = 12              // 拖动判定纵向半径
GUIDE_X_PADDING    = 20              // 拖动判定横向扩展
GUIDE_DEFAULT_Y    = H/2             // 进入 SETTING_GUIDE 时虚线初始 Y
HAIR_LEFT_X        = 184             // 虚线左端 ≈ CX - R
HAIR_RIGHT_X       = 416             // 虚线右端 ≈ CX + R
CONFIRM_BTN        = (W/2-60, 470, 120, 36)
SCISSOR_FADE_IN    = 0.5             // AIMING 进入时剪刀淡入时长
```

## 附录 B · 关于 Bezier-Arc 近似公式

`cpLen = (4/3) · R · tan(theta/4)` 的来源：
对单位圆上从 `(1,0)` 到 `(cos θ, sin θ)` 的弧，要找一组三次 Bezier 控制点 `(1, t)` 与 `(cos θ + t sin θ, sin θ - t cos θ)`，使曲线在端点处与圆相切且经过弧中点。解得 `t = (4/3) tan(θ/4)`。把单位圆缩放回半径 R 即得本式。
