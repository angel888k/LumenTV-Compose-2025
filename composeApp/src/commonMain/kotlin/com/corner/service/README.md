# Service层架构说明

## 概述

Service层是从ViewModel中提取的业务逻辑层，负责处理纯业务逻辑，与UI状态管理解耦。

## 目录结构

```
composeApp/src/commonMain/kotlin/com/corner/service/
├── history/
│   ├── HistoryService.kt           # 历史记录服务接口
│   └── HistoryServiceImpl.kt       # 历史记录服务实现
├── episode/
│   ├── EpisodeManager.kt           # 剧集管理服务接口
│   └── EpisodeManagerImpl.kt       # 剧集管理服务实现
├── player/
│   ├── PlayerStrategy.kt           # 播放器策略接口
│   ├── PlayerStrategyFactory.kt    # 策略工厂
│   ├── InniePlayerStrategy.kt      # 内部播放器策略（VLCJ）
│   ├── ExternalPlayerStrategy.kt   # 外部播放器策略
│   └── WebPlayerStrategy.kt        # Web播放器策略
└── di/
    └── ServiceModule.kt            # 依赖注入模块
```

## 设计原则

### 1. 单一职责原则 (SRP)
每个Service只负责一个明确的业务领域：
- `HistoryService`: 历史记录管理
- `EpisodeManager`: 剧集选择和导航
- `PlayerStrategy`: 播放器策略模式（多态播放）

### 2. 依赖倒置原则 (DIP)
- ViewModel依赖抽象（接口），不依赖具体实现
- 通过构造函数注入Service实例
- 播放器策略通过工厂模式创建

### 3. 开闭原则 (OCP)
- 对扩展开放：可以轻松添加新的Service或播放器策略
- 对修改关闭：修改Service实现不影响ViewModel

### 4. 策略模式 (Strategy Pattern)
- 定义统一的`PlayerStrategy`接口
- 三种实现：InniePlayer（VLCJ）、ExternalPlayer、WebPlayer
- 通过`PlayerStrategyFactory`根据配置动态选择策略
- 消除ViewModel中的条件分支，提高可维护性

## 使用方式

### 在ViewModel中使用Service

```kotlin
class DetailViewModel(
    private val historyService: HistoryService,
    private val episodeManager: EpisodeManager
) : BaseViewModel() {
    
    // 使用HistoryService
    suspend fun loadDetail(vod: Vod) {
        val episode = historyService.handlePlaybackHistory(vod, currentEpisodeIndex)
        // ...
    }
    
    // 使用EpisodeManager
    fun nextEpisode() {
        episodeManager.nextEpisode(detail, currentEp) { vod, ep ->
            startPlay(vod, ep)
        }
    }
}
```

### 创建ViewModel实例

```kotlin
// 使用ServiceModule创建Service实例
val historyService = ServiceModule.provideHistoryService { controller }
val episodeManager = ServiceModule.provideEpisodeManager()

// 创建ViewModel
val viewModel = DetailViewModel(historyService, episodeManager)
```

## 播放器策略模式

### 架构设计

```kotlin
// 1. 定义策略接口
interface PlayerStrategy {
    suspend fun play(
        result: Result,
        episode: Episode,
        onPlayStarted: () -> Unit,
        onError: (String) -> Unit
    )
    fun getStrategyName(): String
}

// 2. 实现具体策略
class InniePlayerStrategy(...) : PlayerStrategy { ... }
class ExternalPlayerStrategy(...) : PlayerStrategy { ... }
class WebPlayerStrategy(...) : PlayerStrategy { ... }

// 3. 工厂模式创建策略
object PlayerStrategyFactory {
    fun createStrategy(
        playerType: Int,
        controller: VlcjFrameController,
        lifecycleManager: PlayerLifecycleManager,
        viewModelScope: CoroutineScope
    ): PlayerStrategy {
        return when (playerType) {
            PlayerType.Innie.id -> InniePlayerStrategy(...)
            PlayerType.External.id -> ExternalPlayerStrategy(...)
            else -> WebPlayerStrategy(...)
        }
    }
}

// 4. ViewModel中使用
val strategy = PlayerStrategyFactory.createStrategy(...)
strategy.play(result, episode, 
    onPlayStarted = { /* UI更新 */ },
    onError = { /* 错误处理 */ }
)
```

### 优势

1. **消除条件分支**：ViewModel中不再有`when/switch`判断播放器类型
2. **易于扩展**：添加新播放器只需实现`PlayerStrategy`接口
3. **职责清晰**：每种播放器的逻辑封装在独立的策略类中
4. **便于测试**：可以单独测试每种策略，也可以Mock策略进行测试

### InniePlayerStrategy 实现细节

内部播放器（VLCJ）的播放流程：

```kotlin
override suspend fun play(result, episode, onPlayStarted, onError) {
    // 1. 加载媒体URL
    loadMediaUrl(result, onError)
    
    // 2. 准备播放器状态
    prepareForPlayback(result, onError)
    
    // 3. 启动播放并等待真正开始
    waitForPlaybackToStart(onPlayStarted, onError)
}
```

