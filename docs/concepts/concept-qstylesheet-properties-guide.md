# QSS 属性指南与常见问题 (Properties Guide & Troubleshooting)

本文档详细介绍了 Qt 样式表 (QSS) 中常见属性的适用范围，并针对开发者常遇到的“背景不生效”问题提供了解决方案和系统性总结。

## 1. 常见问题：QWidget 背景不生效

这是 Qt 开发者最常遇到的 QSS 陷阱之一。

### 问题现象
在 QSS 中给 `QWidget`（或其自定义子类）设置 `background-color` 或 `background-image`，但运行时界面背景依然是透明或灰色的，样式未生效。

### 核心原因
`QWidget` 的默认 `paintEvent` 实现是空的。为了性能优化，默认情况下 `QWidget` 不会进行背景绘制操作，而是让其父控件的内容“透”过来，或者直接使用系统默认背景。
即使样式表引擎解析了 `background` 属性，控件本身如果没有执行绘制逻辑，属性也就无法呈现。

### 解决方案

#### 方法一：开启 `WA_StyledBackground` 属性（推荐）
这是最简单且副作用最小的方法。通过设置该属性，告知 Qt 渲染引擎：此控件需要支持样式表背景绘制。

```cpp
// 在自定义控件的构造函数中添加
MyWidget::MyWidget(QWidget *parent) : QWidget(parent) {
    // 关键代码
    this->setAttribute(Qt::WA_StyledBackground, true); 
}
```

#### 方法二：重写 `paintEvent`
在 `paintEvent` 中使用 `QStylePainter` 手动支持样式表绘制。这种方法更底层，适用于需要混合自定义绘制逻辑的场景。

```cpp
void MyWidget::paintEvent(QPaintEvent *event) {
    QStyleOption opt;
    opt.init(this);
    QPainter p(this);
    // 使用 style() 绘制基本图元，它会自动处理 stylesheet
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
}
```

---

## 2. QSS 属性适用性系统总结

QSS 属性并非对所有控件通用。以下按**属性类别**划分，帮助快速判断哪些属性对哪些控件有效。

### A. 盒子模型 (Box Model)
**适用范围**：绝大多数可视控件（`QWidget` 及其子类）。
**注意**：对于 `QWidget`, `QFrame`, `QDialog` 等容器，通常需要配合 `border` 或 `background` 才能明显看出效果。

| 属性 | 描述 | 典型适用控件 | 备注 |
| :--- | :--- | :--- | :--- |
| `background` / `background-color` | 背景色/背景图 | 所有 Widget | 自定义 QWidget 需开启 `WA_StyledBackground` |
| `border` | 边框（样式、宽度、颜色） | 所有 Widget | `QLabel`, `QLineEdit`, `QFrame`, `QPushButton` |
| `padding` | 内边距（内容与边框的距离） | 所有 Widget | 影响文字/内容的位置 |
| `margin` | 外边距（边框外的距离） | 部分 Widget | 在 Layout 中更有效；某些原生控件可能忽略 |
| `min-width` / `max-width` | 最小/最大宽度 | 所有 Widget | 样式表中的限制会影响布局计算 |

### B. 文本与字体 (Text & Font)
**适用范围**：具有文本显示能力的控件。

| 属性 | 描述 | 典型适用控件 | 备注 |
| :--- | :--- | :--- | :--- |
| `color` | 前景色（文字颜色） | `QLabel`, `QPushButton`, `QLineEdit`, `QCheckBox` | |
| `font` / `font-family` / `font-size` | 字体相关 | 所有显示文字的控件 | 会继承给子控件（除非被覆盖） |
| `text-align` | 文字对齐方式 | `QPushButton`, `QProgressBar` | `QLabel` 通常用 `alignment` 属性控制，QSS 支持有限 |
| `selection-background-color` | 选中文字背景色 | `QLineEdit`, `QTextEdit`, `QComboBox` | |
| `selection-color` | 选中文字颜色 | `QLineEdit`, `QTextEdit` | |

### C. 图像与图标 (Images & Icons)
**适用范围**：支持图标或作为装饰的控件。

| 属性 | 描述 | 典型适用控件 | 备注 |
| :--- | :--- | :--- | :--- |
| `image` | 绘制在内容区域的图片 | `QPushButton`, `QToolButton`, `QLabel`, sub-controls | 不会像 background 那样平铺，可以控制位置 |
| `icon-size` | 图标大小 | `QToolButton`, `QTabBar`, `QListView` | |
| `qproperty-icon` | 设置图标属性 | `QPushButton`, `QToolButton` | 使用 Qt 属性系统语法 |

### D. 子控件定制 (Sub-controls)
**适用范围**：复杂控件（由多个部分组成的控件）。

| 选择器 | 描述 | 适用控件 |
| :--- | :--- | :--- |
| `::indicator` | 复选框/单选框的指示器 | `QCheckBox`, `QRadioButton`, `QGroupBox`, `QListView` |
| `::down-arrow` | 下拉箭头 | `QComboBox`, `QSpinBox` |
| `::drop-down` | 下拉按钮区域 | `QComboBox` |
| `::up-button` / `::down-button` | 微调按钮 | `QSpinBox`, `QScrollBar` |
| `::handle` | 滑块手柄 | `QScrollBar`, `QSlider`, `QSplitter` |
| `::chunk` | 进度条的进度块 | `QProgressBar` |
| `::tab` | 标签页 | `QTabBar` |
| `::title` | 分组框标题 | `QGroupBox` |

---

## 3. 典型 QSS 示例代码

### 示例 1: 自定义 QWidget (作为容器)
```css
/* 确保 C++ 代码中设置了 setAttribute(Qt::WA_StyledBackground); */
QWidget#myContainer {
    background-color: #f0f0f0;
    border: 1px solid #cccccc;
    border-radius: 5px; /* 圆角 */
}
```

### 示例 2: 复杂控件 QComboBox
`QComboBox` 是最复杂的控件之一，通常需要定制它的“下拉区域”和“箭头”。

```css
/* 主体样式 */
QComboBox {
    border: 1px solid gray;
    border-radius: 3px;
    padding: 1px 18px 1px 3px; /* 为箭头留出右侧空间 */
    min-width: 6em;
}

/* 下拉按钮区域 */
QComboBox::drop-down {
    subcontrol-origin: padding;
    subcontrol-position: top right;
    width: 15px;
    border-left-width: 1px;
    border-left-color: darkgray;
    border-left-style: solid;
    border-top-right-radius: 3px;
    border-bottom-right-radius: 3px;
    background: qlineargradient(x1:0, y1:0, x2:0, y2:1, stop:0 #f6f7fa, stop:1 #dadbde);
}

/* 下拉箭头本身 */
QComboBox::down-arrow {
    image: url(:/icons/down_arrow.png); /* 需替换为实际图片路径 */
}

/* 箭头在下拉状态偏移，产生点击感 */
QComboBox::down-arrow:on {
    top: 1px;
    left: 1px;
}
```

### 示例 3: 进度条 QProgressBar
```css
QProgressBar {
    border: 2px solid grey;
    border-radius: 5px;
    text-align: center; /* 文字居中 */
}

/* 进度块样式 */
QProgressBar::chunk {
    background-color: #05B8CC;
    width: 20px; /* 如果设置了 width，进度条会显示为一格一格的 */
    margin: 0.5px;
}
```

## 参考资料
- [Qt Style Sheets Reference](https://doc.qt.io/qt-6/stylesheet-reference.html)
