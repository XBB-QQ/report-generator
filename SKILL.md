---
name: "report-generator"
description: "智能护理表单模板生成器"
---

# Report Generator Skill - v7.1

## 硬性约束（违反即导入报错）

以下规则由实际模板验证得出，违反任一条都会导致导入时报错或运行时异常。

### H1: 顶层 `template` 字段必须是 JSON 字符串
- `d['template']` 的值必须是 `json.dumps(template_object, ensure_ascii=False)` 的结果
- 即字符串类型，不是嵌套对象
- 系统会先 `JSON.parse` 外层拿到 `template`（此时是字符串），再 `JSON.parse` 一次拿到模板对象
- 如果直接放嵌套对象，二次 parse 会报错

### H2: `source` 必须是纯二维数组
- 格式：`[[str|null, str|null, ...], ...]`
- 不允许写成 `{"rows": [...], "cols": {...}}` 等嵌套对象
- 所有值只能是字符串或 `null`，不能是数字、布尔值、对象

### H3: `meta` 必须是扁平列表
- 格式：`[{row, col, s, proxyCell, ...}, ...]`，一维列表
- 长度 = 行数 × 列数，每个元素对应一个单元格
- 不允许写成 `{"widgetIds": [...], "name": ...}` 等字典格式

### H4: `scopeConfig` 必须是列表
- 格式：`[{name, defaultValue, type, desc}, ...]`，列表
- 不是字典/对象
- 每个 `name` 必须与对应 widget 的 `scopeField` 完全匹配（区分大小写）

### H5: `eventConfig` 必须是列表且包含全部 7 个标准事件
- 格式：`[{eventName, expressionStatement}, ...]`，列表
- 必须包含：`beforeload`、`afterload`、`beforerender`、`afterrender`、`beforeprint`、`afterprint`、`childReportMsg`
- 即使脚本为空，也必须有 `{"eventName": "xxx", "expressionStatement": ""}`
- 不是字典，不是空数组

### H6: `resized.rows`/`resized.cols` 元素必须是纯数字
- `resized.rows` 是 `[22, 28, 22, ...]` 纯数字列表
- `resized.cols` 是 `[76, 76, ...]` 纯数字列表
- 不允许 `[{"height": 22}]` 对象格式

### H7: `merges` 元素必须是对象
- 每个 merge 元素必须是 `{"startRow": sr, "startColumn": sc, "endRow": er, "endColumn": ec}` 对象
- 不允许使用数组 `[sr, sc, er, ec]` 格式

### H8: `reportConfig` 必须从已验证模板深拷贝
- `reportConfig` 包含 14 个标准字段：`pageConfig`、`splitLayout`、`headerRepeat`、`footerRepeat`、`headerFrozen`、`followUpPrintOpt`、`scopeConfig`、`searchBarConfig`、`eventConfig`、`serviceConfig`、`headerOptions`、`printOptions`、`functionConfig`、`engineConfig`
- 生成新报表时，从已验证可用的模板（如催产素报表）深拷贝整个 `reportConfig`
- 只替换 `scopeConfig` 和 `eventConfig`，其余 12 个字段保持原样
- 不可自行构造或缩减字段

### H9: `proxyCell=true` 的 cell 不能有 widget
- `proxyCell: true` 的 cell 不能包含 `widget` 字段
- 但 `proxyCell: false` 且在 merge 范围内的 cell 可以有 widget（用于特殊布局）
- widget 只应放在 merge 区域的主单元格（左上角）或非合并单元格

### H10: 所有 `proxyCell=true` 的 cell 必须有 `realCellPosition`
- 格式：`{"realCellPosition": {"row": sr, "col": sc}}`
- 指向所属 merge 区域的主单元格（左上角）坐标
- 缺失会导致渲染时找不到真实单元格位置

### H11: `dataMaker` 换行必须用 `\r\n`
- `componentLogic.dataSource.dataMaker` 字符串中的换行符必须是 `\r\n`，不是 `\n`
- 生成时使用 `json.dumps(opts, ensure_ascii=False, indent=2).replace('\n', '\r\n')`

### H12: ZIP 内容必须用 UTF-8 编码
- `.report` 文件内容必须用 `utf-8` 编码写入 ZIP
- JSON 序列化必须 `ensure_ascii=False`（保留中文字符，不转义为 `\uXXXX`）

