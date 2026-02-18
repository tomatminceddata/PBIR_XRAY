# PBIR X-Ray — Power BI Report Content Analyzer

## What this is about

Power BI reports are black boxes. You can open them in Desktop and click through pages, but you can't answer questions like: which measures does this report actually use? Are there hidden visuals? Do the bookmarks still match the visuals they were created for? Which slicers sync across pages and which ones are invisible? What conditional formatting rules are buried inside that matrix?

PBIR X-Ray cracks open the box. It reads the PBIR folder-based format — the JSON files that Power BI Desktop writes when you save in the modern `.pbip` format — and extracts everything into queryable tables. No APIs, no admin permissions, no external tools. Just Power Query reading files from disk.

The extraction pipeline is a set of 14 Power Query M queries that parse every `.json` file in a `.Report` folder and produce structured tables covering visuals, model references, conditional formatting, slicers, bookmarks, buttons, and report settings. A companion Power BI report visualizes the results across six analysis pages. Point it at any PBIR report folder and you get a complete content inventory — the kind of documentation that nobody writes manually but everybody needs when something breaks in production.

## The PBIR folder-based format

The PBIR folder-based format splits the monolithic `report.json` into a folder tree:

- `.platform` → report metadata (displayName)
- `definition/report.json` → global settings (theme, resources)
- `definition/pages/pages.json` → page ordering
- `definition/pages/{pageId}/page.json` → page metadata
- `definition/pages/{pageId}/visuals/{visualId}/visual.json` → each visual
- `definition/bookmarks/bookmarks.json` → bookmark ordering
- `definition/bookmarks/{bookmarkId}.bookmark.json` → each bookmark state snapshot

Visual data is directly accessible JSON — no double-encoded strings, no nested `Json.Document` calls.

Every output query includes a **ReportName** column.

---

# Power Query for the data preparation

## Firewall-proof architecture

Power Query's privacy firewall blocks any query that references other queries **and** directly accesses data sources. The "Ignore Privacy Levels" workaround is not viable for Power BI Service automated refresh.

The architecture uses a single entry point (`fn_pbir_ReportData`) that performs exactly one `Folder.Files` call, pre-parses all JSON files, and returns a self-contained record. Downstream queries are pure transformations with no data source access of their own.

## Why conditional formatting matters for dependency tracking

Conditional formatting rules in Power BI visuals (background color, font color) reference measures or columns that **do not appear in the visual's field wells**. These are hidden dependencies — if the referenced measure breaks, the formatting silently fails with no obvious trail back to the root cause. By surfacing them as `Source = "ConditionalFormatting"` in `ModelReferences_pbir`, the unused objects finder and any dependency visualization automatically capture these dependencies.

---

## Architecture: Query Dependency Graph

```
pq_pbir_ReportFolderPath (parameter)
    │
    └─► fn_pbir_ReportData (single Folder.Files call → buffered record)
            │
            ├─► ReportSettings_pbir
            ├─► Pages_pbir
            ├─► Visuals_pbir ──────────────────────► SlicerInventory_pbir
            ├─► ModelReferences_pbir ──────────────► UnusedObjectsFinder_pbir
            │       │                                       │
            │       └───────────────────────────────► SlicerInventory_pbir
            ├─► ConditionalFormattingRules_pbir
            ├─► Bookmarks_pbir
            ├─► BookmarkActions_pbir
            ├─► Buttons_pbir
            └─► SchemaCoverageAudit_pbir (disabled, on-demand only)

fn_pbir_ParseQueryRef (pure function, no data source)
    │
    ├─► ModelReferences_pbir
    └─► ConditionalFormattingRules_pbir
```

All data source access is isolated in `fn_pbir_ReportData`. Downstream queries are pure transformations — no `File.Contents`, no `Folder.Files`, no firewall conflicts.

---

## Setup Instructions

### 1. Create the parameter

Create a Power Query parameter named `pq_pbir_ReportFolderPath` pointing to the `.Report` folder (the folder containing `.platform` and `definition/`).

### 2. Create the queries

Copy each query below into a new blank query in Power Query Editor. The recommended creation order follows the dependency graph:

1. `pq_pbir_ReportFolderPath` (parameter)
2. `fn_pbir_ParseQueryRef` (helper function)
3. `fn_pbir_ReportData` (data source function)
4. `ReportSettings_pbir` (report-level metadata)
5. `Pages_pbir` (page inventory)
6. `Visuals_pbir` (visual census)
7. `ModelReferences_pbir` (model object references)
8. `ConditionalFormattingRules_pbir` (CF rule details)
9. `UnusedObjectsFinder_pbir` (distinct model references)
10. `SlicerInventory_pbir` (slicer analysis)
11. `Bookmarks_pbir` (bookmark inventory)
12. `BookmarkActions_pbir` (per-visual bookmark state changes)
13. `Buttons_pbir` (button/action inventory)
14. `SchemaCoverageAudit_pbir` (schema gap discovery) — create as **disabled** query; do NOT load to semantic model; right-click → Invoke to run on demand

### 3. Configure the ColorBar column

In `ConditionalFormattingRules_pbir`, the `ColorBar` column contains SVG images encoded as data URIs. To render them inline in a Power BI table:

1. Set the column's **Data Category** to `Image URL` in the semantic model
2. Set the table image column width to **150px** and height to **30px**

---

## Parameter: pq_pbir_ReportFolderPath

Path to the `.Report` folder (e.g. the folder containing `.platform` and `definition/`).

```m
"/path/to/YourReport.Report" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
```

---

## Query: fn_pbir_ParseQueryRef

Helper function to parse `queryRef` strings like `"Table.column"` or `"Sum(Table.column)"`.

