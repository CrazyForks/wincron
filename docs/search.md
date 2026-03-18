# Search 功能优化评估

## 当前实现现状

### 任务搜索

- 入口在 `frontend/components/SearchPalette.vue`。
- 任务搜索直接读取 Pinia 中的 `jobs`，在前端本地做字段拼接、打分、排序和截断，结果上限为 8 条。
- 查询防抖为 180ms，匹配字段主要是任务名、命令、目录、参数。

对应实现：

- `frontend/components/SearchPalette.vue:10-11`
- `frontend/components/SearchPalette.vue:142-164`
- `frontend/components/SearchPalette.vue:193-199`

### 日志搜索

- 日志搜索不是全量历史检索，而是在搜索面板打开或切到日志模式时，直接调用 `ListLogs("", 100)` 拉取最近 100 条日志。
- 拉回前端后，再对 `jobName / commandLine / error / stdout / stderr` 做本地打分、排序和截断，结果上限同样是 8 条。
- 搜索面板内部维护自己的 `allLogs`，没有复用 store 的日志加载与缓存逻辑。

对应实现：

- `frontend/components/SearchPalette.vue:123-135`
- `frontend/components/SearchPalette.vue:166-190`
- `frontend/components/SearchPalette.vue:201-207`

### 与主日志面板的数据关系

- 主日志面板走的是 store 的 `loadLogs(jobId)`，默认也是 100 条，并且在 `jobExecuted` 事件后会刷新当前聚焦日志。
- 搜索面板没有接入这条刷新链路，而是维护一份独立日志快照。

对应实现：

- `frontend/stores/cron/logs.js:32-69`
- `frontend/stores/cron/lifecycle.js:14-36`

## 性能优化点

### 1. 日志搜索仍然是“传大对象到前端后再扫一遍”

当前日志搜索会把最近 100 条日志的完整内容拉到前端，本地再做 `toLowerCase / includes / sort`。而每条日志的 `stdout` 和 `stderr` 在写入前最多各保留 16KB，这意味着一次搜索理论上会搬运数 MB 文本，再在 UI 线程上重复扫描。

对应实现：

- `frontend/components/SearchPalette.vue:64-90`
- `frontend/components/SearchPalette.vue:123-135`
- `cron_service.go:1035-1048`

优化建议：

- 增加专用搜索接口，例如 `SearchLogs(query, limit, offset, filters)`，让后端只返回搜索结果所需的轻量字段。
- 结果 DTO 建议只带 `id / jobId / jobName / commandLine / finishedAt / exitCode / matchedSnippet / matchedField`，不要把完整 `stdout / stderr` 每次都带回来。
- 如果后续日志量持续增长，再评估基于 SQLite 全文索引的方案；至少当前可以先把“日志列表接口”和“日志搜索接口”拆开。

优先级：高

### 2. 搜索时重复做字符串归一化、拼接和排序

当前每次输入后，任务和日志结果都会重新执行：

- 字段 `trim + lower`
- 参数数组拼接
- 打分
- 全量排序

这在数据量不大时能工作，但随着任务数、日志数上升，输入延迟会先出现在这里。

对应实现：

- `frontend/components/SearchPalette.vue:30-31`
- `frontend/components/SearchPalette.vue:64-75`
- `frontend/components/SearchPalette.vue:193-207`

优化建议：

- 对任务搜索先做一层预计算索引，在 `jobs` 刷新后一次性生成小写字段缓存，查询时只做比较。
- 日志搜索如果短期内仍保留前端本地匹配，也建议为 `allLogs` 预构建可搜索字段，避免每次输入都重新 lower/拼接。
- 对“只取前 8 条结果”的场景，可以考虑用 partial sort / top-N 选择，而不是每次都对全量结果完整排序。

优先级：中高

### 3. 日志请求链路没有复用 store 的缓存与并发控制

主界面的日志加载已经有 `lastId / inflight / seq` 这类去重与竞态保护；搜索面板却绕过 store，直接调用 Wails binding。这样会带来两类问题：

- 同一时刻可能出现重复取数；
- 搜索面板与主界面的日志缓存彼此独立，无法共享已有结果。

对应实现：

- `frontend/components/SearchPalette.vue:5`
- `frontend/components/SearchPalette.vue:123-135`
- `frontend/stores/cron/logs.js:6`
- `frontend/stores/cron/logs.js:35-68`

优化建议：

- 短期内至少把搜索面板接到 store/runtime 层，统一超时、错误处理和请求去重。
- 更合理的做法是新增单独的 search store，负责搜索缓存、最近一次查询结果和请求状态。

优先级：中

### 4. 日志顺序处理存在重复工作

后端 `tail()` 先按 `started_at DESC` 查，再在内存里反转；前端日志面板拿到后又重新按时间排序。虽然 100 条数据下问题不大，但这是明显的重复计算。

对应实现：

- `cron_storage.go:132-149`
- `cron_storage.go:201-204`
- `frontend/components/LogsPanel.vue:145-149`

优化建议：

- 统一日志接口的排序约定，后端直接返回调用方真正需要的顺序。
- 如果日志面板始终按时间倒序展示，就没有必要在后端反转后再让前端重排一次。

优先级：中低

### 5. 日志模式下每次打开面板都强制重拉

搜索面板可见时切到日志模式会强制刷新；面板再次打开时如果当前模式还是日志，也会再次强制刷新。这保证了结果较新，但也意味着频繁打开搜索时会反复发请求。

对应实现：

- `frontend/components/SearchPalette.vue:212-217`
- `frontend/components/SearchPalette.vue:228-235`

优化建议：