### H13: 列数由 PDF 识别到的患者信息字段数动态决定
- 解析 PDF/源文件时，先识别患者基本信息区域的字段数量（如床号、姓名、性别、年龄、住院号、入院诊断 = 6个字段）
- 每个字段占 2 列（标签列 + 值列），因此 **总列数 = 患者信息字段数 × 2**
- 示例：6个字段 → 12列，5个字段 → 10列，4个字段 → 8列
- 患者信息必须在**一行内**排列完所有字段，不得拆成多行
- 主体表格区域（教育内容、评价、签名等）通过合并单元格适配此列数
- 不允许固定列数（如硬编码12列）而不考虑实际字段数

## 整体架构

### reportReference 结构
- `source`: 纯二维数组 `[[str|null, ...], ...]`，不是嵌套对象（见 H2）
- `meta`: 扁平列表，每个cell必须有 `row`/`col`/`s`/`proxyCell` 字段（见 H3）
- `merges`: 对象数组，每个元素为 `{"startRow"/"startColumn"/"endRow"/"endColumn"}`（见 H7）
- `resized`: `{rows: [数字, ...], cols: [数字, ...]}`，元素为纯数字（见 H6）
- `hidden`: `{rows: [], cols: []}`
- `floatElements`: `[]`

### reportConfig 结构（见 H8）
- 从已验证模板深拷贝，只替换 `scopeConfig` 和 `eventConfig`
- 14 个标准字段：`pageConfig`、`splitLayout`、`headerRepeat`、`footerRepeat`、`headerFrozen`、`followUpPrintOpt`、`scopeConfig`、`searchBarConfig`、`eventConfig`、`serviceConfig`、`headerOptions`、`printOptions`、`functionConfig`、`engineConfig`

## source 格式规则

### 空白值使用 `null`（Python中的 `None`）
- 空白位置统一使用 `null`，不使用 `' '`（空格字符串）
- 空字符串 `''` 用于特殊位置（如input控件的值字段）
- 不存在使用 `' '`（单个空格）的情况

### 列数不固定
- 列数根据实际表单需求确定（参考模板有9/12/18/20列）
- 不是强制12列
- 列数由PDF/源文件中的实际列位置决定
- **患者信息区域的列数 = 患者字段数 × 2**（见 H13），主体表格区域通过合并单元格适配
- **A4宽度参考**: A4宽度210mm ≈ 794px（96 DPI），扣除pagePadding（左右各5mm）后约763px可用
  - 每列建议30-45px
  - 12列: 每列约63px（可用50/79交替）
  - 18列: 每列约42px（可用40-50px交替）
  - 20列: 每列约30-40px（可用30/30/30/30/30/30/45/30/30/30/40/40+重复8列）
- 列宽设置示例: `[30,30,30,30,30,30,45,30,30,30,40,40]+[30]*8` 表示前12列按实际内容设置，后8列为30px

### 文本值保留原始格式
- 文本中可包含 `\r\n` 换行符（如 `"性别：\r\n"`）
- 部分模板的source行使用 `{占位符}` 格式（如 `'{创业开发区医院}'`）表示需要替换的值
- 部分模板的source行直接将label和placeholder放在同一单元格（如 `['科室:', '科室']`）

### 行结构模式
- 常见模式1: label在偶数列，input在奇数列（如 `['姓名：', None, '性别：', None, ...]`）
- 常见模式2: 合并单元格后直接在合并区域内放置文本或控件
- 部分模板的source行可以是空数组 `[]`（对应全null行）

## meta Cell 格式规则

### 字段存在性规则
- **所有cell**必须有: `row`, `col`, `s`, `proxyCell`
- **Normal cell**: 有 `v`（文本值），部分有 `t`（不是所有都有）
- **Widget cell**: 大多数**没有** `t`/`v` 字段；部分有 `v`/`t`（见下方详细说明）
- **Proxy cell**: 没有 `t`/`v`，必须有 `realCellPosition: {row, col}`（见 H10）

### Widget cell 的 v/t 规则
- **input widget（readonly=false）**: 通常**没有** `t`/`v`
- **input widget（readonly=true）**: 部分模板有 `t: 1, v: "占位文本"`（如 `'医疗机构名称'`），部分没有
- **checkboxgroup widget**: 部分有 `v`（含格式化文本如 `'□ 选项1\r□ 选项2'`），部分没有
- **datePicker widget**: 通常有 `t: 1, v: "占位文本"`