```m
let
    fn = (queryRef as text) as record =>
        let
            AggFunctions = {"Sum", "Avg", "Min", "Max", "Count", "CountNonNull"},
            
            IsAggregation = List.AnyTrue(
                List.Transform(AggFunctions, each Text.StartsWith(queryRef, _ & "(") and Text.EndsWith(queryRef, ")"))
            ),
            
            AggFunctionName = if IsAggregation 
                then List.First(List.Select(AggFunctions, each Text.StartsWith(queryRef, _ & "(")), null)
                else null,
            
            InnerRef = if IsAggregation and AggFunctionName <> null
                then Text.Range(queryRef, Text.Length(AggFunctionName) + 1, Text.Length(queryRef) - Text.Length(AggFunctionName) - 2)
                else queryRef,
            
            DotPos = Text.PositionOf(InnerRef, "."),
            
            TableName = if DotPos >= 0 then Text.Start(InnerRef, DotPos) else "",
            ObjectName = if DotPos >= 0 then Text.Range(InnerRef, DotPos + 1) else InnerRef,
            
            RefType = if IsAggregation then "Aggregation (" & AggFunctionName & ")" else "Column/Measure"
        in
            [
                RefType = RefType,
                TableName = TableName,
                ObjectName = ObjectName,
                AggFunction = AggFunctionName
            ]
in
    fn
```

---

## Query: fn_pbir_ReportData

**The single data source entry point.** Performs one `Folder.Files` call on the `.Report` folder, parses all JSON files, and returns a buffered record. Every output query consumes this function — no query touches a data source directly.

Path separators are normalized to forward slashes internally, so the parameter value works with both Windows backslash paths (`C:\...\Report.Report`) and Mac/Linux forward slash paths (`/Users/.../Report.Report`).

The returned record contains:
- `ReportName` — from `.platform → metadata.displayName`
- `Platform` — the full `.platform` JSON record
- `ReportJson` — the full `definition/report.json` record
- `PageIds` — ordered list of page IDs from `pages.json`
- `PageData` — list of records, one per page, each containing `[PageId, PageJson, VisualJsonList]`
- `BookmarkIds` — ordered list of bookmark IDs from `bookmarks.json`
- `BookmarkData` — list of records, one per bookmark, each containing `[BookmarkId, BookmarkJson]`

```m
let
    fn = () as record =>
        let
            // === SINGLE DATA SOURCE ACCESS ===
            // Folder.Files is recursive — gets every file in the .Report tree
            AllFiles = Folder.Files(pq_pbir_ReportFolderPath),
            
            // --- Normalize folder path separators for cross-platform matching ---
            // On Windows [Folder Path] uses backslashes; on Mac/Linux forward slashes
            WithNormPath = Table.AddColumn(AllFiles, "NormPath", each
                Text.Replace([Folder Path], "\", "/"), type text
            ),
            BasePath = Text.Replace(pq_pbir_ReportFolderPath, "\", "/"),
            
            // --- Helper: find a file by name and normalized folder path suffix ---
            fnGetFile = (tbl as table, fileName as text, folderSuffix as text) as binary =>
                let
                    Matched = Table.SelectRows(tbl, each 
                        [Name] = fileName 
                        and Text.EndsWith([NormPath], folderSuffix)
                    ),
                    Content = try Matched{0}[Content] otherwise null
                in
                    Content,
            
            // --- Parse core metadata files ---
            PlatformBin = fnGetFile(WithNormPath, ".platform", BasePath & "/"),
            Platform = try Json.Document(PlatformBin) otherwise Record.FromList({}, {}),
            ReportName = try Platform[metadata][displayName] otherwise "Unknown",
            
            ReportJsonBin = fnGetFile(WithNormPath, "report.json", "/definition/"),
            ReportJson = try Json.Document(ReportJsonBin) otherwise Record.FromList({}, {}),
            
            PagesJsonBin = fnGetFile(WithNormPath, "pages.json", "/definition/pages/"),
            PagesJson = try Json.Document(PagesJsonBin) otherwise Record.FromList({}, {}),
            PageIds = try PagesJson[pageOrder] otherwise {},
            
            // --- Parse all page.json files ---
            PageJsonFiles = Table.SelectRows(WithNormPath, each [Name] = "page.json"),
            
            // --- Parse all visual.json files ---
            VisualJsonFiles = Table.SelectRows(WithNormPath, each [Name] = "visual.json"),
            
            // --- Build per-page data with visuals grouped by page ---
            PagesBasePath = BasePath & "/definition/pages/",
            
            PageData = List.Transform(PageIds, each
                let
                    pageId = _,
                    PageFolderPath = PagesBasePath & pageId & "/",
                    
                    PageJsonRow = try Table.SelectRows(PageJsonFiles, each
                        [NormPath] = PageFolderPath
                    ){0} otherwise null,
                    PageJson = try Json.Document(PageJsonRow[Content]) otherwise Record.FromList({}, {}),
                    
                    VisualsFolderPrefix = PageFolderPath & "visuals/",
                    PageVisualFiles = Table.SelectRows(VisualJsonFiles, each
                        Text.StartsWith([NormPath], VisualsFolderPrefix)
                    ),
                    
                    VisualJsonList = try List.Transform(
                        Table.ToRecords(PageVisualFiles),
                        each try Json.Document([Content]) otherwise Record.FromList({}, {})
                    ) otherwise {}
                in
                    [
                        PageId = pageId,
                        PageJson = PageJson,
                        VisualJsonList = VisualJsonList
                    ]
            ),
            
            // --- Parse bookmark files ---
            BookmarksBasePath = BasePath & "/definition/bookmarks/",
            
            BookmarksMetaBin = fnGetFile(WithNormPath, "bookmarks.json", "/definition/bookmarks/"),
            BookmarksMeta = try Json.Document(BookmarksMetaBin) otherwise Record.FromList({}, {}),
            BookmarkIds = try List.Transform(BookmarksMeta[items], each [name]) otherwise {},
            
            BookmarkJsonFiles = Table.SelectRows(WithNormPath, each 
                Text.EndsWith([Name], ".bookmark.json")
                and Text.StartsWith([NormPath], BookmarksBasePath)
            ),
            
            BookmarkData = List.Transform(
                Table.ToRecords(BookmarkJsonFiles),
                each
                    let
                        bJson = try Json.Document([Content]) otherwise Record.FromList({}, {}),
                        bId = try bJson[name] otherwise Text.BeforeDelimiter([Name], ".bookmark.json")
                    in
                        [
                            BookmarkId = bId,
                            BookmarkJson = bJson
                        ]
            ),
            
            Result = [
                ReportName = ReportName,
                Platform = Platform,
                ReportJson = ReportJson,
                PageIds = PageIds,
                PageData = PageData,
                BookmarkIds = BookmarkIds,
                BookmarkData = BookmarkData
            ]
        in
            Result
in
    fn
```

