---
name: door-window-legend-template-workflow
description: 基於既有圖例樣板與明細表 (樣板格位對齊與儲存格合併法) 的門窗圖例自動繪製 SOP。明確說明 Revit API 約束 (需預埋空白 Seed 視圖)、明細表專案參數對應、門窗寬高與台度尺寸標註，並提供 C# 完整代碼。
metadata:
  version: "1.6"
  updated: "2026-08-30"
  references:
    - "Revit API 門窗圖例自動化範本規範"
  related:
    - door-window-legend-workflow.md
  referenced_by:
    - door-window-legend-tools
  tags: [door, window, legend, template, slot, cell-merge, project-parameter, dimension]
---

# 樣板格位門窗圖例表工作流程 (Template-Based Legend Workflow)

## 目的與核心理念

本 SOP 說明**基於既有圖面樣板 (Seed Legend View) 與明細表 (Schedule)** 的門窗圖例表自動產生與更新流程。

### 核心哲學：樣板預先放好 ＋ 明細表資料 ＝ 100% 穩定自動繪製

很多時候由程式從零畫線、建字體容易發生缺字型、排版跑掉或標註失敗。
這個方法的邏輯非常簡單白話：
1. **樣板就是一張 A1 或 A3 的空白門窗表**：表格框線、欄位名稱與格子位置都在 Revit 圖例視圖中畫好。
2. **所有字型大小與排版都不會被破壞**：字型的大小、樣式全由你在範本中決定，MCP **只負責替換文字內容**，絕不會把字體改亂。
3. **MCP 一鍵自動複製並填寫**：自動讀取明細表欄位、複製樣板視圖、把文字填入格子、標註門窗寬高尺寸、自動消除同編號隔線（跨欄合併），並依窗台高度把圖例向上移。

---

## Revit API 事實約束與運作機制 (Revit API Constraint)

> [!IMPORTANT]
> **Revit API 硬性限制：API 無法從零建立 Legend 視圖！**
> 
> 在 Revit API 中，官方並未提供憑空建立全新 Legend 視圖的 API（`View.CreateLegend()` 不存在）。
> 因此，自動化流程必須透過 `View.Duplicate()` 從既有視圖複製。

### 🔧 全自動搭建空白樣板 (`mode = scaffold_template`)

雖然 API 無法從零建立 Legend View，但本專案已實作 **`scaffold_template`** 模式，可以從任意既有 Legend 視圖自動搭建出完整的空白門窗圖例樣板！

**使用者只需對 MCP 說：**
> 💬 「MCP，幫我搭建一個 7×3 的窗戶圖例空白樣板」

**MCP 自動完成：**
1. 複製任意既有 Legend 視圖 → 產生新空白視圖
2. 自動畫出 M×N 表格框線（外框 + 標頭/圖例/參數三區分隔線）
3. 自動畫出每格的 FFL 基準線（使用指定線型如 `中級線0.3`）
4. 自動放入佔位標頭文字（W01, W02, ... 或 D01, D02, ...）
5. 自動放入佔位參數區文字（五金門鎖、鉸鏈、把手等）
6. 從專案中找到既有的 Legend Component，自動複製到每個格位作為佔位圖例

> [!NOTE]
> **前提條件**：專案中必須至少存在一個 Legend 視圖（幾乎所有 Revit 樣板都有）。
> 若專案中已有帶 Legend Component 的視圖，佔位圖例會自動放入；否則需要手動在一個格位拖入一個門或窗圖例。

---

### 搭建完成後的後續步驟

1. **微調字體與樣式**：在 Revit 中開啟 `01_門窗圖例表樣板`，調整文字大小、字型、框線粗細等（後續 MCP 只置換文字不改樣式）。
2. **填入實際資料**：對 MCP 說 `用圖例樣板搭配明細表自動生成門窗表`，MCP 會自動複製樣板並填入真實資料。


## 執行前只需告訴 MCP 三個名稱（即可自動執行）

在對話中告訴 MCP 以下三個名稱，MCP 就會直接在 Revit 中自動生成整張門窗表：

1. **排列好的明細表名稱**
   - 例如：`窗明細表` 或 `門明細表`
   - 說明：明細表中已完成編號排序，相同編號但不同台度已分開列項。
2. **空白門窗圖例樣板視圖名稱 (Seed View)**
   - 例如：`01_門窗表-BIMGO AI空白頁` 或 `01_門窗圖例表樣板`
   - 說明：專案中已預先繪製好表格格子、預置佔位門窗圖例與範例文字的圖例視圖。
3. **FFL 線型名稱**
   - 例如：`中級線0.3` 或 `FFL線`
   - 說明：圖例腳下那條用來代表地板（FL）的細部線型名稱。

---

## 樣板 (Template) 白話設定指引

樣板視圖結構說明：