## s 样式规则

### 字体
- 统一使用 `"FangSong"`（仿宋），不是 `"宋体"`

### 完整样式字段集
`s` 对象可包含以下字段：
- `ff`: 字体（font family）- `"FangSong"`
- `fs`: 字号（font size）- 普通文本10，大标题14
- `ht`: 水平对齐（0=左, 1=中, 2=右）
- `vt`: 垂直对齐（0=上, 1=中, 2=下）
- `bl`: 粗体（0/1）
- `it`: 斜体（0/1）
- `ul`: 下划线 `{"s": 0}`
- `st`: 删除线 `{"s": 0}`
- `ol`: 上划线 `{"s": 0}`
- `tr`: 文字旋转 `{"a": 0, "v": 0/1}`
- `td`: 文字装饰（0）
- `tb`: 文字背景（0）
- `bd`: 边框 `{"t"/"l"/"b"/"r": {"s": 1, "cl": {"rgb": "#000000"}}}`
- `pd`: 内边距 `{"t": 0, "b": 2, "l": 2, "r": 2}`
- `n`: 数字格式 `{"pattern": "@@@"}`

### 不同cell类型的s样式
- **大标题行**: `ht: 2, bl: 1, fs: 14`，可能有 `bd`/`pd`/`vt`
- **普通文本cell**: `ff: "FangSong", fs: 10, vt: 2`，有完整的 `bd`/`pd`
- **Widget cell**: 样式根据位置不同而变化，有 `bd`/`pd`/`vt`/`fs` 等
- **Proxy cell**: `s` 继承主单元格的完整样式

### s样式不是统一的
- 每个cell的 `s` 根据实际位置和边框需求独立设置
- 不是简单的 `s: {}` 或 `s: {"ff": "宋体"}`

## Normal Cell格式
```json
{
  "row": r, "col": c,
  "v": "文本值",
  "s": { "ff": "FangSong", "fs": 10, "vt": 2, "bd": {...}, "pd": {...} },
  "t": 1,
  "proxyCell": false
}
```
- 部分normal cell没有 `t` 字段
- `v` 可以是空字符串 `""`（对应source中的 `''`）

## Input Widget格式
```json
{
  "row": r, "col": c,
  "s": { "ff": "FangSong", "fs": 10, "vt": 2, "bd": {...}, "pd": {...} },
  "proxyCell": false,
  "widget": {
    "type": "input",
    "attribute": {
      "clearable": true, "simplify": true, "size": "small",
      "placeholder": "提示文本", "maxlength": 40,
      "type": "input", "readonly": false
    },
    "componentLogic": {
      "insertCell": { "rowstart": 1, "rowend": 1, "scopeField": "" }
    },
    "scopeField": "scope字段名",
    "identifier": "标识符",
    "name": "中文名"
  }
}
```
- `readonly=false` 的input通常**没有** `t`/`v`
- `readonly=true` 的input（自动填充字段）：部分模板有 `t: 1, v: "占位文本"`，部分没有
- `clearable` 可以是 `true` 或 `false`（readonly字段常用 `false`）
- `simplify` 可以是 `true` 或省略
- `componentLogic` 可有 `interaction` 字段（含dependencies/actionConfig）
- `identifier` 格式不固定，可以是拼音缩写（`ym_jgmc`）、描述性名称（`pi_name`）等

## Radiogroup Widget格式
```json
{
  "row": r, "col": c,
  "s": { "ff": "FangSong", "fs": 10, "vt": 2, "bd": {...}, "pd": {...} },
  "proxyCell": false,
  "widget": {
    "type": "radiogroup",
    "attribute": {
      "disabled": false, "vertical": false,
      "props": { "label": "label", "value": "value", "disabled": "disabled" }
    },
    "componentLogic": {
      "insertCell": { "rowstart": 1, "rowend": 1, "scopeField": "" },
      "dataSource": {
        "type": "1",
        "dataMaker": "[{\r\n    \"label\": \"...\",\r\n    \"value\": \"...\"\r\n},...]"
      }
    },
    "scopeField": "scope字段名",
    "identifier": "标识符",
    "name": "中文名"
  }
}
```
- **没有** `t`/`v` 字段
- `dataMaker` 是格式化JSON字符串，包含 `\r\n` 换行
- `vertical: false` 为横向排列，`vertical: true` 为纵向排列