---

## Query: ReportSettings_pbir

Extracts report-level settings and metadata.

```m
let
    Data = fn_pbir_ReportData(),
    ReportName = Data[ReportName],
    
    Report = Data[ReportJson],
    Settings = try Report[settings] otherwise Record.FromList({}, {}),
    Theme = try Report[themeCollection][baseTheme][name] otherwise null,
    SchemaUrl = try Report[#"$schema"] otherwise null,
    
    PageCount = List.Count(Data[PageIds]),
    
    TotalVisuals = List.Sum(
        List.Transform(Data[PageData], each List.Count(_[VisualJsonList]))
    ),
    
    Result = Table.FromRecords({[
        ReportName = ReportName,
        Schema = SchemaUrl,
        ThemeName = Theme,
        UseStylableVisualContainerHeader = try Settings[useStylableVisualContainerHeader] otherwise null,
        ExportDataMode = try Settings[exportDataMode] otherwise null,
        DefaultDrillFilterOtherVisuals = try Settings[defaultDrillFilterOtherVisuals] otherwise null,
        AllowChangeFilterTypes = try Settings[allowChangeFilterTypes] otherwise null,
        UseEnhancedTooltips = try Settings[useEnhancedTooltips] otherwise null,
        UseDefaultAggregateDisplayName = try Settings[useDefaultAggregateDisplayName] otherwise null,
        PageCount = PageCount,
        TotalVisuals = TotalVisuals
    ]})
in
    Result
```

---

## Query: Pages_pbir

Extracts all report pages with metadata.

```m
let
    Data = fn_pbir_ReportData(),
    ReportName = Data[ReportName],
    PageIds = Data[PageIds],
    PageData = Data[PageData],
    
    Pages = List.Transform(
        List.Zip({PageData, List.Positions(PageIds)}),
        each
            let
                pair = _,
                pd = pair{0},
                ordinal = pair{1},
                
                PageJson = pd[PageJson],
                DisplayName = try PageJson[displayName] otherwise pd[PageId],
                Width = try PageJson[width] otherwise 0,
                Height = try PageJson[height] otherwise 0,
                DisplayOption = try PageJson[displayOption] otherwise null,
                PageVisibility = try PageJson[visibility] otherwise null,
                PageType = try PageJson[type] otherwise null,
                
                VisualCount = List.Count(pd[VisualJsonList]),
                
                VisualTypes = List.Transform(
                    pd[VisualJsonList],
                    each try _[visual][visualType] otherwise "unknown"
                ),
                
                SlicerCount = List.Count(List.Select(VisualTypes, each _ = "slicer" or _ = "advancedSlicerVisual" or _ = "listSlicer"))
            in
                [
                    ReportName = ReportName,
                    PageName = DisplayName,
                    PageId = pd[PageId],
                    PageOrdinal = ordinal,
                    Width = Width,
                    Height = Height,
                    DisplayOption = DisplayOption,
                    PageVisibility = PageVisibility,
                    PageType = PageType,
                    VisualCount = VisualCount,
                    SlicerCount = SlicerCount,
                    VisualTypeList = Text.Combine(List.Sort(List.Distinct(VisualTypes)), ", ")
                ]
    ),
    
    Result = Table.FromRecords(Pages),
    Typed = Table.TransformColumnTypes(Result, {
        {"PageOrdinal", Int64.Type},
        {"Width", type number},
        {"Height", type number},
        {"VisualCount", Int64.Type},
        {"SlicerCount", Int64.Type}
    }),
    Sorted = Table.Sort(Typed, {{"PageOrdinal", Order.Ascending}})
in
    Sorted
```

---

## Query: Visuals_pbir

The full visual census — every visual on every page with type, position, configuration metadata, and conditional formatting detection.

### Conditional formatting detection

Conditional formatting in Power BI visuals is encoded inside `objects.values[]` as entries whose properties (`backColor`, `fontColor`) contain `Conditional` expressions with `Cases` arrays. The detection scans all entries and checks the direct JSON path `property.solid.color.expr.Conditional`.