```
+-----------------------------------------------------------------------+
|  Slot 儲存格單元 (容量由範本自訂，如 7x3、6x2、4x2 等皆可)              |
|                                                                       |
|  [編號/型號區]  <-- 預置標頭文字 (範例：W01 / 100 x 150 cm)              |
|  -------------------------------------------------------------------  |
|  [圖例立面與尺寸標註區]                                                |
|            (自動標註門窗寬度 Dimensions)                              |
|                 +-----------------------+                             |
|                 |                       | | (自動標註門窗高度          |
|                 |  佔位用的門窗圖例      | |  Dimensions)               |
|                 |  (MCP自動換成真實門窗) | |                            |
|  ~~~~~~~~~~~~~~~+-----------------------+~~~~~~~~~~~~~~~~~~~~~~~~~~~  |
|  FFL線 (如:中級線0.3) <-- 地板基準線 (MCP 依台度向上位移圖例)           |
|  -------------------------------------------------------------------  |
|  [參數表格區]   <-- 預置內文區塊 (MCP 依位置置換文字)                  |
|   1. 專案參數 1 (例: 五金門鎖)                                         |
|   2. 專案參數 2 (例: 五金鉸鏈)                                         |
|   3. 專案參數 3 (例: 五金把手)                                         |
|   4. 專案參數 4 (例: 五金門弓器)                                       |
|   5. 專案參數 5 (例: 五金其他)                                         |
|   6. 專案參數 6 (例: 玻璃/材料)                                        |
|   7. 專案參數 7 (例: 備註)                                             |
+-----------------------------------------------------------------------+
```

### 1. 什麼是「佔位用的門窗圖例」？ (白話解釋)
- **概念**：在 Revit 圖例視圖中，API 無法憑空建立一個新的圖例元件。因此，我們在樣板的每個格子裡，**先隨便放一個門或窗的圖例作為佔位格子**。
- **作法**：當 MCP 執行時，它會直接將這個佔位圖例的「族群類型」修改為明細表中對應的真實門窗類型，省去從零建立的麻煩。

### 2. 什麼是「FFL 線型名稱」？ (重要！非全表格線型)
- > [!IMPORTANT]
  > **FFL 線（Floor Level）專指「圖例腳下的那條地板基準線」！**
  > 表格的四周框線、欄位分割線與內文線，都可以使用事務所習慣的任意線型（如 `細線`、`中線`、`粗線`）。
  > **千萬不要把整個表格的框線都畫成 FFL 指定線型**，否則程式會分不出哪一條才是地板基準線！

### 3. 動態格子數量 (沒有限制必須 7 欄 × 3 列)
- 程式會自動計算 FL 線的縱橫位置。
- 如果你的窗戶尺寸較大，可以自由繪製 `6 欄 × 2 列` 或 `4 欄 × 2 列` 的表格，MCP 都會自動識別格子總數並在資料多時自動分頁。

### 4. 自動標註門窗寬高與台度尺寸
MCP 放置好真實門窗後，會自動在圖面上繪製 Revit 原生尺寸標註：
1. **寬度標註**：自動在圖例上方標註門窗總寬。
2. **高度標註**：自動在圖例右側標註門窗總高。
3. **台度連續標註**：若窗台高度 $> 0$，自動標註從 **FFL 地板線 $\rightarrow$ 窗底 $\rightarrow$ 窗頂** 的連續尺寸標註。

### 5. 各公司自訂專案參數 (Project Parameters)
- 範例中的「五金門鎖、鉸鏈、把手、玻璃材料」僅為示範。**不同公司可在明細表中放入任何自己習慣的專案參數**。
- MCP 會自動抓取明細表的欄位內容，填入格位相對應的欄位區塊中。

### 6. 同編號多台度自動合併 (Cell Merging)
- 若同一編號出現多筆台度，MCP 會自動在標記加上 `-1`、`-2` 後綴。
- 自動刪除相鄰格子間的垂直隔線，並將專案參數與備註文字區塊**跨欄合併置中**呈現。

---

## 專案已整合之 C# 自動化程式碼 (Backend Implementation)

本專案後端已完整整合此自動化繪製邏輯（`TemplateLegendExecutor`），以下為完整的 C# 核心程式碼：

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using Autodesk.Revit.DB;

namespace RevitMCP.Core
{
    public static class TemplateLegendExecutor
    {
        private sealed class RowData
        {
            public ElementId SymbolId;
            public string Family, Type, Mark, TypeComment, Width, Height, Sill;
            public string Lock, Hinge, Handle, Closer, Other, Glass, Remark;
            public string VariantKey;
            public bool MarkSuffixed;
            public int VariantIndex, VariantCount;
        }

        private sealed class Slot
        {
            public int Row, Column;
            public DetailCurve FlLine;
            public Element Component;
            public double FlX, FlY;
        }

