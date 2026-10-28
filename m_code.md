# Power Query (M) code

Final version: months are combined automatically from folders, so a new month only needs a new file (or a new Google Sheet tab) followed by a refresh.

## Folder layout

```
<main folder>\
  Analysts.xlsx
  Tasks\     September_2026.xlsx, October_2026.xlsx, ...
  Rejected\  Rejected_September_2026.csv, ...
  Time\      Time_September_2026.csv, ...
```

File and tab names must follow `Month_Year` in English (for example `January_2027`); the month and year are read from the name.

Google Sheet URLs, local paths and two name-correction values are replaced with placeholders. The three configuration tables (weights, targets, scoring rubric) were created with *Enter Data*, so their encoded contents are omitted.


## `FolderPath`
Text parameter: the main folder that holds the monthly files and `Analysts.xlsx`. Change it here if the folder moves.

```m
"<LOCAL_FOLDER>\" meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]
```


## `Random Checks`
Quality-check results from a Google Sheet. All months are kept; each row gets a `Month` (first day of the month of its upload date).

```m
let
    Source = GoogleSheets.Contents("<GOOGLE_SHEET_URL>"),
    #"Final Sheet_Table" = Source{[name="Final Sheet",ItemKind="Table"]}[Data],
    #"Promoted Headers" = Table.PromoteHeaders(#"Final Sheet_Table", [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"Analyst", type text}, {"Upload date", type date}, {"Checking date", type date}, {"Doc ID", type text}, {"Comments", Int64.Type}, {"Number of total tags", Int64.Type}, {"Accuracy", Percentage.Type}, {"Checking Analyst", type text}}),
    #"Added Month" = Table.AddColumn(#"Changed Type", "Month", each Date.StartOfMonth([Upload date]), type date)
in
    #"Added Month"
```


## `Worktimal`
Task exports: reads every monthly `.xlsx` in the `Tasks` folder (sheet `task`), takes the month from the file name (e.g. `September_2026.xlsx`), derives the analyst name from the worker email, and splits timestamps into date and time columns.