```m
let
    Data = fn_pbir_ReportData(),
    ReportName = Data[ReportName],
    PageIds = Data[PageIds],
    PageData = Data[PageData],
    
    fnCountConditionalFormatting = (visualRecord as record) as number =>
        let
            ValuesEntries = try visualRecord[objects][values] otherwise {},
            ValuesCFCount = List.Count(
                List.Select(ValuesEntries, each
                    let
                        props = try _[properties] otherwise Record.FromList({}, {}),
                        propNames = try Record.FieldNames(props) otherwise {},
                        cfProps = List.Select(propNames, each _ = "backColor" or _ = "fontColor")
                    in
                        List.AnyTrue(
                            List.Transform(cfProps, (pn) =>
                                let
                                    exprNode = try Record.Field(props, pn)[solid][color][expr] otherwise Record.FromList({}, {}),
                                    exprFields = try Record.FieldNames(exprNode) otherwise {}
                                in
                                    List.ContainsAny(exprFields, {"Conditional", "FillRule", "Measure", "Column"})
                            )
                        )
                )
            ),
            
            ColumnFmtEntries = try visualRecord[objects][columnFormatting] otherwise {},
            DataBarCount = List.Count(
                List.Select(ColumnFmtEntries, each
                    try (_[properties][dataBars] <> null) otherwise false
                )
            )
        in
            ValuesCFCount + DataBarCount,
    
    fnGetVisual = (pageDisplayName as text, pageId as text, pageOrdinal as number, vJson as record) as record =>
        let
            VisualName = try vJson[name] otherwise "",
            
            Pos = try vJson[position] otherwise Record.FromList({}, {}),
            PosX = try Pos[x] otherwise 0,
            PosY = try Pos[y] otherwise 0,
            PosW = try Pos[width] otherwise 0,
            PosH = try Pos[height] otherwise 0,
            PosZ = try Pos[z] otherwise 0,
            
            V = try vJson[visual] otherwise Record.FromList({}, {}),
            VisualType = try V[visualType] otherwise 
                (if try (vJson[visualGroup] <> null) otherwise false then "visualGroup" else "unknown"),
            
            ObjectKeysList = try Record.FieldNames(V[objects]) otherwise {},
            VcObjectKeysList = try Record.FieldNames(V[visualContainerObjects]) otherwise {},
            
            SlicerMode = try V[objects][data]{0}[properties][mode][expr][Literal][Value] otherwise null,
            SlicerOrientation = try V[objects][general]{0}[properties][orientation][expr][Literal][Value] otherwise
                (try V[objects][layout]{0}[properties][orientation][expr][Literal][Value] otherwise null),
            HasSelfFilter = try (V[objects][general]{0}[properties][selfFilterEnabled][expr][Literal][Value] = "true") otherwise false,
            HasDefaultFilter = try (V[objects][general]{0}[properties][filter] <> null) otherwise false,
            StrictSingleSelect = try (V[objects][selection]{0}[properties][strictSingleSelect][expr][Literal][Value] = "true") otherwise false,
            
            ValuesOnRow = try (V[objects][values]{0}[properties][valuesOnRow][expr][Literal][Value] = "true") otherwise false,
            ShowColumnSubtotals = try V[objects][subTotals]{0}[properties][columnSubtotals][expr][Literal][Value] otherwise null,
            ShowRowSubtotals = try V[objects][subTotals]{0}[properties][rowSubtotals][expr][Literal][Value] otherwise null,
            
            LabelDisplayUnits = try V[objects][labels]{0}[properties][labelDisplayUnits][expr][Literal][Value] otherwise null,
            ColumnAdjustment = try V[objects][columnHeaders]{0}[properties][columnAdjustment][expr][Literal][Value] otherwise null,
            HasBorder = try (V[visualContainerObjects][border]{0}[properties][show][expr][Literal][Value] = "true") otherwise false,
            DrillFilterOtherVisuals = try V[drillFilterOtherVisuals] otherwise null,
            SyncGroupName = try V[syncGroup][groupName] otherwise null,
            ParentGroup = try vJson[parentGroupName] otherwise null,
            IsHidden = try vJson[isHidden] otherwise false,
            HowCreated = try vJson[howCreated] otherwise null,
            
            TitleRaw = try V[visualContainerObjects][title]{0}[properties][text][expr][Literal][Value] otherwise null,
            TitleCleaned = if TitleRaw <> null 
                then Text.BetweenDelimiters(TitleRaw, "'", "'") 
                else null,
            VisualTitle = if TitleCleaned <> null 
                then TitleCleaned & " (" & VisualName & ")" 
                else VisualName,
            
            QS = try V[query][queryState] otherwise Record.FromList({}, {}),
            ProjectionRolesList = try Record.FieldNames(QS) otherwise {},
            ProjectionRoles = Text.Combine(ProjectionRolesList, ", "),
            
            FieldCount = try List.Sum(
                List.Transform(ProjectionRolesList, each 
                    try List.Count(Record.Field(QS, _)[projections]) otherwise 0
                )
            ) otherwise 0,
            
            HasFieldParameters = try List.AnyTrue(
                List.Transform(ProjectionRolesList, each
                    try (Record.Field(QS, _)[fieldParameters] <> null) otherwise false
                )
            ) otherwise false,
            
            CFCount = fnCountConditionalFormatting(V),
            
            PageNavigationDetails = if VisualType = "pageNavigator" then
                let
                    HasObjects = try (V[objects] <> null) otherwise false,
                    PagesEntries = if HasObjects then (try V[objects][pages] otherwise {}) else {},
                    
                    ShowHiddenPages = try List.First(
                        List.RemoveNulls(
                            List.Transform(PagesEntries, each
                                try [properties][showHiddenPages][expr][Literal][Value] otherwise null
                            )
                        )
                    ) otherwise "false",
                    ShowTooltipPages = try List.First(
                        List.RemoveNulls(
                            List.Transform(PagesEntries, each
                                try [properties][showTooltipPages][expr][Literal][Value] otherwise null
                            )
                        )
                    ) otherwise "false",
                    
                    ExcludedPageIds = List.RemoveNulls(
                        List.Transform(PagesEntries, each
                            let
                                pageId = try [selector][id] otherwise null,
                                showPage = try [properties][showPage][expr][Literal][Value] otherwise "true"
                            in
                                if pageId <> null and showPage = "false" then pageId else null
                        )
                    ),
                    
                    Detail = "{""showHiddenPages"":" & ShowHiddenPages 
                        & ",""showTooltipPages"":" & ShowTooltipPages 
                        & ",""excludedPages"":[" 
                        & Text.Combine(List.Transform(ExcludedPageIds, each """" & _ & """"), ",") 
                        & "]}"
                in
                    Detail
            else
                null
        in
            [
                ReportName = ReportName,
                VisualId = VisualName,
                VisualTitle = VisualTitle,
                PageName = pageDisplayName,
                PageId = pageId,
                PageOrdinal = pageOrdinal,
                VisualType = VisualType,
                X = PosX,
                Y = PosY,
                Width = PosW,
                Height = PosH,
                ZOrder = PosZ,
                FieldCount = FieldCount,
                ProjectionRoles = ProjectionRoles,
                ObjectKeys = Text.Combine(ObjectKeysList, ", "),
                VcObjectKeys = Text.Combine(VcObjectKeysList, ", "),
                SlicerMode = SlicerMode,
                SlicerOrientation = SlicerOrientation,
                HasSelfFilter = HasSelfFilter,
                HasDefaultFilter = HasDefaultFilter,
                StrictSingleSelect = StrictSingleSelect,
                ValuesOnRow = ValuesOnRow,
                ShowColumnSubtotals = ShowColumnSubtotals,
                ShowRowSubtotals = ShowRowSubtotals,
                LabelDisplayUnits = LabelDisplayUnits,
                ColumnAdjustment = ColumnAdjustment,
                HasBorder = HasBorder,
                DrillFilterOtherVisuals = DrillFilterOtherVisuals,
                HasFieldParameters = HasFieldParameters,
                HasConditionalFormatting = CFCount > 0,
                ConditionalFormattingCount = CFCount,
                SyncGroupName = SyncGroupName,
                ParentGroup = ParentGroup,
                IsHidden = IsHidden,
                HowCreated = HowCreated,
                PageNavigationDetails = PageNavigationDetails
            ],
    
    AllVisuals = List.Combine(
        List.Transform(
            List.Zip({PageData, List.Positions(PageIds)}),
            each
                let
                    pair = _,
                    pd = pair{0},
                    ordinal = pair{1},
                    PageDisplayName = try pd[PageJson][displayName] otherwise pd[PageId],
                    
                    VisualRecords = List.Transform(
                        pd[VisualJsonList],
                        each fnGetVisual(PageDisplayName, pd[PageId], ordinal, _)
                    )
                in
                    VisualRecords
        )
    ),
    
    Result = Table.FromRecords(AllVisuals),
    Sorted = Table.Sort(Result, {{"PageOrdinal", Order.Ascending}, {"ZOrder", Order.Ascending}})
in
    Sorted
```

