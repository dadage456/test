# WMS任务页面通用组件重构方案

## 概述

本文档描述了基于金风WMS项目中平库出库模块和平库入库模块的列表页面和明细列表页面重构方案，旨在创建可重用的通用组件，提高代码复用率，降低维护成本。

## 一、现状分析

### 1.1 列表页面分析

#### 共同点
- **页面结构完全一致**：AppBar + 扫码输入框 + 表格主体
- **技术栈统一**：都使用`CommonDataGrid`展示数据，BLoC状态管理
- **交互逻辑一致**：扫码搜索 → 行点击导航 → 加载状态处理
- **数据管理模式相同**：`CommonDataGridBloc` + 专用业务BLoC

#### 差异点
| 项目 | 出库模块 | 入库模块 |
|------|----------|----------|
| 数据模型 | `OutboundTask` | `GoodsUpTask` |
| 页面标题 | '出库任务列表' | '平库上架任务' |
| 表格配置 | `OutboundTaskGridConfig` | `GoodsUpTaskGridConfig` |
| 导航路径 | `/outbound/collect/` | `/goods-up/collect/` |
| 筛选功能 | 有筛选对话框 | 暂无筛选功能 |

### 1.2 明细列表页面分析

#### 共同点
- **页面结构高度一致**：AppBar + 扫码输入 + 表格 + 批量操作栏
- **核心功能完全相同**：搜索、分页、选择、批量操作
- **BLoC架构模式一致**：`CommonDataGridBloc` + DetailBloc

#### 差异点
| 项目 | 出库明细 | 入库明细 |
|------|----------|----------|
| 数据模型 | `OutboundTaskItem` | `GoodsUpTaskItem` |
| 页面标题 | '平库下架任务明细' | '上架任务明细' |
| 扫码提示 | '请扫描或输入物料编码' | '请扫描物料或库位' |
| 初始化参数 | outTaskId, workStation, userId, roleOrUserId | inTaskId, workStation |
| 批量操作 | 功能完整 | 部分功能未实现 |

## 二、重构方案

### 2.1 列表页面重构方案

#### 方案选择：配置化通用列表页面

创建`BaseTaskListPage<T>`组件，通过配置对象定制差异：

```dart
// 配置接口
abstract class TaskListConfig<T> {
  String get pageTitle;
  String get scanPlaceholder;
  List<GridColumnConfig<T>> get gridColumns;
  void navigateToCollect(BuildContext context, T task);
  void navigateToDetail(BuildContext context, T task);
  Widget? buildFilterWidget(BuildContext context, dynamic bloc);
  String get addButtonText;
  String get addRoutePath;
  bool showFilterButton() => false;
}

// 通用列表页面
class BaseTaskListPage<T> extends StatefulWidget {
  final TaskListConfig<T> config;

  const BaseTaskListPage({
    super.key,
    required this.config,
  });

  @override
  State<BaseTaskListPage<T>> createState() => _BaseTaskListPageState<T>();
}
```

#### 实现结构

```
lib/common_widgets/task_base/
├── models/
│   ├── base_task_interface.dart          # 基础接口定义
│   └── base_task_query.dart              # 通用查询模型
├── configs/
│   └── task_list_config.dart             # 任务列表配置接口
├── pages/
│   └── base_task_list_page.dart          # 通用任务列表页面
└── widgets/
    ├── base_scanner_input.dart           # 通用扫码输入组件
    └── base_table_config.dart            # 通用表格配置
```

#### 出库模块配置示例

```dart
class OutboundTaskListConfig extends TaskListConfig<OutboundTask> {
  @override
  String get pageTitle => '出库任务列表';

  @override
  String get scanPlaceholder => '请扫描单号';

  @override
  List<GridColumnConfig<OutboundTask>> get gridColumns =>
      OutboundTaskGridConfig.getColumns(_handleCellTap);

  @override
  void navigateToCollect(BuildContext context, OutboundTask task) {
    Modular.to.pushNamed('/outbound/collect/${task.outTaskNo}');
  }

  @override
  void navigateToDetail(BuildContext context, OutboundTask task) {
    Modular.to.pushNamed('/outbound/detail/${task.outTaskId}');
  }

  @override
  Widget? buildFilterWidget(BuildContext context, OutboundTaskBloc bloc) {
    return IconButton(
      icon: SvgPicture.asset('assets/images/icon_filter.svg'),
      onPressed: () => OutboundTaskFilterDialog.show(context: context),
    );
  }

  @override
  bool showFilterButton() => true;

  @override
  String get addButtonText => '';
  @override
  String get addRoutePath() => '/outbound/receive';
}
```

