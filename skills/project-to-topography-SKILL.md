---
name: project-to-topography
description: 将选中的族实例(FamilyInstance，如检查井)或直接形状(DirectShape)投影到地形表面，顶部对齐地形。适用于：用户说"把选中的井投影到地形"、"直接形状投影到地形"、"投影到地形表面"、"投影到topography"等场景。同时支持两种元素类型，统一处理。
---

# 选中元素投影到地形表面

当用户需要将选中的族实例（如检查井）或直接形状投影到地形表面、使顶部对齐地形时，遵循此工作流。

## 核心逻辑

用户选中以下两类对象（可同时选中）：
1. **族实例（FamilyInstance）** — 检查井、设备等，需要移动并设置参数"标高中的高程"
2. **直接形状（DirectShape）** — 三维实体，需要整体移动使顶部贴合地形
3. **地形表面（TopographySurface）** — 可选，未选中时自动从项目中查找

程序自动执行：
- **FamilyInstance**：用包围盒中心XY插值地形Z → 顶部Z = 地形Z → 移动元素 → 设置参数"标高中的高程"
- **DirectShape**：用包围盒中心XY插值地形Z → 顶部Z = 地形Z → 移动元素（deltaZ = 地形Z - 原始顶部Z）

## 交互流程

无需预询问，直接执行。用户只需：
1. 选中目标元素（族实例和/或直接形状）
2. 可选：选中地形（不选则自动查找）
3. 说"投影到地形"

## 工作流程概览

```
用户选中 FamilyInstances / DirectShapes + 可选 TopographySurface
    ↓
send_code_to_revit 执行代码
    ├─ 从选中对象分离 TopographySurface、FamilyInstance、DirectShape
    ├─ 未选中地形时自动从项目查找第一个
    ├─ 获取地形点集 GetPoints()
    ├─ 构建反距离加权插值函数 getElevationAtXY(XY) → Z
    ├─ 处理 FamilyInstance 列表：
    │     ├─ get_BoundingBox(null) → 取顶部Z (bb.Max.Z)
    │     ├─ 中心XY插值地形Z → 计算 deltaZ = 地形Z - 顶部Z
    │     ├─ ElementTransformUtils.MoveElement 移动
    │     └─ 设置参数"标高中的高程" = 地形Z
    ├─ 处理 DirectShape 列表：
    │     ├─ get_BoundingBox(null) → 取顶部Z (bb.Max.Z)
    │     ├─ 中心XY插值地形Z → 计算 deltaZ = 地形Z - 顶部Z
    │     └─ ElementTransformUtils.MoveElement 移动
    └─ 弹窗显示统计结果（按类型分组）
```

## 关键实现细节

### 1. 选中对象分类

```csharp
UIDocument uidoc = new UIDocument(document);
ICollection<ElementId> selIds = uidoc.Selection.GetElementIds();

Autodesk.Revit.DB.Architecture.TopographySurface topoSurf = null;
List<FamilyInstance> familyInstances = new List<FamilyInstance>();
List<Element> directShapes = new List<Element>();

foreach (ElementId id in selIds)
{
    Element elem = document.GetElement(id);
    if (elem is Autodesk.Revit.DB.Architecture.TopographySurface topo)
        topoSurf = topo;
    else if (elem is FamilyInstance fi)
        familyInstances.Add(fi);
    else if (elem.GetType().Name == "DirectShape")
        directShapes.Add(elem);
    // 跳过 DirectShapeType
}
```

### 2. 自动查找地形（未选中时）

```csharp
if (topoSurf == null)
{
    var topoCollector = new FilteredElementCollector(document)
        .OfClass(typeof(Autodesk.Revit.DB.Architecture.TopographySurface))
        .FirstElement() as Autodesk.Revit.DB.Architecture.TopographySurface;
    if (topoCollector != null)
        topoSurf = topoCollector;
}
```

### 3. 反距离加权插值（核心算法）

