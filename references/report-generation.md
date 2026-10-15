# 原生移动端代码审查 HTML 报告生成规范

## 目录

1. 强制输出规则
2. 唯一模板
3. 六章大纲
4. 生成流程
5. 模板变量
6. HTML 片段契约
7. 隐藏审查数据 JSON 严格合同
8. 状态映射
9. 安全与静态门禁
10. 控制台与对话输出边界

## 强制输出规则

- 每次完成 `$mobile-code-review-score` 审查都必须生成 HTML 报告，不能只输出对话文本。
- 没有发现问题时仍生成报告，问题清单写“未发现可确认问题”，并说明测试缺口和未覆盖项。
- 因环境、权限、接口或账号阻塞而无法完成校验时仍生成报告，把结论写为“验证受限”，不得伪造通过结果。
- 关键审查范围尚未确认时先询问用户；范围确认前不要生成内容不实的报告。
- 默认输出到被审查项目根目录下的 `review-output/mobile-code-review-report-YYYYMMDD-HHmmss.html`。
- 用户指定输出目录或文件名时优先采用用户要求；不得覆盖现有报告，除非用户明确授权。
- 审查正文必须写入 HTML 文件；报告成功生成后，控制台或对话只允许输出生成状态与报告绝对路径。
- 输出文件必须保留唯一模板中的内嵌 CSS、JavaScript 和 `reviewDataJson` 数据块，生成后可离线打开并在原报告上编辑或删除问题。
- 隐藏数据块 `reviewDataJson` 必须逐层满足本文 13.1 的唯一严格合同；字段缺失、合同外字段、类型错误、非法空值、占位内容或派生数据不一致时禁止生成、保存或下载报告。

## 唯一模板

- 只使用 `templates/CODE_REVIEW_REPORT_TEMPLATE.html`。
- 该文件是唯一报告模板和交互基线；不得把它降级为静态快照、移除内联运行时代码，或另建简化/自适应变体。
- 模板原件不得包含演示人员、提交、问题、目录、分数或代码片段；静态表格使用空态，真实内容只能来自本次审查数据。
- 模板的大纲、章节名称和顺序与仓库后端 `backEnd-cloud/.codex/skills/backend-code-review-score/templates/CODE_REVIEW_REPORT_TEMPLATE.html` 保持一致；表格结构按原生移动端口径调整：提交日志不展示提交明细和接口数量统计。
- 不维护自适应版、简洁版或其他变体，不得复制或临时拼接新模板。
- 原生移动端特有的码表编号、评分维度、目标操作系统、最低系统版本、设备矩阵、框架上下文和自动化校验只能写入六章内部，不得新增顶层章节。

## 六章大纲

| 顺序 | 固定章节 | 原生移动端报告内容 |
| ---: | --- | --- |
| 1 | 审查概览 | 唯一最终得分、问题数、必须修复数、提交数、涉及人员、AI 编码率、涉及目录树、目标操作系统、最低系统版本与设备矩阵口径、审查范围、审查依据和 AI 编码率说明 |
| 2 | 提交日志 | 按人员汇总提交数、代码变更统计（新增/删除行数与有效文件数）、代码总量占比和 AI 编码率；不展示提交明细和接口数量 |
| 3 | 问题清单 | 可折叠的五级问题说明表，以及按级别排序的问题列表 |
| 4 | 问题详情与脱敏代码片段 | 逐项展示码表字段、位置、影响、修复方向和问题代码证据 |
| 5 | 问题跟踪 | 按人员、模块或目录、中文级别汇总问题 |
| 6 | 结论与后续动作 | 门禁、阻断原因、必须修复、建议项、自动化校验限制、真机/模拟器/平台构建验证缺口和复审要求 |

顶层只允许出现以上六个 `<h2>`，名称和顺序不得调整。

## 生成流程

