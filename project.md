# Project Context

## Purpose
[本工程是一个Qt开发Know库，主要记录Qt开发中遇到的问题和解决方法，以及Qt原理、API解释等相关内容]

## Tech Stack
- **Documentation**: [Markdown]
- **Framework**: [Qt (C++, QML)]
- **Build Systems**: [CMake, qmake] (for code examples)
- **Tools**: [OpenSpec] (for specification management)

## Project Conventions

### Code Style
[文档内容都是用中文，代码示例用英文]

### Architecture Patterns

#### 文档分类体系 (Documentation Categories)

为了高效管理和检索 Qt 开发知识，文档按以下类别进行组织：

1.  **故障排查 (Troubleshooting)**
    *   **适用场景**: 编译错误、运行时崩溃、界面异常、逻辑错误等具体问题的解决记录。
    *   **目录**: `troubleshooting/`
    *   **文件命名**: `issue-<简短描述>.md` (例如: `issue-qthread-slot-not-called.md`)

2.  **核心原理 (Concepts & Principles)**
    *   **适用场景**: Qt 核心机制的解释，如信号槽、事件循环、元对象系统(MOC)、绘图引擎、内存管理等。
    *   **目录**: `concepts/`
    *   **文件命名**: `concept-<概念名称>.md` (例如: `concept-event-loop.md`)

3.  **组件与API (Components & API)**
    *   **适用场景**: 特定类、模块或函数的用法详解及示例。
    *   **目录**: `api/`
    *   **文件命名**: `api-<类名或模块名>.md` (例如: `api-qnetworkaccessmanager.md`)

4.  **最佳实践 (Best Practices)**
    *   **适用场景**: 工程架构、性能优化、编码规范、常用设计模式在 Qt 中的应用。
    *   **目录**: `practices/`
    *   **文件命名**: `practice-<主题>.md` (例如: `practice-cmake-project-structure.md`)

#### 文档格式要求 (Formatting Requirements)

所有文档应采用 Markdown 格式，并遵循以下结构模板：

**1. 故障排查类模板**

```markdown
# [问题标题]

## 问题描述 (Issue Description)
- 现象：[描述具体表现]
- 报错信息：[粘贴关键日志或错误代码]
- 环境：[Qt版本, OS, 编译器]

## 原因分析 (Root Cause Analysis)
- [解释导致问题的根本原因，可结合源码或机制分析]

## 解决方案 (Solution)
- [步骤 1]
- [步骤 2]
<!-- 代码块使用 cpp 语言标签 -->
```

## 参考资料 (References)
- [链接到 Qt 官方文档或相关讨论]
```

**2. 核心原理/组件类模板**

```markdown
# [概念/组件名称]

## 简介 (Introduction)
- [简要说明是什么]

## 工作机制/用法 (Mechanism/Usage)
- [详细解释原理或API使用方法]

## 代码示例 (Code Example)
<!-- 完整的最小可复现示例 -->

## 注意事项 (Notes)
- [使用中的坑或限制]
```

### Testing Strategy
[Explain your testing approach and requirements]

### Git Workflow
[Describe your branching strategy and commit conventions]

## Domain Context

AI 助手需要理解以下 Qt 领域的核心概念：

1.  **模块划分**:
    *   **QtCore**: 核心非 GUI 功能（事件循环、IO、多线程）。
    *   **QtGui**: 窗口系统集成、事件处理、OpenGL/Vulkan 集成。
    *   **QtWidgets**: 传统的桌面 UI 控件集合。
    *   **QtQuick/QML**: 现代的、声明式的 UI 技术，适用于触摸屏和动态界面。
    *   **QtNetwork**: 网络编程能力。

2.  **关键机制**:
    *   **Signal & Slot**: Qt 对象间通信的核心机制。
    *   **Meta-Object System (MOC)**: 提供信号槽、反射、属性系统等动态特性的基础。
    *   **Parent-Child Ownership**: Qt 的内存管理机制，父对象自动管理子对象生命周期。

## Important Constraints

1.  **版本兼容性**:
    *   必须明确区分 **Qt 5** 和 **Qt 6** 的差异。如果代码示例在两个版本中表现不同，必须分别说明。
    *   默认以 **Qt 6 (6.2 LTS 或更高)** 为基准，涉及 Qt 5 的旧代码需标注 `[Qt 5 Legacy]`。

2.  **跨平台性**:
    *   Qt 是跨平台框架，但在 Windows, macOS, Linux 上可能存在差异（如路径分隔符、原生事件等）。
    *   涉及平台特定代码时，必须使用 `#ifdef Q_OS_WIN` 等宏进行包裹，或在文档中明确指出“仅限 Windows”。

3.  **准确性优先**:
    *   严禁臆造 API。所有 API 用法必须基于 Qt 官方文档或经过验证的实践。
    *   对于不确定的行为，应标记为“待验证”或查阅源码确认。

## External Dependencies
[主要参考https://doc.qt.io/]
