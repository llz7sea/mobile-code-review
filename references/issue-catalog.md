# 原生移动端问题分类码表

## 目录

1. 文档定位
2. 编号方案
3. 条目字段结构
4. 去重与合并规则
5. 码表条目
6. AI 使用流程
7. 扩展与维护

## 一、文档定位

本文是移动端代码审查的**问题分类字典（码表）**。AI 在触发代码审查时，对照本表识别问题、分配唯一编号、映射级别、执行去重与合并。

- 评分规则、级别定义参见 [scoring-model.md](scoring-model.md)，报告输出参见 [report-generation.md](report-generation.md)，三文档配合使用。
- 每条问题拥有稳定 `id`（如 `SEC-001`），AI 命中后必须输出该 `id`，作为跨报告的稳定引用。
- 本表覆盖原生 iOS 与原生 Android 场景（Swift、Objective-C、Kotlin、Java、UIKit、SwiftUI、Android Views、Jetpack Compose、Xcode、Gradle 及平台 SDK）。移动 Web/H5、Hybrid WebView、uni-app、Taro、微信小程序、Flutter、React Native 跨端层及后端专属问题（SQL 注入、服务端事务、Redis 缓存等）不在本表范围，应在报告中列为未覆盖项；原生代码中的 WebView Bridge 仅按桥接边界专项检查。
- 本表与后端码表使用相同的条目字段结构、去重规则和 AI 输出格式；前缀相同的分类语义对齐，但条目在前后端各自独立编号，不跨仓库引用。

## 二、编号方案

| 前缀 | 分类 | 说明 |
| --- | --- | --- |
| SEC | 安全与漏洞 | XSS、凭证泄露、开放重定向、动态执行、鉴权绕过、JSBridge、Deep Link、传输与上传下载 |
| PRV | 隐私与合规 | 脱敏、埋点上报、设备权限、剪贴板与内部信息暴露 |
| BIZ | 业务逻辑正确性 | 条件、计算、金额精度、时间、边界、状态、字段映射 |
| ROB | 健壮性 | 空值、异常兜底、状态复位、超时取消、重试边界 |
| CON | 异步与竞态 | 请求竞态、重复提交、防抖节流、并行异步 |
| CMP | 状态与生命周期 | View/ViewModel 边界、状态持有、订阅清理、ViewController/Activity 生命周期与回调 |
| UI | 界面与可访问性 | 样式污染、安全区、软键盘、触控手势、横竖屏、字体缩放与可访问性 |
| PERF | 性能 | 启动、包体、循环请求、重复请求、大列表、图片、后台任务、传感器与资源泄漏 |
| LOG | 日志与调试 | 敏感输出、吞异常、调试残留 |
| CFG | 配置 | 硬编码、环境混用、开关残留、Info.plist/AndroidManifest、签名与公共构建配置 |
| DEP | 依赖 | 已知 CVE、版本漂移、冗余依赖 |
| API | 接口调用 | 网络层复用、契约一致性、响应处理、超时与分页 |
| QUAL | 代码质量 | 复制粘贴、死代码、复杂度、命名、注释 |
| TEST | 测试与验证 | 核心变更验证、纯逻辑测试覆盖、平台构建与目标设备矩阵 |

- 原生移动端存在多线程、协程与异步竞态，`CON` 覆盖 GCD/Operation、协程、线程安全、请求时序、重复触发、页面切换与前后台切换。
- 编号一旦分配不再复用；新增条目末尾追加，废弃条目标记 `deprecated: true` 但保留编号。

## 三、条目字段结构

每条问题以列表项组织，字段固定如下（AI 输出时必须按同结构给出命中信息）：

```yaml
- id: SEC-002                                  # 唯一编号，稳定不变
  title: XSS / 危险 HTML 未净化                 # 中文标题
  level: 阻断                                  # 建议级别：阻断/严重/主要/次要/提示；AI 可依上下文在 ±1 级内调整
  rootCauseKey: SEC-002:xss-unsanitized-html   # 去重合并键，AI 必须输出
  signals:                                     # 识别线索，仅供参考，AI 可依上下文自行补充
    keywords: [ "loadHTMLString", "evaluateJavaScript", "addJavascriptInterface" ]
    whitelist: [ "白名单净化", "来源校验", "DOMPurify.sanitize" ]
  description: 原生 WebView 将用户可控内容直接加载或注入脚本，可执行任意脚本或调用未授权能力。
  fixHint: WebView 内容走白名单净化；脚本和桥接参数按来源与能力校验。
```

- **id**：`前缀-三位数字`，AI 输出的每个问题必须携带。
- **title**：中文，简短一句。
- **level**：建议级别，五档中文：阻断/严重/主要/次要/提示；与评分模型英文枚举一一对应（阻断=Blocker、严重=Critical、主要=Major、次要=Minor、提示=Info）。
- **rootCauseKey**：去重合并主键，格式 `<id>:<归一化根因特征>`；AI 必须输出。
- **signals**：识别线索，`regex` / `keywords` / `whitelist` 三选；**仅供参考**，AI 应结合语义判断，不得只做机械匹配。
- **description**：一句话说明 WHY。
- **fixHint**：一句修复方向（≤ 30 字）。
- **downgradeToInfo**（可选）：列出必须降级为“提示”且不扣分的证据不足场景；命中即按提示处理。
- **excludeFromReview**（可选）：列出默认不进入问题清单的业务边界场景；命中即不输出、不扣分。

## 四、去重与合并规则

AI 每次扫描按下列规则计算与合并，保证多次扫描同一份代码结果稳定。

### 4.1 三层键定义

```
rootCauseKey        = "<id>:<归一化根因特征>"   例：SEC-002:xss-unsanitized-html
locationFingerprint = <文件路径> + "#" + <类名/方法名/代码块>   （不含行号）
descriptionNorm     = 去除变量名、行号、字面量后取前 8 位哈希
```

### 4.2 合并与扣分规则

| 编号 | 规则 | 说明 |
| --- | --- | --- |
| R1 | 根因合并 | 同一 `rootCauseKey` 只算 1 条问题；`locations[]` 记录所有 `locationFingerprint`，扣分不因数量线性叠加 |
| R2 | 影响面上浮 | 同一 `rootCauseKey` 命中位置 ≥ 3 处，可在该 id 建议级别范围内上浮 1 档（如 主要 → 严重），不得超过 id 允许上限 |
| R3 | 描述归一 | `rootCauseKey` 相同、`descriptionNorm` 不同的多次记录，视为同一问题，不重复扣分 |
| R4 | 多类别命中 | 同一位置同时命中多个 id，按最高级别 id 扣分，其他 id 仅列为「关联问题」，不再重复扣 |
| R5 | 扫描幂等 | 同一份代码重复扫描，`rootCauseKey` 集合应完全一致；不一致视为扫描不稳定，需回归 |
| R6 | 证据不足 | 无法明确判定的疑似项，标记为「待确认」，不进入扣分池 |
| R7 | 白名单豁免 | 命中条目 `signals.whitelist` 时不入问题池 |
| R8 | 级别冲突 | 同 id 在不同证据下级别不一致时，取该次扫描中的最高级别 |

### 4.3 AI 输出结构约定

AI 在报告中每个问题至少输出：

```yaml
id: CMP-004
rootCauseKey: CMP-004:side-effect-not-cleaned
level: 主要
mustFix: true
locations:
  - src/views/desk/notice.vue#onMounted
  - src/components/custom-chart/index.vue#initChart
occurrenceCount: 2
mergedFrom: 5           # 归一前的原始命中次数
evidence: <脱敏问题代码片段；mustFix=true 时必填>
fixHint: 卸载钩子成对清理；封装自动清理的组合式函数
```