1. 加载 [issue-catalog.md](issue-catalog.md)，按 `id / rootCauseKey` 识别、去重和合并问题。
2. 按 [scoring-model.md](scoring-model.md) 完成扣分、风险封顶和团队聚合计算，得到唯一最终得分。
3. 按 [ai-coding-rate.md](ai-coding-rate.md) 完成整体与按人员的固定特征估算，得到采集变更代码量、排除代码量、实际分析代码量、折算 AI 编码量、AI 编码率和置信程度。
4. 根据门禁结论确定报告状态、标题分数区间样式和中文结论文案。
5. 读取唯一模板全文并生成新报告文件，不修改模板原件；除替换约定变量和数据块外，保留模板内嵌 CSS、JavaScript、编辑弹窗和保存逻辑。
6. 先填充普通文本变量，再生成提交、问题和统计表格行，最后生成问题详情片段。
7. 生成隐藏审查数据 JSON，写入 `reviewDataJson`，按 13.1 逐层核对精确字段集合、类型、允许值、事实来源和派生一致性，并确认其内容与可见报告完全一致。
8. 对所有动态文本进行 HTML 转义；只有本文列出的 HTML 片段变量允许写入标签。
9. 不适用字段统一写“本次审查不适用”；缺少证据的字段写“无法可靠判定”或“未执行：原因”。AI 编码率的边界情况按 [ai-coding-rate.md](ai-coding-rate.md) 统一输出 `0.00%` 并写明原因，不得写“无法估算”。
10. 静态检查六章标题、模板变量、表格列数、级别折叠表、问题代码块、隐藏 JSON 可解析性和状态一致性；不执行报告页面、真机或模拟器验证。

## 模板变量

### 标题与状态

| 变量 | 内容 |
| --- | --- |
| `reportTitle` | 简短审查标题，例如“订单管理当前修改审查” |
| `reviewResult` | 通过、需修改或阻断；低于 60 分但未触发阻断时仍写“需修改”，由 `status-fail` 表达低分状态 |
| `reviewConclusion` | 一句话审查结论摘要 |
| `reviewType` | 当前修改、本人提交、全局时间段或当前项目全部提交审查 |
| `reviewScope` | 审查范围说明 |
| `reviewTarget` | ViewController、Activity、Fragment、SwiftUI/Compose 层、网络/数据层、资源、构建或目录范围；同时写明 iOS/Android、最低系统版本、设备和架构口径 |
| `generatedAt` | 带时区的报告生成时间 |
| `reviewer` | 审查人，只展示 `git config user.name` 或明确的人名 |
| `scoreFormula` | 得分卡片悬浮提示中的固定计算口径；必须与评分模型和模板运行时重算逻辑一致 |
| `reportStatusClass` | `status-pass`、`status-change`、`status-fail`、`status-high-risk` |
| `scoreBandClass` | `score-excellent`、`score-good`、`score-warning`、`score-fail`、`score-high-risk` |
| `gateResult` | 通过、需修改或阻断，必须与 `reviewResult` 一致 |
| `gateClass` | `badge-pass`、`badge-major`、`badge-fail` 或 `badge-high-risk` |

### 审查概览

| 变量 | 内容 |
| --- | --- |
| `score` | 报告唯一展示的最终得分；单次审查取最终总分，聚合审查取风险校正后的团队综合得分 |
| `issueCount` | 已确认问题数量 |
| `mustFixCount` | 必须修复项数量 |
| `commitCount` | 有效提交数；当前修改审查使用 working tree/staged 标识统计 |
| `authorCount` | 有效参与人数 |
| `aiCodingRate` | 整体 AI 编码率，按 [ai-coding-rate.md](ai-coding-rate.md) 的固定代码特征估算，使用折算 AI 编码量 / 实际分析代码量，写两位小数百分比，例如 `43.42%`；不得因缺少 Git / 工具 AI 标识直接写 `0.00%`，也不得写“无法估算” |
| `codingRateBasis` | AI 编码率说明：必须写明估算方法、判断基线（个人历史风格基线或项目整体风格基线）、主要命中特征、采集变更代码量、排除代码量、实际分析代码量、折算 AI 编码量、置信程度和估算限制 |
| `branchInfo` | 当前分支和基线分支 |
| `commitRange` | Commit 范围 |
| `timeRange` | 时间范围 |
| `reviewAuthors` | 涉及人员 |
| `reviewDirectories` | 涉及目录树：按目录层级聚合本次涉及的目录，使用 `├─`、`└─`、`│` 绘制的多行文本树，合并共同父目录、目录名以 `/` 结尾、默认不展示文件；转义后写入模板固定的 `<pre class="dir-tree">`，不得输出扁平分号列表、逗号分隔、JSON 数组或绝对路径 |
| `reviewBasis` | 已读取规则、源码、配置和已执行自动化命令；目标设备、操作系统、最低系统版本、架构、弱网、权限、前后台切换、真机和模拟器验证的已覆盖或未覆盖口径也写在此处 |
| `reviewDataJson` | 隐藏在 HTML 中的完整审查数据 JSON 字符串；必须是合法 JSON，内容与可见报告一致，顶层必须且只能包含 `summary`、`persons`、`issues` 三个字段，并逐层满足 13.1 的严格合同 |

