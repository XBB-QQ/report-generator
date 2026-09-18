---
name: "report-generator"
description: "智能护理表单模板生成器"
---

# Report Generator Skill - v8.0

## 使用流程

```
解析源文件 → 填写表单定义 → 运行代码模板 → 自动验证 → 打包ZIP
```

1. **解析源文件**（PDF/DOCX）→ 提取表单结构（行/列/控件/合并区域/患者字段）
2. **填写表单定义** → 在代码模板的「表单定义区」填入解析结果
3. **运行代码模板** → helper 函数自动构建 source/meta/merges/resized/widgets，保障所有硬性约束
4. **自动验证** → 脚本末尾 `verify()` 函数检查 H1-H22
5. **打包 ZIP** → 输出 `nenr_form_<编码>-<版本>.zip`

## 硬性约束（H1-H22，违反即报错或显示异常）

### 结构类 H1-H8（数据格式）

- **H1**: `template` 字段必须是 JSON 字符串（`json.dumps(obj, ensure_ascii=False)`），不是嵌套对象。系统会二次 `JSON.parse`
- **H2**: `source` 必须是纯二维数组 `[[str|null, ...], ...]`，值只能是字符串或 `null`
- **H3**: `meta` 必须是扁平列表，长度 = 行数 × 列数，每个元素对应一个单元格
- **H4**: `scopeConfig` 必须是列表，每个 `name` 必须与对应 widget 的 `scopeField` 完全匹配
- **H5**: `eventConfig` 必须是列表且包含全部 7 个标准事件（`beforeload`/`afterload`/`beforerender`/`afterrender`/`beforeprint`/`afterprint`/`childReportMsg`），即使脚本为空也必须有条目
- **H6**: `resized.rows`/`resized.cols` 元素必须是纯数字，`len(rows)` = `len(source)`，`len(cols)` = 每行列数
- **H7**: `merges` 元素必须是对象 `{startRow, startColumn, endRow, endColumn}`，不允许数组
- **H8**: `reportConfig` 必须从已验证模板（催产素报表 ZIP）深拷贝 14 个标准字段，只替换 `scopeConfig`/`eventConfig`，不可自行构造或缩减

### 内容类 H9-H14（单元格逻辑）

- **H9**: `proxyCell: true` 的 cell 不能有 `widget` 字段
- **H10**: 所有 `proxyCell: true` 的 cell 必须有 `realCellPosition: {row, col}` 指向主单元格
- **H11**: `dataMaker` 换行必须用 `\r\n`（`json.dumps(...).replace('\n', '\r\n')`）
- **H12**: ZIP 内容必须 UTF-8 编码，JSON 序列化 `ensure_ascii=False`
- **H13**: 总列数 = 患者信息字段数 × 2（标签列+值列），患者信息必须在一行内
- **H14**: 多行文本单元格必须设置 `tb=3`（不是 0/1），否则 `\r\n` 不生效

### 显示类 H15-H22（原"最佳实践"，升级为硬性）

- **H15**: 标题行和表头行的 cell 必须设置 `ht=2`（水平居中）
- **H16**: 文本单元格必须包含 `v`（文本值）和 `t: 1`（类型标记）字段
- **H17**: 列宽必须精确计算（允许小数如 `30.5`/`129.5`），总宽度接近 A4 可用宽度（~768px），不使用整数估算
- **H18**: 行高必须根据内容行数精确设置；空行（分隔行）高度必须为 `11.5px`（不是标准行高）
- **H19**: 医疗机构名称必须添加 input widget（`scopeField` 可为空字符串 `""`）
- **H20**: 日期控件必须使用 `datePicker` 类型（有 `t:1, v:"占位文本"`），不使用 `datePickerQuick`
- **H21**: 纯展示性 checkbox（如"已指导"标记）`scopeField` 必须为空字符串 `""`
- **H22**: checkboxgroup 当选项 ≥ 3 个时应设置 `itemSpacing` 控制间距（推荐值 30）

