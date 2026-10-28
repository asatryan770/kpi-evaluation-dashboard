# DAX measures

Analyst names that were hard-coded in two measures are replaced with `<Analyst A>` / `<Analyst B>`. Line breaks and indentation were re-added automatically for readability.


## Raw measures

Base metrics computed straight from the data tables.

### `Working Days`
Distinct weekdays (Mon–Fri) on which the analyst logged work.

```dax
Working Days =
CALCULATE(
    DISTINCTCOUNT(Worktimal[Working Date]),
    FILTER(
        Worktimal,
        WEEKDAY(Worktimal[Working Date], 2) <= 5
    )
)
```

### `AVG Spreads Done Daily`
Distinct 'Spread DBS' tasks divided by working days.

```dax
AVG Spreads Done Daily =
DIVIDE(
    CALCULATE(
        DISTINCTCOUNT(Worktimal[Task_id]),
        Worktimal[TaskDefinition] = "Spread DBS"
    ),
    [Working Days],
    0
)
```

### `AVG Spread Processing Time`
Average minutes between start and finish of 'Spread DBS' tasks.

```dax
AVG Spread Processing Time =
AVERAGEX(
    FILTER(
        Worktimal,
        Worktimal[TaskDefinition] = "Spread DBS"
    ),
    VAR StartDateTime =
    Worktimal[Start date] + Worktimal[Start time]
    VAR FinishDateTime =
    Worktimal[Finish Date] + Worktimal[Finish time]
    RETURN
    DATEDIFF(StartDateTime, FinishDateTime, SECOND) / 60
)
```

### `AVG Metadata Done Daily`
Distinct 'Check Metadata' tasks per working day.

```dax
AVG Metadata Done Daily =
DIVIDE(
    CALCULATE(
        DISTINCTCOUNT(Worktimal[Task_id]),
        Worktimal[TaskDefinition] = "Check Metadata"
    ),
    [Working Days],
    0
)
```

### `AVG Reports Checked Daily`
Reports checked per checking day; only for the analysts who perform checks, blank for everyone else.

```dax
AVG Reports Checked Daily =
VAR _Analyst =
SELECTEDVALUE(Analysts[Analyst])
RETURN
IF(
    _Analyst IN {"<Analyst A>", "<Analyst B>"},
    CALCULATE(
        DIVIDE(
            COUNTROWS('Random Checks'),
            DISTINCTCOUNT('Random Checks'[Checking date])
        ),
        CROSSFILTER(
            Analysts[Analyst],
            'Random Checks'[Analyst],
            NONE
        ),
        'Random Checks'[Checking Analyst] = _Analyst
    ),
    BLANK()
)
```

### `Spreading accuracy rate`
Average accuracy from the random quality checks.

```dax
Spreading accuracy rate =
AVERAGE('Random Checks'[Accuracy])
```

### `Total Rejected tasks`
Total rejected tasks in the period.

```dax
Total Rejected tasks =
SUM(Rejected[Count of email (Custom SQL Query2)])
```

### `Working Hours`
Total hours between task start and finish.

```dax
Working Hours =
SUMX(
    Worktimal,
    VAR StartDateTime =
    Worktimal[Start date] + Worktimal[Start time]
    VAR FinishDateTime =
    Worktimal[Finish Date] + Worktimal[Finish time]
    VAR TaskSeconds =
    DATEDIFF(StartDateTime, FinishDateTime, SECOND)
    RETURN
    DIVIDE(TaskSeconds, 3600)
)
```

### `WorkingTotal Working Hours`
Same calculation as Working Hours.

```dax
WorkingTotal Working Hours =
SUMX(
    Worktimal,
    VAR StartDateTime =
    Worktimal[Start date] + Worktimal[Start time]
    VAR FinishDateTime =
    Worktimal[Finish Date] + Worktimal[Finish time]
    VAR TaskSeconds =
    DATEDIFF(StartDateTime, FinishDateTime, SECOND)
    RETURN
    DIVIDE(TaskSeconds, 3600)
)
```

