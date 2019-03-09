---
layout: default
title: Gaps and Islands
parent: Articles
nav_order: 6
---

# Solving 'Gaps-and-Islands' for Continuous Medication Usage
{: .no_toc }

## Contents
{: .no_toc }

1. TOC
{:toc}

# Introduction
A common problem found in population health or quality analytics is the need to calculate the duration of an episode when the episode itself is defined by multiple consecutive events. This is further complicated by the inconsistent and usually poor quality of healthcare billing and clinical data. 

Examples of instances where this type of problem must be solved are:
- Determining periods of uninterrupted insurance coverage
- Determining periods of medication supply when the supply for the period is made up of multiple prescriptions (as in opioid usage where prescriptions are limited to 30 days at a time)
- Determining periods of hospice stay (where the stay is billed monthly)

The following example solves a problem of medication usage over a period of time. In this problem, it is necessary to determine if the patient had a period of 90 days of continuous usage of the medication at any point during the measurement period. If a prescription is reordered or refilled before the previous supply is exhausted, assume that the patient continues to use the original supply until exhausted before starting on the new prescription. 


This solution can be easily modified to solve any of the other scenarios presented above.

# Solution
## Set up a test data set
The test data contains two patients. Each row in the data set represents one prescription for the medication. A prescription has a start date and a quantity of days supplied. The end date is calculated from the start date by adding days supplied.

```sql
-- Create a sample data set
DROP TABLE IF EXISTS #prescriptions
CREATE TABLE #prescriptions
    (
        [patient_id] VARCHAR(40),
        [prescription_start_date] DATE,   -- Prescription Start Date
        [days_supplied] INT,      -- Days Supplied
        [prescription_end_date] DATE,    -- Prescription End Date
        [row_order] BIGINT        -- Output of a ROW_NUMBER() function that orders the prescriptions chronologically by start date
    );
INSERT INTO #prescriptions
VALUES
('Z12345', N'2014-05-03T00:00:00', 10, N'2014-05-12T00:00:00', 1),
('Z12345', N'2014-05-11T00:00:00', 60, N'2014-07-09T00:00:00', 2),
('Z12345', N'2014-05-30T00:00:00', 30, N'2014-06-28T00:00:00', 3),
('Z12345', N'2014-07-18T00:00:00', 30, N'2014-08-16T00:00:00', 4),
('Z12345', N'2014-08-10T00:00:00', 30, N'2014-09-08T00:00:00', 5),
('Z12345', N'2014-11-14T00:00:00', 30, N'2014-12-13T00:00:00', 6),
( 'M6789', N'2017-07-22T00:00:00', 6, N'2017-07-27T00:00:00', 1 ), 
( 'M6789', N'2017-07-27T00:00:00', 5, N'2017-07-31T00:00:00', 2 );

```
## Recursively adjust the usage dates of the medication

Next, a recursive CTE is used to find any prescriptions that overlap, changing the start date of the subsequent prescription to be the day after the end date of the previous prescription. This simulates the assumption that the patient exhausts the previous supply before beginning to use the next prescription.

For more information on recursive CTEs, resources like the following are sufficient and don't need to be repeated here.

https://www.essentialsql.com/recursive-ctes-explained/

https://www.mssqltips.com/sqlservertip/5379/sql-server-common-table-expressions-cte-usage-and-examples/

```sql
WITH cte_med
AS
    (	
        SELECT 
            #prescriptions.patient_id,
            #prescriptions.prescription_start_date,
            #prescriptions.days_supplied,
            #prescriptions.prescription_end_date,
            #prescriptions.row_order
        FROM 
            #prescriptions
        WHERE
            #prescriptions.row_order = 1
        UNION ALL
        SELECT 
            cort.patient_id,
            fx.prescription_start_date ,
            cort.days_supplied,
            DATEADD(d,cort.days_supplied-1,fx.prescription_start_date) prescription_end_date,
            cort.row_order
        FROM 
            cte_med INNER JOIN
            #prescriptions cort ON cort.patient_id = cte_med.patient_id
                                AND cort.row_order = cte_med.row_order + 1 OUTER APPLY
            (
                SELECT
                    CAST(CASE WHEN cte_med.prescription_end_date >= cort.prescription_start_date 
                              THEN DATEADD(d,1,cte_med.prescription_end_date) 
                              ELSE cort.prescription_start_date end 
                        AS date) prescription_start_date
            ) fx
    ),
```

`row_order` is an integer field that counts up from 1 for each patient from the first prescription to the last. In this segment, the anchor record for the recursion is the first prescription for each patient `row_order = 1`. Each iteration of the recursion seeks the next prescription for that patient and checks for an overlap between the end date of the prior prescription and the start date of the next prescription.

In cases where the start date of the next prescription is less than or equal to the next prescription, a new start date is calculated. This occurs in the `fx` OUTER APPLY, which is reused in the field list of the SELECT clause to output the new start date. A new end date for the prescription is calculated by adding the days supplied to the new start date. Because the start date of the prescription is considered the first date of use, 1 day is subtracted from the DATEADD formula to arrive at a correct end date for the prescription. `DATEADD(d,cort.days_supplied-1,fx.prescription_start_date)`