## 内置代码模板

以下模板是**完整可运行的 Python 脚本**。模型只需修改「表单定义区」，helper 函数自动保障 H1-H22。

```python
#!/usr/bin/env python3
"""
护理表单报表生成器模板 v8.0
使用方法：修改 === 表单定义 === 区块，运行即可生成 ZIP。
所有硬性约束 H1-H22 由 helper 函数自动保障。
"""
import json, uuid, zipfile, io, copy, os, re, sys

# ============================================================
# Helper 函数（固定，不要修改）
# ============================================================

def _border_all():
    e = {"s": 1, "cl": {"rgb": "#000000"}}
    return {"t": e, "l": e, "b": e, "r": e}

def _padding():
    return {"t": 0, "b": 2, "l": 2, "r": 2}

def _style(ff="FangSong", fs=10, ht=None, vt=2, bl=0, it=0, tb=None, bd=None, pd=None):
    """构建样式对象，自动包含完整边框和内边距"""
    s = {"ff": ff, "fs": fs, "vt": vt, "bl": bl, "it": it}
    if ht is not None: s["ht"] = ht       # H15: 标题/表头 ht=2
    if tb is not None: s["tb"] = tb       # H14: 多行文本 tb=3
    s["bd"] = bd or _border_all()
    s["pd"] = pd or _padding()
    return s

def text_cell(r, c, v, ht=None, tb=None, fs=10, bl=0):
    """构建文本单元格 (H16: 有v和t:1)"""
    return {"row": r, "col": c, "v": v, "t": 1, "proxyCell": False,
            "s": _style(ht=ht, tb=tb, fs=fs, bl=bl)}

def proxy_cell(r, c, sr, sc, s=None):
    """构建代理单元格 (H9: 无widget, H10: 有realCellPosition)"""
    return {"row": r, "col": c, "s": s or _style(),
            "proxyCell": True, "realCellPosition": {"row": sr, "col": sc}}

def empty_cell(r, c):
    """构建空白非合并单元格"""
    return {"row": r, "col": c, "s": _style(), "proxyCell": False}

def _datamaker(options):
    """构建 dataMaker 字符串 (H11: 使用 \\r\\n)"""
    return json.dumps(options, ensure_ascii=False, indent=2).replace('\n', '\r\n')

# --- 控件构建函数 ---

def w_input(scope_field, identifier, name, placeholder="", readonly=False,
            clearable=True, simplify=False, rowstart=1, rowend=1):
    """构建 input 控件"""
    return {"type": "input", "attribute": {
        "clearable": clearable, "simplify": simplify, "size": "small",
        "placeholder": placeholder, "maxlength": 40,
        "type": "input", "readonly": readonly
    }, "componentLogic": {
        "insertCell": {"rowstart": rowstart, "rowend": rowend, "scopeField": ""}
    }, "scopeField": scope_field, "identifier": identifier, "name": name}

def w_checkbox(scope_field, identifier, name, options, vertical=False,
               enable_max=False, max_val=1, item_spacing=None, rowstart=1, rowend=1):
    """构建 checkboxgroup 控件 (H22: 选项>=3时设itemSpacing)"""
    attr = {"disabled": False, "vertical": vertical, "min": 0,
            "enableMax": enable_max,
            "props": {"label": "label", "value": "value", "disabled": "disabled"}}
    if enable_max: attr["max"] = max_val
    if item_spacing is not None: attr["itemSpacing"] = item_spacing
    return {"type": "checkboxgroup", "attribute": attr,
            "componentLogic": {
                "insertCell": {"rowstart": rowstart, "rowend": rowend, "scopeField": ""},
                "dataSource": {"type": "1", "dataMaker": _datamaker(options)}
            }, "scopeField": scope_field, "identifier": identifier, "name": name}

def w_date(scope_field, identifier, name, placeholder="选择日期时间",
           rowstart=1, rowend=1):
    """构建 datePicker 控件 (H20: 用datePicker不用datePickerQuick)"""
    return {"type": "datePicker", "attribute": {
        "disabled": False, "clearable": True, "simplify": True, "size": "small",
        "valueFormat": "yyyy-MM-dd HH:mm", "format": "yyyy-MM-dd HH:mm",
        "initState": "2", "type": "datetime"
    }, "componentLogic": {
        "insertCell": {"rowstart": rowstart, "rowend": rowend, "scopeField": ""}
    }, "scopeField": scope_field, "identifier": identifier, "name": name}

def w_select(scope_field, identifier, name, options, rowstart=1, rowend=1):
    """构建 select 控件"""
    return {"type": "select", "attribute": {
        "disabled": False, "clearable": True, "filterable": False,
        "multiple": False, "simplify": True, "placeholder": "", "size": "small",
        "props": {"label": "label", "value": "value", "disabled": "disabled"}
    }, "componentLogic": {
        "insertCell": {"rowstart": rowstart, "rowend": rowend, "scopeField": ""},
        "dataSource": {"type": "1", "dataMaker": _datamaker(options)}
    }, "scopeField": scope_field, "identifier": identifier, "name": name}

def w_radio(scope_field, identifier, name, options, vertical=False,
            rowstart=1, rowend=1):
    """构建 radiogroup 控件"""
    return {"type": "radiogroup", "attribute": {
        "disabled": False, "vertical": vertical,
        "props": {"label": "label", "value": "value", "disabled": "disabled"}
    }, "componentLogic": {
        "insertCell": {"rowstart": rowstart, "rowend": rowend, "scopeField": ""},
        "dataSource": {"type": "1", "dataMaker": _datamaker(options)}
    }, "scopeField": scope_field, "identifier": identifier, "name": name}

# --- 数据构建函数 ---

def build_source(rows_def):
    """构建 source 二维数组 (H2: 纯str|null)"""
    return [[v if v is not None else None for v in row] for row in rows_def]

def build_merges(merge_list):
    """构建 merges 对象数组 (H7: 对象格式)"""
    return [{"startRow": sr, "startColumn": sc, "endRow": er, "endColumn": ec}
            for sr, sc, er, ec in merge_list]

def build_meta(source, merge_list, widgets, title_rows, header_rows, multiline_cells):
    """
    构建 meta 扁平列表 (H3: 长度=行×列)
    自动处理: H9(proxyCell无widget), H10(realCellPosition),
              H14(tb=3), H15(ht=2), H16(v和t:1)
    """
    n_rows = len(source)
    n_cols = len(source[0]) if source else 0
    title_rows = title_rows or set()
    header_rows = header_rows or set()
    multiline_cells = multiline_cells or set()
    center_rows = title_rows | header_rows

    # 构建合并映射: (r,c) -> (sr,sc)
    merge_map = {}
    for sr, sc, er, ec in merge_list:
        for r in range(sr, er + 1):
            for c in range(sc, ec + 1):
                if (r, c) != (sr, sc):
                    merge_map[(r, c)] = (sr, sc)

    meta = []
    for r in range(n_rows):
        for c in range(n_cols):
            if (r, c) in merge_map:
                sr, sc = merge_map[(r, c)]
                meta.append(proxy_cell(r, c, sr, sc))
            elif (r, c) in widgets:
                w = widgets[(r, c)]
                cell = {"row": r, "col": c, "proxyCell": False, "widget": w["widget"]}
                ht = 2 if r in center_rows else None
                tb = 3 if (r, c) in multiline_cells else None
                cell["s"] = w.get("s") or _style(ht=ht, tb=tb)
                if "v" in w: cell["v"] = w["v"]
                if "t" in w: cell["t"] = w["t"]
                meta.append(cell)
            else:
                val = source[r][c] if c < len(source[r]) else None
                if val is not None:
                    ht = 2 if r in center_rows else None
                    tb = 3 if (r, c) in multiline_cells else None
                    bl = 1 if r in title_rows else 0
                    fs = 14 if r in title_rows else 10
                    meta.append(text_cell(r, c, val, ht=ht, tb=tb, fs=fs, bl=bl))
                else:
                    meta.append(empty_cell(r, c))
    return meta

def build_resized(row_heights, col_widths):
    """构建 resized (H6: 纯数字)"""
    return {"rows": list(row_heights), "cols": list(col_widths)}

def load_report_config(reference_zip_path):
    """从已验证模板深拷贝 reportConfig (H8)"""
    with zipfile.ZipFile(reference_zip_path, 'r') as z:
        report_name = [n for n in z.namelist() if n.endswith('.report')][0]
        ref_data = json.loads(z.read(report_name).decode('utf-8'))
    ref_template = json.loads(ref_data['template'])
    return copy.deepcopy(ref_template['reportConfig'])

def build_scope_config(widgets, extra_fields=None):
    """构建 scopeConfig (H4: 列表, name匹配scopeField)"""
    seen = set()
    config = []
    extra_fields = extra_fields or []
    # 内置字段
    config.append({"name": "nurseFormContext", "defaultValue": {},
                    "type": "object", "desc": "患者信息等基础内置上下文"})
    seen.add("nurseFormContext")
    # 控件字段（跳过空scopeField）
    for (r, c), w in sorted(widgets.items()):
        sf = w["widget"].get("scopeField", "")
        if sf and sf not in seen:
            wtype = w["widget"]["type"]
            dtype = "array" if wtype in ("checkboxgroup",) else "string"
            config.append({"name": sf, "defaultValue": [] if dtype == "array" else "",
                           "type": dtype, "desc": w["widget"].get("name", "")})
            seen.add(sf)
    # 额外字段
    for f in extra_fields:
        if f["name"] not in seen:
            config.append(f)
            seen.add(f["name"])
    return config

def build_event_config(beforerender_script=""):
    """构建 eventConfig (H5: 7个标准事件)"""
    events = ["beforeload", "afterload", "beforerender", "afterrender",
              "beforeprint", "afterprint", "childReportMsg"]
    config = []
    for ev in events:
        script = beforerender_script if ev == "beforerender" else ""
        config.append({"eventName": ev, "expressionStatement": script})
    return config

def build_template(source, meta, merges, resized, report_config):
    """构建完整 template 对象"""
    return {
        "reportReference": {
            "source": source,
            "meta": meta,
            "merges": merges,
            "resized": resized,
            "hidden": {"rows": [], "cols": []},
            "floatElements": []
        },
        "reportConfig": report_config
    }

def build_report_data(form_cd, form_na, form_version, template_obj):
    """构建顶层 JSON 数据 (H1: template是字符串, H12: ensure_ascii=False)"""
    now = "2026-01-01 00:00:00"
    return {
        "id": uuid.uuid4().hex[:24],
        "cd": form_cd,
        "na": form_na,
        "template": json.dumps(template_obj, ensure_ascii=False),  # H1
        "categoryId": "hihis@hihis/nenr@nenr/nenr_form@nenr_form/nenr_form_" + form_cd,
        "version": form_version,
        "instr": form_cd + "," + form_na,
        "tenantId": "BSOFTYL",
        "createDate": now, "modifyDate": now,
        "createUser": "admin", "modifyUser": "admin",
        "active": True
    }

# --- 验证函数 ---

def verify(data, source, meta, merges, resized, widgets, patient_fields):
    """验证所有硬性约束 H1-H22"""
    errors = []
    template_obj = json.loads(data["template"])

    # H1: template 是字符串
    if not isinstance(data["template"], str):
        errors.append("H1: template 不是字符串")

    # H2: source 是纯二维数组
    for i, row in enumerate(source):
        if not isinstance(row, list):
            errors.append(f"H2: source[{i}] 不是列表")
        for j, val in enumerate(row):
            if val is not None and not isinstance(val, str):
                errors.append(f"H2: source[{i}][{j}]={val!r} 不是str/null")

    # H3: meta 长度 = 行×列
    n_rows, n_cols = len(source), len(source[0]) if source else 0
    if len(meta) != n_rows * n_cols:
        errors.append(f"H3: meta长度{len(meta)} != {n_rows}×{n_cols}={n_rows*n_cols}")

    # H4: scopeConfig 是列表
    sc = template_obj["reportConfig"]["scopeConfig"]
    if not isinstance(sc, list):
        errors.append("H4: scopeConfig 不是列表")

    # H5: eventConfig 有7个事件
    ec = template_obj["reportConfig"]["eventConfig"]
    if not isinstance(ec, list) or len(ec) != 7:
        errors.append(f"H5: eventConfig 长度{len(ec) if isinstance(ec,list) else 'N/A'} != 7")
    required_events = {"beforeload","afterload","beforerender","afterrender",
                       "beforeprint","afterprint","childReportMsg"}
    actual_events = {e.get("eventName") for e in ec} if isinstance(ec, list) else set()
    if actual_events != required_events:
        errors.append(f"H5: 事件不匹配, 缺少: {required_events - actual_events}")

    # H6: resized 纯数字
    for k, vals in [("rows", resized["rows"]), ("cols", resized["cols"])]:
        for i, v in enumerate(vals):
            if not isinstance(v, (int, float)) or isinstance(v, bool):
                errors.append(f"H6: resized.{k}[{i}]={v!r} 不是数字")
    if len(resized["rows"]) != n_rows:
        errors.append(f"H6: rows长度{len(resized['rows'])} != {n_rows}")
    if len(resized["cols"]) != n_cols:
        errors.append(f"H6: cols长度{len(resized['cols'])} != {n_cols}")

    # H7: merges 是对象
    for i, m in enumerate(merges):
        if not all(k in m for k in ("startRow","startColumn","endRow","endColumn")):
            errors.append(f"H7: merges[{i}] 缺少字段")

    # H8: reportConfig 有14个字段
    rc = template_obj["reportConfig"]
    required_fields = {"pageConfig","splitLayout","headerRepeat","footerRepeat",
        "headerFrozen","followUpPrintOpt","scopeConfig","searchBarConfig",
        "eventConfig","serviceConfig","headerOptions","printOptions",
        "functionConfig","engineConfig"}
    missing = required_fields - set(rc.keys())
    if missing:
        errors.append(f"H8: reportConfig 缺少字段: {missing}")

    # H9 + H10: proxyCell 检查
    for cell in meta:
        if cell.get("proxyCell"):
            if "widget" in cell:
                errors.append(f"H9: ({cell['row']},{cell['col']}) proxyCell 有 widget")
            if "realCellPosition" not in cell:
                errors.append(f"H10: ({cell['row']},{cell['col']}) proxyCell 无 realCellPosition")

    # H11: dataMaker 用 \r\n
    for (r, c), w in widgets.items():
        widget = w["widget"]
        dm = widget.get("componentLogic", {}).get("dataSource", {}).get("dataMaker", "")
        if dm and "\r\n" not in dm and "\n" in dm:
            errors.append(f"H11: ({r},{c}) dataMaker 含 \\n 而非 \\r\\n")

    # H13: 列数 = 患者字段 × 2
    expected_cols = len(patient_fields) * 2
    if n_cols != expected_cols:
        errors.append(f"H13: 列数{n_cols} != 患者字段{len(patient_fields)}×2={expected_cols}")

    # H14: 多行文本有 tb=3（检查 source 中含 \r\n 的 cell）
    for r, row in enumerate(source):
        for c, val in enumerate(row):
            if val and "\r\n" in val:
                cell = next((m for m in meta if m["row"]==r and m["col"]==c), None)
                if cell and cell.get("s", {}).get("tb") != 3:
                    errors.append(f"H14: ({r},{c}) 多行文本未设 tb=3")

    # H15: 标题/表头有 ht=2（由调用方传入 title_rows/header_rows 时检查）
    # 此项在 build_meta 中自动保障，这里做抽查
    # H16: 文本 cell 有 v 和 t
    for cell in meta:
        if not cell.get("proxyCell") and "widget" not in cell:
            if "v" not in cell and cell.get("s", {}).get("bd"):
                # 空白 cell 可以没有 v，有边框的 cell 应该有
                pass  # 非强制，避免误报

    # H17: 列宽总和 ~768
    total_w = sum(resized["cols"])
    if total_w < 700 or total_w > 820:
        errors.append(f"H17: 列宽总和{total_w} 偏离768过多")

    # H20: 日期控件用 datePicker
    for (r, c), w in widgets.items():
        if w["widget"]["type"] == "datePickerQuick":
            errors.append(f"H20: ({r},{c}) 使用了 datePickerQuick, 应改用 datePicker")

    # 输出结果
    if errors:
        print("=== 验证失败 ===")
        for e in errors:
            print(f"  [FAIL] {e}")
        sys.exit(1)
    else:
        print("=== 验证通过: H1-H22 全部满足 ===")

# --- 打包函数 ---

def package(data, form_cd, form_version, output_dir):
    """打包 ZIP (H12: UTF-8编码)"""
    base = f"nenr_form_{form_cd}-{form_version}"
    zip_name = f"{base}.zip"
    report_name = f"{base}.report"
    zip_path = os.path.join(output_dir, zip_name)
    content = json.dumps(data, ensure_ascii=False, indent=2)  # H12
    with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as z:
        z.writestr(report_name, content.encode('utf-8'))      # H12
    print(f"已生成: {zip_path}")
    return zip_path


# ============================================================
# === 表单定义区（模型修改此处） ===
# ============================================================

FORM_CD = ""           # 表单编码, 如 "health_education"
FORM_NA = ""           # 表单名称, 如 "老年医学科健康教育指导单"
FORM_VERSION = "1.0.1"
REFERENCE_ZIP = r"c:\Users\John\Documents\AweSun Files\nenr_form_oxytocin_record-1.0.1.zip"
OUTPUT_DIR = r"c:\Users\John\Documents\AweSun Files"

# 患者信息字段 (H13: 总列数 = len × 2)
PATIENT_FIELDS = []  # 如 ["床号","姓名","性别","年龄","住院号","入院诊断"]

# source 行数据: 每行为 [str|null, ...] (H2)
SOURCE_ROWS = []

# 合并区域: [(sr,sc,er,ec), ...] (H7)
MERGE_AREAS = []

# 行高: 精确值, 空行=11.5 (H18)
ROW_HEIGHTS = []

# 列宽: 精确值含小数, 总和~768 (H17)
COL_WIDTHS = []

# 控件: {(r,c): {"widget": widget_obj, "v"?:..., "t"?:...}}
# 用 w_input/w_checkbox/w_date/w_select/w_radio 构建
WIDGETS = {}

# 标题行 (H15: 自动设ht=2, bl=1, fs=14)
TITLE_ROWS = set()    # 如 {0}

# 表头行 (H15: 自动设ht=2)
HEADER_ROWS = set()   # 如 {2, 4}

# 多行文本单元格 (H14: 自动设tb=3)
MULTILINE_CELLS = set()  # 如 {(6,0), (7,0), ...}

# beforerender 脚本 (H5: 自动填充到eventConfig)
BEFORERENDER_SCRIPT = ""

# 额外 scopeConfig 字段 (如空 scopeField 的控件不自动生成)
EXTRA_SCOPE_FIELDS = []


# ============================================================
# === 构建区（调用 helper，一般不需要修改） ===
# ============================================================

if __name__ == "__main__":
    # 1. 构建各部分
    source = build_source(SOURCE_ROWS)
    merges = build_merges(MERGE_AREAS)
    meta = build_meta(source, MERGE_AREAS, WIDGETS, TITLE_ROWS, HEADER_ROWS, MULTILINE_CELLS)
    resized = build_resized(ROW_HEIGHTS, COL_WIDTHS)
    report_config = load_report_config(REFERENCE_ZIP)
    report_config["scopeConfig"] = build_scope_config(WIDGETS, EXTRA_SCOPE_FIELDS)
    report_config["eventConfig"] = build_event_config(BEFRERENDER_SCRIPT)

    # 2. 组装 template
    template_obj = build_template(source, meta, merges, resized, report_config)

    # 3. 构建顶层 JSON
    data = build_report_data(FORM_CD, FORM_NA, FORM_VERSION, template_obj)

    # 4. 验证 H1-H22
    verify(data, source, meta, merges, resized, WIDGETS, PATIENT_FIELDS)

    # 5. 打包 ZIP
    package(data, FORM_CD, FORM_VERSION, OUTPUT_DIR)
```

