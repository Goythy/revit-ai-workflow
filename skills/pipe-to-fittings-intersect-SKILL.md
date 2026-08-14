---
name: pipe-to-fittings-intersect
description: 查找并与管件（弯头/三通/雨水口等）空间相交的管道实例并高亮选中。适用于：用户说"选择雨水口的连接管"、"选择与雨水口相连的管道"、"找出和管件相交的管道"、"选中连在管件上的管道"等场景。不依赖选中状态，自动扫描全项目管件。
---

# 管件相交管道检测与高亮

当用户需要找出与管件（弯头、三通、雨水口等）相交的管道时，执行此技能。**该方法基于空间几何检测，不依赖连接器等 API 接口，因此可处理 DirectShape 类型的自定义族（如雨水口）。**

## 核心逻辑

程序自动执行：
1. **全量扫描**所有类别为 `OST_PipeFittings`（CatId = -2008049）的管件元素（含 `DirectShape`、`FamilyInstance` 等）
2. 从**当前三维视图**收集所有 `Pipe` 管道实例
3. 对每个管件 + 管道组合做**双重判据**相交检测
4. 将所有相交的管道在视图中选中高亮

> **注意：** 本技能不需要用户预先选中任何元素——自动全量扫描。如果用户在三维视图中已选中特定管件，也可仅针对这些选中管件检测（见下方扩展用法）。

## 工作流概览

```
启动技能
    ↓
确认当前激活的是 View3D（非平面/剖面视图）
    ↓
从项目全量收集 PipeFittings 类别元素（直接遍历 Categories，不依赖 BuiltInCategory 枚举）
    ↓
从当前 View3D 收集 Pipe 实例
    ↓
双重判据相交检测：
├─ 判据A：包围盒重叠预筛（+45mm 容差）→ 快速排除无关管道
└─ 判据B：管道 LocationCurve 采样点落入放大的管件包围盒 → 确认相交
    ↓
Selection.SetElementIds(命中管道) → TaskDialog 弹窗报告
```

## 关键实现细节

### 1. 获取管件类别（兼容无 BuiltInCategory.OST_PipeFittings 的环境）

```csharp
int pipeFittingCatId = -2008049; // OST_PipeFittings 整数值

// 通过 document.Settings.Categories 遍历匹配
Autodesk.Revit.DB.Category fittingCat = null;
foreach (Autodesk.Revit.DB.Category cat in document.Settings.Categories)
{
    if (cat.Id.IntegerValue == pipeFittingCatId)
    {
        fittingCat = cat;
        break;
    }
}

// 优先用 ElementCategoryFilter
List<Autodesk.Revit.DB.Element> allFittings;
if (fittingCat != null)
{
    allFittings = new FilteredElementCollector(document)
        .WherePasses(new ElementCategoryFilter(fittingCat.Id))
        .WhereElementIsNotElementType()
        .Cast<Autodesk.Revit.DB.Element>()
        .ToList();
}
else
{
    // 回退：直接按整数 Id 过滤
    allFittings = new FilteredElementCollector(document)
        .WhereElementIsNotElementType()
        .Cast<Autodesk.Revit.DB.Element>()
        .Where(e => e.Category != null && e.Category.Id.IntegerValue == pipeFittingCatId)
        .ToList();
}
```

> **为什么不用 `BuiltInCategory.OST_PipeFittings`：** 某些 Revit 版本或编译环境中该枚举不存在，导致编译失败。使用整数 Id 或遍历 Categories 更安全。

### 2. 双重判据相交检测

#### 判据 A — 包围盒重叠（快速预筛）

```csharp
bool BoxesOverlap(Autodesk.Revit.DB.BoundingBoxXYZ a, 
                  Autodesk.Revit.DB.BoundingBoxXYZ b, double tol)
{
    return a.Min.X - tol <= b.Max.X && a.Max.X + tol >= b.Min.X &&
           a.Min.Y - tol <= b.Max.Y && a.Max.Y + tol >= b.Min.Y &&
           a.Min.Z - tol <= b.Max.Z && a.Max.Z + tol >= b.Min.Z;
}
```

> 这是轴对齐包围盒（AABB）的交集测试，带 `tol` 容差（建议 0.15ft ≈ 45mm），覆盖管件/管道表面接触范围。

#### 判据 B — 管道中心线采样穿入（确认相交）