        /// <summary>
        /// 從空白圖例樣板建立並填入門窗明細表數據，包含自動標註門窗寬高與台度尺寸
        /// </summary>
        public static object Execute(Document document, string templateViewName = "01_門窗圖例表樣板", string scheduleName = "窗明細表", string flLineStyleName = "中級線0.3")
        {
            try
            {
                View templateView = new FilteredElementCollector(document)
                    .OfClass(typeof(View))
                    .Cast<View>()
                    .FirstOrDefault(v => !v.IsTemplate && v.ViewType == ViewType.Legend && v.Name == templateViewName);

                ViewSchedule schedule = new FilteredElementCollector(document)
                    .OfClass(typeof(ViewSchedule))
                    .Cast<ViewSchedule>()
                    .FirstOrDefault(v => !v.IsTemplate && v.Name == scheduleName);

                if (templateView == null) return new { success = false, message = "找不到圖例樣板視圖：" + templateViewName + "。" };
                if (schedule == null) return new { success = false, message = "找不到明細表：" + scheduleName };

                List<FamilySymbol> symbols = new FilteredElementCollector(document)
                    .OfClass(typeof(FamilySymbol))
                    .Cast<FamilySymbol>()
                    .ToList();

                List<string> missingFields;
                List<RowData> rows = ReadRows(schedule, symbols, out missingFields);
                if (missingFields.Count > 0) return new { success = false, message = "明細表缺少必要欄位：" + string.Join("、", missingFields) };
                if (rows.Count == 0) return new { success = false, message = "明細表中無可配對的門窗類型資料。" };

                List<Slot> templateSlots = BuildSlots(document, templateView, flLineStyleName);
                if (templateSlots.Count == 0) return new { success = false, message = $"圖例樣板中找不到線型「{flLineStyleName}」的 FFL 基準線。" };

                int capacity = templateSlots.Count;
                int pageCount = Math.Max(1, (int)Math.Ceiling(rows.Count / (double)capacity));
                int generationNumber = FindNextGenerationNumber(document, templateViewName);
                string generatedBaseName = templateViewName + "-" + generationNumber.ToString(CultureInfo.InvariantCulture);

                List<View> pages = new List<View>();
                for (int page = 1; page <= pageCount; page++)
                {
                    string pageName = page == 1 ? generatedBaseName : generatedBaseName + "-" + page.ToString(CultureInfo.InvariantCulture);
                    ElementId id = templateView.Duplicate(ViewDuplicateOption.WithDetailing);
                    View createdView = document.GetElement(id) as View;
                    createdView.Name = pageName;
                    pages.Add(createdView);
                }
                document.Regenerate();

                DimensionType dimType = new FilteredElementCollector(document)
                    .OfClass(typeof(DimensionType))
                    .Cast<DimensionType>()
                    .FirstOrDefault(t => (t.Name ?? "").Contains("對角線")) 
                    ?? new FilteredElementCollector(document).OfClass(typeof(DimensionType)).Cast<DimensionType>().FirstOrDefault();

                double shortTol = document.Application.ShortCurveTolerance;
                int assignedCount = 0, removedDividers = 0, createdDims = 0, createdLines = 0, skippedShort = 0;

                for (int pageIndex = 0; pageIndex < pages.Count; pageIndex++)
                {
                    View view = pages[pageIndex];
                    List<Slot> slots = BuildSlots(document, view, flLineStyleName);
                    AlignFlWithinRows(document, slots);
                    document.Regenerate();
                    RefreshSlotCoordinates(slots);
                    MatchComponentsToSlots(document, view, slots);

                    List<Element> allElements = new FilteredElementCollector(document, view.Id).WhereElementIsNotElementType().ToList();
                    List<DetailCurve> curves = allElements.OfType<DetailCurve>().ToList();
                    List<TextNote> notes = allElements.OfType<TextNote>().ToList();
                    List<Dimension> dimensions = allElements.OfType<Dimension>().ToList();

                    TextNote smallSample = notes.FirstOrDefault(t => Normalize(t.Name).Contains("3.5MM")) ?? notes.FirstOrDefault();
                    TextNote headerSample = notes.FirstOrDefault(t => Normalize(t.Name).Contains("5MM")) ?? notes.FirstOrDefault();

                    ElementId smallType = smallSample?.GetTypeId();
                    ElementId headerType = headerSample?.GetTypeId();

                    double pitch = GetColumnPitch(slots);
                    double componentXOffset = GetComponentXOffset(view, slots);

                    RemoveGroupedCellDividers(document, view, curves, slots, rows, pageIndex, capacity, componentXOffset, ref removedDividers);

                    for (int slotIndex = 0; slotIndex < slots.Count; slotIndex++)
                    {
                        Slot slot = slots[slotIndex];
                        int dataIndex = pageIndex * capacity + slotIndex;
                        double cellCenterX = slot.FlX + componentXOffset;
                        double cellLeft = cellCenterX - pitch * 0.5;
                        double cellRight = cellCenterX + pitch * 0.5;

                        if (dataIndex >= rows.Count)
                        {
                            if (slot.Component != null) { document.Delete(slot.Component.Id); slot.Component = null; }
                            ClearSlotTexts(document, view, notes, slot, cellLeft, cellRight, headerType, smallType);
                            continue;
                        }

                        RowData data = rows[dataIndex];
                        Element component = slot.Component;
                        if (component != null)
                        {
                            Parameter legendParam = component.get_Parameter(BuiltInParameter.LEGEND_COMPONENT);
                            if (legendParam != null && !legendParam.IsReadOnly) legendParam.Set(data.SymbolId);

                            document.Regenerate();
                            BoundingBoxXYZ box = component.get_BoundingBox(view);
                            if (box != null)
                            {
                                double currentCx = (box.Min.X + box.Max.X) * 0.5;
                                if (Math.Abs(cellCenterX - currentCx) > 0.0001)
                                {
                                    ElementTransformUtils.MoveElement(document, component.Id, new XYZ(cellCenterX - currentCx, 0, 0));
                                    document.Regenerate();
                                    box = component.get_BoundingBox(view);
                                }

                                double sillCm;
                                bool sillValid = TryParseCentimeters(data.Sill, out sillCm);
                                if (sillValid)
                                {
                                    double dy = slot.FlY + (sillCm / 30.48) - box.Min.Y;
                                    if (Math.Abs(dy) > 0.0001)
                                    {
                                        ElementTransformUtils.MoveElement(document, component.Id, new XYZ(0, dy, 0));
                                        document.Regenerate();
                                        box = component.get_BoundingBox(view);
                                    }
                                }

                                // 建立門窗寬度與高度/台度尺寸標註 (Dimensions)
                                if (box != null && dimType != null)
                                {
                                    double minX = box.Min.X, maxX = box.Max.X, minY = box.Min.Y, maxY = box.Max.Y;
                                    double widthOffset = 0.9;
                                    double sharedX = Math.Min(maxX + 1.75, cellRight - 0.75);
                                    if (sharedX < maxX + 1.0) sharedX = maxX + 1.0;

                                    // 寬度標註 (Top Width)
                                    DetailCurve wl = GetOrCreateLine(document, view, curves, new XYZ(minX, maxY, 0), new XYZ(minX, maxY + widthOffset, 0), shortTol, ref createdLines, ref skippedShort);
                                    DetailCurve wr = GetOrCreateLine(document, view, curves, new XYZ(maxX, maxY, 0), new XYZ(maxX, maxY + widthOffset, 0), shortTol, ref createdLines, ref skippedShort);
                                    CreateDimensionIfPossible(document, view, dimensions, new XYZ(minX, maxY + widthOffset, 0), new XYZ(maxX, maxY + widthOffset, 0), wl, wr, dimType, shortTol, ref createdDims, ref skippedShort);

                                    // 高度與台度連續標註 (Right Height & Sill Chain)
                                    DetailCurve bottomLine = GetOrCreateLine(document, view, curves, new XYZ(maxX, minY, 0), new XYZ(sharedX, minY, 0), shortTol, ref createdLines, ref skippedShort);
                                    DetailCurve topLine = GetOrCreateLine(document, view, curves, new XYZ(maxX, maxY, 0), new XYZ(sharedX, maxY, 0), shortTol, ref createdLines, ref skippedShort);
                                    bool hasSill = sillValid && Math.Abs(minY - slot.FlY) > shortTol;

                                    if (hasSill)
                                    {
                                        CreateAlignedChainDimension(document, view, dimensions, new XYZ(sharedX, slot.FlY, 0), new XYZ(sharedX, maxY, 0), slot.FlLine, bottomLine, topLine, dimType, shortTol, ref createdDims, ref skippedShort);
                                    }
                                    else
                                    {
                                        CreateDimensionIfPossible(document, view, dimensions, new XYZ(sharedX, minY, 0), new XYZ(sharedX, maxY, 0), bottomLine, topLine, dimType, shortTol, ref createdDims, ref skippedShort);
                                    }
                                }
                            }
                        }

                        WriteSlotTexts(document, view, notes, slot, cellLeft, cellRight, pitch, data, headerType, smallType);
                        assignedCount++;
                    }
                }

                return new { success = true, message = $"成功建立圖例表 {generatedBaseName}。", pages = pages.Select(p => p.Name).ToList(), assignedCount, removedDividers, createdDims };
            }
            catch (Exception ex)
            {
                return new { success = false, message = ex.Message };
            }
        }