**关键步骤**：
1. `controller.loadURL(url, timeout)` - 加载并准备媒体
2. `lifecycleManager.ready()` - 确保播放器就绪
3. `lifecycleManager.start()` - 调用`controller.play()`开始播放
4. 监听`controller.state`流，等待PLAY状态
5. 触发`onPlayStarted`回调，更新UI

**异常处理**：
- 使用自定义异常`PlayStartedException`跳出Flow.collect
- 超时保护（30秒），提供详细诊断信息
- 错误时自动清理资源

## 实施计划

### Phase 1: 基础准备 ✅ (已完成)
- [x] 创建Service包结构
- [x] 定义HistoryService接口
- [x] 定义EpisodeManager接口
- [x] 建立DI机制
- [x] 搭建测试框架

### Phase 2: HistoryService提取 ✅ (已完成)
- [x] 实现HistoryServiceImpl
- [x] 编写单元测试
- [x] 集成到DetailViewModel
- [x] 删除ViewModel中的旧代码
- [x] 回归测试

### Phase 3: EpisodeManager提取 ✅ (已完成)
- [x] 实现EpisodeManagerImpl
- [x] 编写单元测试
- [x] 集成到DetailViewModel
- [x] 删除ViewModel中的旧代码
- [x] 回归测试

### Phase 4: 播放器策略模式重构 ✅ (已完成)
- [x] 定义PlayerStrategy接口
- [x] 实现InniePlayerStrategy（内部播放器）
- [x] 实现ExternalPlayerStrategy（外部播放器）
- [x] 实现WebPlayerStrategy（Web播放器）
- [x] 创建PlayerStrategyFactory工厂类
- [x] 集成到DetailViewModel
- [x] 修复播放器加载问题（loadURL + play）
- [x] 优化日志输出
- [x] 回归测试

## 迁移指南

### 从ViewModel迁移逻辑到Service的步骤

1. **识别待迁移的方法**
   - 查看TODO注释定位需要迁移的代码
   - 确认方法的输入输出

2. **复制逻辑到Service实现**
   - 将ViewModel中的方法复制到ServiceImpl
   - 调整依赖项（通过构造函数注入）

3. **修改ViewModel调用**
   - 将直接调用改为调用Service方法
   - 传递必要的参数

4. **测试验证**
   - 运行单元测试
   - 手动测试功能
   - 确保行为一致

5. **删除旧代码**
   - 确认新功能正常后
   - 删除ViewModel中的旧方法

### 示例：迁移updateHistory方法

**迁移前（ViewModel中）：**
```kotlin
fun updateHistory(it: History) {
    if (StringUtils.isNotBlank(state.value.detail.site?.key)) {
        scope.launch {
            try {
                Db.History.update(it.copy(createTime = Clock.System.now().toEpochMilliseconds()))
            } catch (e: Exception) {
                log.error("历史记录更新失败", e)
            }
        }
    }
}
```

**迁移后（Service中）：**
```kotlin
override suspend fun updateHistory(history: History) {
    if (StringUtils.isNotBlank(history.key)) {
        try {
            Db.History.update(history.copy(createTime = Clock.System.now().toEpochMilliseconds()))
        } catch (e: Exception) {
            log.error("历史记录更新失败", e)
        }
    }
}
```

**ViewModel调用：**
```kotlin
fun updateHistory(history: History) {
    scope.launch {
        historyService.updateHistory(history)
    }
}
```

## 最佳实践

### 1. Service应该做什么
- ✅ 纯业务逻辑
- ✅ 数据持久化操作
- ✅ 数据处理和转换
- ✅ 调用Repository/DAO

### 2. Service不应该做什么
- ❌ 管理UI状态（StateFlow）
- ❌ 直接操作用户界面
- ❌ 包含Compose相关代码
- ❌ 处理用户交互事件

### 3. 命名规范
- 接口：`XxxService` 或 `XxxManager`
- 实现类：`XxxServiceImpl` 或 `XxxManagerImpl`
- 方法名：动词开头，清晰表达意图

### 4. 错误处理
- Service层抛出异常，由ViewModel捕获和处理
- 记录详细的错误日志
- 不要直接在Service中显示UI提示

## 未来扩展

### 可能的Service扩展
- `SearchService`: 搜索逻辑
- `SiteService`: 站点管理逻辑
- `CacheService`: 缓存管理逻辑

### 播放器策略扩展
- `DLNAPlayerStrategy`: DLNA投屏播放器
- `CustomPlayerStrategy`: 自定义第三方播放器
- 支持更多视频格式和协议

### DI框架迁移
当前使用简单的手动DI，后续可以迁移到：
- **Koin**: 轻量级，适合Kotlin Multiplatform
- **Kodein**: 另一个流行的Kotlin DI框架

迁移示例（Koin）：
```kotlin
val serviceModule = module {
    single<HistoryService> { 
        HistoryServiceImpl(get()) 
    }
    single<EpisodeManager> { 
        EpisodeManagerImpl() 
    }
    factory<PlayerStrategy> { (playerType: Int) ->
        PlayerStrategyFactory.createStrategy(playerType, get(), get(), get())
    }
}
```

## 参考资料

- [DetailViewModel_逻辑提取可行性分析.md](../ui/nav/vm/DetailViewModel_逻辑提取可行性分析.md)