## 交互模板契约

- 报告是单个可离线打开的 HTML 文件；CSS、JavaScript 和隐藏 JSON 必须内嵌，禁止加载远程脚本、字体、样式或图片。
- 人员与提交日志只读，不生成任何人员编辑、删除或新增控件；问题清单的操作列固定在最右侧，只生成已有问题的“编辑”和“删除”按钮，不生成新增问题入口。
- 编辑弹窗不得修改问题编号；切换问题级别时必须把对应默认扣分自动回填到扣分输入框，提交后立即刷新问题清单、问题详情、问题跟踪和所有分数。
- 编辑或删除前撤销、保存按钮均为禁用状态；存在未保存编辑/删除历史时才启用。撤销恢复到初始数据后再次禁用。保存必须在当前 HTML 上生成包含最新 `reviewDataJson` 的新文件，不改变原报告解析所需的数据块位置和格式。
- 得分卡片必须包含 `scoreFormula` 悬浮提示；提示内容应说明基础分、同人递减、多人聚合和风险封顶口径，不能另占一列或新增侧栏。
- `reviewDataJson` 必须原样写入 `<script id="reviewDataJson" type="application/json" hidden>`；不得 HTML 转义成可见文本，也不得在生成后删除运行时所需的 `summary.codeType`、`persons` 或 `issues`。

### 结论

| 变量 | 内容 |
| --- | --- |
| `blockReason` | 阻断原因；未阻断时写“未触发阻断项” |
| `mustFixItems` | 必须修复项摘要 |
| `suggestionItems` | 非阻断建议 |
| `recheckRequirement` | 修复后的复审命令、范围、目标操作系统/最低版本/设备矩阵和前置条件 |

## HTML 片段契约

以下变量允许写入受控 HTML。片段中的作者、提交说明、路径、代码、终端输出和用户输入仍须逐项转义。

| 变量 | 固定结构 |
| --- | --- |
| `personCommitRowsHtml` | 每人一行，五列：人员（第一行姓名，第二行账号）、提交数、代码变更统计（第一行 `+新增 / -删除`，第二行“有效文件 N 个”）、代码总量占比（保留两位小数）、AI 编码率（第一行两位小数比例，第二行“置信程度 X｜分析 N 行”）；无有效提交时输出五列空态 |
| `issueRowsHtml` | 每个问题一行，八列：码表 id、中文级别、扣分、人员、提交、代码位置、是否必须修复、问题摘要；无问题时输出八列空态 |
| `issueDetailsHtml` | 每个已确认问题一个 `<details class="issue-detail" open>`，展示完整字段和脱敏代码证据；无问题时输出说明残余风险的空态块 |
| `personStatsRowsHtml` | 每人一行，六列：人员、提交数、问题数、必须修复、扣分、得分；不得重复展示 AI 编码率 |
| `moduleStatsRowsHtml` | 每个归属一行，五列：人员、模块或目录、问题数、必须修复、扣分 |
| `severityStatsRowsHtml` | 固定输出阻断、严重、主要、次要、提示五行统计，四列：级别、问题数、必须修复、扣分 |

生成片段时还需要以下统计字段，但它们不作为独立模板变量：