        private static void CreateDimensionIfPossible(Document doc, View view, List<Dimension> dims, XYZ a, XYZ b, DetailCurve first, DetailCurve second, DimensionType type, double shortTol, ref int count, ref int skipped)
        {
            if (first == null || second == null || a.DistanceTo(b) <= shortTol) { skipped++; return; }
            Reference r1 = first.GeometryCurve.Reference, r2 = second.GeometryCurve.Reference;
            if (r1 == null || r2 == null) return;
            ReferenceArray refs = new ReferenceArray(); refs.Append(r1); refs.Append(r2);
            Dimension made = doc.Create.NewDimension(view, Line.CreateBound(a, b), refs, type);
            dims.Add(made); count++;
        }

        private static void CreateAlignedChainDimension(Document doc, View view, List<Dimension> dims, XYZ a, XYZ b, DetailCurve fl, DetailCurve bottom, DetailCurve top, DimensionType type, double shortTol, ref int count, ref int skipped)
        {
            if (fl == null || bottom == null || top == null || a.DistanceTo(b) <= shortTol) { skipped++; return; }
            Reference r0 = fl.GeometryCurve.Reference, r1 = bottom.GeometryCurve.Reference, r2 = top.GeometryCurve.Reference;
            if (r0 == null || r1 == null || r2 == null) return;
            ReferenceArray refs = new ReferenceArray(); refs.Append(r0); refs.Append(r1); refs.Append(r2);
            Dimension made = doc.Create.NewDimension(view, Line.CreateBound(a, b), refs, type);
            dims.Add(made); count++;
        }