### `AVG Working Hours`
Working hours per working day.

```dax
AVG Working Hours =
DIVIDE([Working Hours], [Working Days])
```

### `Working Hours Coverage`
Average daily hours as a share of expected hours (6 h full-time, 4 h part-time).

```dax
Working Hours Coverage =
VAR _Shift =
SELECTEDVALUE ( Analysts[Shift], "Full time" )
VAR _ExpectedHours =
IF (
    _Shift = "Part time",
    4,
    6
)
RETURN
DIVIDE (
    [AVG Working Hours],
    _ExpectedHours
)
```


## Targets and 'target met' flags

Targets are looked up from the KPI Targets table by KPI and analyst level; each flag returns 1 (met) or 0.

### `Spreads Daily Target`
Daily spreads target for the selected level.

```dax
Spreads Daily Target =
VAR _Level =
SELECTEDVALUE ( Analysts[Level] )
RETURN
CALCULATE (
    MAX ( 'KPI Targets'[Target] ),
    'KPI Targets'[KPI] = "Spreads Done Daily",
    'KPI Targets'[Level] = _Level
)
```

### `Processing Time Target`
Processing-time target for the selected level.

```dax
Processing Time Target =
VAR _Level =
SELECTEDVALUE(Analysts[Level])
RETURN
CALCULATE(
    MAX('KPI Targets'[Target]),
    'KPI Targets'[KPI] = "Processing Time",
    'KPI Targets'[Level] = _Level
)
```

### `Spreading Accuracy Target`
Accuracy target for the selected level.

```dax
Spreading Accuracy Target =
VAR _Level =
SELECTEDVALUE(Analysts[Level])
RETURN
CALCULATE(
    MAX('KPI Targets'[Target]),
    'KPI Targets'[KPI] = "Spreading Accuracy",
    'KPI Targets'[Level] = _Level
)
```

### `Acceptance Time Target`
Acceptance-time target for the selected level.

```dax
Acceptance Time Target =
VAR _Level =
SELECTEDVALUE(Analysts[Level])
RETURN
CALCULATE(
    MAX('KPI Targets'[Target]),
    'KPI Targets'[KPI] = "Acceptance Time",
    'KPI Targets'[Level] = _Level
)
```

### `Rejected Tasks Target`
Rejected-tasks target for the selected level.

```dax
Rejected Tasks Target =
VAR _Level =
SELECTEDVALUE(Analysts[Level])
RETURN
CALCULATE(
    MAX('KPI Targets'[Target]),
    'KPI Targets'[KPI] = "Rejected Spreads",
    'KPI Targets'[Level] = _Level
)
```

### `Spreads Target Met`
1 if daily spreads reach the target; part-time analysts get a 4/7 pro-rated target.

```dax
Spreads Target Met =
VAR _Shift =
SELECTEDVALUE ( Analysts[Shift], "Full time" )
VAR _AdjustedTarget =
IF (
    _Shift = "Part time",
    [Spreads Daily Target] * 4 / 7,
    [Spreads Daily Target]
)
RETURN
IF (
    'Raw Measures'[AVG Spreads Done Daily] >= _AdjustedTarget,
    1,
    0
)
```

### `Processing Target Met`
1 if average processing time is at or under target.

```dax
Processing Target Met =
IF(
    'Raw Measures'[AVG Spread Processing Time] <= [Processing Time Target],
    1,
    0
)
```

### `Accuracy Target Met`
1 if accuracy reaches the target.

```dax
Accuracy Target Met =
VAR Accuracy = 'Raw Measures'[Spreading accuracy rate]
RETURN
IF(
    ISBLANK(Accuracy),
    BLANK(),
    IF(
        Accuracy >= [Spreading Accuracy Target],
        1,
        0
    )
)
```

### `Acceptance Time Target Met`
1 if average acceptance time is at or under target.

```dax
Acceptance Time Target Met =
IF(
    MAX('Time'[Avg. Acceptance Time])<= [Acceptance Time Target],
    1,
    0
)
```