- **mustFix**：是否必须修复；阻断、严重问题固定为 `true`，主要问题按实际门禁判断，次要和提示通常为 `false`。
- **evidence**：已确认问题的脱敏问题代码；`mustFix=true` 时必须包含精确位置对应的非空代码片段。

### 4.4 误报排除、降级与证据边界

- 命中安全类关键词但缺少上下文证据时，不得直接按阻断、严重或主要扣分；应先降级为“提示”或标记为“待确认”。
- 命中条目的 `downgradeToInfo` 时只按提示处理且不扣分；命中 `excludeFromReview` 时默认不进入问题清单，也不得写入问题摘要、阻断原因、必须修复项或复审要求。
- 只有能证明外部输入可达该代码路径，并可实际造成脚本执行、凭证泄露、越权访问或数据破坏时，才可按对应安全问题的建议级别扣分。
- 后端与跨端 Web 职责范围内的问题不在本表输出：接口是否具备服务端鉴权、SQL 与事务实现、服务端限流与缓存策略、H5 页面样式和前端框架规则等；原生实现本身形成可利用路径（如仅靠隐藏入口当作鉴权、WebView Bridge 无白名单、深链未校验）时才作为原生移动端问题。
- 仅从命名、注释、目录结构或提交文案推断的“疑似”问题不进入扣分池；租户、组织、人员等业务边界字段的传递按业务规则处理，除用户明确要求业务建议外不作为安全问题输出。
- 重复扫描时，被降级或排除的判定必须保持一致；`rootCauseKey` 集合变化时须在报告中说明是新代码还是误报修正。

---

## 五、码表条目

### 5.1 SEC · 安全与漏洞

```yaml
- id: SEC-001
  title: 明文密钥/Token/密码硬编码进原生代码
  level: 阻断
  rootCauseKey: SEC-001:credential-hardcoded
  signals:
    regex:
      - (?i)(password|passwd|token|secret|access[_-]?key|api[_-]?key|client[_-]?secret)\s*[:=]\s*["'][^"'${}]{4,}["']
    keywords: [ "Authorization: Bearer ", "privateKey", "apiKey", "clientSecret" ]
    whitelist: [ "import.meta.env", "process.env", "VITE_" ]
  description: 有明确证据证明生产真实凭证、高权限凭证或可直接调用付费/管理接口的密钥被硬编码，泄露即事故。
  fixHint: 凭证由后端下发或走环境变量；仓库不得保留明文。
  downgradeToInfo:
    - "设计上即公开的第三方移动 SDK Key，且无证据表明可越权调用或产生资损。"
    - "mock 数据、示例配置、开发代理配置、本地调试账号，且未证明会进入生产构建产物。"
    - "仅凭字段名 token、secret、password 命中，但值为占位符、空串、常量枚举或校验提示文案。"

- id: SEC-002
  title: WebView 内容 / 危险 HTML 未净化
  level: 阻断
  rootCauseKey: SEC-002:xss-unsanitized-html
  signals:
    keywords: [ "loadHTMLString", "loadData", "evaluateJavaScript", "innerHTML", "addJavascriptInterface" ]
    whitelist: [ "WKUserContentController", "DOMPurify.sanitize", "白名单净化", "来源校验" ]
  description: 原生 WebView 将用户可控内容直接加载或注入脚本，或桥接输入未净化，可能执行任意脚本或越权调用原生能力。
  fixHint: WebView 内容走白名单净化；脚本和桥接参数按来源与能力校验。

- id: SEC-003
  title: 动态代码执行（eval/new Function/字符串定时器）
  level: 严重
  rootCauseKey: SEC-003:dynamic-code-execution
  signals:
    keywords: [ "eval(", "new Function(", "setTimeout(\"", "setInterval(\"" ]
  description: 不可信字符串进入动态脚本、反射或表达式执行环境，可被注入任意代码或调用未授权能力。
  fixHint: 禁止不可信动态执行；配置驱动逻辑改用白名单映射表。

- id: SEC-004
  title: 开放重定向 / 不可信跳转
  level: 严重
  rootCauseKey: SEC-004:open-redirect
  signals:
    regex:
      - (location\.href|location\.replace|window\.open)\s*\(?[^;\n]*(query|params|redirect|returnUrl)
    keywords: [ "redirect", "returnUrl" ]
    whitelist: [ "域名白名单校验", "URLComponents", "Uri.parse" ]
  description: 外部 URL、Intent 或 Universal/App Link 的跳转目标未校验，可被钓鱼或越权利用。
  fixHint: 跳转目标做 scheme、host、path 白名单；敏感动作二次确认。

- id: SEC-005
  title: postMessage 通信未校验来源
  level: 严重
  rootCauseKey: SEC-005:postmessage-origin-unchecked
  signals:
    regex:
      - postMessage\s*\([^)]*,\s*['"]\*['"]
    keywords: [ "addEventListener('message'", "window.postMessage" ]
    whitelist: [ "event.origin ===" ]
  description: 原生 WebView 消息监听未校验来源或发送目标，任意页面可伪造消息或窃取桥接数据。
  fixHint: 固定来源和目标；消息处理前校验 origin、页面状态与参数。

- id: SEC-006
  title: 仅客户端隐藏当作鉴权 / 页面权限缺失
  level: 阻断
  rootCauseKey: SEC-006:frontend-only-authz
  signals:
    keywords: [ "isHidden", "setVisibility", "shouldShow", "startActivity", "performSegue" ]
  description: 通过隐藏控件、客户端路由或页面入口代替服务端鉴权，敏感操作仍可被构造请求或深链触发。
  fixHint: 客户端控制仅是体验层；敏感操作必须由服务端鉴权并校验身份。

- id: SEC-007
  title: 敏感数据明文落地原生存储
  level: 严重
  rootCauseKey: SEC-007:sensitive-in-web-storage
  signals:
    keywords: [ "UserDefaults", "NSUserDefaults", "SharedPreferences", "getSharedPreferences", "SQLiteDatabase" ]
    whitelist: [ "Keychain", "Keystore", "EncryptedSharedPreferences", "加密存储工具" ]
  description: 密码、Token、身份证、完整个人信息等明文写入 UserDefaults、SharedPreferences、普通数据库或文件，可被越权读取或备份泄露。
  fixHint: 敏感数据使用 Keychain/Keystore；普通缓存不得存放完整凭证。

- id: SEC-008
  title: 外部链接缺少安全隔离
  level: 次要
  rootCauseKey: SEC-008:target-blank-noopener
  signals:
    regex:
      - (SFSafariViewController|CustomTabsIntent|ACTION_VIEW)\s*\([^\n]*(url|uri)
  description: 原生应用把外部链接交给不受控 WebView、Intent 或浏览器上下文，缺少来源、scheme 和返回链路隔离。
  fixHint: 外链使用系统安全容器；限制 scheme、host 和可回调页面。

- id: SEC-009
  title: 上传/下载或媒体选择绕过统一鉴权与限制
  level: 严重
  rootCauseKey: SEC-009:upload-download-unrestricted
  signals:
    keywords: [ "URLSessionUploadTask", "MultipartBody", "RequestBody", "UIDocumentPicker", "ACTION_OPEN_DOCUMENT" ]
    whitelist: [ "统一上传封装", "headers 携带 Token", "类型、大小与数量校验" ]
  description: 原生上传、下载或文件选择绕过统一鉴权与限制，未校验类型、大小、数量、归属或临时文件生命周期。
  fixHint: 文件走统一网络封装；客户端限制并由服务端二次校验。

- id: SEC-010
  title: URL 参数拼接未编码
  level: 主要
  rootCauseKey: SEC-010:url-param-unencoded
  signals:
    regex:
      - "[?&][A-Za-z0-9_]+=\\s*['\"]?\\s*\\+"
    keywords: [ "URL(string:", "Uri.parse(", "appendQueryParameter", "?id=" ]
    whitelist: [ "URLComponents", "Uri.Builder", "addingPercentEncoding" ]
  description: URL、Deep Link 或请求参数直接拼接，特殊字符破坏请求语义或造成注入。
  fixHint: 使用 URLComponents/Uri.Builder；手工拼接必须编码。

- id: SEC-011
  title: JSBridge 接口未做来源与能力白名单
  level: 严重
  rootCauseKey: SEC-011:jsbridge-untrusted-handler
  signals:
    keywords: [ "window.webkit.messageHandlers", "postMessage", "jsBridge", "wx.miniProgram" ]
    whitelist: [ "origin 白名单", "方法白名单", "schema 校验", "能力授权" ]
  description: WebView Bridge 允许任意页面、任意方法或未校验参数调用敏感原生能力。
  fixHint: 固定来源和方法白名单；参数按能力逐项校验。

- id: SEC-012
  title: Deep Link / URL Scheme 未校验
  level: 严重
  rootCauseKey: SEC-012:deeplink-unvalidated
  signals:
    keywords: [ "application(_:open:", "continueUserActivity", "onNewIntent", "intent.data", "deepLink" ]
    whitelist: [ "路由白名单", "schema 校验", "签名校验", "encodeURIComponent" ]
  description: 外部唤起参数、Universal Link、URL Scheme、App Link 或分享链接未校验路径与参数，可导致开放重定向、越权页面或恶意参数注入。
  fixHint: 入口做路径、来源和参数白名单校验；敏感动作二次确认。

- id: SEC-013
  title: 移动端明文传输或域名校验缺失
  level: 严重
  rootCauseKey: SEC-013:mobile-transport-unverified
  signals:
    keywords: [ "http://", "URLSession", "OkHttpClient", "HostnameVerifier", "TrustManager" ]
    whitelist: [ "HTTPS", "项目统一域名配置", "证书校验" ]
  description: 原生请求、资源或桥接回调使用明文 HTTP，或 TLS 证书、HostnameVerifier 和环境边界未按项目配置收敛。
  fixHint: 统一 HTTPS 与受控域名；正确配置证书校验并按环境隔离。
```