## Checkboxgroup Widget格式
```json
{
  "row": r, "col": c,
  "s": { ... },
  "proxyCell": false,
  "widget": {
    "type": "checkboxgroup",
    "attribute": {
      "disabled": false, "vertical": false,
      "min": 0, "enableMax": false,
      "props": { "label": "label", "value": "value", "disabled": "disabled" }
    },
    "componentLogic": {
      "insertCell": { ... },
      "dataSource": { "type": "1", "dataMaker": "..." }
    },
    "scopeField": "...",
    "identifier": "...",
    "name": "..."
  }
}
```
- 部分有 `v: "..."`, `t: 1`（含格式化选项文本），部分没有
- `vertical: false` 横向排列（适用于少量选项），`vertical: true` 纵向排列（适用于多选项）
- `enableMax: false`, `min: 0`
- **模拟单选**: 使用 `enableMax: true, max: 1` 可实现单选的 checkboxgroup（如 Bishop 评分场景，每行只能选一个评分）

## Select Widget格式
```json
{
  "row": r, "col": c,
  "v": "", "t": 1,
  "s": { ... },
  "proxyCell": false,
  "widget": {
    "type": "select",
    "attribute": {
      "disabled": false, "clearable": true, "filterable": false,
      "multiple": false, "simplify": true, "placeholder": "", "size": "small",
      "props": { "label": "label", "value": "value", "disabled": "disabled" }
    },
    "componentLogic": {
      "insertCell": { ... },
      "dataSource": { "type": "1", "dataMaker": "..." }
    },
    "scopeField": "...",
    "identifier": "...",
    "name": "..."
  }
}
```

## DatePicker Widget格式
```json
{
  "row": r, "col": c,
  "t": 1,
  "v": "占位文本",
  "s": { ... },
  "proxyCell": false,
  "widget": {
    "type": "datePicker",
    "attribute": {
      "disabled": false, "clearable": true, "simplify": true,
      "size": "small",
      "valueFormat": "yyyy-MM-dd HH:mm",
      "format": "yyyy-MM-dd HH:mm",
      "initState": "2",
      "type": "datetime"
    },
    "componentLogic": {
      "insertCell": { "rowstart": 1, "rowend": 1, "scopeField": "" }
    },
    "scopeField": "...",
    "identifier": "...",
    "name": "..."
  }
}
```
- 通常有 `t: 1, v: "占位文本"`
- `initState`: 初始化状态（"0"=空, "1"=当前, "2"=自定义）
- `valueFormat`/`format`: 日期格式

## DatePickerQuick Widget格式
```json
{
  "row": r, "col": c,
  "s": { ... },
  "proxyCell": false,
  "widget": {
    "type": "datePickerQuick",
    "attribute": {
      "input-control": true, "disabled": false, "clearable": true,
      "simplify": true, "size": "medium",
      "valueFormat": "yyyy-MM-dd hh:MM:ss",
      "initState": "0", "type": "datetime"
    },
    "componentLogic": {
      "insertCell": { "rowstart": 1, "rowend": 1, "scopeField": "" }
    },
    "scopeField": "...",
    "name": "...",
    "identifier": "..."
  }
}
```
- **没有** `t`/`v` 字段

## Proxy Cell格式（见 H9、H10）
```json
{
  "row": r, "col": c,
  "s": { /* 继承主单元格完整样式 */ },
  "proxyCell": true,
  "realCellPosition": { "row": sr, "col": sc }
}
```
- **没有** `t`/`v` 字段
- **不能有** `widget` 字段（见 H9）
- 必须有 `realCellPosition`（见 H10）
- `s` 完全继承主单元格(sr,sc)的样式

## 占位符处理规则
- source中通常不使用 `{姓名}`、`[选项]` 等占位符文本
- 大部分情况下需要生成控件的位置，source统一用 `null`
- 标签文本（如"姓名："、"年龄："等）作为普通文本保留在source中
- 部分模板source行使用 `{占位符}` 格式表示需要替换的文本
- 部分模板source行直接将label和placeholder放在同一单元格（如 `['科室:', '科室']`）
- 通过 `input_map` / `radio_map` / `checkbox_map` / `select_map` / `date_map` / `datepicker_map` 配置哪些 (r,c) 位置生成控件
- 控件的具体选项数据（label/value）放在 map 中，不从source文本解析