### `Rejected Tasks Target Met`
1 if rejected tasks are at or under target; part-time target pro-rated 4/7.

```dax
Rejected Tasks Target Met =
VAR _Actual =
'Raw Measures'[Total Rejected tasks]
VAR _Shift =
SELECTEDVALUE ( Analysts[Shift], "Full time" )
VAR _AdjustedTarget =
IF (
    _Shift = "Part time",
    [Rejected Tasks Target] * 4 / 7,
    [Rejected Tasks Target]
)
RETURN
IF (
    ISBLANK ( _Actual ) || ISBLANK ( [Rejected Tasks Target] ),
    BLANK(),
    IF ( _Actual <= _AdjustedTarget, 1, 0 )
)
```


## Scoring measures (1–5)

Convert raw results into a 1–5 score. Bands depend on analyst level where noted.

### `Spreads Done Daily`
Score from daily spreads, banded by Junior / Middle / Senior.

```dax
Spreads Done Daily =
VAR Seniority =
SELECTEDVALUE(Analysts[Level])
VAR Spreads =
[AVG Spreads Done Daily]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(Spreads), BLANK(),
    Seniority = "Junior" && Spreads >= 8, 5,
    Seniority = "Junior" && Spreads >= 7, 4,
    Seniority = "Junior" && Spreads >= 6, 3,
    Seniority = "Junior" && Spreads >= 5, 2,
    Seniority = "Junior", 1,
    Seniority = "Middle" && Spreads >= 9, 5,
    Seniority = "Middle" && Spreads >= 8, 4,
    Seniority = "Middle" && Spreads >= 7, 3,
    Seniority = "Middle" && Spreads >= 6, 2,
    Seniority = "Middle", 1,
    Seniority = "Senior" && Spreads >= 10, 5,
    Seniority = "Senior" && Spreads >= 9, 4,
    Seniority = "Senior" && Spreads >= 8, 3,
    Seniority = "Senior" && Spreads >= 7, 2,
    Seniority = "Senior", 1,
    BLANK()
)
```

### `Spreading Accuracy`
Score from accuracy rate, banded by level.

```dax
Spreading Accuracy =
VAR Seniority =
SELECTEDVALUE(Analysts[Level])
VAR Accuracy =
[Spreading Accuracy Rate]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(Accuracy), BLANK(),
    Seniority = "Junior" && Accuracy >= 0.95, 5,
    Seniority = "Junior" && Accuracy >= 0.94, 4,
    Seniority = "Junior" && Accuracy >= 0.93, 3,
    Seniority = "Junior" && Accuracy >= 0.92, 2,
    Seniority = "Junior", 1,
    Seniority = "Middle" && Accuracy >= 0.97, 5,
    Seniority = "Middle" && Accuracy >= 0.96, 4,
    Seniority = "Middle" && Accuracy >= 0.95, 3,
    Seniority = "Middle" && Accuracy >= 0.94, 2,
    Seniority = "Middle", 1,
    Seniority = "Senior" && Accuracy >= 0.99, 5,
    Seniority = "Senior" && Accuracy >= 0.98, 4,
    Seniority = "Senior" && Accuracy >= 0.97, 3,
    Seniority = "Senior" && Accuracy >= 0.96, 2,
    Seniority = "Senior", 1,
    BLANK()
)
```

### `Processing Time`
Score from average processing time, banded by level.

