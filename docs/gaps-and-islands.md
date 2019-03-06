---
layout: default
title: Gaps and Islands
nav_order: 6
---

'''sql
				WITH cte_med
				AS
					(	-- Recursive CTE that adjusts the date of service and prescription end date if there is an overlapping prescription prior
						-- Resucitvity is necessary to iterate chronologically through the prescriptions and adjust them in order.
						SELECT 
							#cortico.PAT_ID,
							#cortico.FIRST_LBP_DATE,
							#cortico.date_of_service,
							#cortico.DAYS_SUPPLIED,
							#cortico.PRESC_END_DATE,
							#cortico.row_order
						FROM 
							#cortico
						WHERE
							#cortico.row_order = 1
						UNION ALL
						SELECT 
							cort.PAT_ID,
							cort.FIRST_LBP_DATE,
							fx.date_of_service ,
							cort.DAYS_SUPPLIED,
							DATEADD(d,cort.DAYS_SUPPLIED-1,fx.date_of_service) PRESC_END_DATE,
							cort.row_order
						FROM 
							cte_med INNER JOIN
							#cortico cort ON cort.PAT_ID = cte_med.PAT_ID
											 AND cort.row_order = cte_med.row_order + 1 OUTER APPLY
							(
								SELECT
									CAST(CASE WHEN cte_med.PRESC_END_DATE >= cort.date_of_service THEN DATEADD(d,1,cte_med.PRESC_END_DATE) ELSE cort.date_of_service end AS date) date_of_service
							) fx
					),
				add_pseudo AS 
					(	-- Set up a couple of pseudo columns for the next step
						SELECT 
							cte_med.PAT_ID,
							cte_med.FIRST_LBP_DATE,
							cte_med.date_of_service,
							cte_med.DAYS_SUPPLIED,
							cte_med.PRESC_END_DATE,
							cte_med.date_of_service island_start,
							cte_med.PRESC_END_DATE  island_end,
							cte_med.row_order
						FROM 
							cte_med
					),
				cte_islands AS
					(	-- Now that we've adjusted for any overlapping prescirptions, this is a standard gaps and islands problem
						-- The output of this recursive query is still a row for wach prescription, but the last row in any consecutive
						-- set of prescriptions will have the start and end dates of the consecutive period in the island start and island end fields
						-- All the ais necessary is to take the max of the difference between these two columns to see if there were any periods
						-- where there were 90 or mor consecutive days.
						SELECT 
							cte_med.PAT_ID,
							cte_med.FIRST_LBP_DATE,
							cte_med.date_of_service,
							cte_med.DAYS_SUPPLIED,
							cte_med.PRESC_END_DATE,
							cte_med.island_start,
							cte_med.island_end,
							cte_med.row_order
						FROM 
							add_pseudo cte_med
						WHERE
							cte_med.row_order = 1 
						UNION ALL
						SELECT 
							next_record.PAT_ID,
							next_record.FIRST_LBP_DATE,
							next_record.date_of_service,
							next_record.DAYS_SUPPLIED,
							next_record.PRESC_END_DATE,
							CASE WHEN DATEDIFF(d,start_record.island_end,next_record.date_of_service) = 1 THEN start_record.island_start ELSE next_record.date_of_service END island_start,
							next_record.PRESC_END_DATE island_end,
							next_record.row_order
						FROM 
							cte_islands start_record INNER JOIN 
							add_pseudo next_record ON next_record.PAT_ID = start_record.PAT_ID and
												   next_record.row_order = start_record.row_order + 1

					)
				UPDATE pt
				SET CORTICOSTEROID_YN = (CASE WHEN consecutive_meds.longest_consecutive >= 90 THEN 1 END)
				FROM
					#patients pt LEFT OUTER join
                    (
						SELECT
							cte_islands.PAT_ID,
							MAX(DATEDIFF(d,cte_islands.island_start,cte_islands.island_end)+1) longest_consecutive
						FROM
							cte_islands
						GROUP by
							pat_id
					) consecutive_meds ON pt.pat_id = consecutive_meds.pat_id
