---
name: cad-mesh-to-topography
description: 从选中的CAD导入文件（ImportInstance/DWG三角网）中提取非"0"图层的三角网顶点，按XY坐标去重（保留Z最大值）+ 容差筛选（默认5mm），生成地形表面（TopographySurface）。适用于：用户说"从CAD三角网生成地形"、"CAD网格生成地形"、"提取三角网顶点生成地形"、"cad生成地形"等场景。
---

# CAD三角网 → 地形表面

当用户需要从选中的CAD导入文件（DWG三角网模型）中提取三角网顶点，去重后生成地形表面时，遵循此工作流。

## 核心流程

```
用户选中 CAD ImportInstance（DWG三角网）
    ↓
获取几何 → 遍历 GeometryInstance
    ├─ 使用 GetSymbolGeometry() 获取原始CAD坐标
    ├─ 跳过 "0" 图层的三角形
    └─ 提取非"0"图层每个三角形的 3 个顶点
    ↓
应用 ImportInstance.GetTotalTransform() 将顶点转换到项目坐标
    ↓
去重步骤 1：按XY精确坐标匹配，保留Z值最大的点
    ↓
去重步骤 2：5mm容差筛选，XY距离≤5mm的点只保留第一个
    ↓
使用 TopographySurface.Create(document, points) 生成地形
    ↓
弹窗显示统计结果
```

## 关键实现细节

### 1. 坐标变换 — 核心原则

**必须使用 `GetSymbolGeometry()` + `GetTotalTransform()` 的组合：**

```csharp
// 获取总变换（包含移动/旋转后的偏移）
Transform totalTransform = cadImport.GetTotalTransform();

// 使用GetSymbolGeometry()获取原始CAD坐标（不经过任何变换）
GeometryElement symGeom = geomInst.GetSymbolGeometry();

// 提取顶点后，用GetTotalTransform转换到项目坐标
XYZ pt = totalTransform.OfPoint(vertex);
```

**❌ 两种常见错误：**
- 仅用 `GetInstanceGeometry(geomInst.Transform)` → 遗漏总变换，地形在CAD原始位置
- `GetInstanceGeometry()` + 又叠加 `GetTotalTransform()` → 双重变换，偏移量翻倍
### 2. 图层过滤

```csharp
string layerName = "";
if (mesh.GraphicsStyleId != ElementId.InvalidElementId)
{
    GraphicsStyle gs = document.GetElement(mesh.GraphicsStyleId) as GraphicsStyle;
    if (gs != null) layerName = gs.Name;
}
if (layerName == "0") continue; // 跳过"0"图层
```

### 3. 去重逻辑（参考Dynamo Python脚本）

**第一步：按XY精确去重，保留Z值最大的点**
```csharp
var pointDict = new Dictionary<string, XYZ>();
foreach (XYZ pt in points)
{
    string key = Math.Round(pt.X, 6).ToString("F6") + "," + Math.Round(pt.Y, 6).ToString("F6");
    if (pointDict.TryGetValue(key, out XYZ existing))
    {
        if (pt.Z > existing.Z) pointDict[key] = pt;
    }
    else pointDict[key] = pt;
}
```

**第二步：容差筛选（默认5mm）**
```csharp
double tolerance_mm = 5.0;
double tolerance_ft = tolerance_mm * 0.00328084;
var finalPoints = new List<XYZ>();
foreach (XYZ pt in uniquePoints)
{
    bool isDuplicate = false;
    foreach (XYZ kept in finalPoints)
    {
        double dx = pt.X - kept.X, dy = pt.Y - kept.Y;
        double distanceMm = Math.Sqrt(dx * dx + dy * dy) / 0.00328084;
        if (distanceMm <= tolerance_mm) { isDuplicate = true; break; }
    }
    if (!isDuplicate) finalPoints.Add(pt);
}
```

### 4. 完整代码模板