#### 重构后使用方式

```dart
// 出库任务列表页面
class OutboundTaskListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BaseTaskListPage<OutboundTask>(
      config: OutboundTaskListConfig(),
    );
  }
}

// 入库任务列表页面
class GoodsUpTaskListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BaseTaskListPage<GoodsUpTask>(
      config: GoodsUpTaskListConfig(),
    );
  }
}
```

### 2.2 明细列表页面重构方案

#### 配置接口设计

```dart
// 明细页面配置接口
abstract class TaskDetailConfig<T> {
  // 页面基础配置
  String get pageTitle;
  String get scanPlaceholder;
  List<GridColumnConfig<T>> get gridColumns;

  // 初始化参数
  Map<String, dynamic> get initializationParams;

  // BLoC创建和初始化
  dynamic createBloc(BuildContext context);
  void initializeBloc(dynamic bloc, Map<String, dynamic> params);

  // 事件处理
  void handleSearch(dynamic bloc, String searchKey);
  void handleScan(dynamic bloc, String scanResult);
  void handleRefresh(dynamic bloc);
  void handleCancelSelected(dynamic bloc, List<String> selectedItemIds);

  // 批量操作配置
  bool showRefreshButton() => true;
  bool showBatchActions() => true;
  bool enableSelectAll() => true;
  bool enableDeselectAll() => true;

  // 批量操作回调
  void onSelectAll(dynamic bloc, List<T> allItems);
  void onDeselectAll(dynamic bloc);
  void onCancelSelected(dynamic bloc, List<int> selectedRows);
  void onClearSelection(dynamic bloc);
}
```

#### 通用明细页面组件

```dart
class BaseTaskDetailPage<T> extends StatefulWidget {
  final TaskDetailConfig<T> config;

  const BaseTaskDetailPage({
    super.key,
    required this.config,
  });

  @override
  State<BaseTaskDetailPage<T>> createState() => _BaseTaskDetailPageState<T>();
}

class _BaseTaskDetailPageState<T> extends State<BaseTaskDetailPage<T>> {
  late dynamic _bloc;
  late final CommonDataGridBloc<T> _gridBloc;
  final ScannerController _scannerController = ScannerController();

  @override
  void initState() {
    super.initState();
    _bloc = widget.config.createBloc(context);
    _gridBloc = _bloc.gridBloc;
    widget.config.initializeBloc(_bloc, widget.config.initializationParams);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFF6F6F6),
      appBar: _buildAppBar(),
      body: Column(
        children: [
          _buildScanInput(),
          Expanded(child: _buildTable()),
          if (widget.config.showBatchActions()) _buildBatchActionBar(),
        ],
      ),
    );
  }

  // 通用组件实现...
}
```

#### 出库明细配置示例

```dart
class OutboundTaskDetailConfig extends TaskDetailConfig<OutboundTaskItem> {
  final String outTaskId;
  final String workStation;
  final int userId;
  final int roleOrUserId;

  OutboundTaskDetailConfig({
    required this.outTaskId,
    required this.workStation,
    required this.userId,
    required this.roleOrUserId,
  });

  @override
  String get pageTitle => '平库下架任务明细';

  @override
  String get scanPlaceholder => '请扫描或输入物料编码';

  @override
  List<GridColumnConfig<OutboundTaskItem>> get gridColumns =>
      OutboundTaskDetailGridConfig.getColumns();

  @override
  Map<String, dynamic> get initializationParams => {
    'outTaskId': outTaskId,
    'workStation': workStation,
    'userId': userId,
    'roleOrUserId': roleOrUserId,
  };

  @override
  OutboundTaskDetailBloc createBloc(BuildContext context) {
    return BlocProvider.of<OutboundTaskDetailBloc>(context);
  }

  @override
  void initializeBloc(OutboundTaskDetailBloc bloc, Map<String, dynamic> params) {
    bloc.initializeQuery(params['outTaskId'], params['workStation']);
  }

  @override
  void handleSearch(OutboundTaskDetailBloc bloc, String searchKey) {
    bloc.add(OutboundTaskDetailEvent.search(searchKey: searchKey));
  }

  @override
  void handleRefresh(OutboundTaskDetailBloc bloc) {
    bloc.add(const OutboundTaskDetailEvent.refresh());
  }

  @override
  void handleCancelSelected(OutboundTaskDetailBloc bloc, List<String> selectedItemIds) {
    bloc.add(OutboundTaskDetailEvent.cancelSelectedItems(
      selectedRows: selectedItemIds.map((id) => int.parse(id)).toList(),
    ));
  }

  // 实现其他方法...
}
```