- `addedLineCount` / `deletedLineCount`：有效新增和删除行数，排除生成文件、压缩产物、纯格式化、无业务影响的锁文件、无独立差异的合并提交以及成对回滚的提交；展示格式为 `+数字` 和 `-数字`。
- `effectiveFileCount`：有效变更文件数，口径与行数统计一致，第二行展示为“有效文件 N 个”。
- `codeChangeSummary`：代码变更统计的合并展示内容，必须在同一个 `<td>` 内两行展示：第一行 `+新增行数 / -删除行数`，第二行 `有效文件 N 个`。
- `collectedChangedLines` / `excludedGeneratedLines` / `analyzedChangedLines`：采集变更代码量（有效新增 + 有效删除）、排除代码量和实际分析代码量，按 [ai-coding-rate.md](ai-coding-rate.md) 计算。
- `codeTotalRatio`：代码总量占比，等于该人员实际分析代码量 / 本次扫描实际分析代码量，保留两位小数，各人合计 100%；与 AI 编码率共用同一分母。非有效参与人（如变更全部被回滚）写“不计入”。
- `aiEstimatedLines` / `aiCodingRate` / `aiConfidence`：整体与按人员分别按分析单元汇总计算；AI 编码率保留两位小数，团队结果不得简单平均人员比例。
- 团队评分使用的贡献权重按 [scoring-model.md](scoring-model.md) “贡献权重与代码总量占比”一节计算，只参与评分，不作为报告展示列。
- 原生移动端报告不统计接口变更数量，也不输出绝对工作量值。
- `personStatsRowsHtml`、`moduleStatsRowsHtml`、`severityStatsRowsHtml` 的扣分列使用同人堆叠递减后的计入扣分合计，允许各表之间存在 1 分以内的舍入差异。

### 按人员提交统计行结构

`personCommitRowsHtml` 每行使用以下固定结构，所有动态文本逐项转义：

```html
<tr>
  <td>张三<br><span class="cell-sub mono">zhangsan@example.com</span></td>
  <td>7</td>
  <td><span class="diff-add">+3889</span> / <span class="diff-del">-1194</span><br><span class="cell-sub">有效文件 67 个</span></td>
  <td>100.00%</td>
  <td>43.42%<br><span class="cell-sub">置信程度 中等｜分析 4463 行</span></td>
</tr>
```

- 人员列第一行为展示姓名（`git log` 作者名或明确的人名），第二行为账号，取作者邮箱；同一人存在多个账号时全部列出并用逗号分隔，无法获取账号时第二行写“账号未知”。
- 代码变更统计列使用 `diff-add`、`diff-del` 类名着色，第二行“有效文件 N 个”使用 `cell-sub` 类名；不得改用其他类名或内联样式，也不得拆成多列。
- 代码总量占比列写该人员实际分析代码量占本次扫描实际分析代码量的百分比（保留两位小数）；非有效参与人写“不计入”，并可用 `cell-sub` 注明原因（如“变更均已回滚”）。
- AI 编码率列第一行写两位小数比例；第二行固定写“置信程度 X｜分析 N 行”，其中置信程度只能是高、中等、低。实际分析代码量为 0 时第一行写 `0.00%`，第二行写“置信程度 低｜分析 0 行”；全部变更被排除的人员第二行写“置信程度 低｜不计入统计”。
- 审查概览的 `reviewBasis` 必须从基础分开始用中文说明分数计算过程；`codingRateBasis` 必须汇总整体的判断基线、主要命中特征、采集与排除代码量、实际分析代码量、折算 AI 编码量、AI 编码率和置信程度，不得只展示一个无法解释的百分比。

### 主表与详情空态

- `personCommitRowsHtml` 没有有效提交时输出 `<tr class="empty-row"><td colspan="5">暂无有效提交统计</td></tr>`。
- `issueRowsHtml` 没有已确认问题时输出 `<tr class="empty-row"><td colspan="8">未发现可确认问题</td></tr>`。
- `issueDetailsHtml` 没有已确认问题时输出 `<div class="empty-state">未发现可确认问题；残余测试缺口与未覆盖项见审查依据及结论。</div>`，不得把裸文本直接写入表格或保留空白章节。

### 问题跟踪空态

- `personStatsRowsHtml` 没有数据时输出 `<tr class="empty-row"><td colspan="6">暂无人员问题统计</td></tr>`。
- `moduleStatsRowsHtml` 没有数据时输出 `<tr class="empty-row"><td colspan="5">暂无模块问题统计</td></tr>`。
- `severityStatsRowsHtml` 无论是否发现问题都固定输出五个中文级别及零值，不使用空态行。