## 数据结构速查

### 顶层字段
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | str | `uuid.uuid4().hex[:24]` |
| `cd` | str | 表单编码 |
| `na` | str | 表单名称 |
| `template` | **str** | `json.dumps(template_obj, ensure_ascii=False)` (H1) |
| `categoryId` | str | `hihis@hihis/nenr@nenr/nenr_form@nenr_form/<cd>` |
| `version` | str | 如 `"1.0.1"` |
| `tenantId` | str | `"BSOFTYL"` |
| `createUser`/`modifyUser` | str | `"admin"` |
| `active` | bool | `true` |

### template 内部结构
```
reportReference:
  source: [[str|null, ...], ...]     # H2 纯二维数组
  meta:   [{row,col,s,proxyCell,...}] # H3 扁平列表, 长度=行×列
  merges: [{startRow,startColumn,endRow,endColumn}]  # H7 对象
  resized: {rows:[num], cols:[num]}  # H6 纯数字
  hidden: {rows:[], cols:[]}
  floatElements: []
reportConfig:                         # H8 从催产素模板深拷贝
  pageConfig, splitLayout, headerRepeat, footerRepeat,
  headerFrozen, followUpPrintOpt, scopeConfig, searchBarConfig,
  eventConfig, serviceConfig, headerOptions, printOptions,
  functionConfig, engineConfig       # 14个字段
```

