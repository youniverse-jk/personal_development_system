# 人生存档 MVP 轻量架构设计

本文仅覆盖 `PRODUCT_INTENT.md` 中定义的第一阶段 MVP。第二阶段的使用情况监测、通知和 Galgame 对话，以及第三阶段的日程、衣柜、钱包和生活记录，不进入当前架构和数据模型。

## 1. 架构目标与边界

已确认技术栈：

- Kotlin
- Jetpack Compose
- Room / SQLite
- Android 系统图片选择器
- Android 系统文件创建与选择接口
- 应用私有文件目录

应用采用单 APK、单进程、单 Gradle `app` 模块的纯原生结构。MVP 没有网络服务，因此不存在 HTTP 意义上的前端和后端；本文中的 API 指 Compose/ViewModel 与本地数据层之间的应用内部接口，以及应用调用 Android 系统能力的接口。

不引入 React、WebView、Spring Boot、微服务、消息队列、云数据库、后台常驻服务或通用规则引擎。

整体依赖方向：

```text
Compose Screen
    ↓
ViewModel + StateFlow
    ↓
Repository / Manager
    ├── Room DAO
    ├── PrivateMediaStore
    └── Android Photo Picker / Storage Access Framework
```

约束：

- UI 不直接访问 DAO、数据库或文件目录。
- DAO 和文件实现不依赖 Compose。
- Room 是结构化业务数据的唯一来源。
- 用户图片的文件名由 Room 引用，文件内容保存在应用私有目录。
- 不为未来功能预建表、字段或抽象框架。

## 2. 核心业务模块

| 模块 | 职责 | 主要依赖 |
| --- | --- | --- |
| App Shell / Navigation | 应用启动、横屏约束、页面切换、依赖实例创建 | Compose、各功能 ViewModel |
| Home | 组合展示固定用户名、头像、等级、当前 EXP、三项状态、底图和入口 | PlayerRepository、MediaRepository |
| Tasks & Growth | 任务 CRUD、完成确认后的业务操作、累计 EXP 增加、等级计算 | TaskRepository、Room |
| Status | 读取和手动更新饱腹、SAN、活力 | PlayerRepository、Room |
| Personalization | 头像与底图导入、替换、默认资源回退、恢复默认底图 | MediaRepository、PrivateMediaStore、系统图片选择器 |
| Data Management | 数据导出、备份校验、覆盖恢复、删除全部个人数据 | BackupManager、ResetManager、Room、PrivateMediaStore、系统文件接口 |
| Local Persistence | 两张 Room 表、DAO、数据库事务和版本迁移 | Room |
| Private Media Storage | 用户图片的复制、读取、替换和清理 | 应用私有文件目录 |

依赖规则：

- Home 只组合其他模块暴露的只读状态，不拥有独立业务数据。
- Tasks & Growth 是同一模块，因为任务完成和 EXP 增加必须共同提交。
- Personalization 不直接修改 Player 表，通过 MediaRepository 协调文件和数据库引用。
- Data Management 可以协调多个数据来源，但具体 CRUD 仍由相应 DAO 或存储组件完成。
- “日程”“今日安排”“衣柜”“钱包”和静态对话区仅是 MVP 主界面的视觉元素，不创建业务模块、路由或数据表。

## 3. 最小数据模型

MVP 只需要两张 Room 表。

### 3.1 `player_state`

应用只有一个用户，因此该表始终只保留主键为 `1` 的一行。

| 字段 | 类型 | 默认值 | 约束与说明 |
| --- | --- | --- | --- |
| `id` | Int | 1 | 主键，只允许单例记录 |
| `total_exp` | Long | 0 | 不得小于 0；经验的唯一持久化来源 |
| `satiety` | Int | 80 | 0～100 |
| `san` | Int | 80 | 0～100 |
| `energy` | Int | 80 | 0～100 |
| `avatar_file_name` | String? | null | 私有目录内的相对文件名；null 表示通用占位头像 |
| `background_file_name` | String? | null | 私有目录内的相对文件名；null 表示内置默认底图 |