        private static DetailCurve GetOrCreateLine(Document doc, View view, List<DetailCurve> existing, XYZ a, XYZ b, double shortTol, ref int count, ref int skipped)
        {
            if (a.DistanceTo(b) <= shortTol) { skipped++; return null; }
            DetailCurve made = doc.Create.NewDetailCurve(view, Line.CreateBound(a, b));
            existing.Add(made); count++; return made;
        }

        private static List<RowData> ReadRows(ViewSchedule schedule, List<FamilySymbol> symbols, out List<string> missingFields)
        {
            missingFields = new List<string>();
            TableSectionData body = schedule.GetTableData().GetSectionData(SectionType.Body);
            Dictionary<string, int> cols = new Dictionary<string, int>();
            for (int c = 0; c < body.NumberOfColumns; c++)
            {
                string h = Cell(schedule, 0, c);
                if (!cols.ContainsKey(Normalize(h))) cols[Normalize(h)] = c;
            }

            int familyCol = FindColumn(cols, "族群", "Family");
            int typeCol = FindColumn(cols, "類型", "Type");
            if (familyCol < 0) missingFields.Add("族群");
            if (typeCol < 0) missingFields.Add("類型");
            if (missingFields.Count > 0) return new List<RowData>();

            int markCol = FindColumn(cols, "類型標記", "Type Mark");
            int commentCol = FindColumn(cols, "類型備註", "Type Comments");
            int widthCol = FindColumn(cols, "寬度", "Width");
            int heightCol = FindColumn(cols, "高度", "Height");
            int sillCol = FindColumn(cols, "窗台高度", "Sill Height");
            int lockCol = FindColumn(cols, "門窗表-五金門鎖", "五金門鎖");
            int hingeCol = FindColumn(cols, "門窗表-五金鉸鏈", "門窗表-五金鉸鍊", "五金鉸鏈", "五金鉸鍊");
            int handleCol = FindColumn(cols, "門窗表-五金把手", "五金把手");
            int closerCol = FindColumn(cols, "門窗表-五金門弓器", "五金門弓器");
            int otherCol = FindColumn(cols, "門窗表-五金其它", "門窗表-五金其他", "五金其它", "五金其他");
            int glassCol = FindColumn(cols, "門窗表-玻璃/材料", "門窗表-玻璃／材料", "玻璃/材料", "玻璃／材料");
            int remarkCol = FindColumn(cols, "門窗表-備註", "備註");

            List<RowData> result = new List<RowData>();
            for (int r = 1; r < body.NumberOfRows; r++)
            {
                string family = Cell(schedule, r, familyCol);
                string type = Cell(schedule, r, typeCol);
                if (family == "" || type == "") continue;

                FamilySymbol symbol = symbols.FirstOrDefault(s => Normalize(s.FamilyName) == Normalize(family) && Normalize(s.Name) == Normalize(type));
                if (symbol == null) continue;

                result.Add(new RowData
                {
                    SymbolId = symbol.Id, Family = family, Type = type,
                    Mark = Cell(schedule, r, markCol), TypeComment = Cell(schedule, r, commentCol),
                    Width = Cell(schedule, r, widthCol), Height = Cell(schedule, r, heightCol), Sill = Cell(schedule, r, sillCol),
                    Lock = Cell(schedule, r, lockCol), Hinge = Cell(schedule, r, hingeCol), Handle = Cell(schedule, r, handleCol),
                    Closer = Cell(schedule, r, closerCol), Other = Cell(schedule, r, otherCol), Glass = Cell(schedule, r, glassCol), Remark = Cell(schedule, r, remarkCol)
                });
            }
            ApplyDuplicateMarkSuffixes(result);
            return result;
        }