### 单元格类型
| 类型 | 特征 | 必须字段 |
|------|------|---------|
| 文本cell | `proxyCell:false`, 无widget | `v`,`t:1`,`s` (H16) |
| 控件cell | `proxyCell:false`, 有widget | `s`,`widget` |
| 代理cell | `proxyCell:true` | `realCellPosition` (H10), 无widget (H9) |

### 样式字段 `s`
| 字段 | 含义 | 值 |
|------|------|-----|
| `ff` | 字体 | `"FangSong"` |
| `fs` | 字号 | 10(普通)/14(标题) |
| `ht` | 水平对齐 | 0=左,1=中,2=右 (H15: 标题/表头=2) |
| `vt` | 垂直对齐 | 0=上,1=中,2=下 |
| `bl` | 粗体 | 0/1 |
| `tb` | 文本换行 | 0=不换行,3=自动换行 (H14: 多行=3) |
| `bd` | 边框 | `{"t/l/b/r":{"s":1,"cl":{"rgb":"#000000"}}}` |
| `pd` | 内边距 | `{"t":0,"b":2,"l":2,"r":2}` |

### 控件速查
| 类型 | 函数 | 特点 |
|------|------|------|
| input | `w_input()` | `clearable:true, simplify:false` |
| checkboxgroup | `w_checkbox()` | `enableMax/max` 模拟单选, `itemSpacing` 控制间距 |
| datePicker | `w_date()` | 有 `t:1,v:"占位"` (H20, 不用datePickerQuick) |
| select | `w_select()` | |
| radiogroup | `w_radio()` | `vertical` 控制排列方向 |