```dax
Processing Time =
VAR Seniority =
SELECTEDVALUE(Analysts[Level])
VAR ProcessingTime =
[AVG Spread Processing Time]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(ProcessingTime), BLANK(),
    Seniority = "Junior" && ProcessingTime <= 45, 5,
    Seniority = "Junior" && ProcessingTime <= 55, 4,
    Seniority = "Junior" && ProcessingTime <= 60, 3,
    Seniority = "Junior" && ProcessingTime <= 70, 2,
    Seniority = "Junior", 1,
    Seniority = "Middle" && ProcessingTime <= 35, 5,
    Seniority = "Middle" && ProcessingTime <= 44, 4,
    Seniority = "Middle" && ProcessingTime <= 46, 3,
    Seniority = "Middle" && ProcessingTime <= 55, 2,
    Seniority = "Middle", 1,
    Seniority = "Senior" && ProcessingTime <= 30, 5,
    Seniority = "Senior" && ProcessingTime <= 35, 4,
    Seniority = "Senior" && ProcessingTime <= 37, 3,
    Seniority = "Senior" && ProcessingTime <= 45, 2,
    Seniority = "Senior", 1,
    BLANK()
)
```

### `Acceptance Time`
Score from average acceptance time (≤1 min = 5 … >5 min = 1).

```dax
Acceptance Time =
VAR Acceptance =
AVERAGE(Time[Avg. Acceptance Time])
RETURN
SWITCH(
    TRUE(),
    ISBLANK(Acceptance), BLANK(),
    Acceptance <= 1, 5,
    Acceptance <= 2, 4,
    Acceptance <= 3, 3,
    Acceptance <= 5, 2,
    1
)
```

### `Rejected Spreads`
Score from rejected count; blank for Senior framework.

```dax
Rejected Spreads =
VAR _Rejected =
COALESCE ( [Total Rejected tasks], 0 )
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
RETURN
IF (
    _Framework = "Senior",
    BLANK(),
    SWITCH (
        TRUE(),
        _Rejected <= 2, 5,
        _Rejected <= 5, 4,
        _Rejected <= 10, 3,
        _Rejected <= 15, 2,
        1
    )
)
```

### `Reports Checked Daily`
Score from reports checked per day; only for analysts who do checks.

```dax
Reports Checked Daily =
VAR Reports =
[AVG Reports Checked Daily]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(Reports), BLANK(),
    Reports >= 5, 5,
    Reports >= 4, 4,
    Reports >= 3, 3,
    Reports >= 2, 2,
    1
)
```

### `Metadata Done Daily`
Score from metadata tasks per day; only for analysts who do checks.

```dax
Metadata Done Daily =
VAR CeckingAnalyst =
SELECTEDVALUE(Analysts[Analyst])
VAR MetadataTasks =
[AVG Metadata Done Daily]
RETURN
IF(
    CeckingAnalyst IN {"<Analyst A>", "<Analyst B>"},
    SWITCH(
        TRUE(),
        ISBLANK(MetadataTasks), BLANK(),
        MetadataTasks > 10, 5,
        MetadataTasks > 8, 4,
        MetadataTasks > 6, 3,
        MetadataTasks > 4, 2,
        1
    ),
    BLANK()
)
```

### `Working Hours Score`
Score from working-hours coverage.

```dax
Working Hours Score =
VAR Coverage =
[Working Hours Coverage]
RETURN
SWITCH(
    TRUE(),
    ISBLANK(Coverage), BLANK(),
    Coverage >= 0.99, 5,
    Coverage >= 0.95, 4,
    Coverage >= 0.90, 3,
    Coverage >= 0.85, 2,
    1
)
```

### `Promptness Score`
Average team-lead rating.

```dax
Promptness Score =
AVERAGE ( 'Team Lead Assessment'[Promptness] )
```

### `Activity in Channels Score`
Average team-lead rating.

```dax
Activity in Channels Score =
AVERAGE ( 'Team Lead Assessment'[Activity in the Channels] )
```

### `Willingness to Help Score`
Average team-lead rating.

```dax
Willingness to Help Score =
AVERAGE ( 'Team lead assessment'[Willingness to Help] )
```

### `Responsiveness Score`
Average team-lead rating.

```dax
Responsiveness Score =
AVERAGE (
    'Team lead assessment'[Responsiveness]
)
```

### `Flexibility Score`
Average team-lead rating.

```dax
Flexibility Score =
AVERAGE (
    'Team lead assessment'[Flexibility]
)
```