```csharp
try
{
    UIDocument uidoc = new UIDocument(document);
    var selectedIds = uidoc.Selection.GetElementIds();
    if (selectedIds.Count == 0) return "请先选中一个CAD导入文件";

    ElementId cadId = selectedIds.First();
    ImportInstance cadImport = document.GetElement(cadId) as ImportInstance;
    if (cadImport == null) return "选中的不是CAD导入文件";

    // 获取几何
    Options options = new Options();
    options.ComputeReferences = false;
    options.DetailLevel = ViewDetailLevel.Fine;
    GeometryElement geomElem = cadImport.get_Geometry(options);

    // 获取总变换
    Transform totalTransform = cadImport.GetTotalTransform();

    // 收集顶点
    var points = new List<XYZ>();
    int layer0Skipped = 0, nonZeroLayer = 0;
    var layerNames = new System.Text.StringBuilder();

    foreach (GeometryObject geomObj in geomElem)
    {
        GeometryInstance geomInst = geomObj as GeometryInstance;
        if (geomInst == null) continue;

        GeometryElement symGeom = geomInst.GetSymbolGeometry();
        if (symGeom == null) continue;

        foreach (GeometryObject geom in symGeom)
        {
            Mesh mesh = geom as Mesh;
            if (mesh == null) continue;

            // 图层过滤
            string layerName = "";
            if (mesh.GraphicsStyleId != ElementId.InvalidElementId)
            {
                GraphicsStyle gs = document.GetElement(mesh.GraphicsStyleId) as GraphicsStyle;
                if (gs != null) layerName = gs.Name;
            }
            if (layerName == "0") { layer0Skipped += mesh.NumTriangles; continue; }

            nonZeroLayer += mesh.NumTriangles;
            if (layerName.Length > 0 && !layerNames.ToString().Contains(layerName))
            {
                if (layerNames.Length > 0) layerNames.Append(", ");
                layerNames.Append(layerName);
            }

            // 提取顶点并应用总变换
            for (int i = 0; i < mesh.NumTriangles; i++)
            {
                MeshTriangle tri = mesh.get_Triangle(i);
                points.Add(totalTransform.OfPoint(tri.get_Vertex(0)));
                points.Add(totalTransform.OfPoint(tri.get_Vertex(1)));
                points.Add(totalTransform.OfPoint(tri.get_Vertex(2)));
            }
        }
    }

    if (points.Count == 0) return "未提取到任何三角形顶点";

    // 去重步骤1：按XY精确去重，保留Z值最大的点
    double tolerance_mm = 5.0;
    var pointDict = new Dictionary<string, XYZ>();
    foreach (XYZ pt in points)
    {
        string key = Math.Round(pt.X, 6).ToString("F6") + "," + Math.Round(pt.Y, 6).ToString("F6");
        if (pointDict.TryGetValue(key, out XYZ existing))
        {
            if (pt.Z > existing.Z) pointDict[key] = pt;
        }
        else pointDict[key] = pt;
    }
    var uniquePoints = new List<XYZ>(pointDict.Values);

    // 去重步骤2：容差筛选
    var finalPoints = new List<XYZ>();
    foreach (XYZ pt in uniquePoints)
    {
        bool isDuplicate = false;
        foreach (XYZ kept in finalPoints)
        {
            double dx = pt.X - kept.X, dy = pt.Y - kept.Y;
            double distanceMm = Math.Sqrt(dx * dx + dy * dy) / 0.00328084;
            if (distanceMm <= tolerance_mm) { isDuplicate = true; break; }
        }
        if (!isDuplicate) finalPoints.Add(pt);
    }

    if (finalPoints.Count < 3)
        return string.Format("去重后的点不足3个（{0}个），无法创建地形", finalPoints.Count);

    if (finalPoints.Count > 500)
    {
        TaskDialog.Show("提示", string.Format("本次操作将使用 {0} 个点创建地形，Revit可能会短暂无响应，请耐心等待完成弹窗。", finalPoints.Count));
    }

    Autodesk.Revit.DB.Architecture.TopographySurface topography = null;
    using (SubTransaction sub = new SubTransaction(document))
    {
        sub.Start();
        try
        {
            topography = Autodesk.Revit.DB.Architecture.TopographySurface.Create(document, finalPoints);
            sub.Commit();
        }
        catch { sub.RollBack(); throw; }
    }

    string msg = string.Format("图层信息：\n'0'图层跳过：{0}个三角形\n非'0'图层处理：{1}个三角形\n涉及图层：{2}\n\n提取顶点：{3}个\nXY去重后：{4}个\n容差(5mm)筛选后：{5}个\n\n地形已创建成功！地形ID: {6}",
        layer0Skipped, nonZeroLayer, layerNames.ToString(), points.Count, uniquePoints.Count, finalPoints.Count, topography.Id.IntegerValue);
    TaskDialog.Show("结果", msg);
    return msg;
}
catch (Exception ex)
{
    TaskDialog.Show("错误", ex.Message);
    return ex.Message;
}
```

## 与 `revit-expert.md` 规则的一致性

| 规则项 | 本技能遵守情况 |
|--------|-------------|
| try-catch + return | ✅ 全部代码在 try-catch 内，每个路径返回 string |
| 最多1个 TaskDialog | ✅ 正常路径 1 个，异常/空选中 1 个即返回 |
| StringBuilder 汇总 | ✅ 使用 `System.Text.StringBuilder` 汇总图层名 |
| 批量前预提示 | ✅ 超过500个点提前告知 |
| 返回值长度控制 | ✅ 仅返回简短统计 |
| `document` 而非 `doc` | ✅ 全部使用 `document` |
| 变量名不冲突 | ✅ 使用 `pt`、`kept` 等 camelCase 变量 |
| 字符串插值 | ✅ 全部使用 `string.Format` 替代 `$"..."` |
| UIDocument | ✅ 显式创建 `new UIDocument(document)` |
| TopographySurface | ✅ 使用完全限定名 |
| ImportInstance坐标变换 | ✅ `GetSymbolGeometry()` + `GetTotalTransform()` |

## 交互流程

1. **用户说"从CAD生成地形"或类似触发**
2. **→ 确认CAD已选中，用户确认后发送代码执行**
3. **弹窗显示统计结果**（图层信息、顶点数、去重数、地形ID）
4. **数据量大时可能超时** → 告知用户等待Revit弹窗完成即可