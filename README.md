# CSS 精灵图生成工具

基于 React + TypeScript + Vite 构建的 CSS 精灵图（Sprite）生成与拆分工具。

---

## 精灵图生成算法核心计算逻辑详解

本章节深入解析 [spriteGenerator.ts](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts) 中的核心算法，完整说明从用户配置参数到最终画布绘制每一步的计算过程。

### 一、输入参数定义

用户通过 `SpriteConfig` 配置精灵图生成参数，核心字段如下（参见 [types/index.ts](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/types/index.ts#L23-L29)）：

| 参数 | 类型 | 含义 |
|------|------|------|
| `columns` | number | 用户指定的列数 |
| `spacing` | number | 图标之间的间距（像素） |
| `bgColor` | string | 背景颜色（支持 'transparent'） |
| `classPrefix` | string | CSS 类名前缀 |
| `retina` | boolean | 是否启用 Retina 高清模式 |

同时输入一组 `IconItem[]`，每个图标携带自身原始尺寸 `width` / `height`。

---

### 二、Step 1：Retina 缩放系数与实际间距

**代码位置**：[spriteGenerator.ts#L38-L40](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L38-L40)

```typescript
const scale = retina ? 2 : 1;
const actualSpacing = spacing * scale;
```

#### 逻辑说明：

- **非 Retina 模式**：`scale = 1`，画布按 1:1 物理像素绘制。
- **Retina 模式**：`scale = 2`，画布尺寸和所有坐标整体翻倍，生成 2x 高清图，最终在 CSS 中通过 `background-size` 压缩回逻辑尺寸显示，保证在高密度屏幕上的清晰度。
- `actualSpacing` 是实际参与坐标计算的间距，已经经过缩放处理。

> **关键理解**：Retina 模式的本质是「画布按 2x 像素绘制，CSS 输出时再统一除以 2」。坐标、尺寸、间距全部参与翻倍。

---

### 三、Step 2：单元格尺寸（cellWidth / cellHeight）计算

**代码位置**：[spriteGenerator.ts#L46-L47](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L46-L47)

```typescript
const cellWidth = Math.max(...loadedImages.map((img) => img.width)) * scale;
const cellHeight = Math.max(...loadedImages.map((img) => img.height)) * scale;
```

#### 计算公式：

```
cellWidth  = max(所有图标原始宽度)  × scale
cellHeight = max(所有图标原始高度)  × scale
```

#### 逻辑说明：

1. 遍历所有已加载的图标，取最大宽度和最大高度作为基准单元格尺寸。
2. 乘以 `scale` 得到实际绘制用的单元格像素尺寸。
3. 所有图标无论自身大小，都被放入相同尺寸的单元格中，保证网格对齐整齐。

#### 示例：

假设 3 个图标尺寸分别为 16×16、24×20、32×32，`retina=true`：

```
cellWidth  = max(16, 24, 32) × 2 = 32 × 2 = 64 px
cellHeight = max(16, 20, 32) × 2 = 32 × 2 = 64 px
```

即每个单元格为 64×64 像素。

---

### 四、Step 3：实际行列数计算

**代码位置**：[spriteGenerator.ts#L49-L50](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L49-L50)

```typescript
const numCols = Math.min(columns, icons.length);
const numRows = Math.ceil(icons.length / numCols);
```

#### 计算公式：

```
numCols = min(用户指定列数, 图标总数)
numRows = ceil(图标总数 / numCols)
```

#### 逻辑说明：

- `numCols` 取用户指定列数与图标数的较小值，避免图标少于列数时出现多余空列。
- `numRows` 使用向上取整（`ceil`），保证最后一行即使不满也能容纳剩余图标。

#### 示例：

共 7 个图标，用户指定 `columns=3`：

```
numCols = min(3, 7) = 3
numRows = ceil(7 / 3) = ceil(2.333...) = 3
```

即排布为 3 列 × 3 行，最后一行只有 1 个图标。

---

### 五、Step 4：画布总尺寸计算

**代码位置**：[spriteGenerator.ts#L52-L53](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L52-L53)

```typescript
const totalWidth = numCols * cellWidth + (numCols + 1) * actualSpacing;
const totalHeight = numRows * cellHeight + (numRows + 1) * actualSpacing;
```

#### 计算公式：

```
总宽度  = 列数 × 单元格宽度  + (列数 + 1) × 实际间距
总高度  = 行数 × 单元格高度  + (行数 + 1) × 实际间距
```

#### 逻辑说明：

间距分布在 **单元格之间 + 画布四周外边界**，即：

```
┌───────────────────────────────────────────┐
│  spacing  │  cell  │  spacing  │  cell  │  spacing  │   ← 横向
├───────────────────────────────────────────┤
│  spacing                                   │
│  ┌──────┐          ┌──────┐               │
│  │ cell │ spacing  │ cell │               │
│  └──────┘          └──────┘               │
│  spacing                                   │   ← 纵向
│  ┌──────┐          ┌──────┐               │
│  │ cell │ spacing  │ cell │               │
│  └──────┘          └──────┘               │
│  spacing                                   │
└───────────────────────────────────────────┘
```

因此 `(numCols + 1)` 对应「左外边距 + 列间间距 + 右外边距」共 `numCols + 1` 段间距，纵向同理。

#### 示例：

`numCols=3`，`cellWidth=64`，`actualSpacing=10`：

```
totalWidth = 3 × 64 + (3 + 1) × 10 = 192 + 40 = 232 px
```

---

### 六、Step 5：每个图标精确坐标计算

**代码位置**：[spriteGenerator.ts#L67-L73](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L67-L73)

```typescript
const row = Math.floor(index / numCols);
const col = index % numCols;
const x = actualSpacing + col * (cellWidth + actualSpacing);
const y = actualSpacing + row * (cellHeight + actualSpacing);
```

#### 计算公式：

对第 `i` 个图标（索引从 0 开始）：

```
行号 row = floor(i / numCols)
列号 col = i % numCols

单元格左上角 X = 外边距间距 + col × (单元格宽 + 间距)
单元格左上角 Y = 外边距间距 + row × (单元格高 + 间距)
```

#### 逻辑说明：

- `row` / `col` 将一维图标列表映射为二维网格坐标。
- 每个单元格之间间隔 `actualSpacing`，所以横向步进为 `cellWidth + actualSpacing`。
- 最外层有一段 `actualSpacing` 作为画布外边距。

#### 示例：

`numCols=3`，`cellWidth=64`，`actualSpacing=10`，第 4 个图标（index=3）：

```
row = floor(3 / 3) = 1
col = 3 % 3 = 0

x = 10 + 0 × (64 + 10) = 10
y = 10 + 1 × (64 + 10) = 84
```

即该图标单元格左上角位于画布的 (10, 84) 位置。

---

### 七、Step 6：图标尺寸小于单元格时的居中偏移量

**代码位置**：[spriteGenerator.ts#L75-L80](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L75-L80)

```typescript
const drawWidth = img.width * scale;
const drawHeight = img.height * scale;
const offsetX = (cellWidth - drawWidth) / 2;
const offsetY = (cellHeight - drawHeight) / 2;

ctx.drawImage(img, x + offsetX, y + offsetY, drawWidth, drawHeight);
```

#### 计算公式：

```
实际绘制宽度  = 图标原始宽度  × scale
实际绘制高度  = 图标原始高度  × scale

水平居中偏移 = (单元格宽 - 绘制宽) / 2
垂直居中偏移 = (单元格高 - 绘制高) / 2

图标最终绘制 X = 单元格左上角 X + 水平偏移
图标最终绘制 Y = 单元格左上角 Y + 垂直偏移
```

#### 逻辑说明：

由于单元格尺寸取的是所有图标的最大值，较小的图标会有多余空间。通过将差值对分，使图标在单元格中水平和垂直居中，保证视觉对齐整齐。

#### 示例：

单元格 64×64，某图标原始尺寸 24×20，`retina=true`（scale=2）：

```
drawWidth  = 24 × 2 = 48
drawHeight = 20 × 2 = 40

offsetX = (64 - 48) / 2 = 8
offsetY = (64 - 40) / 2 = 12
```

若单元格左上角为 (10, 84)，则该图标实际绘制位置为：

```
绘制 X = 10 + 8  = 18
绘制 Y = 84 + 12 = 96
```

---

### 八、Step 7：Retina 模式下 CSS 输出的还原逻辑

**代码位置**：[spriteGenerator.ts#L109-L148](file:///d:/code/gy/45/45-未归类-22/45-未归类-22/src/services/spriteGenerator.ts#L109-L148)

CSS 生成时需要将画布的物理像素尺寸还原为逻辑显示尺寸：

```typescript
const scale = retina ? 2 : 1;
const displayWidth = Math.round(totalWidth / scale);
const displayHeight = Math.round(totalHeight / scale);
const displayCellW = Math.round(cellWidth / scale);
const displayCellH = Math.round(cellHeight / scale);
```

对每个图标位置同样做缩放：

```typescript
const x = Math.round(pos.x / scale);
const y = Math.round(pos.y / scale);
```

#### 计算公式：

```
CSS 显示尺寸 = Canvas 物理尺寸 / scale
CSS 背景位置偏移 = Canvas 物理坐标 / scale
```

#### 逻辑说明：

- Retina 模式下图实际是 2x 像素图片，通过 CSS 的 `background-size` 将整张精灵图压缩到 1x 的逻辑尺寸显示。
- 每个图标的 `background-position` 负偏移同样需要除以 scale，才能在压缩后的图上准确定位。
- 这样做使得开发者在使用 CSS 时按 1x 逻辑像素设置元素尺寸即可，浏览器会自动利用 2x 图片的高清像素。

#### 完整示例（Retina 模式）：

假设 `totalWidth=232`，`totalHeight=232`，`retina=true`：

```css
.sprite {
  background-size: 116px 116px;  /* 232 / 2 */
  width: 32px;                    /* 64 / 2 单元格逻辑宽 */
  height: 32px;                   /* 64 / 2 单元格逻辑高 */
}
.sprite-home {
  background-position: -9px -48px; /* 物理坐标 (18, 96) / 2 → 取整 */
  width: 24px;                     /* 48 / 2 图标逻辑宽 */
  height: 20px;                    /* 40 / 2 图标逻辑高 */
}
```

---

### 九、完整计算流程汇总

```
用户配置参数
    │
    ├─→ 计算 scale = retina ? 2 : 1
    │
    ├─→ 计算 actualSpacing = spacing × scale
    │
    ├─→ 加载所有图标，获取原始尺寸
    │
    ├─→ cellWidth  = max(所有图标宽)  × scale
    │   cellHeight = max(所有图标高)  × scale
    │
    ├─→ numCols = min(columns, icons.length)
    │   numRows = ceil(icons.length / numCols)
    │
    ├─→ totalWidth  = numCols × cellWidth  + (numCols + 1) × actualSpacing
    │   totalHeight = numRows × cellHeight + (numRows + 1) × actualSpacing
    │
    ├─→ 创建 Canvas(totalWidth, totalHeight)，填充背景色
    │
    └─→ 对每个图标 index i：
            row = floor(i / numCols)
            col = i % numCols
            cellX = actualSpacing + col × (cellWidth + actualSpacing)
            cellY = actualSpacing + row × (cellHeight + actualSpacing)
            drawW = img.width × scale
            drawH = img.height × scale
            offsetX = (cellWidth - drawW) / 2
            offsetY = (cellHeight - drawH) / 2
            绘制位置 = (cellX + offsetX, cellY + offsetY)
            记录 IconPosition(x, y, width, height)

    最后生成 CSS / SCSS：
        所有尺寸与坐标统一除以 scale 得到逻辑像素
        background-size = (totalWidth/scale, totalHeight/scale)
        background-position = (-x/scale, -y/scale)
```

---

## 项目基础说明

### 技术栈

- **前端框架**: React 18 + TypeScript
- **构建工具**: Vite 5
- **样式方案**: TailwindCSS 3 + CSS Variables
- **状态管理**: Zustand
- **图标库**: lucide-react
- **文件处理**: JSZip (批量导出ZIP)
- **存储**: localStorage (元数据) + IndexedDB (图片Blob)

### 目录结构

```
src/
├── components/        # UI 组件
├── pages/             # 页面 (Generator / Splitter / Library)
├── services/          # 核心服务
│   ├── spriteGenerator.ts   # 精灵图合成
│   └── spriteSplitter.ts    # 精灵图拆分
├── store/             # Zustand 状态管理
├── types/             # TypeScript 类型定义
├── utils/             # 工具函数
└── __tests__/         # 测试用例
```

### 开发命令

```bash
npm install    # 安装依赖
npm run dev    # 启动开发服务器
npm run build  # 构建生产版本
npm run test   # 运行测试
```