### 问题详情代码证据

- 每个问题详情至少包含 `id`、`rootCauseKey`、中文级别、扣分、评分维度、`locations`、`mergedFrom`、人员、提交、影响、`fixHint` 和复审方式。
- 所有已确认问题必须展示最小且可定位的脱敏问题代码，使用 `<pre><code>` 并完成 HTML 转义。
- `mustFix=true` 时代码证据为强制字段，不得只输出问题描述或修复方向。
- 证据不足时标记为“待确认”且不扣分，不得为了填满模板虚构代码。
- `codeDisplayPolicy` 只控制无问题上下文、完整文件或大段辅助代码，不得隐藏已确认问题的代码片段。

### 五级问题说明表

第三章必须在问题列表前保留固定的 `<details class="severity-guide">`，默认收起，摘要为“问题等级说明（点击展开）”。表格不使用占位符，固定展示：

| 级别 | 单个问题扣分 | 必须修复 | 含义 |
| --- | ---: | --- | --- |
| 阻断 | 20-30 | 是 | 导致系统不可用、数据损坏、重大安全风险、合规阻断或无法上线 |
| 严重 | 10-19 | 是 | 高概率线上故障、严重性能问题、权限绕过或核心业务错误 |
| 主要 | 5-9 | 视情况 | 明确影响功能正确性、维护性、扩展性或可测试性 |
| 次要 | 1-4 | 否 | 局部可读性、边界处理或轻量规范问题 |
| 提示 | 0 | 否 | 不扣分建议、风格提示或可选优化 |

## 13.1 隐藏审查数据 JSON 严格合同

`reviewDataJson` 必须渲染为完整、合法、可被 `JSON.parse` 解析的 JSON 对象，并原样隐藏在模板末尾的 `<script id="reviewDataJson" type="application/json" hidden>` 数据块中。这里定义的是**唯一严格合同**，不是推荐结构：对象字段顺序不影响 JSON 语义，但每一层字段集合必须完全一致，不得缺漏、不得多生成、不得重复键，也不得依赖 DTO、序列化器或解析器静默忽略未知字段。

### 13.1.1 顶层固定字段

顶层必须且只能包含以下三个字段：

| 字段 | 类型 | 生成规则 |
| --- | --- | --- |
| `summary` | object | 报告概览、评分、范围、AI 编码率和结论数据，字段合同见 13.1.2 |
| `persons` | array | 按人员汇总的数据，元素合同见 13.1.3；无人员时允许 `[]`，不得写 `null` |
| `issues` | array | 顶层问题事实源，元素合同见 13.1.4；无问题时必须为 `[]` |

固定结构骨架：

```json
{
  "summary": {},
  "persons": [],
  "issues": []
}
```

不得临时增加 `nodeId` 或其他扩展字段。若业务协议确需新增字段，必须先修改本合同，并同步报告生成器、模板校验器、DTO、持久化结构和所有读取方后才能输出；只修改单一环节或依赖未知字段被静默接收均视为门禁失败。

### 13.1.2 `summary` 固定字段

`summary` 必须且只能包含以下字段，所有字段都必须显式生成：