---

## Query: ModelReferences_pbir

Every model object (table, column, measure, aggregation) referenced by each visual — including hidden dependencies from conditional formatting rules and slicer label measures.

### Hidden dependency sources

| Source | Description |
|---|---|
| `Projection` | Fields in visual field wells (queryState) |
| `FieldParameter` | Field parameter references at role level |
| `ConditionalFormatting` | Measures/columns driving CF rules, gradients, or field value colors |
| `Label` | Measures displayed as labels on button/list slicers |

```m
let
    Data = fn_pbir_ReportData(),
    ReportName = Data[ReportName],
    PageIds = Data[PageIds],
    PageData = Data[PageData],
    
    fnExtractCFRef = (node as record) as record =>
        let
            MEntity = try node[Measure][Expression][SourceRef][Entity] otherwise null,
            MProp = try node[Measure][Property] otherwise null,
            CEntity = try node[Column][Expression][SourceRef][Entity] otherwise null,
            CProp = try node[Column][Property] otherwise null,
            Entity = if MEntity <> null then MEntity else CEntity,
            Property = if MProp <> null then MProp else CProp
        in
            [Entity = Entity, Property = Property],
    
    fnGetRefs = (pageDisplayName as text, pageOrdinal as number, vJson as record) as list =>
        let
            VisualName = try vJson[name] otherwise "",
            V = try vJson[visual] otherwise Record.FromList({}, {}),
            VisualType = try V[visualType] otherwise "unknown",
            
            QS = try V[query][queryState] otherwise Record.FromList({}, {}),
            RoleNames = try Record.FieldNames(QS) otherwise {},
            
            ProjectionRefs = List.Combine(
                List.Transform(RoleNames, each
                    let
                        role = _,
                        roleData = try Record.Field(QS, role) otherwise Record.FromList({}, {}),
                        items = try roleData[projections] otherwise {}
                    in
                        List.Transform(items, each
                            let
                                qr = try _[queryRef] otherwise "",
                                parsed = fn_pbir_ParseQueryRef(qr)
                            in
                                [
                                    ReportName = ReportName,
                                    VisualId = VisualName,
                                    PageName = pageDisplayName,
                                    PageOrdinal = pageOrdinal,
                                    VisualType = VisualType,
                                    Role = role,
                                    QueryRef = qr,
                                    RefType = parsed[RefType],
                                    TableName = parsed[TableName],
                                    ObjectName = parsed[ObjectName],
                                    AggFunction = parsed[AggFunction],
                                    IsActive = try _[active] otherwise null,
                                    Source = "Projection"
                                ]
                        )
                )
            ),
            
            FieldParamRefs = List.Combine(
                List.Transform(RoleNames, each
                    let
                        role = _,
                        roleData = try Record.Field(QS, role) otherwise Record.FromList({}, {}),
                        fpList = try roleData[fieldParameters] otherwise {}
                    in
                        List.Transform(fpList, each
                            let
                                entity = try _[parameterExpr][Column][Expression][SourceRef][Entity] otherwise "",
                                prop = try _[parameterExpr][Column][Property] otherwise ""
                            in
                                [
                                    ReportName = ReportName,
                                    VisualId = VisualName,
                                    PageName = pageDisplayName,
                                    PageOrdinal = pageOrdinal,
                                    VisualType = VisualType,
                                    Role = role & " (FieldParameter)",
                                    QueryRef = entity & "." & prop,
                                    RefType = "FieldParameter",
                                    TableName = entity,
                                    ObjectName = prop,
                                    AggFunction = null,
                                    IsActive = null,
                                    Source = "FieldParameter"
                                ]
                        )
                )
            ),
            
            ValuesEntries = try V[objects][values] otherwise {},
            
            CFRefs = List.Combine(
                List.Transform(ValuesEntries, each
                    let
                        entry = _,
                        props = try entry[properties] otherwise Record.FromList({}, {}),
                        propNames = try Record.FieldNames(props) otherwise {},
                        cfProps = List.Select(propNames, each _ = "backColor" or _ = "fontColor")
                    in
                        List.Combine(
                            List.Transform(cfProps, (pn) =>
                                let
                                    exprNode = try Record.Field(props, pn)[solid][color][expr] otherwise Record.FromList({}, {}),
                                    exprFields = try Record.FieldNames(exprNode) otherwise {}
                                in
                                    if List.Contains(exprFields, "Conditional") then
                                        let
                                            Cases = try exprNode[Conditional][Cases] otherwise {},
                                            FirstCase = try Cases{0} otherwise Record.FromList({}, {}),
                                            Cond = try FirstCase[Condition] otherwise Record.FromList({}, {}),
                                            HasAnd = try (Cond[And] <> null) otherwise false,
                                            CompNode = if HasAnd then
                                                    try Cond[And][Left][Comparison][Left] otherwise Record.FromList({}, {})
                                                else
                                                    try Cond[Comparison][Left] otherwise Record.FromList({}, {}),
                                            Extracted = fnExtractCFRef(CompNode)
                                        in
                                            if Extracted[Entity] <> null and Extracted[Property] <> null then
                                                {[
                                                    ReportName = ReportName, VisualId = VisualName, PageName = pageDisplayName,
                                                    PageOrdinal = pageOrdinal, VisualType = VisualType,
                                                    Role = "Values (ConditionalFormatting)", QueryRef = Extracted[Entity] & "." & Extracted[Property],
                                                    RefType = "Column/Measure", TableName = Extracted[Entity], ObjectName = Extracted[Property],
                                                    AggFunction = null, IsActive = null, Source = "ConditionalFormatting"
                                                ]}
                                            else {}
                                    else if List.Contains(exprFields, "FillRule") then
                                        let
                                            InputNode = try exprNode[FillRule][Input] otherwise Record.FromList({}, {}),
                                            Extracted = fnExtractCFRef(InputNode)
                                        in
                                            if Extracted[Entity] <> null and Extracted[Property] <> null then
                                                {[
                                                    ReportName = ReportName, VisualId = VisualName, PageName = pageDisplayName,
                                                    PageOrdinal = pageOrdinal, VisualType = VisualType,
                                                    Role = "Values (ConditionalFormatting)", QueryRef = Extracted[Entity] & "." & Extracted[Property],
                                                    RefType = "Column/Measure", TableName = Extracted[Entity], ObjectName = Extracted[Property],
                                                    AggFunction = null, IsActive = null, Source = "ConditionalFormatting"
                                                ]}
                                            else {}
                                    else if List.Contains(exprFields, "Measure") or List.Contains(exprFields, "Column") then
                                        let
                                            Extracted = fnExtractCFRef(exprNode)
                                        in
                                            if Extracted[Entity] <> null and Extracted[Property] <> null then
                                                {[
                                                    ReportName = ReportName, VisualId = VisualName, PageName = pageDisplayName,
                                                    PageOrdinal = pageOrdinal, VisualType = VisualType,
                                                    Role = "Values (ConditionalFormatting)", QueryRef = Extracted[Entity] & "." & Extracted[Property],
                                                    RefType = "Column/Measure", TableName = Extracted[Entity], ObjectName = Extracted[Property],
                                                    AggFunction = null, IsActive = null, Source = "ConditionalFormatting"
                                                ]}
                                            else {}
                                    else {}
                            )
                        )
                )
            ),
            
            LabelEntries = try V[objects][label] otherwise {},
            LabelRefs = List.Combine(
                List.Transform(LabelEntries, each
                    let
                        mEntity = try [properties][field][expr][Measure][Expression][SourceRef][Entity] otherwise null,
                        mProp = try [properties][field][expr][Measure][Property] otherwise null,
                        cEntity = try [properties][field][expr][Column][Expression][SourceRef][Entity] otherwise null,
                        cProp = try [properties][field][expr][Column][Property] otherwise null,
                        Entity = if mEntity <> null then mEntity else cEntity,
                        Prop = if mProp <> null then mProp else cProp
                    in
                        if Entity <> null and Prop <> null then
                            {[
                                ReportName = ReportName, VisualId = VisualName, PageName = pageDisplayName,
                                PageOrdinal = pageOrdinal, VisualType = VisualType,
                                Role = "Values (Label)", QueryRef = Entity & "." & Prop,
                                RefType = "Column/Measure", TableName = Entity, ObjectName = Prop,
                                AggFunction = null, IsActive = null, Source = "Label"
                            ]}
                        else {}
                )
            ),
            
            Combined = List.Combine({ProjectionRefs, FieldParamRefs, CFRefs, LabelRefs})
        in
            Combined,
    
    AllRefs = List.Combine(
        List.Transform(
            List.Zip({PageData, List.Positions(PageIds)}),
            each
                let
                    pair = _,
                    pd = pair{0},
                    ordinal = pair{1},
                    PageDisplayName = try pd[PageJson][displayName] otherwise pd[PageId],
                    
                    VisualRefs = List.Combine(
                        List.Transform(
                            pd[VisualJsonList],
                            each try fnGetRefs(PageDisplayName, ordinal, _) otherwise {}
                        )
                    )
                in
                    VisualRefs
        )
    ),
    
    Result = Table.FromRecords(AllRefs),
    Sorted = Table.Sort(Result, {{"PageOrdinal", Order.Ascending}, {"VisualId", Order.Ascending}, {"Role", Order.Ascending}})
in
    Sorted
```