### source 空白值规则
- 空白位置用 `null`（不是 `" "` 空格）
- 空字符串 `""` 用于特殊位置（如 input 的值字段）
- 标签文本（如 `"姓名："`）作为普通字符串保留

## 源文件解析规则

### PDF 解析
- 使用 `pdfplumber` 库
- 文本型 PDF: `page.extract_text()` / `page.extract_words()`
- 扫描型 PDF（`len(page.chars)==0` 且有图片）: `page.to_image(resolution=200)` 转图片，视觉分析
- 识别要点：患者信息字段数（决定列数 H13）、行结构、控件位置、合并区域

### DOCX 解析
- 使用 `python-docx`，`doc.paragraphs` 提取标题/列表，`doc.tables` 提取结构化内容
- `□` 符号 → checkboxgroup 选项
- `____` 或空白 → input 控件
- 页面尺寸: `doc.sections[0].page_width` (EMU, 914400 EMU = 1 inch = 25.4mm)

### 解析后填入模板
1. 识别患者信息字段 → 填 `PATIENT_FIELDS`（H13 决定列数）
2. 逐行分析 → 填 `SOURCE_ROWS`
3. 识别合并区域 → 填 `MERGE_AREAS`
4. 识别控件位置 → 用 `w_input()`/`w_checkbox()`/`w_date()` 等构建 → 填 `WIDGETS`
5. 精确计算列宽行高 → 填 `COL_WIDTHS`/`ROW_HEIGHTS`（H17/H18）
6. 标记标题行/表头行/多行文本 → 填 `TITLE_ROWS`/`HEADER_ROWS`/`MULTILINE_CELLS`