固定用户名“郑雅雯”是应用资源或常量，不写入数据库，也不进入编辑 API。

等级和当前经验是派生值，不单独持久化：

```text
level = 1 + totalExp / 100
currentExp = totalExp % 100
```

使用 `Long` 保存累计 EXP，避免长期使用后发生普通 `Int` 溢出。

### 3.2 `tasks`

| 字段 | 类型 | 默认值 | 约束与说明 |
| --- | --- | --- | --- |
| `id` | Long | 自动生成 | 主键 |
| `title` | String | 无 | 去除首尾空格后不能为空 |
| `exp_reward` | Int | 无 | 0～500，包含边界 |
| `is_completed` | Boolean | false | 完成后不得恢复为 false |
| `created_at` | Long | 当前时间 | Unix epoch milliseconds |
| `completed_at` | Long? | null | 完成时写入；未完成时必须为 null |

关键约束：

- 只有未完成任务可以编辑或删除。
- 已完成任务的名称、奖励、状态和完成时间全部只读。
- 完成任务只能从 `false` 单向变为 `true`。
- 完成状态变化与 `player_state.total_exp` 增加必须处于同一个 Room 事务。
- 数据库更新使用带条件的语句，例如只更新 `id` 匹配且 `is_completed = false` 的记录；受影响行数为 0 时不得增加 EXP。

### 3.3 实体关系

两张表不需要外键：

```text
player_state（固定一行）
    ↑
    │ 任务完成时，在事务中增加 total_exp
    │
tasks（多行，彼此独立）
```

任务奖励是完成时使用的业务输入，不表示数据库外键关系。MVP 不增加用户表、任务分类表、经验流水表、任务层级表或媒体表。

### 3.4 文件与备份模型

用户头像和自定义底图不是数据库实体。数据库只保存相对文件名，不能保存安装目录的绝对路径，因为重新安装或恢复后应用私有目录可能变化。

建议私有文件结构：

```text
files/
  media/
    avatar_<unique>.<ext>
    background_<unique>.<ext>
  restore_staging/
```

备份包采用带版本的 ZIP：

```text
backup.zip
  manifest.json
  data.json
  media/
    avatar.<ext>       # 存在自定义头像时
    background.<ext>   # 存在自定义底图时
```

`manifest.json` 至少包含 `formatVersion` 和导出时间；`data.json` 包含 `player_state` 与全部任务。MVP 不导出原始 SQLite 文件，以避免数据库内部版本与备份格式强耦合。

## 4. 应用内部 API

### 4.1 API 统一规则

- 持续读取使用 `Flow<T>` 或 `StateFlow<T>`，数据库变化后 UI 自动更新。
- 写操作使用挂起函数并返回明确的成功值或结构化错误。
- Repository 不返回 UI 文案，只返回错误类型；ViewModel 将错误映射为中文提示。
- 用户在系统选择器中取消选择属于正常取消，不记录为业务错误。
- 简单 CRUD 不额外建立空洞的 UseCase 接口；有事务边界的完成任务、恢复和重置操作由专门方法封装。

建议统一结果形式：

```text
AppResult<T>
  Success<T>
  Failure(AppError)
```

主要错误类型：

| 错误 | 含义 |
| --- | --- |
| `ValidationError` | 名称为空、EXP 或状态值越界 |
| `NotFound` | 指定任务不存在 |
| `TaskAlreadyCompleted` | 对已完成任务执行完成、编辑或删除 |
| `MediaReadFailed` | 无法读取用户选择的图片 |
| `MediaWriteFailed` | 图片复制或保存失败 |
| `BackupInvalid` | 备份结构、数据或必要文件不完整 |
| `BackupVersionUnsupported` | 备份格式版本不受支持 |
| `StorageUnavailable` | 目标文件无法打开、空间不足或写入失败 |
| `RestoreFailed` | 校验通过后应用恢复失败，原数据保持不变 |
| `PersistenceError` | 其他数据库读写失败 |

