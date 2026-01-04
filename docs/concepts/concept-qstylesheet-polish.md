# QStyleSheet 刷新机制 (Polish & Unpolish)

## 简介 (Introduction)
在 Qt 中，`polish` (打磨) 和 `unpolish` (去光/复原) 是 `QStyle` 类的两个核心虚函数，用于初始化和清理控件的样式。

当使用 **Qt Style Sheets (QSS)** 时，Qt 内部使用了一个特殊的 `QStyle` 实现（通常是 `QStyleSheetStyle`）。理解这两个方法对于解决 **动态属性改变后样式未刷新** 的问题至关重要。

## 核心概念 (Core Concepts)

### 1. polish(QWidget *widget)
*   **作用**: "打磨" 控件。这是样式初始化的地方。
*   **运行时机**: 
    *   控件第一次创建并显示之前。
    *   调用 `QWidget::setStyleSheet` 后。
    *   显式调用 `style()->polish(widget)` 时。
*   **QSS 中的行为**: 当 `polish` 被调用时，QSS 引擎会根据控件当前的属性（类名、ObjectName、动态属性等）重新计算选择器（Selectors），解析出适用的样式规则，并将计算出的调色板（Palette）、字体（Font）和样式表规则应用到控件上。

### 2. unpolish(QWidget *widget)
*   **作用**: 清除样式设置，将控件恢复到标准状态。
*   **运行时机**:
    *   控件被销毁前。
    *   样式被更改或移除时。
*   **QSS 中的行为**: 撤销 QSS 对该控件造成的所有影响（例如恢复默认的调色板和字体），确保下一次 `polish` 能在一个干净的状态下运行。

## 常见问题：动态属性与样式刷新
**问题描述**: 
假设你在 QSS 中使用了[动态属性选择器](https://doc.qt.io/qt-6/stylesheet-examples.html#customizing-using-dynamic-properties)：
```css
/* 当属性 isError 为 true 时，背景变红 */
QLineEdit[isError="true"] {
    background-color: red;
}
```
当你在 C++ 代码中调用 `lineEdit->setProperty("isError", true);` 时，你会发现 **界面并没有自动变红**。

**原因**: 
Qt 的样式表计算是**惰性**且**一次性**的。仅仅改变属性值（Property）不会自动触发样式系统的重新计算（Re-evaluation）。Qt 不知道这个属性变化是否影响了样式，为了性能，它默认什么都不做。

## 解决方案：手动触发刷新 (The Polish Loop)

要强制 QSS 引擎重新评估当前控件的样式，必须遵循 **"Unpolish -> Polish"** 的惯用模式。

### 代码示例 (Code Example)

封装一个静态辅助函数来处理样式刷新：

```cpp
#include <QStyle>
#include <QWidget>

// StyleUtils.h
class StyleUtils {
public:
    static void refreshStyle(QWidget *widget) {
        if (!widget) return;
        
        // 1. 获取控件当前的 Style（可能是 QStyleSheetStyle）
        QStyle *style = widget->style();
        
        // 2. 清理旧样式信息
        style->unpolish(widget);
        
        // 3. 重新应用样式（此时会根据最新的属性值重新匹配 QSS 选择器）
        style->polish(widget);
        
        // 4. (可选) 强制重绘，确保视觉更新
        // 通常 polish 会触发 update，但有时显式调用更保险
        widget->update(); 
    }
};

// 使用场景
void MyForm::onErrorOccurred() {
    ui->myLineEdit->setProperty("isError", true);
    
    // 必须手动刷新！
    StyleUtils::refreshStyle(ui->myLineEdit);
}
```

### 为什么不能只调 `polish`？
如果只调用 `polish` 而不先 `unpolish`，Qt 的内部缓存可能认为该控件已经初始化过了，或者在某些实现中可能会导致状态叠加不正确。标准的做法是先清理再初始化，确保完全的重新计算。

## 自动刷新机制 (Auto-Refresh)
虽然自定义属性需要手动刷新，但某些特定的 Qt 内置状态变化会自动触发刷新，因为 Qt 内核对它们做了特殊处理：
*   `enabled` 状态变化 (`setEnabled`)
*   `checked` 状态变化 (对于 QAbstractButton)
*   `hover` (鼠标悬停)
*   `pressed` (鼠标按下)

对于这些标准伪状态（Pseudo-states），你不需要手动调用 `unpolish`/`polish`。只有当你使用自定义的 `Q_PROPERTY` 或动态属性 (`setProperty`) 作为 QSS 选择器条件时，才需要手动干预。

## 注意事项 (Notes)
1.  **性能开销**: `unpolish`/`polish` 过程涉及 CSS 解析匹配和调色板计算，相对昂贵。不要在高频事件（如 `mouseMoveEvent` 或动画循环）中频繁调用。
2.  **子控件级联**: 对父控件调用 `unpolish`/`polish` **不会** 自动递归刷新所有子控件。如果样式属性是继承的（如字体），或者选择器涉及到子控件结构，可能需要手动遍历刷新子控件。
3.  **QEvent::StyleChange**: 调用 `polish` 可能会触发 `QEvent::StyleChange` 事件，如果你的控件重写了 `event()` 方法，需留意此事件。

## 参考资料 (References)
- [Qt Style Sheets Reference](https://doc.qt.io/qt-6/stylesheet-reference.html)
- [QStyle::polish](https://doc.qt.io/qt-6/qstyle.html#polish)
- [Working with Custom Properties](https://doc.qt.io/qt-6/stylesheet-examples.html#customizing-using-dynamic-properties)