| 字段 | 类型 | 内容要求 |
| --- | --- | --- |
| `codeType` | string | 固定为 `frontend` |
| `totalScore` | number | 最终风险分，必须等于可见 `score` |
| `finalRiskScore` | number | 最终风险分，必须等于 `totalScore` |
| `score` | number | 模板交互同步分，必须等于 `totalScore` |
| `branch` | string | 实际审查分支；确实无法取得时使用已核验的分支或基线说明，不得写通用占位文本 |
| `aiCodingRate` | string | 按规则计算的百分比，格式 `0.00%` |
| `aiCodingMethod` | string | 实际采用的 AI 编码率估算方法 |
| `aiEstimatedLines` | number | 折算 AI 编码量，非负数 |
| `collectedChangedLines` | number | 采集变更代码量，非负整数 |
| `analyzedChangedLines` | number | 实际分析代码量，非负整数 |
| `excludedGeneratedLines` | number | 排除代码量，非负整数 |
| `aiConfidence` | string | 只允许 `高`、`中等`、`低` |
| `aiCodingBasis` | string | 基于真实证据生成的中文 AI 编码率判断依据 |
| `reviewBasis` | string | 与可见报告一致的纯文本中文计分依据，不得包含 HTML、知识库文件名、原始变量名或英文字段名 |
| `reportTitle` | string | 实际报告标题 |
| `reviewScope` | string | 实际审查范围 |
| `reviewTarget` | string | 实际审查对象 |
| `reviewType` | string | 四种审查类型之一 |
| `generatedAt` | string | 实际生成时间，格式必须在生成器内统一 |
| `reviewer` | string | 实际审查人或明确的审查主体 |
| `reviewConclusion` | string | 与当前门禁结果和问题事实一致的中文结论 |
| `reviewResult` | string | 只允许 `通过`、`需修改`、`阻断` |
| `issueCount` | number | 当前确认问题数量，必须等于顶层有效问题数量 |
| `mustFixCount` | number | 顶层有效问题中 `mustFix === true` 的数量 |
| `commitCount` | number | 纳入范围的实际提交数，非负整数 |
| `authorCount` | number | 必须等于 `persons.length` |
| `reviewAuthors` | string | 按人员名称生成；确无人员时固定为 `无` |
| `branchInfo` | string | 可见概览使用的真实分支或基线信息 |
| `commitRange` | string | 真实提交范围 |
| `timeRange` | string | 真实时间范围；审查类型不适用时间范围时写明确的“不适用”说明 |
| `reviewDirectories` | string | 本次实际涉及目录的紧凑文本树，不得写绝对路径或 JSON 数组 |
| `codingRateBasis` | string | 与可见报告一致的 AI 编码率中文判断说明 |

`summary.totalScore`、`summary.finalRiskScore` 和 `summary.score` 必须始终相等。多人场景必须写多人聚合后的最终风险分，禁止写按全部问题累计扣分得到的中间分。所有统计字段必须由当前事实源重新计算，不能沿用删除或改派前的旧值。

### 13.1.3 `persons[]` 固定字段

每个 `persons[]` 元素必须且只能包含以下字段：

| 字段 | 类型 | 内容要求 |
| --- | --- | --- |
| `id` | string | 稳定且非空的人员标识，供顶层问题的 `personId` 关联 |
| `name` | string | Git 作者名、提交人名或已确认人员名，非空 |
| `email` | string | Git 提交邮箱；确实缺失时允许空字符串，不得臆造 |
| `score` | number | 按评分模型从当前顶层有效问题重新计算的个人分 |
| `issueCount` | number | 必须等于本人员 `issues.length` |
| `issues` | array | 人员问题快照，元素合同见 13.1.5；无问题时必须为 `[]` |
| `commitCount` | number | 本人员实际提交数，非负整数 |
| `addedLines` | number | 有效新增行数，非负整数 |
| `deletedLines` | number | 有效删除行数，非负整数 |
| `effectiveFileCount` | number | 有效代码文件数，非负整数 |
| `analyzedChangedLines` | number | 实际分析代码量，非负整数 |
| `excludedGeneratedLines` | number | 排除代码量，非负整数 |
| `aiEstimatedLines` | number | 折算 AI 编码量，非负数 |
| `aiCodingRate` | string | 由 `aiEstimatedLines / analyzedChangedLines` 计算，格式 `0.00%`；分母为 0 时固定为 `0.00%` |
| `aiConfidence` | string | 只允许 `高`、`中等`、`低` |
| `aiCodingBasis` | string | 基于本人员真实代码特征生成的中文判断依据 |
| `codeTotalRatio` | string | 本人员实际分析代码量占整体比例，格式 `0.00%` |
| `codeAmount` | string | 固定格式：`+新增行数 / -删除行数\n有效文件 N 个` |

`includedById`、`includedDeduction`、`mustFixCount` 等仅用于模板运行时计算的临时字段不得写入最终隐藏 JSON。人员名称、邮箱、提交数和代码量必须来自实际 Git 证据；只有合同明确允许的情况才能输出空字符串或零值，不能把默认值当作未采集数据的替代品。

### 13.1.4 顶层 `issues[]` 固定字段

顶层 `issues[]` 是问题与评分的唯一事实源。每项必须且只能包含以下字段：