### 5.2 PRV · 隐私与合规

```yaml
- id: PRV-001
  title: 个人敏感信息未脱敏展示或导出
  level: 严重
  rootCauseKey: PRV-001:pii-not-masked
  signals:
    keywords: [ "phone", "idCard", "bankCard", "身份证", "手机号", "TextField", "TextView" ]
    whitelist: [ "maskPhone", "desensitize", "脱敏工具" ]
  description: 手机号、身份证、银行卡等在原生界面、日志、分享、截图或导出文件中未脱敏。
  fixHint: 展示统一脱敏工具；完整值仅在授权场景按需请求。

- id: PRV-002
  title: 埋点 / 监控 SDK 上报敏感数据
  level: 严重
  rootCauseKey: PRV-002:tracking-sensitive-data
  signals:
    keywords: [ "track(", "Sentry.setUser", "captureException", "FirebaseAnalytics", "Crashlytics" ]
  description: 埋点或崩溃监控把 Token、身份证、手机号、设备标识等上报到第三方。
  fixHint: 上报前统一过滤敏感字段；监控 SDK 配置脱敏钩子。

- id: PRV-003
  title: 错误提示暴露内部信息
  level: 主要
  rootCauseKey: PRV-003:internal-info-exposed
  signals:
    keywords: [ "error.localizedDescription", "Throwable.message", "error.stack", "showError" ]
    whitelist: [ "统一错误文案", "错误码映射" ]
  description: 接口地址、堆栈、内部结构或原始异常直接透出到用户可见的错误提示。
  fixHint: 用户侧展示业务文案；技术细节仅进监控。

- id: PRV-004
  title: 设备权限请求缺少告知与最小化
  level: 严重
  rootCauseKey: PRV-004:device-permission-overreach
  signals:
    keywords: [ "requestAuthorization", "requestWhenInUseAuthorization", "checkSelfPermission", "requestPermissions", "CAMERA", "RECORD_AUDIO" ]
    whitelist: [ "用途说明", "用户主动触发", "最小权限", "拒绝降级" ]
  description: 未说明用途、未由用户主动触发或一次性申请与功能无关的系统权限，造成隐私与合规风险。
  fixHint: 按功能按需申请；拒绝后提供可用降级路径。

- id: PRV-005
  title: 敏感信息写入剪贴板或截图区域
  level: 主要
  rootCauseKey: PRV-005:sensitive-clipboard-or-screenshot
  signals:
    keywords: [ "UIPasteboard", "ClipboardManager", "setPrimaryClip", "UIGraphicsImageRenderer", "MediaProjection" ]
    whitelist: [ "用户明确复制", "脱敏展示", "复制后清理" ]
  description: Token、密码、完整身份证号等敏感信息被静默写入系统剪贴板、分享内容或可被截屏的界面区域。
  fixHint: 仅在用户明确操作时复制；默认脱敏并设置清理策略。
```

### 5.3 BIZ · 业务逻辑正确性

```yaml
- id: BIZ-001
  title: 条件分支错误
  level: 主要
  rootCauseKey: BIZ-001:branch-logic-error
  signals:
    regex:
      - if\s*\([^)]*==\s*null[^)]*&&
    keywords: [ "TODO", "FIXME" ]
  description: 逻辑运算符误用、空值判断顺序颠倒、分支缺失。
  fixHint: 明确前置条件；补齐 else 分支；用例覆盖每条分支。

- id: BIZ-002
  title: 金额 / 浮点精度错误
  level: 严重
  rootCauseKey: BIZ-002:money-precision
  signals:
    keywords: [ "* 100", "/ 100", "toFixed(", "parseFloat(" ]
    whitelist: [ "Big.js", "decimal.js", "currency.js", "分为单位整数运算" ]
  description: 金额用浮点直接运算或依赖 toFixed 舍入，出现 0.1+0.2 类误差。
  fixHint: 金额按分为单位整数运算或用 decimal 库；展示层再格式化。

- id: BIZ-003
  title: 时间 / 时区 / 日期边界错误
  level: 主要
  rootCauseKey: BIZ-003:time-timezone-boundary
  signals:
    keywords: [ "new Date('", "getTime()", "toISOString(" ]
    whitelist: [ "dayjs(", "统一日期工具" ]
  description: 字符串直接解析日期存在系统版本、Locale 或时区差异；时区混用；区间闭开错误。
  fixHint: 使用明确格式的 DateFormatter/java.time；跨时区以后端为准；区间左闭右开。

- id: BIZ-004
  title: 提交字段映射 / 取值错误
  level: 严重
  rootCauseKey: BIZ-004:payload-field-mapping
  signals:
    keywords: [ "params", "data", "form" ]
  description: 表单字段与接口契约不一致、单位或枚举取值错误，写入错误数据。
  fixHint: 对照接口文档逐字段核对名称、类型、单位与枚举。

- id: BIZ-005
  title: 边界 / 空数据处理错误
  level: 主要
  rootCauseKey: BIZ-005:boundary-error
  signals:
    regex:
      - \.length\s*-\s*1
    keywords: [ "[0]", "split(" ]
  description: off-by-one、空数组取下标、分页边界、除零未防护。
  fixHint: 空集合走占位分支；下标访问前判长度；边界值单独用例。

- id: BIZ-006
  title: 状态流转 / 操作可用性错误
  level: 主要
  rootCauseKey: BIZ-006:state-transition
  signals:
    keywords: [ "status", "state", "disabled" ]
  description: 未按前置状态控制操作可用性，允许非法流转或重复流转。
  fixHint: 状态与可用操作集中映射管理；操作前校验当前状态。

- id: BIZ-007
  title: 参数组合不合法未校验
  level: 主要
  rootCauseKey: BIZ-007:param-combination-invalid
  signals:
    keywords: [ "rules", "validator" ]
  description: 单字段合法但组合非法（如开始时间晚于结束时间）未做跨字段校验。
  fixHint: 表单增加跨字段 validator；提交前统一组合校验。

- id: BIZ-008
  title: 危险操作缺少二次确认
  level: 主要
  rootCauseKey: BIZ-008:dangerous-action-no-confirm
  signals:
    keywords: [ "handleDelete", "batchDelete", "remove(" ]
    whitelist: [ "$confirm", "ElMessageBox.confirm" ]
  description: 删除、批量变更等不可逆操作无确认，或确认取消后仍执行。
  fixHint: 危险操作统一确认弹窗；确认结果必须阻断后续调用。
```

