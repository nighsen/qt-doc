# QAbstractNativeEventFilter

## 简介 (Introduction)
`QAbstractNativeEventFilter` 是 Qt 提供的一个接口类，用于访问底层的操作系统原生事件（Native Events）。它允许开发者在 Qt 的事件系统将这些事件转换为 Qt 事件（如 `QMouseEvent`, `QKeyEvent`）之前，对它们进行拦截和处理。

这个类通常用于处理 Qt 框架自身未封装或无法直接处理的平台特定消息，例如 Windows 的 `WM_COPYDATA`、`WM_HOTKEY`，或者 Linux X11/XCB 的特定事件。

## 工作机制/用法 (Mechanism/Usage)

1.  **继承接口**: 创建一个类继承自 `QAbstractNativeEventFilter`。
2.  **实现虚函数**: 重写 `nativeEventFilter` 函数。
    *   该函数接收事件类型、消息指针和结果指针。
    *   返回 `true` 表示停止事件进一步传播（即 Qt 不会再处理该事件）。
    *   返回 `false` 表示事件继续传递给 Qt 的标准事件处理流程。
3.  **安装过滤器**: 在 `QCoreApplication` (或 `QApplication`, `QGuiApplication`) 上调用 `installNativeEventFilter` 方法来启用过滤器。
    *   **注意**: 过滤器是全局的，会拦截应用程序接收到的所有原生事件。

### 函数签名差异
由于 Qt 5 和 Qt 6 在底层类型定义上的变化，`nativeEventFilter` 的签名有所不同，开发时需注意兼容性。

*   **Qt 5**: `virtual bool nativeEventFilter(const QByteArray &eventType, void *message, long *result);`
*   **Qt 6**: `virtual bool nativeEventFilter(const QByteArray &eventType, void *message, qintptr *result);`

## 代码示例 (Code Example)

以下示例展示了如何在 Windows 平台上捕获 `WM_DISPLAYCHANGE` (显示器分辨率改变) 消息。代码使用了宏来处理 Qt 5 和 Qt 6 的签名差异。

### 头文件 (NativeEventFilter.h)

```cpp
#ifndef NATIVEEVENTFILTER_H
#define NATIVEEVENTFILTER_H

#include <QAbstractNativeEventFilter>
#include <QByteArray>
#include <QDebug>

class NativeEventFilter : public QAbstractNativeEventFilter
{
public:
    // 根据 Qt 版本选择正确的签名
#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
    bool nativeEventFilter(const QByteArray &eventType, void *message, qintptr *result) override;
#else
    bool nativeEventFilter(const QByteArray &eventType, void *message, long *result) override;
#endif
};

#endif // NATIVEEVENTFILTER_H
```

### 源文件 (NativeEventFilter.cpp)

```cpp
#include "NativeEventFilter.h"

// 引入平台相关的头文件
#ifdef Q_OS_WIN
#include <windows.h>
#endif

#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
bool NativeEventFilter::nativeEventFilter(const QByteArray &eventType, void *message, qintptr *result)
#else
bool NativeEventFilter::nativeEventFilter(const QByteArray &eventType, void *message, long *result)
#endif
{
    // 1. 判断事件类型
    // Windows 平台通常是 "windows_generic_MSG" 或 "windows_dispatcher_MSG"
    if (eventType == "windows_generic_MSG" || eventType == "windows_dispatcher_MSG") {
        
#ifdef Q_OS_WIN
        // 2. 将 void* 转换为平台特定的结构体
        MSG *msg = static_cast<MSG *>(message);

        // 3. 处理特定的消息
        if (msg->message == WM_DISPLAYCHANGE) {
            qDebug() << "Detected display resolution change!";
            
            // 可以在这里获取更多信息
            int newWidth = LOWORD(msg->lParam);
            int newHeight = HIWORD(msg->lParam);
            qDebug() << "New Resolution:" << newWidth << "x" << newHeight;

            // 如果返回 true，Qt 将不会收到这个事件（通常对于观察类需求，建议返回 false）
            // return true; 
        }
#endif
    }

    // 返回 false 让 Qt 继续处理该事件
    return false;
}
```

### 安装过滤器 (main.cpp)

```cpp
#include <QApplication>
#include "NativeEventFilter.h"

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    // 实例化并安装过滤器
    NativeEventFilter nativeFilter;
    a.installNativeEventFilter(&nativeFilter);

    // ... 创建主窗口等 ...

    return a.exec();
}
```

## 注意事项 (Notes)

1.  **平台依赖性**: `nativeEventFilter` 处理的是裸的操作系统消息（如 Windows 的 `MSG`，X11 的 `xcb_generic_event_t`）。使用此功能意味着你的代码将变得不可移植（Not Portable），必须使用 `#ifdef Q_OS_...` 宏进行隔离。
2.  **性能影响**: 由于此过滤器会在所有原生事件进入 Qt 之前被调用，因此其中的逻辑必须非常高效。避免在 `nativeEventFilter` 中执行耗时操作，否则会阻塞整个 UI 线程。
3.  **返回值的副作用**: 如果返回 `true`，Qt 内部的事件处理机制（包括 Widget 的绘制、鼠标交互等）将不会收到该事件。除非你明确知道要屏蔽该消息，否则应始终返回 `false`。
4.  **eventType 字符串**:
    *   Windows: `"windows_generic_MSG"`, `"windows_dispatcher_MSG"`
    *   macOS: `"mac_generic_NSEvent"`
    *   Linux (X11/XCB): `"xcb_generic_event_t"`
    *   建议在代码中打印 `eventType` 来确认当前平台使用的标识符。