```csharp
bool CurveHitsBox(Autodesk.Revit.DB.Curve crv, 
                  Autodesk.Revit.DB.BoundingBoxXYZ box, double pad)
{
    if (crv == null) return false;
    double len = crv.Length;
    if (len < 1e-6) return false;
    
    int n = (int)Math.Ceiling(len / 0.5); // 每约 500mm 采样
    if (n < 5) n = 5;
    if (n > 100) n = 100;

    double minX = box.Min.X - pad, maxX = box.Max.X + pad;
    double minY = box.Min.Y - pad, maxY = box.Max.Y + pad;
    double minZ = box.Min.Z - pad, maxZ = box.Max.Z + pad;

    for (int i = 0; i <= n; i++)
    {
        var pt = crv.Evaluate(i / (double)n, false);
        if (pt.X >= minX && pt.X <= maxX &&
            pt.Y >= minY && pt.Y <= maxY &&
            pt.Z >= minZ && pt.Z <= maxZ)
            return true;
    }
    return false;
}
```

> **原理：** 管道 `LocationCurve` 是其中心线，将中心线离散化采样后检查是否有任何采样点落在放大的管件包围盒内。如果有 → 表示管道穿入/端接了管件区域。

#### 遍历执行

```csharp
var hitPipeIds = new HashSet<Autodesk.Revit.DB.ElementId>();
double padFeet = 0.15; // 约 45mm 放大量

foreach (var fit in allFittings)
{
    var fitBox = fit.get_BoundingBox(activeView);
    if (fitBox == null) continue;

    foreach (var pipe in pipes)
    {
        if (hitPipeIds.Contains(pipe.Id)) continue; // 已命中跳过

        var pipeBox = pipe.get_BoundingBox(activeView);
        if (pipeBox == null) continue;

        // 判据A
        if (!BoxesOverlap(fitBox, pipeBox, padFeet)) continue;

        // 判据B
        var locCurve = pipe.Location as Autodesk.Revit.DB.LocationCurve;
        if (locCurve == null || locCurve.Curve == null) continue;

        if (CurveHitsBox(locCurve.Curve, fitBox, padFeet))
            hitPipeIds.Add(pipe.Id);
    }
}
```

### 3. 选中并报告结果

```csharp
if (hitPipeIds.Count == 0)
{
    string msg0 = "未找到与任意管件相交的管道实例。";
    TaskDialog.Show("提示", msg0);
    return msg0;
}

List<Autodesk.Revit.DB.ElementId> idList = hitPipeIds.ToList();
uidoc.Selection.SetElementIds(idList);

string summary = "共找到 " + hitPipeIds.Count + " 根与管件相交的管道，已在视图中选中。";
TaskDialog.Show("结果", summary);
return summary;
```

## 性能参考

| 条件 | 表现 |
|------|------|
| ~300+ 管件 × ~500+ 管道 | Revit 短暂无响应约 10-30s，之后弹窗显示结果 |
| MCP 超时（>30s） | 可能发生，但代码仍在 Revit 内执行，等待弹窗即可 |
| 无需分批次提交 | 单次执行完全可行 |

## 扩展用法（可选）— 仅针对已选中的管件

如果用户已选中特定管件（如只想看某个区域的雨水口连接管），可将上述 `allFittings` 替换为：

```csharp
int pipeFittingCatId = -2008049;
var selectedIds = uidoc.Selection.GetElementIds();
var selectedFittings = new List<Autodesk.Revit.DB.Element>();

foreach (var id in selectedIds)
{
    var e = document.GetElement(id);
    if (e != null && e.Category != null && e.Category.Id.IntegerValue == pipeFittingCatId)
        selectedFittings.Add(e);
}

// 然后将 allFittings 改为 selectedFittings 继续检测
```

这样仅找出与**当前选中管件**相交的管道。

## 已知限制与注意事项

| 情况 | 说明 | 处理方式 |
|------|------|---------|
| 管道从管件正上方/正下方穿过但不连接 | 可能被误识别为相交 | 正常行为——因 DirectShape 无连接器信息，只能靠空间几何判定 |
| 复杂弯头多段管线 | 可能检测到多个交点 | 管道只记录一次（HashSet），不影响结果 |
| 管道类型为 AnalysisPipe 或非 LocationCurve | 无法提取中心线 | 被自动跳过 |

## 代码执行规范

遵循 revit-expert.md 规则：

- 使用文件工作流（先写 `.cs` → `read_file` → `send_code_to_revit`）避免 JSON 编码问题
- 全部逻辑包裹 `try-catch`，每个路径必须 `return string`
- 使用英文分号，变量名避免与 Revit API 类型冲突（用 `crv` 而非 `Curve`）
- 批量操作前注释说明："即将处理 N 个管件 × M 根管道，Revit 可能出现短暂无响应，请耐心等待完成弹窗。"
- 结果仅返回统计摘要（数量），不输出逐条 ID

## 交互流程

1. **用户触发指令** → 直接发送代码执行（无需额外选择）
2. **执行完成后** → 弹窗显示命中的管道数量，管道已在视图中选中高亮
3. **如需调整精度** → 修改 `padFeet` 参数（增大更宽松，减小更严格）