        private static void ApplyDuplicateMarkSuffixes(List<RowData> rows)
        {
            foreach (RowData row in rows) row.VariantKey = Normalize(row.Mark != "" ? "MARK|" + row.Mark : "TYPE|" + row.Family + "|" + row.Type);
            foreach (var group in rows.GroupBy(r => r.VariantKey))
            {
                List<RowData> variants = group.ToList();
                if (variants.Count <= 1) continue;
                for (int i = 0; i < variants.Count; i++)
                {
                    RowData row = variants[i];
                    row.VariantIndex = i + 1;
                    row.VariantCount = variants.Count;
                    string mark = (row.Mark ?? "").Trim();
                    row.Mark = (mark == "" ? row.Type : mark) + "-" + (i + 1).ToString(CultureInfo.InvariantCulture);
                    row.MarkSuffixed = true;
                }
            }
        }

        private static List<Slot> BuildSlots(Document document, View view, string flLineStyleName)
        {
            List<DetailCurve> flLines = new FilteredElementCollector(document, view.Id)
                .WhereElementIsNotElementType()
                .OfType<DetailCurve>()
                .Where(d => IsFlLine(d, flLineStyleName))
                .ToList();

            var groups = new List<List<DetailCurve>>();
            foreach (DetailCurve line in flLines.OrderByDescending(d => MidY(d.GeometryCurve)))
            {
                double y = MidY(line.GeometryCurve);
                var g = groups.FirstOrDefault(x => Math.Abs(MidY(x[0].GeometryCurve) - y) < 0.5);
                if (g == null) { g = new List<DetailCurve>(); groups.Add(g); }
                g.Add(line);
            }
            groups = groups.OrderByDescending(g => g.Average(d => MidY(d.GeometryCurve))).ToList();

            List<Slot> slots = new List<Slot>();
            for (int r = 0; r < groups.Count; r++)
            {
                var row = groups[r].OrderBy(d => MidX(d.GeometryCurve)).ToList();
                for (int c = 0; c < row.Count; c++)
                    slots.Add(new Slot { Row = r, Column = c, FlLine = row[c], FlX = MidX(row[c].GeometryCurve), FlY = MidY(row[c].GeometryCurve) });
            }
            return slots;
        }

        private static bool IsFlLine(DetailCurve d, string lineStyleName)
        {
            Curve c = d.GeometryCurve;
            if (c == null || !c.IsBound) return false;
            XYZ a = c.GetEndPoint(0), b = c.GetEndPoint(1);
            string n = Normalize(d.LineStyle == null ? "" : d.LineStyle.Name);
            return n.Contains(Normalize(lineStyleName)) && Math.Abs(a.Y - b.Y) < 0.01;
        }

        private static void RemoveGroupedCellDividers(Document document, View view, List<DetailCurve> curves, List<Slot> slots, List<RowData> rows, int pageIndex, int capacity, double componentXOffset, ref int removed)
        {
            for (int i = 0; i < slots.Count - 1; i++)
            {
                Slot a = slots[i], b = slots[i + 1];
                if (a.Row != b.Row || b.Column != a.Column + 1) continue;
                int ia = pageIndex * capacity + i, ib = ia + 1;
                if (ia >= rows.Count || ib >= rows.Count) continue;

                RowData ra = rows[ia], rb = rows[ib];
                if (ra.VariantCount <= 1 || ra.VariantKey != rb.VariantKey) continue;

                double boundary = ((a.FlX + componentXOffset) + (b.FlX + componentXOffset)) * 0.5;
                double lower = a.FlY - 9.5, upper = a.FlY + 21.5;

                List<DetailCurve> dividers = curves.Where(c => IsVertical(c.GeometryCurve) && Math.Abs(MidX(c.GeometryCurve) - boundary) < 0.25 && OverlapsY(c.GeometryCurve, lower, upper)).ToList();
                foreach (DetailCurve divider in dividers)
                {
                    document.Delete(divider.Id);
                    curves.Remove(divider);
                    removed++;
                }
            }
        }