### 5.4 ROB · 健壮性

```yaml
- id: ROB-001
  title: 空值 / Optional 访问
  level: 主要
  rootCauseKey: ROB-001:null-undefined-access
  signals:
    regex:
      - (response|result|data)\.[A-Za-z0-9_]+\.[A-Za-z0-9_]+
    keywords: [ "force unwrap", "!!", "as!", "first()", "get(0)" ]
    whitelist: [ "?.", "??", "guard let", "requireNotNull" ]
  description: 深层取值、强制解包或集合下标未防空，接口异常或边界数据导致崩溃。
  fixHint: 使用 guard/可选链/默认值；集合访问前校验边界。

- id: ROB-002
  title: 异步异常无兜底
  level: 主要
  rootCauseKey: ROB-002:async-error-unhandled
  signals:
    keywords: [ "async", "await", "launch", "async(", "Task {" ]
    whitelist: [ "do-catch", "try", "Result", "CoroutineExceptionHandler" ]
  description: Swift concurrency、协程、Future 或异步回调失败后无人处理，导致状态悬挂、崩溃或静默失败。
  fixHint: 异步链路统一处理错误；失败给出可感知反馈并收口状态。

- id: ROB-003
  title: 失败分支状态未复位
  level: 主要
  rootCauseKey: ROB-003:state-not-reset-on-failure
  signals:
    keywords: [ "isLoading", "isSubmitting", "isEnabled", "setLoading", "finally" ]
    whitelist: [ "defer", "finally", "onCompletion" ]
  description: 加载、提交锁、按钮状态或页面占用状态在失败、取消或超时分支未复位，界面持续不可操作。
  fixHint: 用 defer/finally/完成回调统一收口成功、失败和取消分支。

- id: ROB-004
  title: 类型转换 / 隐式比较错误
  level: 主要
  rootCauseKey: ROB-004:type-coercion-error
  signals:
    keywords: [ "as!", "toInt(", "Integer(", "== null", "isAssignableFrom" ]
    whitelist: [ "as?", "guard let", "toIntOrNull", "=== null" ]
  description: 强制类型转换、隐式类型转换或数值比较不符合数据契约，导致崩溃或业务判断错误。
  fixHint: 使用可失败转换；比较前明确类型、单位和空值语义。

- id: ROB-005
  title: 请求超时 / 取消缺失
  level: 主要
  rootCauseKey: ROB-005:timeout-cancel-missing
  signals:
    keywords: [ "URLSession", "OkHttpClient", "Retrofit", "Alamofire", "Call" ]
    whitelist: [ "timeoutInterval", "withTimeout", "cancel()", "dispose()" ]
  description: 长请求无超时，页面离开、生命周期结束或任务取消后仍占用线程和网络资源。
  fixHint: 统一网络层设超时；生命周期结束时取消并释放请求。

- id: ROB-006
  title: 失败自动重试 / 轮询无边界
  level: 严重
  rootCauseKey: ROB-006:retry-unbounded
  signals:
    keywords: [ "retry", "Timer.scheduledTimer", "Handler.postDelayed", "轮询" ]
    whitelist: [ "maxRetries", "指数退避", "invalidate()", "removeCallbacks" ]
  description: 失败重试、定时器或轮询无最大次数、退避和生命周期停止条件，形成请求风暴或后台耗电。
  fixHint: 重试设上限加退避；离开页面或进入后台立即停止。

- id: ROB-007
  title: JSON / 存储解析未防错
  level: 次要
  rootCauseKey: ROB-007:unsafe-parse
  signals:
    keywords: [ "JSON.parse(" ]
    whitelist: [ "try {", "安全解析工具" ]
  description: JSONDecoder、JSONObject、Bundle 或本地存储内容直接强制解析，脏数据即抛错或丢失状态。
  fixHint: 解析包 try/catch 并给默认值；封装安全解析工具。

- id: ROB-008
  title: 弱网 / 离线状态无恢复策略
  level: 主要
  rootCauseKey: ROB-008:offline-recovery-missing
  signals:
    keywords: [ "NWPathMonitor", "ConnectivityManager", "NetworkCallback", "offline", "retry" ]
    whitelist: [ "离线占位", "手动重试", "指数退避", "幂等键" ]
  description: 网络抖动、断网或恢复后没有可感知状态、重试边界、请求取消或数据恢复策略，用户误以为提交成功或界面卡死。
  fixHint: 明确离线态与重试入口；写操作保证幂等和结果可恢复。

- id: ROB-009
  title: 前后台切换状态未恢复
  level: 主要
  rootCauseKey: ROB-009:foreground-background-state-loss
  signals:
    keywords: [ "sceneDidEnterBackground", "sceneWillEnterForeground", "applicationDidEnterBackground", "onStart", "onStop", "onResume" ]
    whitelist: [ "恢复任务", "取消过期请求", "状态快照" ]
  description: 应用进入后台、页面隐藏或恢复前台后仍使用过期数据、重复提交、遗留任务或锁状态，导致状态错乱。
  fixHint: 前后台切换收口请求与订阅；恢复时校验数据新鲜度。
```

### 5.5 CON · 异步与竞态

```yaml
- id: CON-001
  title: 请求竞态覆盖（过期响应回写）
  level: 严重
  rootCauseKey: CON-001:stale-response-overwrite
  signals:
    keywords: [ "URLSessionDataTask", "Job", "DispatchWorkItem", "requestId" ]
    whitelist: [ "cancel()", "请求序号比对", "最新请求标记" ]
  description: 快速切换页面、筛选或请求时旧响应后到，覆盖新状态或回写已销毁对象。
  fixHint: 发新请求前取消旧任务，或以请求序号丢弃过期响应。

- id: CON-002
  title: 重复提交未防护
  level: 严重
  rootCauseKey: CON-002:duplicate-submit
  signals:
    keywords: [ "onClick", "setOnClickListener", "submit", "send(" ]
    whitelist: [ "loading 锁", "isEnabled = false", "防抖", "幂等键" ]
  description: 提交、支付或危险操作可被连点，产生重复请求与重复数据。
  fixHint: 操作期间禁用入口；必要时配合后端幂等键。

- id: CON-003
  title: 高频事件未防抖 / 节流
  level: 主要
  rootCauseKey: CON-003:debounce-throttle-missing
  signals:
    keywords: [ "scrollViewDidScroll", "doOnTextChanged", "addTextChangedListener", "onSensorChanged" ]
    whitelist: [ "debounce", "throttle", "sample", "removeObserver" ]
  description: 滚动、输入、布局、传感器或回调直接触发请求和重计算，高频抖动造成卡顿或耗电。
  fixHint: 输入防抖、滚动/传感器节流；生命周期结束时移除监听。

- id: CON-004
  title: 并行异步结果处理错误
  level: 主要
  rootCauseKey: CON-004:parallel-async-mishandled
  signals:
    keywords: [ "TaskGroup", "async let", "DispatchGroup", "awaitAll", "joinAll" ]
    whitelist: [ "supervisorScope", "Result", "部分失败处理" ]
  description: 并行任务部分失败未处理、线程组未等待或取消未传播，导致状态不一致和资源泄漏。
  fixHint: 明确部分失败策略；等待、取消和异常传播必须成对设计。

- id: CON-005
  title: 页面切换后过期响应回写
  level: 严重
  rootCauseKey: CON-005:page-lifecycle-stale-response
  signals:
    keywords: [ "viewDidDisappear", "onDestroy", "onStop", "URLSessionDataTask", "Job" ]
    whitelist: [ "取消旧请求", "请求序号比对", "页面仍激活" ]
  description: 页面离开、Fragment/Activity 销毁或前后台切换后，旧任务仍回写已销毁或非当前页面状态。
  fixHint: 生命周期结束时取消任务；回写前校验实例和请求序号。
```