- 加一个很轻的 TTL 缓存，例如 5 到 15 秒内直接复用结果，同时后台静默刷新。
- 或者保留旧结果先展示，再异步更新，减少“打开搜索先等一下”的感觉。

优先级：中

## 体验优化点

### 1. 日志搜索范围需要明确告知用户

现在日志搜索实际上只覆盖“最近 100 条全局日志”，但 UI 上没有明确说明。用户看到空结果时，很容易误以为是“历史里不存在”，而不是“搜索范围有限”。

对应实现：

- `frontend/components/SearchPalette.vue:123-129`
- `frontend/stores/cron/logs.js:46-47`

优化建议：

- 在日志模式顶部增加范围提示，例如“当前仅搜索最近 100 条日志”。
- 空结果下提供更明确的文案，例如“未在最近 100 条日志中找到匹配项”。
- 如果后续做服务端搜索，可以再补“搜索更多历史”的入口。

优先级：高

### 2. 搜索结果在面板打开期间可能变旧

主界面会在 `jobExecuted` 事件后刷新任务和日志，但搜索面板自己的 `allLogs` 不会自动跟进。结果是：面板开着时新日志已经生成，搜索结果里却看不到。

对应实现：

- `frontend/components/SearchPalette.vue:26`
- `frontend/components/SearchPalette.vue:123-135`
- `frontend/stores/cron/lifecycle.js:15-35`

优化建议：

- 搜索面板打开期间订阅 `jobExecuted`，把新 entry 追加到本地结果集或触发一次轻量刷新。
- 如果不想自动更新，也至少增加“刷新结果”动作，避免用户先关掉再重新打开。

优先级：高

### 3. 每次打开都会清空查询词，不利于连续操作

目前只要 `visible` 变化，就会执行 `resetSearch()`，这会清空 query、debouncedQuery 和选中项。对于反复切回搜索面板的人来说，连续查找成本偏高。

对应实现：

- `frontend/components/SearchPalette.vue:116-121`
- `frontend/components/SearchPalette.vue:228-235`

优化建议：

- 至少在一次会话内保留上一次查询词。
- 如果担心旧查询造成困扰，可以只在模式切换时保留、应用关闭后清空。
- 再往前一步可以增加 recent searches / recent results。

优先级：中

### 4. 结果缺少“为什么命中”的解释

当前结果虽然有 snippet，但没有高亮匹配词，也没有告诉用户命中的是任务名、命令、目录，还是日志输出。尤其日志结果里 `stdout/stderr/error` 混在一起时，可读性一般。

对应实现：

- `frontend/components/SearchPalette.vue:77-90`
- `frontend/components/SearchPalette.vue:157`
- `frontend/components/SearchPalette.vue:184`
- `frontend/components/SearchPalette.vue:332-348`

优化建议：

- 对匹配词做高亮。
- 显示命中字段标签，例如“命中：命令 / 目录 / stderr / error”。
- 日志结果可以把失败状态、时间、任务名和匹配片段的层级拉开，减少用户二次判断。

优先级：中

### 5. 键盘流还可以更完整

现在支持 `Ctrl/Cmd + F` 打开、方向键切换、`Enter` 跳转、`Ctrl + Enter` 直接运行任务，但这些提示基本只隐藏在 placeholder 和设置页说明里；搜索面板本身没有显式的快捷键提示，也没有更丰富的切换动作。

对应实现：

- `frontend/pages/MainPage.vue:16-31`
- `frontend/components/SearchPalette.vue:283-293`
- `frontend/components/SettingsShortcutDialog.vue:43-52`

优化建议：

- 在搜索面板底部加轻量键盘提示：`Enter` 打开，`Ctrl+Enter` 运行，`Tab` 切换任务/日志，`Esc` 关闭。
- 增加 `Tab` 切换 scope，会比鼠标点切换更顺手。
- 如果后续增加筛选器，可以继续扩展为“纯键盘完成搜索”。

优先级：中

### 6. 缺少更强的筛选维度

当前搜索主要依赖自由文本，没有状态、目录、时间范围等辅助筛选。日志场景下，很多时候用户真正想找的是“最近失败的某个任务”而不是全文检索。

优化建议：

- 任务搜索补充 `enabled/disabled`、folder 过滤。
- 日志搜索补充 `success/fail`、最近时间范围、任务名过滤。
- 可以先做轻量 chips，不必一步上复杂语法。

优先级：中

## 建议优先级

### 第一阶段：低风险、见效快

1. 在日志搜索 UI 上明确“仅搜索最近 100 条日志”。
2. 搜索面板打开期间接入 `jobExecuted` 刷新，解决结果变旧。
3. 为任务搜索做预计算字段缓存，减少输入时重复 lower/拼接。
4. 给日志模式加短 TTL 缓存或先展示旧结果再后台刷新。

### 第二阶段：结构优化

1. 新增专用日志搜索接口，返回轻量结果而不是完整日志正文。
2. 统一搜索请求入口，避免搜索面板绕过 store。
3. 补充状态、目录、时间范围等基础筛选。

### 第三阶段：历史规模上来后的升级项

1. 评估 SQLite 侧全文搜索能力。
2. 支持分页或“继续搜索更多历史”。
3. 为搜索结果增加 recent history / saved filters。

## 结论

search 功能还有比较明确的优化空间，而且不是“为了优化而优化”。

- 性能上，最值得优先处理的是“日志搜索把大文本搬到前端再扫”和“每次查询重复做字符串处理”。
- 体验上，最值得优先处理的是“日志搜索范围不透明”和“面板打开后结果会变旧”。

如果只做一轮小改动，建议先从“范围提示 + 实时刷新 + 前端缓存/预计算”开始；如果准备做一轮结构优化，优先补一个真正的日志搜索接口。