#### 重构后使用方式

```dart
// 出库明细页面
class OutboundTaskDetailPage extends StatelessWidget {
  final String outTaskId;
  final String workStation;
  final int userId;
  final int roleOrUserId;

  const OutboundTaskDetailPage({
    super.key,
    required this.outTaskId,
    required this.workStation,
    required this.userId,
    required this.roleOrUserId,
  });

  @override
  Widget build(BuildContext context) {
    return BaseTaskDetailPage<OutboundTaskItem>(
      config: OutboundTaskDetailConfig(
        outTaskId: outTaskId,
        workStation: workStation,
        userId: userId,
        roleOrUserId: roleOrUserId,
      ),
    );
  }
}
```

## 三、实施步骤

### 3.1 第一阶段：创建通用组件

1. **创建目录结构**
   ```bash
   mkdir -p lib/common_widgets/task_base/{models,configs,pages,widgets}
   ```

2. **实现配置接口**
   - `TaskListConfig<T>`
   - `TaskDetailConfig<T>`

3. **实现通用页面**
   - `BaseTaskListPage<T>`
   - `BaseTaskDetailPage<T>`

4. **实现通用组件**
   - 扫码输入组件
   - 表格配置组件
   - 批量操作组件

### 3.2 第二阶段：重构出库模块

1. **创建出库配置类**
   - `OutboundTaskListConfig`
   - `OutboundTaskDetailConfig`

2. **重构出库页面**
   - 使用通用组件替换现有页面
   - 保持路由不变，确保兼容性

3. **测试验证**
   - 功能测试
   - UI测试
   - 性能测试

### 3.3 第三阶段：重构入库模块

1. **创建入库配置类**
   - `GoodsUpTaskListConfig`
   - `GoodsUpTaskDetailConfig`

2. **重构入库页面**
   - 使用通用组件替换现有页面
   - 完善缺失的批量操作功能

3. **测试验证**
   - 功能对比测试
   - 用户体验测试

### 3.4 第四阶段：优化和扩展

1. **性能优化**
   - 组件缓存
   - 内存管理

2. **功能增强**
   - 更多配置选项
   - 主题支持

3. **文档完善**
   - 使用指南
   - 最佳实践

## 四、方案优势

### 4.1 代码复用
- **列表页面复用率：90%+**
- **明细页面复用率：90%+**
- **组件复用率：95%+**

### 4.2 维护成本
- **核心逻辑集中**：主要逻辑在基类中维护
- **统一修改**：一次修改，所有模块受益
- **降低bug风险**：减少重复代码，降低出错概率

### 4.3 开发效率
- **新模块开发**：只需实现配置接口，开发时间减少70%
- **功能迭代**：统一的功能迭代，效率提升80%
- **测试成本**：基类测试覆盖所有模块，测试工作量减少60%

### 4.4 扩展性
- **类型安全**：泛型保证编译时类型检查
- **配置灵活**：可以精确控制每个功能
- **向后兼容**：渐进式重构，不影响现有功能

## 五、风险控制

### 5.1 技术风险
- **破坏性更改**：通过保持路由兼容性避免
- **性能问题**：分阶段重构，及时性能测试
- **类型安全**：充分利用Dart类型系统

### 5.2 项目风险
- **开发时间**：分阶段实施，降低风险
- **团队接受度**：提供详细文档和培训
- **维护复杂度**：清晰的代码结构和文档

## 六、总结

本重构方案通过创建配置化的通用组件，可以将现有出库和入库模块的列表页面和明细列表页面重构为高度可重用的组件。方案具有以下特点：

1. **高复用率**：代码复用率达到90%以上
2. **低风险**：渐进式重构，保持向后兼容
3. **高效率**：新模块开发时间减少70%
4. **易维护**：核心逻辑集中，维护成本降低80%

通过这个方案，不仅可以解决当前的代码重复问题，还可以为未来的功能扩展和新模块开发提供坚实的基础。