```m
let
    Files = Folder.Files(FolderPath & "Tasks"),
    Monthly = Table.SelectRows(Files, each Text.EndsWith([Name], ".xlsx") and not Text.StartsWith([Name], "~$")),
    WithData = Table.AddColumn(Monthly, "Data", each Table.PromoteHeaders(Excel.Workbook([Content], null, true){[Name="task"]}[Data], [PromoteAllScalars=true])),
    WithMonth = Table.AddColumn(WithData, "Month", each Date.FromText("1 " & Text.Replace(Text.BeforeDelimiter([Name], "."), "_", " "), "en-US"), type date),
    Slim = Table.SelectColumns(WithMonth, {"Month", "Data"}),
    #"Promoted Headers1" = Table.ExpandTableColumn(Slim, "Data", Table.ColumnNames(Slim{0}[Data])),
    #"Changed Type3" = Table.TransformColumnTypes(#"Promoted Headers1",{{"Task_id", type text}, {"Index", Int64.Type}, {"Status", type text}, {"StatusTimestamp", type text}, {"TaskDefinition", type text}, {"EntityId", type text}, {"NumberOfPages", Int64.Type}, {"Slo", type datetime}, {"Sla", type datetime}, {"Worker", type text}, {"UnableToProceedTimestamp", type text}, {"UnableToProceedReason", type text}, {"StartTimestamp", type text}, {"FinishTimestamp", type text}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Changed Type3",{{"Task_id", type text}, {"Index", Int64.Type}, {"Status", type text}, {"StatusTimestamp", type text}, {"TaskDefinition", type text}, {"EntityId", type text}, {"NumberOfPages", Int64.Type}, {"Slo", type datetime}, {"Sla", type datetime}, {"Worker", type text}, {"UnableToProceedTimestamp", type text}, {"UnableToProceedReason", type text}, {"StartTimestamp", type text}, {"FinishTimestamp", type text}}),
    #"Added Custom Column" = Table.AddColumn(#"Changed Type", "Custom", each let splitWorker = Splitter.SplitTextByDelimiter(".", QuoteStyle.None)([Worker]), splitsplitWorker1 = Splitter.SplitTextByDelimiter("@", QuoteStyle.None)(splitWorker{1}?) in Text.Combine({Text.Proper(splitWorker{0}?), " ", Text.Proper(splitsplitWorker1{0}?)}), type text),
    #"Renamed Columns" = Table.RenameColumns(#"Added Custom Column",{{"Custom", "Analyst Name"}}),
    #"Reordered Columns" = Table.ReorderColumns(#"Renamed Columns",{"Task_id", "Index", "Status", "StatusTimestamp", "TaskDefinition", "EntityId", "NumberOfPages", "Slo", "Sla", "Worker", "Analyst Name", "UnableToProceedTimestamp", "UnableToProceedReason", "StartTimestamp", "FinishTimestamp"}),
    #"Split Column by Delimiter" = Table.SplitColumn(#"Reordered Columns", "StartTimestamp", Splitter.SplitTextByEachDelimiter({"+"}, QuoteStyle.Csv, false), {"StartTimestamp.1", "StartTimestamp.2"}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Split Column by Delimiter",{{"StartTimestamp.1", type datetime}, {"StartTimestamp.2", type text}}),
    #"Removed Columns" = Table.RemoveColumns(#"Changed Type1",{"StartTimestamp.2"}),
    #"Split Column by Delimiter1" = Table.SplitColumn(#"Removed Columns", "FinishTimestamp", Splitter.SplitTextByEachDelimiter({"+"}, QuoteStyle.Csv, false), {"FinishTimestamp.1", "FinishTimestamp.2"}),
    #"Changed Type2" = Table.TransformColumnTypes(#"Split Column by Delimiter1",{{"FinishTimestamp.1", type datetime}, {"FinishTimestamp.2", type text}}),
    #"Removed Columns1" = Table.RemoveColumns(#"Changed Type2",{"FinishTimestamp.2"}),
    #"Renamed Columns1" = Table.RenameColumns(#"Removed Columns1",{{"StartTimestamp.1", "StartTimestamp"}, {"FinishTimestamp.1", "FinishTimestamp"}}),
    #"Inserted Date" = Table.AddColumn(#"Renamed Columns1", "Date", each DateTime.Date([StartTimestamp]), type date),
    #"Reordered Columns1" = Table.ReorderColumns(#"Inserted Date",{"Task_id", "Index", "Status", "StatusTimestamp", "TaskDefinition", "EntityId", "NumberOfPages", "Slo", "Sla", "Worker", "Analyst Name", "UnableToProceedTimestamp", "UnableToProceedReason", "Date", "StartTimestamp", "FinishTimestamp"}),
    #"Renamed Columns2" = Table.RenameColumns(#"Reordered Columns1",{{"Date", "Start date"}}),
    #"Extracted Time" = Table.TransformColumns(#"Renamed Columns2",{{"StartTimestamp", DateTime.Time, type time}}),
    #"Renamed Columns3" = Table.RenameColumns(#"Extracted Time",{{"StartTimestamp", "Start time"}}),
    #"Inserted Date1" = Table.AddColumn(#"Renamed Columns3", "Date", each DateTime.Date([FinishTimestamp]), type date),
    #"Renamed Columns4" = Table.RenameColumns(#"Inserted Date1",{{"Date", "Finish Date"}}),
    #"Extracted Time1" = Table.TransformColumns(#"Renamed Columns4",{{"FinishTimestamp", DateTime.Time, type time}}),
    #"Renamed Columns5" = Table.RenameColumns(#"Extracted Time1",{{"FinishTimestamp", "Finish time"}}),
    #"Reordered Columns2" = Table.ReorderColumns(#"Renamed Columns5",{"Task_id", "Index", "Status", "StatusTimestamp", "TaskDefinition", "EntityId", "NumberOfPages", "Slo", "Sla", "Worker", "Analyst Name", "UnableToProceedTimestamp", "UnableToProceedReason", "Start date", "Start time", "Finish Date", "Finish time"})
in
    #"Reordered Columns2"
```


## `Rejected`
Rejected tasks: reads every `Rejected_<Month>_<Year>.csv` in the `Rejected` folder, takes the month from the file name, and builds a rejection date from the year / month name / day columns.

```m
let
    Files = Folder.Files(FolderPath & "Rejected"),
    Csvs = Table.SelectRows(Files, each Text.EndsWith([Name], ".csv")),
    WithData = Table.AddColumn(Csvs, "Data", each Table.PromoteHeaders(Csv.Document([Content], [Delimiter=",", Columns=5, Encoding=65001, QuoteStyle=QuoteStyle.None]), [PromoteAllScalars=true])),
    WithMonth = Table.AddColumn(WithData, "Month", each Date.FromText("1 " & Text.Replace(Text.Replace(Text.BeforeDelimiter([Name], "."), "Rejected_", ""), "_", " "), "en-US"), type date),
    Slim = Table.SelectColumns(WithMonth, {"Month", "Data"}),
    #"Promoted Headers" = Table.ExpandTableColumn(Slim, "Data", Table.ColumnNames(Slim{0}[Data])),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"Name_Surname", type text}, {"Year of Rejection Date", Int64.Type}, {"Month of Rejection Date", type text}, {"Day of Rejection Date", Int64.Type}, {"Count of email (Custom SQL Query2)", Int64.Type}}),
    #"Added Custom" = Table.AddColumn(#"Changed Type", "Date of Rejection", each #date( [Year of Rejection Date], Date.Month(Date.FromText("1 " & [Month of Rejection Date] & " 2000", "en-US")), [Day of Rejection Date] ))
in
    #"Added Custom"
```


## `Time`
Average acceptance and processing times: reads every `Time_<Month>_<Year>.csv` in the `Time` folder, takes the month from the file name, and fixes name spellings so they match the Analysts table.