| 字段 | 类型 | 内容要求 |
| --- | --- | --- |
| `id` | string | 稳定问题码，非空且在同一报告内唯一 |
| `severity` | string | 只允许 `blocker`、`critical`、`major`、`minor`、`info` |
| `deduction` | number | 原始扣分，非负数 |
| `confirmed` | boolean | 是否确认；只有 `true` 的问题参与可见报告和评分，不得省略 |
| `personId` | string | 必须精确关联现有 `persons[].id` |
| `commit` | string | 可复核的真实提交标识；工作区问题使用规范中明确的工作区标识，不得写占位文本 |
| `location` | string | 主代码位置，格式为仓库相对路径加行号 |
| `locations` | array | 归一后的全部代码位置字符串；至少包含 `location` 且不得重复 |
| `module` | string | 实际模块或仓库相对目录 |
| `summary` | string | 与可见问题清单一致的非空中文摘要 |
| `impact` | string | 非空中文影响说明 |
| `fixHint` | string | 非空中文修复建议 |
| `recheck` | string | 非空中文复审方式 |
| `code` | string | 非空、最小且已脱敏的问题代码证据 |
| `mustFix` | boolean | 是否必须修复 |
| `scoringDimension` | string | 非空中文评分维度；统一使用该字段名，不得同时或改用 `dimension` |
| `rootCauseKey` | string | 归一化根因键，非空 |
| `mergedFrom` | number | 归一前合并命中数，正整数 |

顶层问题不得重复键或重复 `id`。`confirmed === false` 的待确认问题可以保留为事实记录，但不得进入 `persons[].issues`、问题数量、个人扣分、团队评分和可见问题列表；删除的问题必须从顶层数组物理移除，不能只从可见区域隐藏。

### 13.1.5 `persons[].issues[]` 固定八字段

每项必须且只能包含以下八个字段：

| 字段 | 类型 | 内容要求 |
| --- | --- | --- |
| `id` | string | 对应顶层有效问题的 `id` |
| `level` | string | 只允许中文 `阻断`、`严重`、`主要`、`次要`、`提示` |
| `deduction` | number | 对应顶层问题的原始扣分 |
| `person` | string | 对应人员的 `name` |
| `commit` | string | 对应顶层问题的 `commit` |
| `location` | string | 对应顶层问题的 `location` |
| `mustFix` | boolean | 对应顶层问题的 `mustFix` |
| `summary` | string | 对应顶层问题的 `summary` |

人员问题数组必须由 `confirmed === true` 且 `personId` 匹配的顶层问题实时派生，不得独立维护。`persons[].issueCount === persons[].issues.length`；无问题人员必须显式输出 `"issues": []`，不得省略、写 `null` 或空字符串。

### 13.1.6 完整生成、重算与静态门禁

- 首次生成、首次渲染、编辑、删除、撤销、改派、保存和下载前，必须从顶层 `issues[]` 完整重建 `persons[].issues[]`、个人扣分、个人分、人员问题数、团队聚合分、风险封顶、最终风险分和 `summary` 派生字段。
- 删除问题后，该问题不得继续出现在顶层问题、人员问题快照、问题清单、问题详情或问题跟踪中，其扣分不得残留；撤销删除时恢复同一问题事实并再次全量重算。
- 整个隐藏 JSON 的内容必须与可见 HTML 一致。字段值必须来自实际 Git 证据、问题事实源、评分算法或 AI 编码率计算；不得用 `null`、空字符串、占位文本、默认零、伪造值或旧缓存绕过生成。只有本合同明确允许的空值、空数组或零值才能输出。
- 必须在对象构造完成后按字段白名单重建最终 JSON，再执行深度校验：逐层校验精确字段集合、类型、枚举、唯一性、外键关联、数组元素、评分派生关系和可见报告一致性。任何未知字段都必须报错，禁止静默丢弃。
- JSON 字符串必须完成 JSON 转义和脚本上下文安全转义，至少处理引号、反斜杠、换行、`<`、`>`、`&` 和可能破坏 `</script>` 的内容。
- 隐藏 JSON 不得包含除 Git 提交邮箱外的未脱敏敏感信息；可见报告已经脱敏的内容，隐藏 JSON 必须采用同等或更严格的脱敏口径。
- 以下任一情况均为静态门禁失败，必须禁止生成、保存或下载报告：字段缺失、合同外字段、重复键、类型或枚举错误、非法空值、占位或伪造内容、问题关联失效、人员问题快照不同步、分数未重算、可见报告与隐藏 JSON 不一致、未知字段被静默忽略。

