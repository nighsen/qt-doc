# QStyleSheet 层级与覆盖机制 (Hierarchy & Cascading)

## 简介 (Introduction)
Qt 样式表 (QSS) 的设计深受 CSS 启发，支持级联（Cascading）和继承。理解 QSS 的应用层级和覆盖规则，对于管理大型应用程序的统一外观至关重要。

QSS 可以应用在三个不同的层级：
1.  **全局级**: `QApplication::setStyleSheet`
2.  **窗口/容器级**: `QWidget::setStyleSheet` (父控件)
3.  **组件级**: `QWidget::setStyleSheet` (控件本身)

## 作用范围 (Scope)

### 1. 全局级 (Application Level)
*   **设置方式**: `qApp->setStyleSheet("...");`
*   **范围**: 影响整个应用程序中的所有控件。
*   **用途**: 定义应用程序的主题、通用色调、字体和默认控件样式。

### 2. 父控件级 (Parent Level)
*   **设置方式**: `myDialog->setStyleSheet("...");`
*   **范围**: 影响该父控件本身，以及它的**所有**子孙控件。
*   **用途**: 为特定的模块、对话框或面板设定独立的样式风格。

### 3. 组件级 (Widget Level)
*   **设置方式**: `myButton->setStyleSheet("...");`
*   **范围**: 仅影响该控件本身（如果是容器，也会级联影响其子控件）。
*   **用途**: 对特定控件进行微调或高亮显示。

## 覆盖与优先级规则 (Cascading & Priority)

Qt 处理样式表时，会将所有层级的样式表**合并**（Merge）成一个大的样式表，然后根据 CSS 的权重规则来决定最终效果。

**核心原则：权重 (Specificity) > 来源 (Source)**

### 规则 1: 来源优先级 (Source Order)
在**权重相同**（Selector Specificity is equal）的情况下，定义得“越近”的样式优先级越高。
逻辑上的合并顺序是：`Application Styles` -> `Parent Styles` -> `Widget Styles`。
因此，后定义的覆盖先定义的：

> **Widget 本身 > 父控件 > QApplication**

**示例**:
```cpp
// 1. App 级别: 所有按钮文字红色
qApp->setStyleSheet("QPushButton { color: red; }");

// 2. Button 级别: 这个按钮文字蓝色
myButton->setStyleSheet("color: blue;"); // 相当于 "QPushButton { color: blue; }"
```
**结果**: `myButton` 显示为**蓝色**。

### 规则 2: 选择器权重 (Selector Specificity)
如果选择器的特异性不同，则**权重高者胜出**，无论它定义在哪里。
权重计算规则（由高到低）：
1.  **ID 选择器**: `QPushButton#okButton`
2.  **类选择器 / 属性选择器**: `.QPushButton`, `[enabled="true"]`
3.  **类型选择器**: `QPushButton`
4.  **通配符**: `*`

**示例**:
```cpp
// 1. App 级别: 使用 ID 选择器 (权重高)
qApp->setStyleSheet("QPushButton#okButton { color: red; }");

// 2. Button 级别: 通用规则 (权重低)
okButton->setStyleSheet("color: blue;"); // 隐式相当于 "QPushButton { color: blue; }"
```
**结果**: `okButton` 显示为**红色**！
**原因**: App 级别的 `#okButton` 选择器权重高于 Button 自身设置的类型选择器权重。

## 继承性 (Inheritance)

与 Web CSS 不同，QSS 的属性继承行为比较特殊：

1.  **不自动继承**: 大多数样式属性（如 `border`, `background`）**不会**自动从父控件继承到子控件。
    *   例如：给 `QFrame` 设置红背景，其内部的 `QPushButton` 不会变成红背景，除非按钮背景是透明的。
2.  **可继承属性**: 只有 `font`, `palette` (如 `color`) 等少数属性会从父控件传递给子控件。
3.  **强制继承**: 
    *   QSS 会通过级联机制（Cascading）将父控件的 stylesheet 应用于子控件。
    *   如果父控件设置了 `QPushButton { background: red; }`，这个规则会匹配其所有子按钮。

## 常见陷阱 (Common Pitfalls)

### 1. 意外的全局影响
在父控件上写通用选择器会污染子控件。
```cpp
// 错误写法
frame->setStyleSheet("background: white;"); 
// 这实际上等同于 "* { background: white; }"
// 导致 frame 内部的所有子控件（按钮、标签、输入框）背景都变成了白色。

// 正确写法
frame->setStyleSheet(".QFrame { background: white; }");
// 或者针对特定对象
frame->setStyleSheet("#myFrame { background: white; }");
```

### 2. 权重误判
当你发现 `widget->setStyleSheet(...)` 不生效时，通常是因为 Application 或父级设置了一个**权重更高**的选择器（比如用了 ID 选择器）。
*   **调试技巧**: 使用更具体的选择器来提升权重，或者在 App 级别尽量少用 ID 选择器。

## 总结 (Summary)

| 比较维度 | 说明 |
| :--- | :--- |
| **合并逻辑** | App样式 + 父级样式 + 自身样式 = 最终样式表 |
| **同权重优先级** | 自身(Widget) > 父级(Parent) > 全局(App) |
| **最高优先级** | ID 选择器 (`#id`) > 类/属性 (`.class`) > 类型 (`Type`) |
| **最佳实践** | 全局定基调，局部做微调；局部尽量用类名限定，防止污染子控件。 |

## 参考资料 (References)
- [Qt Style Sheets: The Style Sheet Syntax](https://doc.qt.io/qt-6/stylesheet-syntax.html#conflict-resolution)
- [Qt Style Sheets Examples](https://doc.qt.io/qt-6/stylesheet-examples.html)