```m
let
    Files = Folder.Files(FolderPath & "Time"),
    Csvs = Table.SelectRows(Files, each Text.EndsWith([Name], ".csv")),
    WithData = Table.AddColumn(Csvs, "Data", each Table.PromoteHeaders(Csv.Document([Content], [Delimiter=",", Columns=6, Encoding=65001, QuoteStyle=QuoteStyle.None]), [PromoteAllScalars=true])),
    WithMonth = Table.AddColumn(WithData, "Month", each Date.FromText("1 " & Text.Replace(Text.Replace(Text.BeforeDelimiter([Name], "."), "Time_", ""), "_", " "), "en-US"), type date),
    Slim = Table.SelectColumns(WithMonth, {"Month", "Data"}),
    #"Promoted Headers" = Table.ExpandTableColumn(Slim, "Data", Table.ColumnNames(Slim{0}[Data])),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"Name Surname", type text}, {"Avg. Acceptance Time", type number}, {"Avg. number_of_pages", type number}, {"Avg. Processing Time", type number}, {"Avg. Processing Time_1", type number}, {"Count of document_id", Int64.Type}}),
    #"Replaced Value" = Table.ReplaceValue(#"Changed Type", "<name as in source>", "<corrected name>", Replacer.ReplaceText, {"Name Surname"}),
    #"Replaced Value1" = Table.ReplaceValue(#"Replaced Value", "<name as in source>", "<corrected name>", Replacer.ReplaceText, {"Name Surname"})
in
    #"Replaced Value1"
```


## `Analysts`
Analyst list (level, shift, KPI framework) from `Analysts.xlsx` in the main folder.

```m
let
    Source = Excel.Workbook(File.Contents(FolderPath & "Analysts.xlsx"), null, true),
    Sheet1_Sheet = Source{[Item="Sheet1",Kind="Sheet"]}[Data],
    #"Changed Type" = Table.TransformColumnTypes(Sheet1_Sheet,{{"Column1", type text}, {"Column2", type text}}),
    #"Promoted Headers" = Table.PromoteHeaders(#"Changed Type", [PromoteAllScalars=true]),
    #"Changed Type1" = Table.TransformColumnTypes(#"Promoted Headers",{{"Analyst", type text}, {"Level", type text}})
in
    #"Changed Type1"
```


## `Team lead assessment`
Team-lead ratings from a Google Sheet with one tab per month. Tabs must be named `<Month>_<Year>` (e.g. `September_2026`); tabs with other names are ignored. Each tab's name becomes the `Month`.

```m
let
    Source = GoogleSheets.Contents("<GOOGLE_SHEET_URL>"),
    Tabs = Table.SelectRows(Source, each [ItemKind] = "Table"),
    WithMonth = Table.AddColumn(Tabs, "Month", each try Date.FromText("1 " & Text.Replace([name], "_", " "), "en-US") otherwise null, type date),
    MonthTabs = Table.SelectRows(WithMonth, each [Month] <> null),
    WithData = Table.AddColumn(MonthTabs, "Table", each Table.PromoteHeaders([Data], [PromoteAllScalars=true])),
    Slim = Table.SelectColumns(WithData, {"Month", "Table"}),
    #"Promoted Headers" = Table.ExpandTableColumn(Slim, "Table", Table.ColumnNames(Slim{0}[Table])),
    #"Renamed Columns" = Table.RenameColumns(#"Promoted Headers",{{"Flexibility ", "Flexibility"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"Analyst", type text}, {"Level", type text}, {"Promptness", Int64.Type}, {"Activity in the Channels", Int64.Type}, {"Willingness to Help", Int64.Type}, {"Responsiveness", Int64.Type}, {"Flexibility", Int64.Type}, {"Organizational skills", Int64.Type}})
in
    #"Changed Type"
```


## `KPI Weights`
Entered-data table: Section, KPI, Analyst Weight, Senior Weight. Values omitted here.

```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("<ENCODED_TABLE_DATA>", BinaryEncoding.Base64), Compression.Deflate)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [Section = _t, KPI = _t, #"Analyst Weight" = _t, #"Senior Weight" = _t]),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Section", type text}, {"KPI", type text}, {"Analyst Weight", Percentage.Type}, {"Senior Weight", Percentage.Type}})
in
    #"Changed Type"
```


## `KPI Targets`
Entered-data table: target and termination threshold per KPI and level. Values omitted here.

```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("<ENCODED_TABLE_DATA>", BinaryEncoding.Base64), Compression.Deflate)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [KPI = _t, Level = _t, Target = _t, #"Termination Threshold" = _t]),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"KPI", type text}, {"Level", type text}, {"Target", type number}, {"Termination Threshold", type number}})
in
    #"Changed Type"
```


## `KPI scores`
Entered-data table: the 1–5 scoring rubric (KPI, Level, Score, Criteria). Values omitted here.

```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("<ENCODED_TABLE_DATA>", BinaryEncoding.Base64), Compression.Deflate)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [KPI = _t, Level = _t, Score = _t, Criteria = _t]),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"KPI", type text}, {"Level", type text}, {"Score", Int64.Type}, {"Criteria", type text}})
in
    #"Changed Type"
```