---

## Query: ConditionalFormattingRules_pbir

Every CF rule on every visual, across all four CF types. The query code is unchanged from the internal README — see the `.pbip` file for the full M code, or copy from the Power Query Editor.

### CF types detected

| CFType | JSON Location | Expression Key | Description |
|---|---|---|---|
| `Rules` | `objects.values[]` | `Conditional.Cases` | Threshold-based color bands (e.g. red/yellow/green) |
| `Gradient` | `objects.values[]` | `FillRule` | Continuous color scale (`linearGradient2` or `linearGradient3`) |
| `FieldValue` | `objects.values[]` | direct `Measure`/`Column` | Color driven by a DAX measure returning hex/color values |
| `DataBars` | `objects.columnFormatting[]` | `dataBars` | Horizontal bars proportional to cell values |

The full query code is included in the `.pbip` project file.

---

## Query: UnusedObjectsFinder_pbir

Distinct set of `Table.Object` references from the report — designed for joining with semantic model metadata to find unused objects.

```m
let
    Source = ModelReferences_pbir,
    
    UniqueRefs = Table.Distinct(
        Table.SelectColumns(Source, {"ReportName", "TableName", "ObjectName", "RefType"})
    ),
    
    WithKey = Table.AddColumn(UniqueRefs, "ModelObjectKey", 
        each [TableName] & "." & [ObjectName], type text
    ),
    
    Sorted = Table.Sort(WithKey, {{"TableName", Order.Ascending}, {"ObjectName", Order.Ascending}})
in
    Sorted
```