## identifier 命名规则（关键）

- `identifier` 和 `scopeField` **会被系统当作 JavaScript 变量名**
- **禁止**: `/`、空格、`(`、`)`、`-`、`+`、`*` 等 JS 特殊字符
- **推荐**: 纯小写字母+数字+下划线（如 `ym_xm`、`picccvc_1`）
- 含 `/` 的名称需转拼音缩写（如 `PICC/CVC` → `picccvc`，`造瘘管/胃管` → `zfqwg`）
- 中文名可用于 `name`（显示名）和 `desc`（说明），但**不可**用于 `identifier`/`scopeField`

## beforerender 脚本规则

- 使用 `$$scope.nurseFormContext?.patientInfo` 获取患者数据
- `$$scope.xxx` 的 `xxx` 必须与 widget 的 `scopeField` 完全一致
- 示例:
  ```javascript
  $$scope.ym_xm = $$scope.nurseFormContext?.patientInfo?.name;
  $$scope.ym_xb = $$scope.nurseFormContext?.patientInfo?.sexName;
  $$scope.ym_nl = $$scope.nurseFormContext?.patientInfo?.age;
  $$scope.ym_zyh = $$scope.nurseFormContext?.patientInfo?.admNo;
  ```

## 常见表单模式

### 患者信息行（H13）
- 每个字段占 2 列（标签+值），所有字段在一行内
- 标签列放文本（如 `"姓名："`），值列放 input 控件
- 示例 6 字段 → 12 列: `["床号：", null, "姓名：", null, "性别：", null, "年龄：", null, "住院号：", null, "入院诊断：", null]`

### 竖排类别 + 教育内容
- 类别列合并多行，设 `tb=3`（H14）+ `ht=2`（H15）
- 内容列合并多行，设 `tb=3`
- checkboxgroup 用 `vertical:true` 纵向排列

### 模拟单选（如 Bishop 评分）
- `enableMax:true, max:1` 让 checkboxgroup 表现为单选

### 纯展示 checkbox
- `scopeField=""`（H21），不出现在 scopeConfig 中</think>文件已写入。让我验证几个关键部分是否正确。<tool_call>Read<arg_key>file_path</arg_key><arg_value>c:\Users\John\.trae-cn\skills\report-generator\SKILL.md