### `Organizational Skills Score`
Average team-lead rating (mapped to 'Participation' in the weights table).

```dax
Organizational Skills Score =
AVERAGE (
    'Team lead assessment'[Organizational Skills]
)
```


## Section KPIs

Weighted score per section, using the Analyst or Senior weight column depending on the analyst's KPI framework. The 'Normalized' versions divide by the weight of KPIs that actually have a score, so a missing KPI doesn't drag the result down.

### `Spreading Skills KPI`
Weighted sum of Spreading Skills scores.

```dax
Spreading Skills KPI =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
RETURN
SUMX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Spreading Skills"
    ),
    VAR _KPI =
    'KPI Weights'[KPI]
    VAR _Weight =
    IF (
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    )
    VAR _Score =
    SWITCH (
        _KPI,
        "Spreads Done Daily",
        CALCULATE (
            [Spreads Done Daily],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Spreading Accuracy",
        CALCULATE (
            [Spreading Accuracy],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Processing Time",
        CALCULATE (
            [Processing Time],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Acceptance Time",
        CALCULATE (
            [Acceptance Time],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Rejected Spreads",
        CALCULATE (
            [Rejected Spreads],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Reports Checked Daily",
        CALCULATE (
            [Reports Checked Daily],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Metadata Done Daily",
        CALCULATE (
            [Metadata Done Daily],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        BLANK()
    )
    RETURN
    IF (
        NOT ISBLANK ( _Score )
        && _Weight > 0,
        _Score * _Weight,
        BLANK()
    )
)
```

### `Spreading Skills KPI Normalized`
Spreading Skills KPI divided by the weight of KPIs that have data.

```dax
Spreading Skills KPI Normalized =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
VAR _ApplicableWeight =
SUMX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Spreading Skills"
    ),
    VAR _KPI =
    'KPI Weights'[KPI]
    VAR _Weight =
    IF (
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    )
    VAR _Score =
    SWITCH (
        _KPI,
        "Spreads Done Daily",
        CALCULATE (
            [Spreads Done Daily],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Spreading Accuracy",
        CALCULATE (
            [Spreading Accuracy],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Processing Time",
        CALCULATE (
            [Processing Time],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Acceptance Time",
        CALCULATE (
            [Acceptance Time],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Rejected Spreads",
        CALCULATE (
            [Rejected Spreads],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Reports Checked Daily",
        CALCULATE (
            [Reports Checked Daily],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        "Metadata Done Daily",
        CALCULATE (
            [Metadata Done Daily],
            REMOVEFILTERS ( 'KPI Weights' )
        ),
        BLANK()
    )
    RETURN
    IF (
        NOT ISBLANK ( _Score ),
        _Weight,
        0
    )
)
RETURN
DIVIDE (
    [Spreading Skills KPI],
    _ApplicableWeight
)
```

### `Organizational KPI`
Weighted sum of Organizational scores.

```dax
Organizational KPI =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
RETURN
SUMX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Organizational"
    ),
    VAR _KPI =
    'KPI Weights'[KPI]
    VAR _Weight =
    IF (
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    )
    VAR _Score =
    SWITCH (
        _KPI,
        "Responsiveness", [Responsiveness Score],
        "Flexibility", [Flexibility Score],
        "Working Hours", [Working Hours Score],
        "Participation", [Organizational Skills Score],
        BLANK()
    )
    RETURN
    _Score * _Weight
)
```

### `Organizational KPI Normalized`
Organizational KPI re-scaled to the weights that apply.