### 4.2 Player / Home API

| API | 请求 | 响应 | 主要错误 |
| --- | --- | --- | --- |
| `observePlayerState()` | 无 | `Flow<PlayerState>` | 数据库读取失败 |
| `observeHomeSnapshot()` | 无 | `Flow<HomeSnapshot>`，包含固定用户名、派生等级/EXP、状态值及媒体引用 | 数据库或媒体引用读取失败；缺图时回退默认资源 |
| `updateVitals(request)` | `satiety`、`san`、`energy`，均为 0～100 | 更新后的 `PlayerState` | `ValidationError`、`PersistenceError` |

`HomeSnapshot` 是界面读取模型，不是 Room 实体。等级和当前 EXP 在映射时由 `totalExp` 计算。

### 4.3 Task API

| API | 请求 | 响应 | 主要错误 |
| --- | --- | --- | --- |
| `observeTasks()` | 无 | `Flow<List<Task>>`，包含未完成和已完成任务 | 数据库读取失败 |
| `createTask(request)` | `title`、`expReward` | 新建的 `Task` | `ValidationError`、`PersistenceError` |
| `updateTask(request)` | `id`、`title`、`expReward` | 更新后的 `Task` | `ValidationError`、`NotFound`、`TaskAlreadyCompleted` |
| `deleteTask(taskId)` | 任务 ID | 成功时无数据 | `NotFound`、`TaskAlreadyCompleted`、`PersistenceError` |
| `completeTask(taskId)` | 任务 ID；UI 已完成二次确认 | `CompletionResult` | `NotFound`、`TaskAlreadyCompleted`、`PersistenceError` |

`CompletionResult` 至少包含：

- `taskId`
- `rewardGranted`
- `totalExp`
- `level`
- `currentExp`
- `levelsGained`

### 4.4 Personalization API

图片 URI 来自 Android 系统图片选择器。Repository 必须在方法执行期间完成读取和私有副本创建，不能把临时 URI 作为长期数据保存。

| API | 请求 | 响应 | 主要错误 |
| --- | --- | --- | --- |
| `replaceAvatar(sourceUri)` | 系统选择器返回的 URI | 新的头像媒体引用 | `MediaReadFailed`、`MediaWriteFailed`、`PersistenceError` |
| `replaceBackground(sourceUri)` | 系统选择器返回的 URI | 新的底图媒体引用 | 同上 |
| `restoreDefaultBackground()` | 无 | `background_file_name = null` 后的状态 | `PersistenceError`；旧文件删除失败只记录并稍后清理 |

图片选择器取消时 ViewModel 不调用替换 API。

### 4.5 Backup / Restore / Reset API

覆盖恢复和删除全部数据的二次确认由 Compose UI 完成；只有用户确认后才调用实际写操作。Manager 不接受一个容易被误传的 `confirmed = true` 参数。

| API | 请求 | 响应 | 主要错误 |
| --- | --- | --- | --- |
| `exportBackup(destinationUri)` | 系统文件创建器返回的目标 URI | `BackupSummary`，包含导出时间和任务数量 | `StorageUnavailable`、`MediaReadFailed`、`PersistenceError` |
| `inspectBackup(sourceUri)` | 系统文件选择器返回的 URI | `BackupInspection`，包含版本、数据摘要和是否可恢复 | `BackupInvalid`、`BackupVersionUnsupported`、`StorageUnavailable` |
| `restoreBackup(sourceUri)` | 已检查且用户已确认的备份 URI | 恢复后的 `PlayerState` 和任务数量 | `BackupInvalid`、`BackupVersionUnsupported`、`RestoreFailed` |
| `resetAllData()` | 无；UI 已完成二次确认 | 默认 `PlayerState` | `PersistenceError` |

