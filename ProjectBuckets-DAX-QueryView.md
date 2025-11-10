# Project Buckets Debugging in DAX Query View

This file contains the full DAX query broken into numbered sections for easier walkthrough and narration.

```DAX
-- 1. Wrap in EVALUATE
EVALUATE

-- 2. Define Buckets (Static Reference Table)
VAR Buckets =
    DATATABLE(
        "BucketName", STRING,
        "BucketOrder", INTEGER,
        {
            { "Recent Week", 1 },
            { "Last 2 Weeks", 2 },
            { "Last 4 Weeks", 3 },
            { "Current Month", 4 },
            { "Previous Month", 5 },
            { "Last Quarter", 6 },
            { "Last 2 Quarters", 7 },
            { "Current Year", 8 },
            { "Previous Year", 9 },
            { "Last 3 Years", 10 },
            { "Older Projects", 11 },
            { "Upcoming Projects", 12 }
        }
    )

RETURN
SELECTCOLUMNS(
    FILTER(

        -- 3. Cross Projects with Buckets
        CROSSJOIN(
            SELECTCOLUMNS(
                Projects,
                "Project", Projects[ProjectID],
                "BaseDate", Projects[ActualStartDateProper],
                "Status", Projects[Status]
            ),
            Buckets
        ),

        -- 4. Apply Filtering Logic
        VAR baseDate = [BaseDate]
        VAR bucketName = [BucketName]
        VAR projectstatus = [Status]
        RETURN
            (projectstatus = "Not Started" && bucketName = "Upcoming Projects")
            ||
            (projectstatus <> "Not Started" &&
                SWITCH(
                    TRUE(),
                    bucketName = "Recent Week", DATEDIFF(baseDate, TODAY(), DAY) <= 7,
                    bucketName = "Last 2 Weeks", DATEDIFF(baseDate, TODAY(), DAY) <= 14,
                    bucketName = "Last 4 Weeks", DATEDIFF(baseDate, TODAY(), DAY) <= 28,
                    bucketName = "Current Month", MONTH(baseDate) = MONTH(TODAY()) && YEAR(baseDate) = YEAR(TODAY()),
                    bucketName = "Previous Month", MONTH(baseDate) = MONTH(EDATE(TODAY(), -1)) && YEAR(baseDate) = YEAR(EDATE(TODAY(), -1)),
                    bucketName = "Last Quarter", DATEDIFF(baseDate, TODAY(), MONTH) <= 3,
                    bucketName = "Last 2 Quarters", DATEDIFF(baseDate, TODAY(), MONTH) <= 6,
                    bucketName = "Current Year", YEAR(baseDate) = YEAR(TODAY()),
                    bucketName = "Previous Year", YEAR(baseDate) = YEAR(TODAY()) - 1,
                    bucketName = "Last 3 Years", DATEDIFF(baseDate, TODAY(), YEAR) <= 3,
                    bucketName = "Older Projects", DATEDIFF(baseDate, TODAY(), YEAR) > 3,
                    FALSE
                )
            )
    ),

    -- 5. Return Final Table
    "Project", [Project],
    "BucketName", [BucketName],
    "BucketOrder", [BucketOrder]
)