```dax
Organizational KPI Normalized =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
VAR _WeightedScore =
SUMX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Organizational"
    ),
    VAR _KPI =
    'KPI Weights'[KPI]
    VAR _Weight =
    IF (
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    )
    VAR _Score =
    SWITCH (
        _KPI,
        "Responsiveness", [Responsiveness Score],
        "Flexibility", [Flexibility Score],
        "Working Hours", [Working Hours Score],
        "Participation", [Organizational Skills Score],
        BLANK()
    )
    RETURN
    IF (
        NOT ISBLANK ( _Score ),
        _Score * _Weight
    )
)
VAR _ApplicableWeight =
SUMX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Organizational"
    ),
    VAR _KPI =
    'KPI Weights'[KPI]
    VAR _Weight =
    IF (
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    )
    VAR _Score =
    SWITCH (
        _KPI,
        "Responsiveness", [Responsiveness Score],
        "Flexibility", [Flexibility Score],
        "Working Hours", [Working Hours Score],
        "Participation", [Organizational Skills Score],
        BLANK()
    )
    RETURN
    IF (
        NOT ISBLANK ( _Score ),
        _Weight
    )
)
RETURN
DIVIDE (
    _WeightedScore,
    _ApplicableWeight
)
```

### `Engagement KPI`
Weighted sum of Engagement scores.

```dax
Engagement KPI =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
RETURN
SUMX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Engagement"
    ),
    VAR _KPI =
    'KPI Weights'[KPI]
    VAR _Weight =
    IF (
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    )
    VAR _Score =
    SWITCH (
        _KPI,
        "Promptness", [Promptness Score],
        "Activity in Channels", [Activity in Channels Score],
        "Willingness to Help", [Willingness to Help Score],
        BLANK()
    )
    RETURN
    _Score * _Weight
)
```

### `Engagement KPI Normalized`
Engagement KPI re-scaled to the weights that apply.

```dax
Engagement KPI Normalized =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
VAR _Promptness =
[Promptness Score]
VAR _Activity =
[Activity in Channels Score]
VAR _Willingness =
[Willingness to Help Score]
VAR _PromptnessWeight =
CALCULATE (
    IF (
        _Framework = "Senior",
        MAX ( 'KPI Weights'[Senior Weight] ),
        MAX ( 'KPI Weights'[Analyst Weight] )
    ),
    'KPI Weights'[Section] = "Engagement",
    'KPI Weights'[KPI] = "Promptness"
)
VAR _ActivityWeight =
CALCULATE (
    IF (
        _Framework = "Senior",
        MAX ( 'KPI Weights'[Senior Weight] ),
        MAX ( 'KPI Weights'[Analyst Weight] )
    ),
    'KPI Weights'[Section] = "Engagement",
    'KPI Weights'[KPI] = "Activity in Channels"
)
VAR _WillingnessWeight =
CALCULATE (
    IF (
        _Framework = "Senior",
        MAX ( 'KPI Weights'[Senior Weight] ),
        MAX ( 'KPI Weights'[Analyst Weight] )
    ),
    'KPI Weights'[Section] = "Engagement",
    'KPI Weights'[KPI] = "Willingness to Help"
)
VAR _WeightedScore =
IF (
    NOT ISBLANK ( _Promptness ),
    _Promptness * _PromptnessWeight,
    0
)
+
IF (
    NOT ISBLANK ( _Activity ),
    _Activity * _ActivityWeight,
    0
)
+
IF (
    NOT ISBLANK ( _Willingness ),
    _Willingness * _WillingnessWeight,
    0
)
VAR _ApplicableWeight =
IF (
    NOT ISBLANK ( _Promptness ),
    _PromptnessWeight,
    0
)
+
IF (
    NOT ISBLANK ( _Activity ),
    _ActivityWeight,
    0
)
+
IF (
    NOT ISBLANK ( _Willingness ),
    _WillingnessWeight,
    0
)
RETURN
DIVIDE (
    _WeightedScore,
    _ApplicableWeight
)
```


## Final KPI

### `Final KPI`
Combines every section's KPIs in one pass: pulls scores and weights per KPI, drops KPIs with no score or zero weight, and divides by the weight that applies.

