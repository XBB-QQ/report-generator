# report-generator

智能护理表单模板生成器，当前版本 **v8.8**。

本仓库是一个 Trae Skill，用于根据 PDF、DOCX 等源文件生成护理表单 `.report`，并打包为可导入的 ZIP；也可在严格限制修改范围的前提下修复已有模板。

> 完整的生成规则、代码模板和 H1-H35 硬性约束以 [SKILL.md](SKILL.md) 为准。

## 核心能力

- 从 PDF、DOCX 中解析表单结构、合并单元格、控件和患者信息字段。
- 生成标准 `.report` JSON，并打包为 `<cd>-1.0.1.zip`。
- 构建 input、select、checkboxgroup、radiogroup、datePicker 等控件。
- 支持 PC 与 PDA 双平台：PDA 使用 vantInput 等控件和两列布局，禁止混用。
- 支持评分计算、护士签名事件、条件显隐和系统字典数据源。
- 支持根据控件值动态显示/隐藏明细行（change 事件 + afterload 初始状态）。
- 通过 `verify()` 在打包前校验 H1-H35，发现违规配置直接停止。
- 修改已有模板时进行最小化差异控制，不做未经授权的额外调整。

## 仓库结构

```text
report-generator/
├── SKILL.md    # Skill 主文件，包含规则、代码模板、verify() 和 package()
└── README.md   # 仓库说明
```

## 安装与加载

本仓库不是独立 CLI 或 Python 包，需要作为 Trae Skill 加载。

- Trae CN 全局 Skill：`~/.trae-cn/skills/report-generator/`
- 项目级 Skill：`<项目目录>/.trae/skills/report-generator/`

加载后，可直接提出类似需求：

```text
根据这个 PDF 生成护理表单模板 ZIP。
```

```text
仅修复这个模板的重复表头配置，其他内容不要改。
```

## 使用流程

```text
解析源文件
  → 填写表单定义
  → 构建 source/meta/merges/resized/widgets
  → verify() 校验 H1-H35
  → package() 打包 ZIP
```

1. **解析源文件**：提取行、列、文本、合并区域、控件和患者字段。
2. **填写表单定义**：设置 `FORM_CD`、`FORM_NA`、`PLATFORM`、行列数据、控件、`EVENT_SCRIPTS` 及尺寸。
3. **构建模板**：使用 `SKILL.md` 内置 helper 函数生成标准结构。
4. **自动验证**：`verify()` 检查数据结构、布局、控件、标识符、打印格式和平台一致性。
5. **打包输出**：验证通过后生成 ZIP。

## 输出结构

```text
<cd>-1.0.1.zip
└── <cd>-1.0.1.report
```

`.report` 顶层关键属性示例：

```json
{
  "cd": "nenr_form_admission_nursing_assessment",
  "na": "入院护理评估单",
  "version": "1.0.1",
  "instr": "nenr_form_admission_nursing_assessment,入院护理评估单"
}
```

其中 `template` 必须是 JSON 字符串，系统会对其进行二次 `JSON.parse`。

## 硬性约束总览

| 规则 | 范围 | 核心要求 |
|---|---|---|
| H1-H8 | 数据结构 | 规范 `template`、`source`、`meta`、`merges`、`resized`、事件和 `reportConfig` |
| H9-H14 | 单元格逻辑 | 代理单元格、换行、UTF-8、患者信息列数及多行文本 |
| H15-H22 | 显示 | 标题对齐、文本类型、精确行列尺寸、日期及复选控件 |
| H23 | 页面结构 | PC 第一行为合并空行；PDA 不保留空行，直接放业务字段 |
| H24 | 交互计算 | 计算逻辑放在 `componentLogic.interaction`，PC/PDA 输入、下拉、多选控件通用 |
| H25-H26 | 签名与条件属性 | PC 签名 focus+dblclick、PDA 仅 focus；支持条件背景和内容显隐 |
| H27 | 字典数据源 | 命中固定字典语义时必须使用 `dataSource.type="3"` |
| H28 | 修改边界 | 直接覆盖出错 ZIP，只修改用户明确要求的内容 |
| H29 | 标识符 | `identifier` 必须支持 JavaScript 点号属性访问 |
| H30-H32 | 顶层属性 | 固定 `cd`、`version="1.0.1"` 和 `instr=cd,na` 规则 |
| H33 | 打印格式 | `pageConfig.printFormat` 必须固定为 `"pdf"` |
| H34 | PDA 平台 | `PLATFORM` 二选一，控件映射、两列布局、数组 scope、cell 扩展字段 |
| H35 | 动态行显隐 | change 事件与 afterload 初始状态必须同时配置，禁止调试代码 |