---

## Query: SlicerInventory_pbir

Detailed slicer analysis — mode, filters, field parameters, sync groups, and interaction settings.

```m
let
    Source = Visuals_pbir,
    SlicersOnly = Table.SelectRows(Source, each [VisualType] = "slicer" or [VisualType] = "advancedSlicerVisual" or [VisualType] = "listSlicer"),
    
    Refs = ModelReferences_pbir,
    SlicerRefs = Table.SelectRows(Refs, each [Role] = "Values" and [Source] = "Projection"),
    
    Joined = Table.NestedJoin(SlicersOnly, {"VisualId"}, SlicerRefs, {"VisualId"}, "RefData", JoinKind.LeftOuter),
    Expanded = Table.ExpandTableColumn(Joined, "RefData", {"TableName", "ObjectName", "QueryRef"}, {"SlicerTable", "SlicerField", "SlicerQueryRef"}),
    
    Selected = Table.SelectColumns(Expanded, {
        "ReportName", "VisualId", "PageName", "PageOrdinal",
        "SlicerTable", "SlicerField", "SlicerQueryRef",
        "SlicerMode", "SlicerOrientation",
        "HasSelfFilter", "HasDefaultFilter", "StrictSingleSelect",
        "HasFieldParameters", "SyncGroupName",
        "Width", "Height"
    }),
    
    Sorted = Table.Sort(Selected, {{"PageOrdinal", Order.Ascending}})
in
    Sorted
```

---

## Query: Bookmarks_pbir

Bookmark inventory — one row per bookmark with metadata, target page, and visual impact summary. Cross-references trigger visuals and affected visuals.

### Why bookmarks matter for governance

Bookmarks are frozen snapshots of visual state scattered across separate JSON files, with cross-references by opaque IDs back to visuals and pages. When someone renames a visual, moves it, changes its type, or deletes it, the bookmark silently becomes stale. There is no built-in way to audit which bookmarks are still valid.

The full query code is included in the `.pbip` project file.

---

## Query: BookmarkActions_pbir

Per-visual state changes for each bookmark — one row per visual per bookmark. Shows what the bookmark does to each visual: hides it, changes its filters, modifies its projections, or alters object properties.

### Staleness detection

The `IsStale` column compares the `visualType` stored in the bookmark's `explorationState` against the current `visualType` from the live `visual.json` file. If someone changed a bar chart to a line chart, or deleted the visual entirely, `IsStale = true`.

The full query code is included in the `.pbip` project file.

---

## Query: Buttons_pbir

Button and action inventory — every visual that carries a navigation action via `visualLink`. Covers native `actionButton` visuals, `image` visuals repurposed as buttons, and any other visual type with a link action.

### Link types

`Bookmark`, `Back`, `PageNavigation`, `Drillthrough`, `WebUrl`, `ClearAllSlicers`

The full query code is included in the `.pbip` project file.

---

## Query: SchemaCoverageAudit_pbir

On-demand diagnostic query — recursively walks every JSON file in the `.Report` folder, enumerates all key paths, and compares them against the extraction pipeline's known paths. Discovers new or unused PBIR fields without manual JSON inspection.

**Create as a disabled query. Do NOT load to the semantic model.** Right-click → Invoke to run on demand.

### Configuration boundary

The recursive walker stops descending into expression grammar nodes (`expr`, `Literal`, `Measure`, `Column`, `Conditional`, `FillRule`, etc.). These encode values using a shared recursive grammar — the same structure appears on every visual. The interesting gaps live above that boundary: structural settings like `sortDefinition`, `filterConfig`, `expansionStates`, `position.tabOrder`.

### Why the row count looks alarming

The walker emits a row for every path segment, not just leaf settings. The extraction map registers paths at coarse levels (e.g. `objects.general[]`), so intermediate nodes appear as unextracted even though the pipeline reads their children via direct path access. The genuinely actionable gaps are paths where the parent is also unextracted — nodes the pipeline never touches at all.

The full query code is included in the `.pbip` project file.

---

# The semantic model

## Bookmarks and Buttons — how the tables relate

Bookmarks and buttons are complementary halves of the same mechanism. A bookmark is a frozen snapshot of visual state. A button is the trigger that activates it. Understanding their governance implications requires crossing three tables.

### Join keys

```
Buttons_pbir.VisualId    ──►  Visuals_pbir.VisualId     (1:1, the button IS a visual)
Buttons_pbir.PageId      ──►  Visuals_pbir.PageId       (use both columns for unique join)
Buttons_pbir.BookmarkId  ──►  Bookmarks_pbir.BookmarkId (M:1, many buttons can trigger the same bookmark)
Bookmarks_pbir.BookmarkId ──► BookmarkActions_pbir.BookmarkId (1:M)
```

### Governance scenarios

**Orphaned buttons** — a button whose `BookmarkId` doesn't match any `Bookmarks_pbir.BookmarkId`. The bookmark was deleted but the button still references it.