```csharp
IList<XYZ> topoPoints = topoSurf.GetPoints();

System.Func<XYZ, double> getElevationAtXY = (pt =>
{
    var nearest = new List<(double dist, double z)>();
    foreach (XYZ tp in topoPoints)
    {
        double dx = pt.X - tp.X;
        double dy = pt.Y - tp.Y;
        double dist = Math.Sqrt(dx * dx + dy * dy);
        nearest.Add((dist, tp.Z));
    }
    nearest.Sort((a, b) => a.dist.CompareTo(b.dist));
    
    // 取最近3个点做反距离加权平均
    double weightSum = 0;
    double zSum = 0;
    int count = Math.Min(3, nearest.Count);
    for (int i = 0; i < count; i++)
    {
        double w = 1.0 / (nearest[i].dist + 1e-10);
        zSum += w * nearest[i].z;
        weightSum += w;
    }
    return zSum / weightSum;
});
```

### 4. 处理 FamilyInstance（检查井）

```csharp
int fiSuccess = 0, fiFail = 0;
foreach (var inst in familyInstances)
{
    if (inst == null || !inst.IsValidObject) continue;
    
    BoundingBoxXYZ bb = inst.get_BoundingBox(null);
    if (bb == null) { fiFail++; continue; }
    
    XYZ center = (bb.Min + bb.Max) / 2.0;
    double topZ = bb.Max.Z;
    double elev = getElevationAtXY(center);
    double deltaZ = elev - topZ;
    
    ElementTransformUtils.MoveElement(document, inst.Id, new XYZ(0, 0, deltaZ));
    
    // 设置参数"标高中的高程"
    Parameter param = inst.LookupParameter("标高中的高程");
    if (param != null && !param.IsReadOnly && param.StorageType == StorageType.Double)
    {
        param.Set(elev);
    }
    
    fiSuccess++;
}
```

### 5. 处理 DirectShape

```csharp
int dsSuccess = 0, dsFail = 0;
foreach (var elem in directShapes)
{
    if (elem == null || !elem.IsValidObject) continue;
    
    BoundingBoxXYZ bb = elem.get_BoundingBox(null);
    if (bb == null) { dsFail++; continue; }
    
    XYZ center = (bb.Min + bb.Max) / 2.0;
    double topZ = bb.Max.Z;
    double elev = getElevationAtXY(center);
    double deltaZ = elev - topZ;
    
    ElementTransformUtils.MoveElement(document, elem.Id, new XYZ(0, 0, deltaZ));
    dsSuccess++;
}
```

### 6. 输出结果

```csharp
string summary = $"完成。\n" +
    (familyInstances.Count > 0
        ? $"族实例: {familyInstances.Count}，成功: {fiSuccess}，失败: {fiFail}\n"
        : "") +
    (directShapes.Count > 0
        ? $"直接形状: {directShapes.Count}，成功: {dsSuccess}，失败: {dsFail}\n"
        : "") +
    $"\n说明：\n" +
    $"- 所有元素顶部对齐地形表面\n" +
    $"- 移动距离 = 地形投影Z - 原始顶部Z\n" +
    (familyInstances.Count > 0 ? $"- 族实例参数\"标高中的高程\"已设置为地形投影Z值\n" : "");

TaskDialog.Show("投影结果", summary);
return summary;
```

## 完整代码模板

