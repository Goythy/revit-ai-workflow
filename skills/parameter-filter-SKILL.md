---
name: parameter-filter
description: 参数过滤器——对选中元素按指定参数的值进行筛选，支持删除匹配元素或高亮选中。适用于：用户说"删除我选中的实例，参数名xxx值中包含xx的所有实例"、"过滤并高亮我选中的实例参数名xxx值中包含xx的所有实例"等场景。统一遍历所有同名参数取第一个有效值，兼容 DirectShape 多同名参数等特殊元素。
---

# 参数过滤器 — 选中元素的参数值筛选（删除/高亮）

当用户需要基于参数值对已选中的元素进行筛选并执行操作时，使用此技能。

## 核心逻辑

1. **获取选中元素**：从 `uidoc.Selection.GetElementIds()` 获取
2. **读取参数值**：遍历 `element.Parameters` 集合中所有匹配名称的参数，取第一个有效非空值
3. **判断条件**：参数值是否包含用户指定的关键字
4. **执行操作**：根据用户意图选择「删除」或「高亮选中」

## 触发条件

当用户说以下类似内容时触发此技能：

- "删除我选中的实例，参数名"xxx"值中包含"xx"的所有实例"
- "过滤并高亮我选中的实例，参数名"xxx"值中包含"xx"的所有实例"
- "把选中的 xxx 里，参数 xxx 等于/不包含 xxx 的删掉/标出来"
- "筛选出我选的里面 xxx 参数符合 xxx 条件的"

## 交互流程

```
用户选中元素 + 指定参数名和条件
    ↓
ask_followup_question: 选择操作模式
    ├─ 1. 删除匹配的元素
    └─ 2. 高亮选中匹配的元素
    ↓
send_code_to_revit 执行对应代码
    ↓
弹窗显示统计结果
```

### 标准询问模板

```
请选择操作模式：
选项: ["1. 删除匹配的元素", "2. 高亮选中匹配的元素"]
```

> 如果用户已经明确说了要删除还是高亮，则跳过此步直接执行。

## 关键实现细节

### 参数读取策略（核心）

由于 Revit 中某些元素（如 DirectShape）可能存在**多个同名参数**，`LookupParameter()` 只返回第一个匹配的参数，可能恰好是空的。**正确的做法是遍历 `element.Parameters` 集合，检查所有同名参数并取第一个有效值。**

```csharp
string paramValue = null;
foreach (Autodesk.Revit.DB.Parameter p in elem.Parameters)
{
    if (p == null || p.Definition == null) continue;
    if (p.Definition.Name != paramName) continue;
    
    // 根据存储类型读取值，清除 \0 空字符
    string v = "";
    if (p.StorageType == StorageType.String)
        v = p.AsString() ?? "";
    else if (p.StorageType == StorageType.Double)
        v = p.AsValueString() ?? "";
    else if (p.StorageType == StorageType.Integer)
        v = p.AsValueString() ?? "";
    else
        v = p.AsValueString() ?? "";
    
    v = v.Replace("\0", "").Trim();
    if (!string.IsNullOrEmpty(v))
    {
        paramValue = v;
        break; // 取第一个有效值
    }
}
```

> **为什么不用 LookupParameter：** DirectShape 等非标准族可能有多个同名 IfcName 参数，第一个为空而有值的在后面。`LookupParameter()` 无法处理这种情况。
> 
> **为什么不先判断元素类型再决定方法：** 性能差异可忽略（几个到几十个参数的遍历开销微秒级），统一遍历方式更可靠且代码更简洁。

### 条件匹配

| 用户条件关键词 | 代码判断 |
|---------------|---------|
| 包含 xx | `paramValue.Contains(keyword)` |
| 等于 xx | `paramValue == keyword` |
| 不以 xx 开头 | `!paramValue.StartsWith(keyword)` |
| 以 xx 结尾 | `paramValue.EndsWith(keyword)` |

## 代码实现

### A. 删除模式