## 控件生成逻辑
1. 遍历source每个cell，val = source[r][c]
2. 如果 val 是 null 或空字符串:
   - 检查 (r,c) 是否在各类map中 -> 生成对应控件
   - 都不在 -> 检查是否是merge区域 -> proxyCell / 普通空白cell
3. 如果 val 是字符串 -> 普通文本cell

## Merges规则（见 H7）
- 每个元素必须是对象: `{"startRow": sr, "startColumn": sc, "endRow": er, "endColumn": ec}`
- 不允许使用数组 `[sr, sc, er, ec]` 格式
- 合并非常精细（参考模板有41个merge），按实际内容需求合并
- 每个merge区域内，除主单元格(sr,sc)外的所有cell设置 `proxyCell: true`
- proxyCell的 `s` 完全继承主单元格样式
- proxyCell 必须有 `realCellPosition`（见 H10）

## 列宽/行高规则（见 H6）
- `resized.rows`: 纯数字列表，按实际需要设置不同行高（如 `[24, 24, 22, 22, 22, ...]`）
- `resized.cols`: 按实际需要设置不同列宽（如 `[50, 79, 50, 79, ...]`）
- 不是统一固定值
- **关键约束**: `len(resized.rows)` 必须严格等于 `len(source)`，`len(resized.cols)` 必须等于 source 每行的列数
  - 不匹配会导致 `TypeError: Cannot read properties of undefined (reading '0')` 运行时错误
  - 推荐使用动态计算: `[28,28,24,22]+[24]*(len(source)-4)` 而非硬编码
- **列宽单位是像素(px)**（非毫米），pageW/pageH 单位是毫米
  - A4宽度 210mm ≈ 794px（96 DPI），扣除pagePadding后约 775px 可用
  - 参考模板列宽总计: 674px(18列) ~ 748px(9列)
  - 12列布局建议: 交替 label(50px) + input(79px) = 774px

## 顶层数据结构格式
- `id`: UUID格式（hex，24字符，如 `uuid.uuid4().hex[:24]`）
- `cd`: 表单编码
- `na`: 表单名称
- `template`: **JSON 字符串**（`json.dumps(template_object, ensure_ascii=False)`），不是嵌套对象（见 H1）
- `categoryId`: 分类ID（如 `"hihis@hihis/nenr@nenr/nenr_form@nenr_form/nenr_form_xyz"`）
- `version`: 版本号（如 `"1.0.1"`, `"1.0.6"`）
- `instr`: 包含cd、na、拼音缩写的字符串
- `tenantId`: 租户ID（如 `"BSOFTYL"`）
- `createDate`/`modifyDate`: `"yyyy-MM-dd HH:mm:ss"` 格式
- `createUser`/`modifyUser`: 字符串（如 `"admin"`，不需要 UUID 格式）
- `active`: true

## scopeConfig规则（见 H4）
- **必须是列表**，不是字典
- 每个控件对应一个scope_config条目
- 格式: `{"name": "scope字段名", "defaultValue": "", "type": "string", "desc": "中文说明"}`
- `desc` 是已验证模板使用的字段名（推荐使用 `desc`，不用 `remark`）
- `name` 使用小写scope字段名
- **`name` 必须与对应 widget 的 `scopeField` 完全匹配**（区分大小写）
- `type` 可以是 `"string"`, `"array"`, `"object"` 等
- `defaultValue` 对于array类型可以是 `[]`，对于object类型可以是 `{}`
- 部分模板包含内置scope字段如 `nurseFormContext`（类型"object"，描述"患者信息等基础内置上下文"）

## eventConfig规则（见 H5）
- **必须是列表**，包含全部 7 个标准事件，不是字典，不是空数组
- 每个事件格式: `{"eventName": "事件名", "expressionStatement": "脚本或空字符串"}`
- 7 个标准事件: `beforeload`, `afterload`, `beforerender`, `afterrender`, `beforeprint`, `afterprint`, `childReportMsg`
- 即使脚本为空，也必须有 `{"eventName": "xxx", "expressionStatement": ""}`
- `beforerender` 中常包含自动填充患者信息的脚本
- `expressionStatement` 中的 `$$scope.xxx` 必须与对应 widget 的 `scopeField` 完全一致（区分大小写）
- 示例beforerender脚本:
  ```javascript
  $$scope.ym_xm = $$scope.nurseFormContext?.patientInfo?.name;
  $$scope.ym_xb = $$scope.nurseFormContext?.patientInfo?.sexName;
  $$scope.ym_nl = $$scope.nurseFormContext?.patientInfo?.age;
  $$scope.ym_zyh = $$scope.nurseFormContext?.patientInfo?.admNo;
  ```