        private static void WriteSlotTexts(Document document, View view, List<TextNote> notes, Slot slot, double left, double right, double pitch, RowData data, ElementId headerType, ElementId smallType)
        {
            bool grouped = data.VariantCount > 1;
            bool last = !grouped || data.VariantIndex == data.VariantCount;
            int dummy = 0;

            SetTextInBox(document, view, notes, JoinNonEmpty(data.Mark, data.VariantIndex == 1 ? data.Type : ""), headerType, (left + right) * 0.5, slot.FlY + 20.39, left, right, slot.FlY + 17.83, slot.FlY + 21.10, true, ref dummy);

            double mergedLeft = grouped ? right - pitch * data.VariantCount : left;
            double mergedRight = right;
            double commonLeft = last ? mergedLeft : left;
            double commonRight = last ? mergedRight : right;
            double commonCenter = (commonLeft + commonRight) * 0.5;

            SetTextInBox(document, view, notes, last ? data.TypeComment : "", headerType, commonCenter, slot.FlY + 17.54, commonLeft, commonRight, slot.FlY + 14.5, slot.FlY + 17.82, true, ref dummy);

            string[] values = last ? new[] { data.Lock, data.Hinge, data.Handle, data.Closer, data.Other, data.Glass, data.Remark } : new[] { "", "", "", "", "", "", "" };
            const double firstTop = -1.6404199475065786, rowHeight = 1.148293963254593;
            for (int i = 0; i < values.Length; i++)
            {
                double top = slot.FlY + firstTop - i * rowHeight, bottom = top - rowHeight;
                SetTextInBox(document, view, notes, values[i], smallType, commonCenter, (top + bottom) * 0.5, commonLeft, commonRight, bottom, top, true, ref dummy);
            }
        }

        private static void ClearSlotTexts(Document document, View view, List<TextNote> notes, Slot slot, double left, double right, ElementId headerType, ElementId smallType)
        {
            WriteSlotTexts(document, view, notes, slot, left, right, right - left, new RowData { VariantCount = 1, VariantIndex = 1 }, headerType, smallType);
        }

        private static void SetTextInBox(Document document, View view, List<TextNote> notes, string text, ElementId preferredType, double x, double y, double left, double right, double bottom, double top, bool center, ref int counter)
        {
            string value = (text ?? "").Trim();
            TextNote note = FindNoteInBox(view, notes, left, right, bottom, top);
            if (value == "")
            {
                if (note != null) { document.Delete(note.Id); notes.Remove(note); }
                return;
            }
            if (note == null)
            {
                if (preferredType != null && preferredType != ElementId.InvalidElementId)
                {
                    note = TextNote.Create(document, view.Id, new XYZ(x, y, 0), value, preferredType);
                    notes.Add(note);
                }
            }
            else
            {
                if ((note.Text ?? "").Trim() != value) note.Text = value;
                if (preferredType != null && preferredType != ElementId.InvalidElementId && note.GetTypeId() != preferredType) note.ChangeTypeId(preferredType);
            }
            document.Regenerate();
            if (center && note != null) CenterNote(document, view, note, x);
        }

        private static TextNote FindNoteInBox(View view, List<TextNote> notes, double left, double right, double bottom, double top)
        {
            return notes.FirstOrDefault(note => {
                BoundingBoxXYZ b = note.get_BoundingBox(view);
                if (b == null) return false;
                double cx = (b.Min.X + b.Max.X) * 0.5, cy = (b.Min.Y + b.Max.Y) * 0.5;
                return cx > left && cx < right && cy > bottom && cy < top;
            });
        }

        private static void CenterNote(Document document, View view, TextNote note, double x)
        {
            BoundingBoxXYZ b = note.get_BoundingBox(view);
            if (b == null) return;
            double dx = x - (b.Min.X + b.Max.X) * 0.5;
            if (Math.Abs(dx) > 0.0001) { ElementTransformUtils.MoveElement(document, note.Id, new XYZ(dx, 0, 0)); document.Regenerate(); }
        }

        private static void AlignFlWithinRows(Document document, List<Slot> slots)
        {
            foreach (var row in slots.GroupBy(s => s.Row))
            {
                double y = row.Average(s => s.FlY);
                foreach (Slot s in row)
                {
                    double dy = y - s.FlY;
                    if (Math.Abs(dy) > 0.0001) ElementTransformUtils.MoveElement(document, s.FlLine.Id, new XYZ(0, dy, 0));
                }
            }
        }

        private static void RefreshSlotCoordinates(List<Slot> slots)
        {
            foreach (Slot s in slots) { s.FlX = MidX(s.FlLine.GeometryCurve); s.FlY = MidY(s.FlLine.GeometryCurve); }
        }