恢复必须重新验证输入，不能只依赖此前 `inspectBackup` 的结果，因为文件在两次调用之间可能发生变化。

## 5. 跨模块一致性

### 5.1 完成任务并增加 EXP

这是 MVP 最重要的数据库事务：

1. 在 Room 事务中读取任务。
2. 若任务不存在，返回 `NotFound`。
3. 若任务已完成，返回 `TaskAlreadyCompleted`，不增加 EXP。
4. 使用条件更新将任务标记完成并写入 `completed_at`。
5. 只有条件更新影响一行时，才给 `player_state.total_exp` 增加奖励。
6. 提交事务后返回新的等级和当前 EXP。

UI 防连点只能改善体验，不能替代事务和条件更新。

### 5.2 替换头像或底图

Room 与文件系统不能共享同一个事务，因此采用“先准备、后引用、最后清理”：

1. 将源图片复制为私有目录中的新唯一文件。
2. 确认新文件可以读取。
3. 更新 Room 中的相对文件名。
4. 数据库提交成功后删除旧文件。
5. 数据库更新失败时删除新文件并保留旧引用。

如果应用在第 3、4 步之间退出，最多留下无引用文件，不会让数据库引用一个尚未写入的文件。无引用文件可在下次启动或下次替换时安全清理。

### 5.3 导出备份

1. 在 Room 只读事务中取得一致的 `player_state` 和任务快照。
2. 根据快照中的媒体文件名读取对应文件。
3. 写入临时备份内容和 manifest。
4. ZIP 完整生成后再写入用户选择的目标 URI。
5. 任何步骤失败都返回错误，不修改应用数据。

导出期间应避免同时替换头像或底图。可在 Data Management 和 Personalization 共享一个简单的协程 `Mutex`，不需要事件总线或分布式锁。

### 5.4 覆盖恢复

1. 将备份解压到私有临时目录。
2. 校验格式版本、JSON 结构、字段范围、任务完成状态与图片完整性。
3. 把新图片复制为私有目录中的新唯一文件，但暂不删除旧图片。
4. 在单个 Room 事务中清空并写入新的 `player_state` 和任务。
5. 事务成功后删除旧用户图片和临时目录。
6. 数据库事务失败时删除新图片并保留旧数据与旧图片。

恢复后媒体路径根据新私有文件名重建，不能沿用备份设备中的绝对路径。

### 5.5 删除全部个人数据

1. 在单个 Room 事务中删除任务，并将 `player_state` 恢复为默认值。
2. 事务成功后删除所有用户头像和自定义底图。
3. 文件删除失败时数据库仍保持默认引用；残留文件没有业务引用，可稍后再次清理。

不能先删除图片再重置数据库，否则数据库失败时会留下指向已删除图片的记录。

## 6. 建议项目目录

```text
app/
  src/main/
    AndroidManifest.xml
    java/<package>/
      MainActivity.kt
      LifeArchiveApp.kt

      app/
        AppContainer.kt
        AppNavigation.kt
        AppRoutes.kt

      core/
        model/
          PlayerState.kt
          Task.kt
          AppResult.kt
          AppError.kt
        growth/
          GrowthCalculator.kt
        time/
          TimeProvider.kt

      data/
        local/
          AppDatabase.kt
          entity/
            PlayerStateEntity.kt
            TaskEntity.kt
          dao/
            PlayerStateDao.kt
            TaskDao.kt
        repository/
          PlayerRepository.kt
          TaskRepository.kt
          MediaRepository.kt
        media/
          PrivateMediaStore.kt
          ImageImporter.kt
        backup/
          BackupManager.kt
          BackupManifest.kt
          BackupValidator.kt
        reset/
          ResetManager.kt

      feature/
        home/
          HomeScreen.kt
          HomeViewModel.kt
          HomeUiState.kt
        tasks/
          TaskScreen.kt
          TaskViewModel.kt
          TaskUiState.kt
          TaskEditorDialog.kt
          CompleteTaskDialog.kt
        status/
          StatusScreen.kt
          StatusViewModel.kt
          StatusUiState.kt
        personalization/
          PersonalizationViewModel.kt
        dataManagement/
          DataManagementScreen.kt
          DataManagementViewModel.kt
          DataManagementUiState.kt

      ui/
        components/
        theme/

    res/
      drawable-nodpi/
        default_background.*
      drawable/
        avatar_placeholder.*
      values/
        strings.xml
        colors.xml
        themes.xml

  src/test/
    ...
  src/androidTest/
    ...
```