- 使用 `$$scope.nurseFormContext?.patientInfo` 获取患者数据

## serviceConfig规则
- `datasourceType`: "3"
- `dbConfig`: `[]`
- `inputsType`: -1

## 源文件解析规则

### PDF解析
- 使用 `pdfplumber` 库
- 文本型PDF: `page.extract_text()` / `page.extract_words()` 直接提取
- 扫描型PDF（无文本层）: `page.to_image(resolution=200)` 转图片，通过视觉分析
- 判断方法: `len(page.chars) == 0` 且 `len(page.images) > 0` → 扫描件

### DOCX解析
- 使用 `python-docx` 库（`from docx import Document`）
- **段落** (`doc.paragraphs`): 提取标题、说明文字、列表项（如措施清单）
  - `p.style.name` 可区分正文/列表（`List Bullet`）
  - `□` 符号表示checkbox选项
- **表格** (`doc.tables`): 提取结构化表单内容
  - `table.rows` / `table.columns` 获取行列数
  - `cell.text` 获取单元格文本（含 `\n` 换行）
  - 合并单元格的文本会在所有被合并的cell中重复出现
- **页面尺寸** (`doc.sections`): `page_width` / `page_height` 单位为EMU（914400 EMU = 1 inch = 25.4mm）
- **解析策略**:
  1. 先遍历段落获取标题和列表项
  2. 再遍历表格获取结构化内容
  3. 表格中的 `□` 符号对应checkbox控件
  4. 表格中的 `____` 或空白对应input控件
  5. 多个表格可能属于同一表单的不同区域（如页眉信息表+评分表）

### 多部分文档处理
- 一个源文件可能包含多个独立表单（如产前+LATCH+母婴分离）
- 每个部分有独立的标题行和表头
- 生成一个合并模板，各部分用空行分隔
- 每部分独立编号行，但source/meta/merges全局统一

### checkboxgroup 批量措施处理
- 当一个单元格内有多个 `□` 选项时，使用 `checkboxgroup` 控件（`vertical: true`）
- 选项数据通过 `dataMaker` 传入，每个选项为 `{label, value}` 格式
- 不要为每个选项创建独立的checkboxgroup单元格

## 输出格式规则（见 H1、H12）
- **最终输出是 ZIP 压缩包**，不是单个 `.report` 文件
- ZIP 包命名格式：`nenr_form_<编码>-<版本>.zip`（如 `nenr_form_oxytocin_record-1.0.1.zip`）
- ZIP 包内只包含一个文件，命名格式：`nenr_form_<编码>-<版本>.report`（如 `nenr_form_oxytocin_record-1.0.1.report`）
- `.report` 文件内容就是完整的 JSON 数据（顶层对象含 `id`/`cd`/`na`/`template` 等字段）
- `.report` 只是 JSON 的扩展名，内容仍是标准 JSON
- **`template` 字段的值必须是 JSON 字符串**，不是嵌套对象（见 H1）
- **ZIP 内文件必须用 UTF-8 编码**，JSON 序列化必须 `ensure_ascii=False`（见 H12）

## identifier 命名规则（关键）
- `identifier` 和 `scopeField` 字段**会被系统当作 JavaScript 变量名使用**
- **禁止包含** `/`、空格、`(`、`)`、`-`、`+`、`*` 等 JS 运算符或特殊字符
- `/` 会被解析为除法运算符，导致 `ReferenceError: XXX is not defined`
- 推荐使用纯小写字母+数字+下划线（如 `picccvc_1`，不要用 `PICC/CVC_1`）
- 中文名称可用作 `name`（显示名）和 `desc`（说明），但**不可**用作 `identifier`/`scopeField`
- 含 `/` 的名称（如 `PICC/CVC`、`造瘘管/胃管/导尿管`）需转成拼音缩写（如 `picccvc`、`zfqg`）