完整细则及示例见 [SKILL.md](SKILL.md)。

## 关键配置规范

### 表单标识

- `cd` 格式：`nenr_form_xxx`。
- `xxx` 使用简洁、语义一致的英文，优先采用适用的医疗术语。
- `cd` 必须匹配：

```text
^nenr_form_[a-z][a-z0-9]*(?:_[a-z0-9]+)*$
```

- `version` 固定为字符串 `"1.0.1"`。
- `instr` 必须严格等于 `cd + "," + na`。

### 控件标识符

`identifier` 必须匹配：

```text
^[A-Za-z_$][A-Za-z0-9_$]*$
```

禁止以数字开头，禁止包含 `/`、空格、连字符、括号等 JavaScript 特殊字符。

### 页面打印格式（H33）

模板内部必须满足：

```json
{
  "reportConfig": {
    "pageConfig": {
      "printFormat": "pdf"
    }
  }
}
```

`load_report_config()` 会默认写入 `"pdf"`，`verify()` 会拒绝 `"html"` 或其他值。

该规则只针对 `pageConfig.printFormat`，不改变 `printOptions.printFormat`。

## 系统字典数据源

当控件语义命中固定字典关系时，禁止用本地 `type="1"` 写死选项，必须使用系统字典：

```json
{
  "type": "3",
  "dataMaker": [
    "5d3156deb7cef02f8897f8ac",
    "5d3159eab7cef02f88980166",
    "hi.his.marry"
  ]
}
```

选项映射固定为：

```json
{
  "label": "text",
  "value": "cd",
  "disabled": "disabled"
}
```

常见字典包括：

| 中文语义 | 字典编码 |
|---|---|
| 婚姻 | `hi.his.marry` |
| 民族 | `hi.his.nation` |
| 职业 | `hi.his.occu` |
| 文化程度 | `sys.per.educationallevel` |
| 患者性别 | `hi.his.sex` |
| 证件类型 | `hi.his.pernidtp` |
| 联系人与患者的关系 | `hi.his.milRelationType` |

完整字典对应关系及前缀规则见 [SKILL.md](SKILL.md)。

## 修改已有模板的边界

修改已有报表时必须遵守 H28：

- 直接覆盖当前出错的同一个 ZIP，保持原文件名和固定版本号。
- 禁止擅自递增版本号或生成额外修复副本。
- 参考示例只能用于用户明确指定的字段或结构。
- 不顺手修改页面尺寸、行列宽、控件、字典或其他无关配置。
- 修改后必须核对差异，确保每处变化都能追溯到明确需求。

## 运行依赖

核心代码模板使用 Python 3 和标准库。解析源文件时按需安装：

```powershell
pip install pdfplumber python-docx
```

- `pdfplumber`：解析文本型 PDF；扫描件需要转图片后识别。
- `python-docx`：解析 DOCX 段落、表格和页面信息。

## 校验说明

`verify()` 是打包前的硬性检查，不是可选建议：

- 校验失败时输出对应规则编号并停止打包。
- 校验通过后才会调用 `package()` 生成 ZIP。
- H33 会单独检查 `pageConfig.printFormat`。
- H34 会检查 PC/PDA 控件不混用、PDA 布局和数组 scope。
- H35 会检查行显隐的 change 事件与 afterload 初始状态成对存在。
- 生成或修改模板后，还应在目标业务系统中进行实际导入和运行验证。