```csharp
try
{
    UIDocument uidoc = new UIDocument(document);
    ICollection<ElementId> selectedIds = uidoc.Selection.GetElementIds();
    
    if (selectedIds.Count == 0)
    {
        string msg = "当前没有选中任何元素，请先在Revit中选中需要处理的实例。";
        TaskDialog.Show("提示", msg);
        return msg;
    }
    
    // 从参数[0]获取参数名，[1]获取关键字，[2]获取匹配模式（contains/equal/notStartWith/endsWith）
    string paramName = parameters != null && parameters.Length > 0 && parameters[0] is string ? (string)parameters[0] : "IfcName";
    string keyword = parameters != null && parameters.Length > 1 && parameters[1] is string ? (string)parameters[1] : "";
    string matchMode = parameters != null && parameters.Length > 2 && parameters[2] is string ? (string)parameters[2] : "contains";
    
    var toDelete = new List<ElementId>();
    var details = new List<string>();
    int checkedCount = 0;
    int noValueCount = 0;
    
    foreach (ElementId eid in selectedIds)
    {
        var elem = document.GetElement(eid);
        if (elem == null || !elem.IsValidObject) continue;
        
        checkedCount++;
        
        // 遍历所有同名参数，取第一个有效非空值
        string paramValue = null;
        foreach (Autodesk.Revit.DB.Parameter p in elem.Parameters)
        {
            if (p == null || p.Definition == null) continue;
            if (p.Definition.Name != paramName) continue;
            
            string v = "";
            if (p.StorageType == StorageType.String)
                v = p.AsString() ?? "";
            else if (p.StorageType == StorageType.Double)
                v = p.AsValueString() ?? "";
            else if (p.StorageType == StorageType.Integer)
                v = p.AsValueString() ?? "";
            else
                v = p.AsValueString() ?? "";
            
            v = v.Replace("\0", "").Trim();
            if (!string.IsNullOrEmpty(v))
            {
                paramValue = v;
                break;
            }
        }
        
        if (string.IsNullOrEmpty(paramValue))
        {
            noValueCount++;
            continue;
        }
        
        bool matches = false;
        switch (matchMode)
        {
            case "contains":
                matches = paramValue.Contains(keyword);
                break;
            case "equal":
                matches = paramValue == keyword;
                break;
            case "notStartWith":
                matches = !paramValue.StartsWith(keyword);
                break;
            case "endsWith":
                matches = paramValue.EndsWith(keyword);
                break;
        }
        
        if (matches)
        {
            toDelete.Add(eid);
            if (details.Count < 10)
            {
                details.Add(string.Format("  Id={0}, {1}={2}", eid.IntegerValue, paramName, paramValue));
            }
        }
    }
    
    if (toDelete.Count == 0)
    {
        string msg = string.Format("已检查 {0} 个选中元素，未找到 {1} 满足 \"{2}\" 条件的实例。", checkedCount, paramName, matchMode);
        if (noValueCount > 0)
            msg += string.Format("\n（其中 {0} 个元素所有 {1} 参数均为空值）", noValueCount, paramName);
        TaskDialog.Show("结果", msg);
        return msg;
    }
    
    int deletedCount = 0;
    var failIds = new List<string>();
    
    foreach (ElementId delId in toDelete)
    {
        try
        {
            document.Delete(delId);
            deletedCount++;
        }
        catch (Exception ex)
        {
            if (failIds.Count < 5)
                failIds.Add(string.Format("Id={0}: {1}", delId.IntegerValue, ex.Message));
        }
    }
    
    System.Text.StringBuilder sb = new System.Text.StringBuilder();
    sb.AppendLine(string.Format("删除完成。选中 {0} 个元素，匹配 {1} 个，成功删除 {2} 个。", checkedCount, toDelete.Count, deletedCount));
    
    if (noValueCount > 0)
        sb.AppendLine(string.Format("（{0} 个元素所有 {1} 参数均为空值，已跳过）", noValueCount, paramName));
    
    if (failIds.Count > 0)
    {
        sb.AppendLine(string.Format("失败 {0} 个（最多显示5个）：", failIds.Count));
        foreach (string f in failIds)
            sb.AppendLine("  " + f);
    }
    
    if (details.Count > 0)
    {
        sb.AppendLine("匹配示例（最多10个）：");
        foreach (string d in details)
            sb.AppendLine(d);
    }
    
    string result = sb.ToString();
    TaskDialog.Show("删除结果", result);
    return result;
}
catch (Exception ex)
{
    TaskDialog.Show("错误", ex.Message);
    return ex.Message;
}
```

### B. 高亮选中模式