        private static void MatchComponentsToSlots(Document document, View view, List<Slot> slots)
        {
            foreach (Slot s in slots) s.Component = null;
            List<Element> comps = new FilteredElementCollector(document, view.Id)
                .WhereElementIsNotElementType()
                .Where(e => e.get_Parameter(BuiltInParameter.LEGEND_COMPONENT) != null)
                .ToList();

            HashSet<Slot> used = new HashSet<Slot>();
            foreach (Element e in comps)
            {
                BoundingBoxXYZ b = e.get_BoundingBox(view);
                if (b == null) continue;
                double cx = (b.Min.X + b.Max.X) * 0.5, cy = (b.Min.Y + b.Max.Y) * 0.5;
                Slot best = slots.Where(s => !used.Contains(s)).OrderBy(s => Math.Abs(cx - s.FlX) * 2 + Math.Abs(cy - (s.FlY + 7.5))).FirstOrDefault();
                if (best != null) { best.Component = e; used.Add(best); }
            }
        }

        private static double GetColumnPitch(List<Slot> slots)
        {
            List<double> pitchList = new List<double>();
            foreach (var row in slots.GroupBy(s => s.Row))
            {
                List<double> xs = row.OrderBy(s => s.Column).Select(s => s.FlX).ToList();
                for (int i = 1; i < xs.Count; i++) if (xs[i] - xs[i - 1] > 0.1) pitchList.Add(xs[i] - xs[i - 1]);
            }
            return pitchList.Count == 0 ? 16 : pitchList.OrderBy(x => x).ElementAt(pitchList.Count / 2);
        }

        private static double GetComponentXOffset(View view, List<Slot> slots)
        {
            List<double> offsets = new List<double>();
            foreach (Slot s in slots.Where(x => x.Component != null))
            {
                BoundingBoxXYZ b = s.Component.get_BoundingBox(view);
                if (b != null) offsets.Add((b.Min.X + b.Max.X) * 0.5 - s.FlX);
            }
            return offsets.Count == 0 ? -0.34 : offsets.OrderBy(x => x).ElementAt(offsets.Count / 2);
        }

        private static int FindNextGenerationNumber(Document document, string templateName)
        {
            List<string> names = new FilteredElementCollector(document).OfClass(typeof(View)).Cast<View>().Where(v => !v.IsTemplate && v.ViewType == ViewType.Legend).Select(v => v.Name ?? "").ToList();
            string prefix = templateName + "-";
            int max = 0;
            foreach (string name in names)
            {
                if (!name.StartsWith(prefix, StringComparison.OrdinalIgnoreCase)) continue;
                string suffix = name.Substring(prefix.Length);
                int number;
                if (int.TryParse(suffix, NumberStyles.Integer, CultureInfo.InvariantCulture, out number) && number > max) max = number;
            }
            return max + 1;
        }

        private static int FindColumn(Dictionary<string, int> columns, params string[] names)
        {
            foreach (string n in names) { if (columns.TryGetValue(Normalize(n), out int v)) return v; }
            return -1;
        }

        private static string Cell(ViewSchedule s, int r, int c)
        {
            if (c < 0) return "";
            try { return (s.GetCellText(SectionType.Body, r, c) ?? "").Trim(); } catch { return ""; }
        }

        private static bool IsVertical(Curve c) { return c != null && c.IsBound && Math.Abs(c.GetEndPoint(0).X - c.GetEndPoint(1).X) < 0.01 && Math.Abs(c.GetEndPoint(0).Y - c.GetEndPoint(1).Y) > 0.01; }
        private static bool OverlapsY(Curve c, double lower, double upper) { double a = c.GetEndPoint(0).Y, b = c.GetEndPoint(1).Y; return Math.Max(a, b) > lower + 0.01 && Math.Min(a, b) < upper - 0.01; }
        private static double MidX(Curve c) => (c.GetEndPoint(0).X + c.GetEndPoint(1).X) * 0.5;
        private static double MidY(Curve c) => (c.GetEndPoint(0).Y + c.GetEndPoint(1).Y) * 0.5;
        private static string Normalize(string v) => new string((v ?? "").Where(c => !char.IsWhiteSpace(c)).ToArray()).Replace("／", "/").ToUpperInvariant();
        private static string JoinNonEmpty(string a, string b) => string.IsNullOrEmpty(a?.Trim()) ? b?.Trim() ?? "" : (string.IsNullOrEmpty(b?.Trim()) ? a.Trim() : a.Trim() + "　" + b.Trim());

        private static bool TryParseCentimeters(string text, out double value)
        {
            value = 0;
            string raw = (text ?? "").Trim().Replace(",", "");
            string numeric = new string(raw.Where(c => char.IsDigit(c) || c == '.' || c == '-').ToArray());
            if (!double.TryParse(numeric, NumberStyles.Float, CultureInfo.InvariantCulture, out double n)) return false;
            string lower = raw.ToLowerInvariant();
            if (lower.Contains("mm")) n /= 10; else if (lower.Contains("m") && !lower.Contains("cm")) n *= 100;
            value = n;
            return true;
        }
    }
}
```