### 5.6 CMP · 状态与生命周期

```yaml
- id: CMP-001
  title: View 状态或共享模型越权修改
  level: 主要
  rootCauseKey: CMP-001:props-mutation
  signals:
    keywords: [ "inout", "var model", "state =", "mutableStateOf", "notifyDataSetChanged" ]
    whitelist: [ "单向数据流", "StateFlow", "ViewModel", "只读模型" ]
  description: View、ViewController/Activity 或共享模型直接越权修改状态，破坏单向数据流和生命周期边界。
  fixHint: 通过 ViewModel/事件更新状态；共享模型保持明确读写边界。

- id: CMP-002
  title: 列表标识缺失或不稳定
  level: 主要
  rootCauseKey: CMP-002:vfor-unstable-key
  signals:
    regex:
      - (ForEach|LazyColumn|RecyclerView)[^\n]*(index|position)
    keywords: [ "IndexPath", "getItemId", "key = { index }" ]
  description: 列表使用位置索引代替稳定业务标识，增删排序或复用后行状态与数据错位。
  fixHint: 使用稳定业务主键或实现稳定 getItemId；避免位置作为身份。

- id: CMP-003
  title: 状态观察循环或副作用
  level: 严重
  rootCauseKey: CMP-003:reactive-loop
  signals:
    keywords: [ "observe", "collect", "didSet", "onChange", "LaunchedEffect" ]
  description: 状态观察回调再次修改被观察源，或在派生状态中发请求、改状态，形成循环和重复任务。
  fixHint: 派生状态保持纯函数；观察回调必须有明确收敛条件。

- id: CMP-004
  title: 副作用未清理（监听 / 定时器 / Observer）
  level: 主要
  rootCauseKey: CMP-004:side-effect-not-cleaned
  signals:
    keywords: [ "NotificationCenter.addObserver", "registerReceiver", "Timer.scheduledTimer", "CoroutineScope", "addTextChangedListener" ]
    whitelist: [ "removeObserver", "unregisterReceiver", "invalidate()", "cancel()", "dispose()" ]
  description: 监听、定时器、广播、协程、Observer 或事件总线订阅未随界面销毁清理，造成泄漏或跨页串扰。
  fixHint: 生命周期钩子成对注册与清理；统一封装自动取消。

- id: CMP-005
  title: 全局状态越界 / 重复维护
  level: 主要
  rootCauseKey: CMP-005:global-state-misuse
  signals:
    keywords: [ "Singleton", "static var", "GlobalScope", "SharedPreferences", "UserDefaults" ]
    whitelist: [ "Repository", "ViewModel", "单一数据源", "生命周期作用域" ]
  description: 界面直接改全局状态或同一事实多处重复维护（如登录态双份），导致身份和页面状态不一致。
  fixHint: 状态变更走明确的 ViewModel/Repository；同一事实只留一个权威来源。

- id: CMP-006
  title: 表单提交 / 交互回调未收口
  level: 主要
  rootCauseKey: CMP-006:avue-callback-not-closed
  signals:
    keywords: [ "submit", "onConfirm", "onCancel", "completion", "isLoading" ]
  description: 表单提交、权限授权、媒体选择、系统回调或弹窗只处理成功分支，失败、取消和超时后状态未恢复。
  fixHint: 成功、失败、取消、超时与 completion 分支统一收口状态。

- id: CMP-007
  title: 状态观察或绑定失效
  level: 主要
  rootCauseKey: CMP-007:reactivity-broken
  signals:
    keywords: [ "notifyDataSetChanged", "reloadData", "mutableStateOf", "LiveData", "StateFlow" ]
    whitelist: [ "DiffUtil", "snapshotFlow", "observeAsState", "主线程更新" ]
  description: 状态变更未通知界面、在错误线程更新 UI，或绑定对象生命周期已失效，导致界面陈旧或崩溃。
  fixHint: 使用平台状态容器和主线程更新；绑定随生命周期收集。

- id: CMP-008
  title: 生命周期误用
  level: 次要
  rootCauseKey: CMP-008:lifecycle-misuse
  signals:
    keywords: [ "viewDidLoad", "viewWillAppear", "onCreate", "onStart", "onResume" ]
  description: 在错误生命周期阶段访问界面、重复初始化任务，或在 ViewController/Activity 已销毁后继续使用上下文。
  fixHint: 按平台生命周期放置初始化和刷新；销毁时释放界面与任务。

- id: CMP-009
  title: 移动页面生命周期订阅重复或未清理
  level: 主要
  rootCauseKey: CMP-009:mobile-lifecycle-subscription-leak
  signals:
    keywords: [ "viewWillAppear", "viewWillDisappear", "onStart", "onStop", "onResume", "onPause" ]
    whitelist: [ "生命周期清理", "removeObserver", "订阅幂等" ]
  description: 界面反复出现、隐藏和恢复时重复注册监听、消息通道、协程或定时器，或销毁时未清理。
  fixHint: 生命周期成对注册与清理；重复进入保持订阅幂等。
```

### 5.7 UI · 界面与可访问性

