# Model/View/Delegate (MVD)

## 简介 (Introduction)
Qt 的 **Model/View/Delegate (MVD)** 架构是对经典 **MVC (Model-View-Controller)** 模式的一种改编。在 Qt 中，View 和 Controller 的角色被合并，并引入了 **Delegate (代理)** 的概念。

这种架构将数据的存储（Model）与数据的展示（View）分离开来，使得同一个数据模型可以在多个视图中以不同的方式显示，同时通过 Delegate 提供了灵活的自定义渲染和编辑能力。

*   **Model (模型)**: 负责存储数据，并提供标准接口供 View 和 Delegate 访问。它不关心数据如何显示。
*   **View (视图)**: 负责将 Model 中的数据展示给用户。
*   **Delegate (代理)**: 负责在 View 中绘制每一个数据项（Item），并处理用户的编辑操作（如在单元格中创建下拉框、微调框等）。

## 核心组件 (Core Components)

### 1. Model (模型)
所有模型都继承自 `QAbstractItemModel`。
*   **`QAbstractListModel`**: 用于一维列表数据。
*   **`QAbstractTableModel`**: 用于二维表格数据。
*   **`QStandardItemModel`**: Qt 提供的通用模型，适合简单数据，但在大数据量下性能较差。

### 2. View (视图)
所有视图都继承自 `QAbstractItemView`。
*   **`QListView`**: 显示一维列表。
*   **`QTableView`**: 显示二维表格。
*   **`QTreeView`**: 显示层级树状数据。

### 3. Delegate (代理)
所有代理都继承自 `QAbstractItemDelegate`，通常继承 **`QStyledItemDelegate`**。
*   **`paint()`**: 自定义绘制内容（例如画进度条、图标、富文本）。
*   **`createEditor()`**: 创建编辑控件（例如 QComboBox, QSpinBox）。
*   **`setEditorData()` / `setModelData()`**: 在 Model 和 Editor 之间交换数据。

## 工作机制 (Mechanism)

1.  **数据读取**: View 通过 Model 的 `data()` 接口获取数据。View 会请求不同的 **Role**（角色），例如：
    *   `Qt::DisplayRole`: 显示的文本。
    *   `Qt::DecorationRole`: 图标/图片。
    *   `Qt::BackgroundRole`: 背景色。
    *   `Qt::UserRole`: 用户自定义数据。
2.  **数据更新**: 当数据发生变化时，Model 发出 `dataChanged()` 信号，通知 View 刷新。
3.  **交互编辑**: 当用户双击单元格时，View 请求 Delegate 创建一个编辑器（Editor）。编辑完成后，Delegate 将新数据写入 Model。

## 代码示例 (Code Example)

以下示例展示了一个只读的自定义 Model 和一个自定义 Delegate（将数字渲染为星级评分）。

### 1. 自定义 Model (ReadOnlyTableModel.h)

```cpp
#include <QAbstractTableModel>
#include <vector>

struct Movie {
    QString title;
    int rating; // 1-5
};

class MovieModel : public QAbstractTableModel {
    Q_OBJECT
public:
    explicit MovieModel(QObject *parent = nullptr) : QAbstractTableModel(parent) {
        m_data = {
            {"The Matrix", 5},
            {"Inception", 5},
            {"Sharknado", 1}
        };
    }

    // 返回行数
    int rowCount(const QModelIndex &parent = QModelIndex()) const override {
        return m_data.size();
    }

    // 返回列数
    int columnCount(const QModelIndex &parent = QModelIndex()) const override {
        return 2; // Title, Rating
    }

    // 核心：提供数据
    QVariant data(const QModelIndex &index, int role = Qt::DisplayRole) const override {
        if (!index.isValid()) return QVariant();

        const auto &movie = m_data[index.row()];

        if (role == Qt::DisplayRole) {
            if (index.column() == 0) return movie.title;
            if (index.column() == 1) return movie.rating;
        }
        
        return QVariant();
    }

    // 表头
    QVariant headerData(int section, Qt::Orientation orientation, int role) const override {
        if (role == Qt::DisplayRole && orientation == Qt::Horizontal) {
            return section == 0 ? "Movie Title" : "Rating";
        }
        return QVariant();
    }

private:
    std::vector<Movie> m_data;
};
```

### 2. 自定义 Delegate (StarDelegate.h)

```cpp
#include <QStyledItemDelegate>
#include <QPainter>

class StarDelegate : public QStyledItemDelegate {
public:
    using QStyledItemDelegate::QStyledItemDelegate;

    void paint(QPainter *painter, const QStyleOptionViewItem &option, const QModelIndex &index) const override {
        if (index.column() == 1) { // 仅针对 Rating 列
            int rating = index.data().toInt();
            
            painter->save();
            
            // 绘制背景（处理选中状态）
            if (option.state & QStyle::State_Selected)
                painter->fillRect(option.rect, option.palette.highlight());

            // 绘制星星
            QString stars;
            for(int i=0; i<rating; ++i) stars += "★";
            
            painter->drawText(option.rect, Qt::AlignCenter, stars);
            
            painter->restore();
        } else {
            // 其他列使用默认渲染
            QStyledItemDelegate::paint(painter, option, index);
        }
    }
};
```

### 3. 组装 (main.cpp)

```cpp
#include <QApplication>
#include <QTableView>
#include "ReadOnlyTableModel.h"
#include "StarDelegate.h"

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QTableView view;
    MovieModel model;
    StarDelegate delegate;

    view.setModel(&model);
    // 为第 1 列（Rating）设置自定义代理
    view.setItemDelegateForColumn(1, &delegate);
    
    view.show();
    return app.exec();
}
```

## 最佳实践 (Best Practices)

1.  **Model 的选择**:
    *   对于静态或少量数据，可以使用 `QStandardItemModel`（基于 `QStandardItem` 对象，开销较大）。
    *   对于大数据量或自定义数据结构，**必须**继承 `QAbstractTableModel` / `QAbstractListModel` 并实现 `data()`，这样可以避免数据复制，实现“按需加载”。
2.  **避免在 `data()` 中执行耗时操作**: `data()` 会被 View 频繁调用（包括滚动、鼠标移动时），如果其中包含数据库查询或文件 IO，会导致界面卡顿。
3.  **使用 `Qt::UserRole`**: 如果需要传递非显示数据给 Delegate（例如 ID、状态标志），可以使用 `Qt::UserRole + n` 定义自定义 Role。
4.  **Delegate 的绘制**: 在 `paint()` 方法中要尽可能高效。避免在 `paint()` 中创建大量对象。

## 参考资料 (References)
- [Qt Documentation: Model/View Programming](https://doc.qt.io/qt-6/model-view-programming.html)
- [QAbstractItemModel Class](https://doc.qt.io/qt-6/qabstractitemmodel.html)