The result of this CTE is an updated list of the medication orders, with revised start/end dates for each prescription that reflect the assumed actual usage of the medication by the patient.

|patient_id	|prescription_start_date	|days_supplied	|prescription_end_date	|row_order  |
|:----------|:--------------------------|:--------------|:----------------------|:----------|
|Z12345	    |2014-05-03	                |10	            |2014-05-12	            |1          |
|Z12345	    |2014-05-13	                |60	            |2014-07-11	            |2          |
|Z12345	    |2014-07-12	                |30	            |2014-08-10	            |3          |
|Z12345	    |2014-08-11	                |30	            |2014-09-09	            |4          |
|Z12345	    |2014-09-10	                |30	            |2014-10-09	            |5          |
|Z12345	    |2014-11-14	                |30	            |2014-12-13	            |6          |
|M6789	    |2017-07-22	                |6	            |2017-07-27	            |1          |
|M6789	    |2017-07-28	                |5	            |2017-08-01	            |2          |

## Identify 'islands' of continuous medication supply

From here, the rest of the problem is a classic gaps-and-islands problems In this case, the goal is to identify the islands and their duration in days. This is solved by a second recursive CTE. The first step is to add a set of columns to store the island start/end dates as the recursion progresses. There are more succint ways to do this than an intermediate CTE, but this is easier to understand.

```sql
add_pseudo AS 
    (	-- Set up a couple of pseudo columns for the next step
        SELECT 
            cte_med.patient_id,
            cte_med.prescription_start_date,
            cte_med.days_supplied,
            cte_med.prescription_end_date,
            cte_med.prescription_start_date island_start,
            cte_med.prescription_end_date  island_end,
            cte_med.row_order
        FROM 
            cte_med
    ),
```
Two columns are added `island_start` and `island_end`. These columns are initially populated with the start/end date of the prescirption for that row. Next, a recursive CTE computes the islands. An island is a period of unbroken daily medication supply.

```sql
cte_islands AS
    (	
        SELECT 
            cte_med.patient_id,
            cte_med.prescription_start_date,
            cte_med.days_supplied,
            cte_med.prescription_end_date,
            cte_med.island_start,
            cte_med.island_end,
            cte_med.row_order
        FROM 
            add_pseudo cte_med
        WHERE
            cte_med.row_order = 1 
        UNION ALL
        SELECT 
            next_record.patient_id,
            next_record.prescription_start_date,
            next_record.days_supplied,
            next_record.prescription_end_date,
            CASE WHEN DATEDIFF(d,start_record.island_end,next_record.prescription_start_date) = 1 THEN start_record.island_start ELSE next_record.prescription_start_date END island_start,
            next_record.prescription_end_date island_end,
            next_record.row_order
        FROM 
            cte_islands start_record INNER JOIN 
            add_pseudo next_record ON next_record.patient_id = start_record.patient_id and
                                    next_record.row_order = start_record.row_order + 1
    )
```

For this recursive CTE, the anchor is again the first prescription (chronologically) for each patient. If the next prescription in the chronological sequence starts the day after the end of the previous prescription, then the `island_start` of the subsequent prescription is set to the start_date of the previous prescription. The output of this recursion is still a single row for each prescription, but the final row in each island will contain the full duration of that island. 

|patient_id	|prescription_start_date    |days_supplied	|prescription_end_date	|island_start	|island_end	|row_order|
|:----------|:--------------------------|:--------------|:----------------------|:--------------|:----------|:--------|
|Z12345	    |2014-05-03	                |10	            |2014-05-12	            |2014-05-03	    |2014-05-12	|1|
|Z12345	    |2014-05-13	                |60	            |2014-07-11	            |2014-05-03	    |2014-07-11	|2|
|Z12345	    |2014-07-12	                |30	            |2014-08-10	            |2014-05-03	    |2014-08-10	|3|
|Z12345	    |2014-08-11	                |30	            |2014-09-09	            |2014-05-03	    |2014-09-09	|4|
|Z12345	    |2014-09-10	                |30	            |2014-10-09	            |2014-05-03	    |2014-10-09	|5|
|Z12345	    |2014-11-14	                |30	            |2014-12-13	            |2014-11-14	    |2014-12-13	|6|
|M6789	    |2017-07-22	                |6	            |2017-07-27	            |2017-07-22	    |2017-07-27	|1|
|M6789	    |2017-07-28	                |5	            |2017-08-01	            |2017-07-22	    |2017-08-01	|2|

## Output the results

From here, it is trivial to answer a number of questions about the medication supply for the patient. In this case, the question asked is whether the patient had, at any time, 90 unbroken days of medication supply. Add 1 to the output of the DATEDIFF function to get a true count of medication days.

```sql
select
	patient_id,
	max(DATEDIFF(DAY,island_start,island_end)+1) max_duration
from
	cte_islands
group by
	patient_id
```
Output: 
|patient_id	|max_duration|
|:----------|:-----------|
|Z12345	    |160|
|M6789	    |11|