说明：

- 仍然只有一个 Gradle `app` 模块；目录只是代码组织，不是独立构建模块。
- Repository 可先使用具体类，不必为每个类创建只有一个实现的接口。
- `GrowthCalculator` 是纯函数，便于验证连续升级。
- `TimeProvider` 仅用于让任务时间相关测试可控，不引入复杂时钟框架。

## 7. 可能导致大规模返工的问题

### 7.1 把等级、当前 EXP 和累计 EXP 分别保存

三者会产生不一致。必须只保存 `total_exp`，另外两个始终派生。

### 7.2 只在 UI 层阻止重复完成任务

快速点击、协程并发或进程恢复仍可能重复奖励。不可变规则必须落实在 Repository 的 Room 事务和条件更新中。

### 7.3 保存相册绝对路径或临时 URI

原图移动、删除、权限失效或重新安装后都会失效。必须立即复制到私有目录，并只存相对文件名。

### 7.4 直接备份原始 SQLite 文件

会让备份格式与 Room 内部 schema 紧密耦合，后续迁移困难。MVP 采用版本化的结构化数据与媒体压缩包。

### 7.5 恢复时边读边覆盖正式数据

损坏备份会造成半恢复状态。必须先暂存和完整校验，再替换正式数据。

### 7.6 为未来任务层级预建复杂模型

长期目标、章节、前置依赖目前没有确认。MVP 只保留单层任务，未来通过 Room migration 增加新表或字段。

### 7.7 默认素材仍引用项目外绝对路径

`PRODUCT_INTENT.md` 中的参考图片位于用户 Downloads 目录，不能成为构建依赖。正式实现前必须把最终可用的默认底图和占位头像复制到项目资源目录，并确认素材使用权。

### 7.8 任意底图破坏 UI 可读性

用户图片不可控。主界面信息面板需要稳定的半透明底色、对比度和系统安全区域处理；MVP 不需要自动取色或图像分析。

### 7.9 Android 系统自动云备份违背本地优先

应用应关闭 Android 自动云备份或通过数据提取规则明确排除数据库和用户图片。用户主动通过系统文件创建器导出备份不属于应用自动上传。

### 7.10 数据管理入口尚未在主界面中定义

MVP 已要求头像、底图、导出、恢复和删除数据，但 Product Intent 尚未明确这些操作从哪个 UI 入口进入。这不影响数据库架构，但在制作可点击原型前需要在 UI 线框中确定，避免页面完成后重新调整导航。

## 8. MVP 架构验收要点

- 工程只包含原生 Kotlin/Compose/Room 实现，不出现 WebView 或网络后端。
- Room 只有 MVP 必需的 `player_state` 与 `tasks` 两张业务表。
- 任务完成与累计 EXP 增加通过一次事务完成，并能抵抗重复调用。
- 头像和底图复制到应用私有目录，数据库不保存外部绝对路径或临时 URI。
- 备份可以完整包含结构化数据与用户图片，损坏备份不会破坏现有数据。
- 覆盖恢复和重置操作失败时不会产生业务可见的半完成状态。
- 默认素材进入项目资源目录，不依赖开发机器 Downloads 路径。
- 应用不自动把数据库或用户图片上传到云端。