```yaml
- id: UI-001
  title: 全局主题或样式污染
  level: 主要
  rootCauseKey: UI-001:global-style-pollution
  signals:
    keywords: [ "UIAppearance", "appearance", "setBackgroundColor", "MaterialTheme", "Theme.Material" ]
    whitelist: [ "局部主题", "ViewController 作用域", "主题覆盖说明" ]
  description: 通过全局 Appearance、主题或 Window 样式修改局部界面，导致其他页面或系统组件视觉回归。
  fixHint: 主题配置集中管理；局部样式限定 View/Activity 范围。

- id: UI-002
  title: 页面状态缺失（loading/empty/error/无权限）
  level: 主要
  rootCauseKey: UI-002:page-state-missing
  signals:
    keywords: [ "ProgressView", "CircularProgressIndicator", "placeholder", "ErrorView", "EmptyView" ]
  description: 缺少加载、空数据、错误、无权限、重试等必要界面状态，异常时界面空白或用户无法恢复。
  fixHint: 页面与列表补齐加载、空、错、无权限态；异常提供重试入口。

- id: UI-003
  title: 布局溢出 / 遮挡
  level: 主要
  rootCauseKey: UI-003:layout-overflow
  signals:
    keywords: [ "NSLayoutConstraint", "layoutIfNeeded", "clipToBounds", "WindowInsets", "adjustResize" ]
  description: 约束、Insets、键盘或屏幕尺寸变化导致界面遮挡、裁剪、内容撑破或关键操作不可见。
  fixHint: 使用约束和 Insets 适配；覆盖键盘、横竖屏和长文本场景。

- id: UI-004
  title: 可访问性缺失
  level: 次要
  rootCauseKey: UI-004:accessibility-missing
  signals:
    keywords: [ "accessibilityLabel", "contentDescription", "UIAccessibility", "importantForAccessibility" ]
  description: 控件缺少可识别标签、操作顺序或状态描述，VoiceOver/TalkBack/键盘等辅助技术无法完成主要操作。
  fixHint: 补齐 accessibilityLabel/contentDescription、焦点顺序和状态播报。

- id: UI-005
  title: 安全区与系统栏适配缺失
  level: 次要
  rootCauseKey: UI-005:mobile-rules-in-desktop
  signals:
    keywords: [ "safeAreaInsets", "WindowInsets", "fitsSystemWindows", "statusBar", "navigationBar" ]
  description: 固定头部、底部操作栏或全屏页面未处理刘海、状态栏、导航栏和底部手势区域，关键内容被遮挡。
  fixHint: 使用 safeAreaInsets/WindowInsets；固定栏补齐系统栏内边距。

- id: UI-006
  title: 触控目标或手势交互不适配
  level: 主要
  rootCauseKey: UI-006:touch-target-gesture-mismatch
  signals:
    keywords: [ "UITapGestureRecognizer", "UIPanGestureRecognizer", "onTouchEvent", "onInterceptTouchEvent", "setOnClickListener" ]
    whitelist: [ "触控目标规范", "gesture recognizer", "点击反馈", "performHapticFeedback" ]
  description: 关键控件触控区域过小、滚动与横滑手势冲突、事件穿透或没有按平台提供触觉与视觉反馈。
  fixHint: 扩大可点击区域；明确手势优先级并验证取消与反馈。

- id: UI-007
  title: 软键盘遮挡输入与提交操作
  level: 主要
  rootCauseKey: UI-007:keyboard-viewport-obstruction
  signals:
    keywords: [ "keyboardWillShow", "keyboardDidShow", "WindowInsets", "adjustResize", "contentInset" ]
    whitelist: [ "键盘适配", "安全区变量", "聚焦滚动" ]
  description: iOS/Android 软键盘弹起后输入框、错误提示或提交按钮被遮挡，或窗口 Insets 处理导致不可滚动。
  fixHint: 使用键盘通知/WindowInsets 适配；聚焦时滚动到可视区域。

- id: UI-008
  title: 横竖屏与系统字体缩放兼容缺失
  level: 次要
  rootCauseKey: UI-008:orientation-font-scale-missing
  signals:
    keywords: [ "orientation", "UIContentSizeCategory", "fontScale", "configuration.orientation" ]
    whitelist: [ "Auto Layout", "约束布局", "横屏布局", "字体缩放测试" ]
  description: 横竖屏切换、系统大字体或动态字体开启后出现裁剪、重叠或文字不可读。
  fixHint: 使用约束布局和可换行文本；覆盖方向与字体缩放场景。
```

### 5.8 PERF · 性能

```yaml
- id: PERF-001
  title: 循环内逐条请求
  level: 主要
  rootCauseKey: PERF-001:request-in-loop
  signals:
    keywords: [ "forEach(async", "map(async", "for" ]
  description: 循环内逐条发起网络请求、磁盘访问或图片解码，延迟叠加且占用线程与电量。
  fixHint: 改批量接口或合并任务；确需并行用受控并发。

- id: PERF-002
  title: 重复请求未合并 / 未缓存
  level: 主要
  rootCauseKey: PERF-002:duplicate-request
  signals:
    keywords: [ "viewDidLoad", "onCreate", "load", "fetch", "Repository" ]
  description: 同一数据被多个界面或重复生命周期回调反复拉取，公共数据无缓存或请求合并。
  fixHint: 公共数据入 Repository/缓存；同参请求合并并设置失效策略。

- id: PERF-003
  title: 大列表无分页 / 虚拟化
  level: 主要
  rootCauseKey: PERF-003:large-list-unbounded
  signals:
    keywords: [ "RecyclerView", "LazyColumn", "List", "pageSize", "fetchAll" ]
  description: 一次拉取并创建全量数据、图片或 View，滚动卡顿、内存飙升甚至崩溃。
  fixHint: 接口分页；列表使用复用/虚拟化并限制单页数据量。

- id: PERF-004
  title: 大依赖整包引入 / 未懒加载
  level: 主要
  rootCauseKey: PERF-004:heavy-import
  signals:
    keywords: [ "import *", "framework", "Firebase", "GoogleMap", "Realm" ]
    whitelist: [ "按需引入", "模块化依赖", "构建产物分析" ]
  description: 整包引入重依赖、启动阶段无条件初始化不必要 SDK，启动时间和包体显著膨胀。
  fixHint: 依赖按需引入；延迟初始化非首屏能力；用构建产物验证。

- id: PERF-005
  title: 内存 / 资源泄漏
  level: 主要
  rootCauseKey: PERF-005:resource-leak
  signals:
    keywords: [ "Data(contentsOf:", "BitmapFactory", "UIImage(data:", "MediaPlayer", "AVAudioPlayer" ]
    whitelist: [ "recycle()", "release()", "close()", "deinit", "onCleared" ]
  description: 图片、媒体、数据库、文件句柄、线程或订阅未释放，大对象常驻导致内存持续增长。
  fixHint: 生命周期结束时释放资源；大对象采用流式处理并避免长时间持有。

- id: PERF-006
  title: 渲染抖动 / 不必要重渲染
  level: 次要
  rootCauseKey: PERF-006:render-thrash
  signals:
    keywords: [ "notifyDataSetChanged", "reloadData", "layoutIfNeeded", "StateFlow" ]
  description: 频繁整体刷新大列表、重复触发布局或广播全量状态，造成主线程抖动和无必要重绘。
  fixHint: 使用差量更新；合并刷新并把计算移出主线程。

- id: PERF-007
  title: 静态资源未优化
  level: 次要
  rootCauseKey: PERF-007:asset-unoptimized
  signals:
    keywords: [ ".png", ".jpg", "base64", "BitmapFactory.decodeResource", "UIImage(named:" ]
    whitelist: [ "压缩资源", "采样解码", "缓存策略" ]
  description: 大图、动画、字体或音视频资源未压缩、未按需加载或一次性解码，造成包体、内存和启动压力。
  fixHint: 压缩并按需解码；使用缓存和资源分级策略。

- id: PERF-008
  title: 移动端启动或安装包过重
  level: 主要
  rootCauseKey: PERF-008:mobile-first-screen-heavy
  signals:
    keywords: [ "application(_:didFinishLaunching", "Application.onCreate", "startup", "MainActivity", "AppDelegate" ]
    whitelist: [ "延迟初始化", "资源预加载策略", "构建体积分析" ]
  description: 启动阶段同步初始化重依赖、数据库、图片或远程配置，或安装包包含大量无关资源，导致启动和首屏交互过慢。
  fixHint: 延迟初始化非关键能力；清理无关资源并用构建数据验证。

- id: PERF-009
  title: 定位 / 传感器 / Bridge 高频调用
  level: 主要
  rootCauseKey: PERF-009:sensor-bridge-frequency
  signals:
    keywords: [ "startUpdatingLocation", "watchPosition", "SensorManager", "TYPE_ACCELEROMETER", "Handler.postDelayed" ]
    whitelist: [ "采样节流", "页面可见时订阅", "停止监听" ]
  description: 定位、传感器、推送或 WebView Bridge 在后台或不可见界面持续高频运行，造成电量、性能和隐私风险。
  fixHint: 按页面可见性启停并节流；离开页面立即释放能力。
```