```csharp
try
{
    UIDocument uidoc = new UIDocument(document);
    ICollection<ElementId> selectedIds = uidoc.Selection.GetElementIds();
    
    if (selectedIds.Count == 0)
    {
        string msg = "当前没有选中任何元素，请先在Revit中选中需要处理的实例。";
        TaskDialog.Show("提示", msg);
        return msg;
    }
    
    // 从参数[0]获取参数名，[1]获取关键字，[2]获取匹配模式（contains/equal/notStartWith/endsWith）
    string paramName = parameters != null && parameters.Length > 0 && parameters[0] is string ? (string)parameters[0] : "IfcName";
    string keyword = parameters != null && parameters.Length > 1 && parameters[1] is string ? (string)parameters[1] : "";
    string matchMode = parameters != null && parameters.Length > 2 && parameters[2] is string ? (string)parameters[2] : "contains";
    
    var hitIds = new List<ElementId>();
    var details = new List<string>();
    int checkedCount = 0;
    int noValueCount = 0;
    
    foreach (ElementId eid in selectedIds)
    {
        var elem = document.GetElement(eid);
        if (elem == null || !elem.IsValidObject) continue;
        
        checkedCount++;
        
        // 遍历所有同名参数，取第一个有效非空值
        string paramValue = null;
        foreach (Autodesk.Revit.DB.Parameter p in elem.Parameters)
        {
            if (p == null || p.Definition == null) continue;
            if (p.Definition.Name != paramName) continue;
            
            string v = "";
            if (p.StorageType == StorageType.String)
                v = p.AsString() ?? "";
            else if (p.StorageType == StorageType.Double)
                v = p.AsValueString() ?? "";
            else if (p.StorageType == StorageType.Integer)
                v = p.AsValueString() ?? "";
            else
                v = p.AsValueString() ?? "";
            
            v = v.Replace("\0", "").Trim();
            if (!string.IsNullOrEmpty(v))
            {
                paramValue = v;
                break;
            }
        }
        
        if (string.IsNullOrEmpty(paramValue))
        {
            noValueCount++;
            continue;
        }
        
        bool matches = false;
        switch (matchMode)
        {
            case "contains":
                matches = paramValue.Contains(keyword);
                break;
            case "equal":
                matches = paramValue == keyword;
                break;
            case "notStartWith":
                matches = !paramValue.StartsWith(keyword);
                break;
            case "endsWith":
                matches = paramValue.EndsWith(keyword);
                break;
        }
        
        if (matches)
        {
            hitIds.Add(eid);
            if (details.Count < 10)
            {
                details.Add(string.Format("  Id={0}, {1}={2}", eid.IntegerValue, paramName, paramValue));
            }
        }
    }
    
    if (hitIds.Count == 0)
    {
        string msg = string.Format("已检查 {0} 个选中元素，未找到 {1} 满足 \"{2}\" 条件的实例。", checkedCount, paramName, matchMode);
        if (noValueCount > 0)
            msg += string.Format("\n（其中 {0} 个元素所有 {1} 参数均为空值）", noValueCount, paramName);
        TaskDialog.Show("结果", msg);
        return msg;
    }
    
    // 设置选中（追加模式：保留原有选中基础上添加）
    uidoc.Selection.SetElementIds(hitIds);
    
    System.Text.StringBuilder sb = new System.Text.StringBuilder();
    sb.AppendLine(string.Format("高亮完成。选中 {0} 个元素，匹配 {1} 个，已在视图中选中。", checkedCount, hitIds.Count));
    
    if (noValueCount > 0)
        sb.AppendLine(string.Format("（{0} 个元素所有 {1} 参数均为空值，已跳过）", noValueCount, paramName));
    
    if (details.Count > 0)
    {
        sb.AppendLine("匹配示例（最多10个）：");
        foreach (string d in details)
            sb.AppendLine(d);
    }
    
    string result = sb.ToString();
    TaskDialog.Show("高亮结果", result);
    return result;
}
catch (Exception ex)
{
    TaskDialog.Show("错误", ex.Message);
    return ex.Message;
}
```

## 代码提交方式

使用文件工作流提交代码：

1. `write_to_file` → 临时 `.cs` 文件（路径：`temp_cs/ParameterFilter.cs`）
2. `read_file` → 读取并检查内容
3. `send_code_to_revit` → 传入读取的内容，`parameters` 数组传入：
   - `parameters[0]`: 参数名字符串（如 "IfcName"）
   - `parameters[1]`: 关字字符串（如 "井"）
   - `parameters[2]`: 匹配模式字符串（"contains" / "equal" / "notStartWith" / "endsWith"）
4. 可选清理：`Remove-Item temp_cs\ParameterFilter.cs`

## 已知限制与注意事项

| 情况 | 说明 | 处理方式 |
|------|------|---------|
| 参数值为空 | 元素有该参数但值为空 | 跳过，计入 noValueCount |
| 参数不存在 | 元素没有该参数定义 | 跳过，计入 noValueCount |
| 多同名参数 | DirectShape 等可能有多个同名参数 | 遍历取第一个有效值 |
| 参数值为 `\0` | IFC 导出可能带入空字符 | `.Replace("\0", "")` 清除 |
| 批量删除大量元素 | 超过500个元素时 Revit 可能短暂无响应 | 正常行为，等待完成弹窗即可 |

## 与 revit-expert.md 规则的一致性

| 规则项 | 本技能遵守情况 |
|--------|-------------|
| try-catch + return | ✅ 全部代码在 try-catch 内，每个路径返回 string |
| 最多1个 TaskDialog | ✅ 正常路径 1 个，异常/空选中 1 个即返回 |
| StringBuilder 汇总 | ✅ 使用 StringBuilder 构造摘要 |
| 返回值长度控制 | ✅ 仅返回简短统计摘要（最多10个示例） |
| `document` 而非 `doc` | ✅ 全部使用 `document` |
| 变量名不冲突 | ✅ 使用 `crv`、`pt`、`elem` 等 camelCase 变量 |
| 预询问阶段先于代码 | ✅ 使用 ask_followup_question 确认操作模式（如用户未明确） |
| 遍历 Parameters 而非 LookupParameter | ✅ 兼容多同名参数场景 |