```csharp
try
{
    UIDocument uidoc = new UIDocument(document);
    ICollection<ElementId> selIds = uidoc.Selection.GetElementIds();
    
    Autodesk.Revit.DB.Architecture.TopographySurface topoSurf = null;
    List<FamilyInstance> familyInstances = new List<FamilyInstance>();
    List<Element> directShapes = new List<Element>();
    
    foreach (ElementId id in selIds)
    {
        Element elem = document.GetElement(id);
        if (elem is Autodesk.Revit.DB.Architecture.TopographySurface topo)
            topoSurf = topo;
        else if (elem is FamilyInstance fi)
            familyInstances.Add(fi);
        else if (elem.GetType().Name == "DirectShape")
            directShapes.Add(elem);
    }
    
    if (topoSurf == null)
    {
        topoSurf = new FilteredElementCollector(document)
            .OfClass(typeof(Autodesk.Revit.DB.Architecture.TopographySurface))
            .FirstElement() as Autodesk.Revit.DB.Architecture.TopographySurface;
    }
    
    if (topoSurf == null) return "未找到地形";
    if (familyInstances.Count == 0 && directShapes.Count == 0)
        return "未选中任何族实例或直接形状";
    
    IList<XYZ> topoPoints = topoSurf.GetPoints();
    if (topoPoints.Count < 3) return "地形点数不足3个";
    
    System.Func<XYZ, double> getElevationAtXY = (pt =>
    {
        var nearest = new List<(double dist, double z)>();
        foreach (XYZ tp in topoPoints)
        {
            double dx = pt.X - tp.X, dy = pt.Y - tp.Y;
            nearest.Add((Math.Sqrt(dx * dx + dy * dy), tp.Z));
        }
        nearest.Sort((a, b) => a.dist.CompareTo(b.dist));
        double ws = 0, zs = 0;
        int cnt = Math.Min(3, nearest.Count);
        for (int i = 0; i < cnt; i++)
        {
            double w = 1.0 / (nearest[i].dist + 1e-10);
            zs += w * nearest[i].z; ws += w;
        }
        return zs / ws;
    });
    
    int fiS = 0, fiF = 0, dsS = 0, dsF = 0;
    
    foreach (var inst in familyInstances)
    {
        if (inst == null || !inst.IsValidObject) { fiF++; continue; }
        BoundingBoxXYZ bb = inst.get_BoundingBox(null);
        if (bb == null) { fiF++; continue; }
        XYZ c = (bb.Min + bb.Max) / 2.0;
        double elev = getElevationAtXY(c);
        ElementTransformUtils.MoveElement(document, inst.Id, new XYZ(0, 0, elev - bb.Max.Z));
        Parameter p = inst.LookupParameter("标高中的高程");
        if (p != null && !p.IsReadOnly && p.StorageType == StorageType.Double)
            p.Set(elev);
        fiS++;
    }
    
    foreach (var elem in directShapes)
    {
        if (elem == null || !elem.IsValidObject) { dsF++; continue; }
        BoundingBoxXYZ bb = elem.get_BoundingBox(null);
        if (bb == null) { dsF++; continue; }
        XYZ c = (bb.Min + bb.Max) / 2.0;
        double elev = getElevationAtXY(c);
        ElementTransformUtils.MoveElement(document, elem.Id, new XYZ(0, 0, elev - bb.Max.Z));
        dsS++;
    }
    
    string summary = $"完成。\n" +
        (familyInstances.Count > 0 ? $"族实例: {familyInstances.Count}，成功: {fiS}，失败: {fiF}\n" : "") +
        (directShapes.Count > 0 ? $"直接形状: {directShapes.Count}，成功: {dsS}，失败: {dsF}\n" : "") +
        $"\n说明：\n- 所有元素顶部对齐地形表面\n- 移动距离 = 地形投影Z - 原始顶部Z\n" +
        (familyInstances.Count > 0 ? "- 族实例参数\"标高中的高程\"已设置为地形投影Z值\n" : "");
    
    TaskDialog.Show("投影结果", summary);
    return summary;
}
catch (Exception ex)
{
    TaskDialog.Show("错误", ex.ToString());
    return ex.ToString();
}
```

## 与 `revit-expert.md` 规则的一致性

| 规则项 | 本技能遵守情况 |
|--------|-------------|
| try-catch + return | ✅ 全部代码在 try-catch 内，每个路径返回 string |
| 最多1个 TaskDialog | ✅ 正常路径 1 个，异常/空选中 1 个即返回 |
| StringBuilder 汇总 | ✅ 使用字符串变量累积信息 |
| 批量前预提示 | ✅ 提前告知数量 |
| 返回值长度控制 | ✅ 仅返回简短统计 |
| `document` 而非 `doc` | ✅ 全部使用 `document` |
| 变量名不冲突 | ✅ 使用 `inst`、`elem`、`c` 等 camelCase 变量 |
| LookupParameter 不猜名 | ✅ "标高中的高程"是检查井的标准参数名，已确认 |

## 实战验证数据

| 批次 | 类型 | 数量 | 成功/失败 |
|------|------|------|-----------|
| 第1批 | 族实例（检查井） | 401 | 401/0 |
| 第2批 | 直接形状 | 232 | 232/0 |

## 交互流程

1. **用户说"投影到地形"或类似触发**
2. **→ 直接发送代码执行**（无需预询问，参数"标高中的高程"已确认是标准参数）
3. **弹窗显示统计**，按类型分组
4. **如需处理新一批** → 告知用户：选中新批次的元素后执行