### 5.9 LOG · 日志与调试

```yaml
- id: LOG-001
  title: 敏感信息输出到控制台 / 日志
  level: 严重
  rootCauseKey: LOG-001:sensitive-in-console
  signals:
    regex:
      - (print|NSLog|os_log|Log\.[divew])\s*\([^)]*(token|password|idCard|phone|secret)
    keywords: [ "print(response)", "NSLog", "Log.d", "Log.e" ]
  description: 原生日志打印 Token、密码、身份证、完整响应、设备标识或用户隐私内容。
  fixHint: 删除敏感打印；必要日志走统一封装并脱敏。

- id: LOG-002
  title: 异常被吞
  level: 主要
  rootCauseKey: LOG-002:swallow-exception
  signals:
    regex:
      - catch\s*(\([^)]*\))?\s*\{\s*\}
    keywords: [ "catch { }", "catch (e) { }", "try?" ]
  description: 空 catch、try? 或异常回调不提示不上报，失败被静默吞掉。
  fixHint: 记录脱敏上下文并给出用户反馈；不可恢复的错误继续抛出。

- id: LOG-003
  title: 调试代码残留
  level: 次要
  rootCauseKey: LOG-003:debug-code-left
  signals:
    keywords: [ "print(", "NSLog(", "Log.d(", "Log.v(", "debugger" ]
    whitelist: [ "构建剔除配置", "统一日志封装", "DEBUG 条件编译" ]
  description: 调试打印、断点或测试开关残留到生产构建路径。
  fixHint: 提交前清理；生产构建统一关闭调试日志和断点。
```

### 5.10 CFG · 配置

```yaml
- id: CFG-001
  title: 接口地址 / 环境配置硬编码
  level: 主要
  rootCauseKey: CFG-001:config-hardcoded
  signals:
    regex:
      - "http[s]?://[a-zA-Z0-9.-]+(:\\d+)?"
    keywords: [ "localhost:", "127.0.0.1", "192.168." ]
    whitelist: [ "BuildConfig", "xcconfig", "Info.plist 配置", "环境配置" ]
  description: 接口地址、端口、第三方地址或环境凭证硬编码进原生源码或资源。
  fixHint: 收敛到 xcconfig/BuildConfig/受控配置；按环境隔离。

- id: CFG-002
  title: 环境判断散落业务代码
  level: 严重
  rootCauseKey: CFG-002:env-config-mixed
  signals:
    keywords: [ "#if DEBUG", "BuildConfig.DEBUG", "flavor", "xcconfig", "environment" ]
    whitelist: [ "构建配置内环境分支", "统一环境工具" ]
  description: 业务代码散落环境分支，或测试地址、调试证书和开关混入生产构建。
  fixHint: 环境差异收敛到构建配置层；业务代码不感知环境。

- id: CFG-003
  title: Mock / 调试开关残留
  level: 主要
  rootCauseKey: CFG-003:debug-switch-left
  signals:
    keywords: [ "mock = true", "if (true)", "// debug", "isMock", "BuildConfig.DEBUG" ]
  description: Mock 开关、临时强制分支、调试代理或测试账号残留到生产路径。
  fixHint: 上线前关闭 Mock；开关走构建配置并默认关闭。

- id: CFG-004
  title: 公共构建 / 工程配置破坏性修改
  level: 主要
  rootCauseKey: CFG-004:build-config-broken
  signals:
    keywords: [ "xcodebuild", "Build Settings", "build.gradle", "build.gradle.kts", "lintOptions", "proguardFiles" ]
  description: 为个别模块关闭静态检查、修改公共构建配置、签名、混淆或网络安全策略，影响全工程。
  fixHint: 公共配置改动需说明影响面；局部问题局部解决。

- id: CFG-005
  title: 原生入口 / 权限或平台配置漂移
  level: 主要
  rootCauseKey: CFG-005:platform-config-drift
  signals:
    keywords: [ "Info.plist", "Entitlements", "AndroidManifest.xml", "uses-permission", "intent-filter", "BuildConfig" ]
    whitelist: [ "平台矩阵说明", "配置校验脚本", "按平台隔离" ]
  description: App 入口、权限声明、Intent/Universal Link、后台能力、签名或构建变体与代码不一致，导致特定系统能力不可用或过度授权。
  fixHint: 配置与代码同源校验；逐平台构建并检查权限、签名和能力清单。
```

### 5.11 DEP · 依赖

```yaml
- id: DEP-001
  title: 已知漏洞版本依赖
  level: 严重
  rootCauseKey: DEP-001:vulnerable-dependency
  signals:
    keywords: [ "pod outdated", "swift package", "gradle dependencies", "已知 CVE 版本" ]
  description: CocoaPods、Swift Package、Maven 或 Gradle 依赖版本存在公开 CVE 且社区已发布修复版。
  fixHint: 升级到修复版；不可升级时评估替代或缓解方案。

- id: DEP-002
  title: 版本漂移 / 锁文件缺失
  level: 主要
  rootCauseKey: DEP-002:version-drift
  signals:
    keywords: [ "latest", "SNAPSHOT", "version.ref", "Package.resolved", "Podfile.lock" ]
    whitelist: [ "锁文件已提交", "版本目录统一管理" ]
  description: 依赖版本漂移、使用 SNAPSHOT/latest，或 Package.resolved/Podfile.lock/Gradle 锁定信息缺失，构建不可复现。
  fixHint: 固定版本并提交锁定文件；依赖版本集中管理。

- id: DEP-003
  title: 冗余 / 重复依赖
  level: 次要
  rootCauseKey: DEP-003:redundant-dependency
  signals:
    keywords: [ "Alamofire", "URLSession", "OkHttp", "Retrofit", "Gson", "Moshi" ]
  description: 同类网络、JSON、图片或日志库并存，或引入未使用的原生依赖，增加包体和维护成本。
  fixHint: 同类库收敛为项目既定一个；定期清理未用依赖。
```

### 5.12 API · 接口调用

```yaml
- id: API-001
  title: 绕过统一网络层
  level: 主要
  rootCauseKey: API-001:bypass-request-layer
  signals:
    keywords: [ "URLSession.shared", "URLSession(configuration:", "OkHttpClient(", "Retrofit.Builder", "HttpURLConnection" ]
    whitelist: [ "项目统一网络封装", "平台请求适配层" ]
  description: 自建网络客户端或直接调用平台请求 API，绕过 Token、环境域名、错误处理、证书校验和弱网策略。
  fixHint: 一律走项目统一网络封装；特殊需求在封装层扩展。

- id: API-002
  title: URL/method/params 与契约不一致
  level: 严重
  rootCauseKey: API-002:contract-mismatch
  signals:
    keywords: [ "httpMethod", "URLRequest", "Request.Builder", "@GET", "@POST" ]
  description: HTTP 方法、URL、请求体、Header、字段与接口文档不一致。
  fixHint: 对照接口文档逐项核对方法、路径、Header、字段、类型和单位。

- id: API-003
  title: 响应层级 / 错误码处理错误
  level: 主要
  rootCauseKey: API-003:response-handling-error
  signals:
    keywords: [ "response.data", "response.body", "statusCode", "isSuccessful" ]
  description: 响应解码层级错误、未按业务 code 判定成功、把 HTTP 200/2xx 直接当业务成功。
  fixHint: 按项目响应结构统一取值；成功判定沿用既有约定。

- id: API-004
  title: 列表查询分页或增量参数缺失
  level: 主要
  rootCauseKey: API-004:pagination-missing
  signals:
    keywords: [ "page", "pageSize", "limit", "offset", "loadMore" ]
  description: 列表查询不传分页、游标或增量参数，或把 limit 设为极大值变相全量。
  fixHint: 列表使用分页/游标；全量需求走导出或专用接口。

- id: API-005
  title: 接口地址 / 页面权限或平台能力靠猜测
  level: 主要
  rootCauseKey: API-005:api-guessed
  signals:
    keywords: [ "/api/", "permission", "scope.", "Info.plist", "AndroidManifest" ]
  description: 未核对接口文档、原生权限或平台能力配置，凭经验编造 URL、参数、权限编码或能力范围。
  fixHint: 以接口文档与平台配置为准；资料缺失时先确认。

- id: API-006
  title: 移动平台请求域名 / Header 配置不一致
  level: 主要
  rootCauseKey: API-006:mobile-request-platform-config
  signals:
    keywords: [ "baseURL", "baseUrl", "Host", "Cookie", "Authorization", "Network Security Config" ]
    whitelist: [ "统一网络封装", "平台域名清单", "环境配置" ]
  description: iOS/Android 不同构建变体的域名、Header、Cookie、Token、ATS/Network Security Config 或证书策略不一致，导致请求失败或绕过安全策略。
  fixHint: 统一网络封装和平台配置；按环境验证域名、凭证和证书策略。
```