```dax
Final KPI =
VAR _Analyst =
SELECTEDVALUE(Analysts[Analyst])
VAR _Framework =
SELECTEDVALUE(Analysts[KPI Framework])
VAR _KPITable =
ADDCOLUMNS(
    'KPI Weights',
    "@Weight",
    IF(
        _Framework = "Senior",
        'KPI Weights'[Senior Weight],
        'KPI Weights'[Analyst Weight]
    ),
    "@Score",
    VAR _KPI = 'KPI Weights'[KPI]
    RETURN
    SWITCH(
        _KPI,
        /* Spreading Skills */
        "Spreads Done Daily",
        [Spreads Done Daily],
        "Spreading Accuracy",
        [Spreading Accuracy],
        "Processing Time",
        [Processing Time],
        "Acceptance Time",
        [Acceptance Time],
        "Rejected Spreads",
        [Rejected Spreads],
        //"Essential Errors",
        //[Essential Errors Score],
        "Reports Checked Daily",
        [Reports Checked Daily],
        "Metadata Done Daily",
        [Metadata Done Daily],
        /* Engagement */
        "Promptness",
        IFERROR(
            VALUE(
                SELECTEDVALUE(
                    'Team lead assessment'[Promptness]
                )
            ),
            BLANK()
        ),
        "Activity in Channels",
        IFERROR(
            VALUE(
                SELECTEDVALUE(
                    'Team lead assessment'[Activity in the Channels]
                )
            ),
            BLANK()
        ),
        "Willingness to Help",
        IFERROR(
            VALUE(
                SELECTEDVALUE(
                    'Team lead assessment'[Willingness to Help]
                )
            ),
            BLANK()
        ),
        /* Organizational */
        "Responsiveness",
        IFERROR(
            VALUE(
                SELECTEDVALUE(
                    'Team lead assessment'[Responsiveness]
                )
            ),
            BLANK()
        ),
        "Flexibility",
        IFERROR(
            VALUE(
                SELECTEDVALUE(
                    'Team lead assessment'[Flexibility]
                )
            ),
            BLANK()
        ),
        "Working Hours",
        [Working Hours Score],
        "Participation",
        IFERROR(
            VALUE(
                SELECTEDVALUE(
                    'Team lead assessment'[Organizational skills]
                )
            ),
            BLANK()
        ),
        BLANK()
    )
)
VAR _ApplicableKPIs =
FILTER(
    _KPITable,
    NOT ISBLANK([@Score])
    && [@Weight] > 0
)
VAR _WeightedScore =
SUMX(
    _ApplicableKPIs,
    [@Score] * [@Weight]
)
VAR _ApplicableWeight =
SUMX(
    _ApplicableKPIs,
    [@Weight]
)
RETURN
IF(
    ISBLANK(_Analyst) || ISBLANK(_Framework),
    BLANK(),
    DIVIDE(
        _WeightedScore,
        _ApplicableWeight
    )
)
```

### `Team Final KPI`
Average of Final KPI across analysts.

```dax
Team Final KPI =
AVERAGEX (
    VALUES ( Analysts[Analyst] ),
    CALCULATE ( [Final KPI] )
)
```

### `Debug Spreading KPI`
Helper that lists each Spreading Skills KPI score as text, for troubleshooting.

```dax
Debug Spreading KPI =
VAR _Framework =
SELECTEDVALUE ( Analysts[KPI Framework] )
RETURN
CONCATENATEX (
    FILTER (
        'KPI Weights',
        'KPI Weights'[Section] = "Spreading Skills"
    ),
    VAR _KPI = 'KPI Weights'[KPI]
    VAR _Score =
    SWITCH (
        _KPI,
        "Spreads Done Daily", [Spreads Done Daily],
        "Spreading Accuracy", [Spreading Accuracy],
        "Processing Time", [Processing Time],
        "Acceptance Time", [Acceptance Time],
        "Rejected Spreads", [Rejected Spreads],
        "Reports Checked Daily", [Reports Checked Daily],
        "Metadata Done Daily", [Metadata Done Daily],
        BLANK()
    )
    RETURN
    _KPI & "=" &
    IF (
        ISBLANK ( _Score ),
        "BLANK",
        FORMAT ( _Score, "0.00" )
    ),
    " | "
)
```
