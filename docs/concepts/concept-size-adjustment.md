# Qt 大小调节机制详解

在 Qt 界面开发中，理解控件的大小调节机制（Size Adjustment Mechanism）对于构建灵活且响应式的 UI 至关重要。Qt 提供了两套主要的大小管理方式：**布局管理器（Layout Management）** 和 **手动几何管理（Manual Geometry Management）**。

本文将详细解释 `adjustSize()` 和 `updateGeometry()` 这两个核心 API，并介绍与之相关的其他重要 API。

## 1. 核心 API 详解

### adjustSize()

`void QWidget::adjustSize()`

**作用**：
`adjustSize()` 用于根据控件的内容自动调整控件的大小。

**机制**：
- 如果控件在一个布局中，该函数通常不需要手动调用，因为布局管理器会自动处理。
- 当控件是顶层窗口或者没有父布局时，调用此函数会使控件调整为 `sizeHint()` 返回的大小。
- 它是 "一次性" 的调整。它会计算最佳大小并直接调用 `resize()`。

**使用场景**：
- 在初始化一个没有布局管理的顶层窗口或自定义控件时，确保其大小适应内容。
- 当动态改变了控件的内容（例如设置了很长的文本标签），且该控件未受布局管理器完全控制时，调用它来强制更新大小。

### updateGeometry()

`void QWidget::updateGeometry()`

**作用**：
`updateGeometry()` 用于通知布局系统：该控件的几何提示（Geometry Hints，主要是 `sizeHint` 和 `sizePolicy`）已经发生变化。

**机制**：
- 它本身**不会**立即改变控件的大小。
- 它会通知父控件（通常是父布局），告诉它："我的大小需求变了，请重新计算布局"。
- 布局管理器收到通知后，会在适当的时候重新布局，可能会调用该控件的 `resize()`。

**使用场景**：
- 自定义控件（Subclassing QWidget）时，如果内部状态改变导致推荐大小（sizeHint）变化（例如，改变了字体大小、内容变多），必须调用此函数。
- 典型的例子是 `QLabel::setText()` 内部就调用了 `updateGeometry()`，以便布局能自动适应新文本的长度。

---

## 2. 布局系统与大小提示 (Size Hints)

Qt 的布局系统依赖于控件提供的 "提示" 来决定每个控件该多大。

### sizeHint()
`QSize QWidget::sizeHint() const`
- 控件的 "推荐大小"。
- 布局管理器会尽量尊重这个大小，但受限于 `sizePolicy` 和最小/最大尺寸限制。
- 自定义控件通常需要重写此函数来告诉布局它需要多大空间。

### minimumSizeHint()
`QSize QWidget::minimumSizeHint() const`
- 控件能被压缩到的最小推荐大小。
- 布局管理器通常不会把控件压缩到比这个更小（除非 `sizePolicy` 允许忽略它）。

### sizePolicy (大小策略)
`QSizePolicy QWidget::sizePolicy() const`
- 决定了控件在布局中如何随窗口缩放。
- 常见策略：
    - `Fixed`: 固定为 `sizeHint()`，不伸缩。
    - `Preferred`: 优先使用 `sizeHint()`，但可以伸缩。
    - `Expanding`: 尽量占用更多空间。
    - `Ignored`: 忽略 `sizeHint()`，完全由布局分配。

---

## 3. 相关 API 对比

| API | 描述 | 触发布局更新? | 是否立即生效? |
| :--- | :--- | :--- | :--- |
| **resize(w, h)** | 强制设置控件大小。如果控件在布局中，可能会被布局在下一次刷新时覆盖。 | 否 (通常) | 是 |
| **setGeometry(x, y, w, h)** | 设置位置和大小。同上。 | 否 | 是 |
| **adjustSize()** | 根据 `sizeHint()` 设置大小。 | 否 | 是 |
| **updateGeometry()** | **仅仅通知** 布局系统大小提示已改变。 | **是** (异步) | 否 (等待布局刷新) |
| **layout()->activate()** | 强制重新计算布局（通常不需要手动调用，Qt 会自动处理）。 | 是 | 是 |

## 4. 最佳实践总结

1. **自定义控件**：
    - 总是重写 `sizeHint()` 返回合理的大小。
    - 当影响大小的属性改变时，调用 `updateGeometry()`。

2. **使用布局时**：
    - 尽量不要手动调用 `resize()` 或 `setGeometry()`，让布局管理器接管。
    - 如果需要改变控件大小，考虑修改其 `FixedSize`、`MinimumSize` 或调整 `Stretch Factor`。

3. **不使用布局时**：
    - 使用 `adjustSize()` 来快速适配内容大小。
    - 需要手动处理 `resizeEvent` 来管理子控件位置。

## 5. 示例代码

```cpp
// 自定义控件示例
class MyWidget : public QWidget {
    QString m_text;
public:
    void setText(const QString &text) {
        m_text = text;
        // 1. 内容改变，导致需要的空间改变，通知布局系统
        updateGeometry(); 
    }

    QSize sizeHint() const override {
        // 2. 布局系统询问时，返回基于内容计算的大小
        QFontMetrics fm(font());
        return fm.size(Qt::TextShowMnemonic, m_text);
    }
};

// 使用示例
void setup() {
    MyWidget *w = new MyWidget;
    w->setText("Hello World");
    
    // 如果没有父布局，可以手动调整
    w->adjustSize(); 
    w->show();
}
```