**Stale bookmark targets** — `BookmarkActions_pbir.IsStale = true` means the visual type stored in the bookmark snapshot no longer matches the live visual.

**Image buttons** — `Buttons_pbir` rows where `VisualType = "image"` and `ImageSourceMeasure` is populated. These bypass standard button formatting and are easy to overlook.

---

# DAX Measures

## Slicer Sync Analysis

These measures analyze slicer synchronization across report pages. They require a **disconnected `SyncGroups` table**:

```dax
SyncGroups = DISTINCT( SELECTCOLUMNS( FILTER( Visuals_pbir, Visuals_pbir[SyncGroupName] <> BLANK() ), "SyncGroupName", Visuals_pbir[SyncGroupName] ) )
```

**Do not create a relationship** between `SyncGroups` and any other table.

### Slicer Sync Status

```dax
Slicer Sync Status = 
VAR _selectedGroup =
    SELECTEDVALUE( SyncGroups[SyncGroupName] )
VAR _slicers =
    FILTER(
        Visuals_pbir,
        ( Visuals_pbir[VisualType] = "slicer" || Visuals_pbir[VisualType] = "advancedSlicerVisual" || Visuals_pbir[VisualType] = "listSlicer" )
            && Visuals_pbir[PageId] IN VALUES( Pages_pbir[PageId] )
    )
VAR _inGroup =
    FILTER(
        _slicers,
        Visuals_pbir[SyncGroupName] = _selectedGroup
    )
VAR _countInGroup = COUNTROWS( _inGroup )
VAR _isHidden =
    MAXX( _inGroup, Visuals_pbir[IsHidden] )
RETURN
    SWITCH(
        TRUE(),
        _countInGroup > 0 && _isHidden,
            "Hidden, Synced",
        _countInGroup > 0,
            "Visible, Synced",
        "No Slicer"
    )
```

### Slicer Sync HTML

The HTML measure for the **HTML Content (lite)** visual is included in the `.pbip` project file.

## Page Navigation HTML

The page navigation HTML measure is included in the `.pbip` project file.

---

# Good to Know

At the time of developing this solution, there was no official documentation for the PBIR format beyond the JSON schema definitions in Microsoft's [fabric/item/report](https://github.com/microsoft/json-schemas/tree/main/fabric/item/report) repository. The schemas define the structure but not the behavior — they tell you which fields can exist, not when Power BI chooses to write or omit them.

### Power BI strips default values from visual.json

When all configuration options for a visual feature are at their default values, Power BI Desktop removes the entire JSON node from `visual.json` rather than writing out the defaults explicitly. This means absence of a JSON node does not mean "feature not configured" — it means "configured with all defaults." Extraction code must define its own defaults via `try ... otherwise <default>`.

### The `try...otherwise` discipline

Every field access in these queries uses `try...otherwise`. This is essential because not every visual type has every field, Power BI strips default values, schema versions differ between visuals in the same report, and custom visuals use completely different object structures. Without `try...otherwise`, a single missing field on a single visual would crash the entire query.

---

# PBIR Reference

## Visual Type Reference

| visualType | Description |
|---|---|
| `tableEx` | Table |
| `pivotTable` | Matrix |
| `slicer` | Slicer (classic) |
| `advancedSlicerVisual` | Slicer (button/tile) |
| `listSlicer` | Slicer (list) |
| `card` | Card (classic) |
| `multiRowCard` | Multi-row Card (classic) |
| `cardVisual` | Card (new) |
| `barChart` | Bar Chart |
| `columnChart` | Column Chart |
| `lineChart` | Line Chart |
| `lineClusteredColumnComboChart` | Line and Clustered Column |
| `scatterChart` | Scatter Chart |
| `pieChart` | Pie Chart |
| `donutChart` | Donut Chart |
| `treemap` | Treemap |
| `waterfallChart` | Waterfall |
| `funnel` | Funnel |
| `gauge` | Gauge |
| `kpi` | KPI |
| `decompositionTreeVisual` | Decomposition Tree |
| `keyDriversVisual` | Key Influencers |
| `textbox` | Text Box |
| `image` | Image |
| `basicShape` | Shape |
| `actionButton` | Button |
| `bookmarkNavigator` | Bookmark Navigator |
| `pageNavigator` | Page Navigator |
| `visualGroup` | Visual Group (container) |

## PBIR JSON Encoding Patterns

**Literal value suffixes:** `D` (decimal), `L` (long integer), `M` (currency)

**Color encoding:** Single-quoted hex strings `"'#F8696B'"`. The `#` character requires URL-encoding as `%23` in SVG data URIs. Theme colors use `ThemeDataColor` with `ColorId` and `Percent`.

**Slicer sync groups:** Encoded in `visual.syncGroup` with `groupName` (auto-derived from field name), `fieldChanges`, and `filterChanges`. The Sync Slicers pane in Power BI Desktop is entirely a derived UI view with no separate configuration file.

---

# Backlog

- **`cardVisual` settings extraction** in `Visuals_pbir` — the modern card visual stores layout and callout configuration in `objects.layout[].properties.columnCount` and `objects.value[].properties.fontSize`. Currently `Visuals_pbir` does not extract any `cardVisual`-specific properties.
- **SyncGroups (BLANK) value** — the disconnected `SyncGroups` slicer shows a `(BLANK)` value. Investigate whether this is a Power Query output issue or a slicer behavior with disconnected tables.

---

# Resources

- [Microsoft PBIR JSON Schemas](https://github.com/microsoft/json-schemas/tree/main/fabric/item/report) — official schema definitions for the PBIR folder-based format

## Sample data

The included sample report uses the **Contoso 1M row dataset** from SQLBI's Contoso Data Generator V2. Download the ready-to-use data from [github.com/sql-bi/Contoso-Data-Generator-V2-data/releases/tag/ready-to-use-data](https://github.com/sql-bi/Contoso-Data-Generator-V2-data/releases/tag/ready-to-use-data).