原生移动端补充：审查对象虽然是 Swift、Objective-C、Kotlin 或 Java 等原生代码，输出仍必须使用上述统一字段合同，`summary.codeType` 固定为 `frontend`，不得改为 `mobile`、`ios` 或 `android`。目标操作系统、最低系统版本、CPU 架构、设备矩阵、弱网、权限、前后台切换以及真机/模拟器覆盖情况写入 `summary.reviewScope`、`summary.reviewTarget` 或 `summary.reviewBasis`；不得为这些维度临时增加 JSON 字段。原生移动端不统计接口变更数量，也不输出绝对工作量值。

## 状态映射

按以下优先级选择唯一状态：

```text
status-high-risk > status-fail > status-change > status-pass
```

| 条件 | 报告状态 | 标题样式 | 审查结果 |
| --- | --- | --- | --- |
| 存在阻断、严重或其他明确阻断项 | `status-high-risk` | `score-high-risk` | 阻断 |
| 最终得分低于 60 且没有更高优先级状态 | `status-fail` | `score-fail` | 需修改 |
| 存在需要修改的问题但未阻断 | `status-change` | 按最终得分选择 | 需修改 |
| 无必须修改项，或仅有提示 | `status-pass` | 按最终得分选择 | 通过 |

## 安全与静态门禁

- 转义 `&`、`<`、`>`、`"` 和 `'`，尤其是代码、commit message、作者、路径、终端输出和接口响应。
- 不把被审查代码直接拼入 HTML 标签或属性；代码只允许放入转义后的 `<pre><code>`。
- 类名变量只能使用本文列出的状态类和 `badge-blocker`、`badge-critical`、`badge-major`、`badge-minor`、`badge-info`，不得接受任意输入。
- 不加载远程脚本、字体、样式或图片，保证报告可离线打开。
- 模板文件允许保留约定占位符；生成后的最终文件中 `{{[A-Za-z0-9_]+}}` 匹配数必须为 0（`reviewDataJson` 必须已写入合法 JSON）。
- 使用静态解析确认顶层 `<h2>` 恰好为六章且顺序正确，模板目录中只能存在唯一的 `CODE_REVIEW_REPORT_TEMPLATE.html`。
- 检查所有必须修复项在 `issueDetailsHtml` 中存在文件位置和非空 `<pre><code>`；检查问题数、必须修复数、最终得分、状态类和门禁结论相互一致。
- 检查隐藏数据块 `reviewDataJson` 可被 `JSON.parse` 解析，并完整通过 13.1 的逐层严格合同：精确字段集合、类型、枚举、唯一性、人员关联、人员问题快照、AI 指标、问题数量和所有派生分数必须与可见报告一致；未知字段必须报错，不得静默忽略。任一项失败时禁止生成、保存或下载报告。
- 检查隐藏 JSON 内不存在未转义的 `</script>`，且 `persons[].codeAmount` 中的换行以 `\n` 形式写入 JSON 字符串。
- 检查模板内存在 `data-edit-issue`、`data-delete-issue`、`undoButton`、`saveButton` 和级别变更回填逻辑；生成后的报告不得残留这些控件所依赖的占位符。
- 本 Skill 不执行报告页面、被审查应用的真机、模拟器或实际网络验证。

## 控制台与对话输出边界

- 报告成功生成后，使用固定格式：`移动端代码审查报告已生成：<报告绝对路径>`。
- 不得在控制台、过程消息或最终回复中展开得分、风险摘要、问题清单、代码片段、门禁结论或验证明细；这些内容全部写入 HTML 报告。
- 审查执行期间不要提前输出中间发现，避免报告内容通过控制台或对话旁路泄露。
- 报告生成失败时可以输出失败状态和必要原因；关键范围缺失时可以提出完成审查所必需的问题。此时不得声称报告已生成，也不得提供虚假路径。