### 5.13 QUAL · 代码质量

```yaml
- id: QUAL-001
  title: 大段复制粘贴
  level: 主要
  rootCauseKey: QUAL-001:copy-paste-code
  signals:
    keywords: [ "// copy", "同上" ]
  description: 多处代码高度相似，修改遗漏导致行为分叉。
  fixHint: 抽公共函数、UseCase、Repository 或可复用 View。

- id: QUAL-002
  title: 死代码 / 注释代码块
  level: 次要
  rootCauseKey: QUAL-002:dead-code
  signals:
    keywords: [ "// eslint-disable", "unused" ]
  description: 未使用的类、View、函数、变量、资源或成块注释代码长期留存。
  fixHint: 直接删除，历史靠 Git 追溯。

- id: QUAL-003
  title: 上帝类 / 超长方法
  level: 主要
  rootCauseKey: QUAL-003:god-component
  signals:
    keywords: [ "UIViewController", "Activity", "Fragment", "ViewModel", "Repository" ]
  description: 单个 ViewController、Activity、Fragment、ViewModel 或类职责混杂、体量过大（> 800 行）或单方法过长（> 80 行）。
  fixHint: 按职责拆分界面、状态、网络和数据层；下沉纯逻辑。

- id: QUAL-004
  title: 圈复杂度过高
  level: 主要
  rootCauseKey: QUAL-004:high-complexity
  signals:
    keywords: [ "if", "else if", "switch" ]
  description: 深层嵌套 if、三元链，单函数圈复杂度过高难测试。
  fixHint: 卫语句提前返回；分支表或策略映射替代深嵌套。

- id: QUAL-005
  title: 魔法值 / 硬编码字面量
  level: 次要
  rootCauseKey: QUAL-005:magic-number
  signals:
    regex:
      - ===\s*\d{2,}
    keywords: [ "status === 1" ]
  description: 状态码、类型值等业务字面量散落各处，含义不清。
  fixHint: 提炼常量或枚举并集中管理。

- id: QUAL-006
  title: 命名 / 语义模糊
  level: 次要
  rootCauseKey: QUAL-006:naming-ambiguous
  signals:
    regex:
      - \b(temp|tmp|obj|info|flag|foo|bar|test)\d?\b
  description: 变量、函数、类、View、资源或组件使用无语义命名。
  fixHint: 命名体现业务含义；遵循项目命名规范。

- id: QUAL-007
  title: 复杂逻辑缺少必要注释
  level: 次要
  rootCauseKey: QUAL-007:comment-missing-stale
  signals:
    keywords: [ "reduce(", "正则", "位运算", "DispatchQueue", "CoroutineScope", "Bridge" ]
  description: 复杂业务、系统版本兼容、线程切换、平台桥接或安全分支缺少说明，或注释与实现不符。
  fixHint: 补充业务含义与关键分支注释；失效注释同步删除。
```

### 5.14 TEST · 测试与验证

```yaml
- id: TEST-001
  title: 核心变更缺少验证
  level: 主要
  rootCauseKey: TEST-001:core-change-unverified
  signals:
    keywords: [ "AppDelegate", "SceneDelegate", "Application", "URLSession", "ViewModel", "permission" ]
  description: App 入口、网络层、导航权限、全局状态、线程或平台配置等核心设施改动未运行项目已有校验或定向回归。
  fixHint: 按项目脚本执行 lint/static check/test/xcodebuild/Gradle 与定向回归。

- id: TEST-002
  title: 复杂纯逻辑无测试覆盖
  level: 次要
  rootCauseKey: TEST-002:pure-logic-untested
  signals:
    keywords: [ "utils", "工具函数", "Formatter", "Mapper", "Validator" ]
  description: 新增可单测的纯逻辑（计算、转换、校验、解析、映射）无任何测试。
  fixHint: 纯函数补单测覆盖主路径与边界。

- id: TEST-003
  title: 目标平台或设备矩阵未验证
  level: 主要
  rootCauseKey: TEST-003:device-platform-matrix-unverified
  signals:
    keywords: [ "iOS", "Android", "xcodebuild", "Gradle", "横屏", "真机", "模拟器", "API level" ]
    whitelist: [ "平台构建", "设备矩阵", "真机回归", "模拟器回归" ]
  description: 原生核心变更只完成静态检查或单一平台构建，没有覆盖项目声明的操作系统、最低版本、屏幕方向、弱网、权限或代表性设备状态。
  fixHint: 按项目支持矩阵执行 Xcode/Gradle 构建与代表性模拟器、真机回归。
```

---

## 六、AI 使用流程

AI 触发代码审查时，按以下步骤对照本表：

1. **对照识别**：扫描代码文件，逐段与本表 signals 对照，命中即候选。
2. **归一根因**：为每个候选计算 `rootCauseKey`，同键合并。
3. **计算位置指纹**：为每个 `rootCauseKey` 汇总 `locations[]`。
4. **描述归一去重**：对同键多描述做 `descriptionNorm` 归一，取最详尽一条作为主描述。
5. **级别裁定**：按本表 `level` 建议，可依上下文 ±1 级调整，多类别命中同位置按最高级别扣分（R4）。
6. **影响面上浮**：`locations` ≥ 3 处时按 R2 上浮一档。
7. **输出**：按 [scoring-model.md](scoring-model.md) 计分并按 [report-generation.md](report-generation.md) 结构输出，每条问题携带 `id / rootCauseKey / level / mustFix / locations / mergedFrom / evidence / fixHint`；所有已确认问题展示脱敏问题代码，`mustFix=true` 时不得省略。
8. **幂等校验**：重复扫描时对比上次 `rootCauseKey` 集合，不一致必须给出差异说明（新代码 / 误报修正）。

## 七、扩展与维护

- **新增条目**：末尾追加，编号连续，不复用已废弃编号；本表未收录的问题按临近分类临时定级，并在报告中说明属于码表外新增。
- **废弃条目**：保留编号并标 `deprecated: true`，说明原因与替代 id。
- **级别调整**：以本表 `level` 为建议基线；某项目/团队约定不同的，在项目根 `AGENTS.md` 或 `.codex/rules` 里做覆盖说明，不直接改本文件。
- **命中率反馈**：AI 生成报告时应记录本次命中的 id 集合，用于后续调整 signals 与级别。
