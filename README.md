
### Developer  
```sql
SELECT DISTINCT
	owner as developer,
	count_table.all_story_count,
	completed_table.completed_story_count,
	estimat_table.avg_first_estimate_deviation,
	estimat_table.avg_second_estimate_deviation,
	rejected_table.avg_pulls_qa_rejected_count,
	rejected_table.avg_pulls_reviewer_rejected_count,
	part_table.avg_qa_part,
	part_table.avg_review_part,
	time_story_table.avg_time_on_story,
	reviewed_table.review_story_count,
	waiting_table.avg_wait_review,
	review_spandings_table.avg_spandings_for_review
FROM stats
LEFT JOIN(
	SELECT 
		developer, 
		COUNT(story_id) AS all_story_count
	FROM(
		SELECT DISTINCT 
		story_id,
		IIF(actual_dev_spendings_member == "", owner, actual_dev_spendings_member ) AS developer 
		FROM stats
		WHERE state_changes_to_in_development > 0
	)
	GROUP BY developer
) as count_table
ON count_table.developer == owner
LEFT JOIN(
	SELECT 
		developer, 
		COUNT(story_id) AS completed_story_count
	FROM(
		SELECT DISTINCT 
		story_id,
		IIF(actual_dev_spendings_member == "", owner, actual_dev_spendings_member ) AS developer
		FROM stats
		WHERE state == "Completed"
	)
	GROUP BY developer
) as completed_table
ON completed_table.developer == owner
LEFT JOIN(
	SELECT 
		owner as developer,
		avg( first_estimate_delta ) avg_first_estimate_deviation,
		avg( second_estimate_delta ) avg_second_estimate_deviation
	FROM ( -- большой разброс в значений
		SELECT DISTINCT
		story_id,
		owner,
		(actual_dev - estimate_first_value) as first_estimate_delta,
		(actual_dev - estimate_second_value) as second_estimate_delta
		FROM stats
	)
	GROUP BY owner
) as estimat_table
ON estimat_table.developer == owner
LEFT JOIN(
	SELECT  
		owner as developer,
		AVG(pulls_qa_rejected_count) AS avg_pulls_qa_rejected_count,
		AVG(pulls_reviewer_rejected_count) AS avg_pulls_reviewer_rejected_count
	FROM (
		SELECT 
		story_id,
		owner,
			SUM(pulls_qa_rejected_count) AS pulls_qa_rejected_count,
			SUM(pulls_reviewer_rejected_count) AS pulls_reviewer_rejected_count
		FROM (
			SELECT DISTINCT
			story_id, 
			owner,
			pulls_repository,
			pulls_pull_id,
			pulls_qa_rejected_count,
			pulls_reviewer_rejected_count
			FROM stats
		)
		GROUP BY story_id
	)
	GROUP BY owner
) as rejected_table
ON rejected_table.developer == owner
LEFT JOIN(
	SELECT   
		owner as developer,
		AVG(qa_part) AS avg_qa_part,
		AVG(review_part) AS avg_review_part
	FROM (
		SELECT 
		owner,
			story_id,
			IIF(actual_qa>actual_dev, 1 , 
				IIF(actual_qa == 0, 0, CAST(actual_qa AS REAL) / CAST(actual_dev AS REAL))
			) as qa_part,
			IIF(actual_review>actual_dev, 1 , 
				IIF(actual_review == 0, 0, CAST(actual_review AS REAL) / CAST(actual_dev AS REAL))
			) as review_part
		FROM (
			SELECT DISTINCT
			owner,
			story_id,
			actual_dev - 0 as actual_dev,
			actual_qa - 0 as actual_qa,
			actual_review - 0 as actual_review
			FROM stats
		)
	)
	GROUP BY owner
) as part_table
ON part_table.developer == owner
LEFT JOIN(
	SELECT 
		owner as developer,
		avg(time_on_story) as avg_time_on_story
	FROM (
		SELECT DISTINCT
			story_id,
			owner,
			JULIANDAY(first_move_to_in_development)- JULIANDAY(story_created_at) as time_on_story
		FROM stats
		WHERE owner != ""
	)
	GROUP BY owner
) as time_story_table
ON time_story_table.developer == owner
LEFT JOIN(
	SELECT 
		reviewer as developer,
		COUNT(story_id) as review_story_count
	FROM (
		SELECT DISTINCT
			story_id,
			reviewer
		FROM stats
		WHERE reviewer != ""
	)
	GROUP BY reviewer
) as reviewed_table
ON reviewed_table.developer == owner
LEFT JOIN( 
	SELECT 
		reviewer as developer,
		AVG(total_days_ready_for_review) as avg_wait_review
	FROM (
		SELECT DISTINCT
			story_id,
			total_days_ready_for_review,
			reviewer
		FROM stats
		WHERE reviewer != ""
	)
	GROUP BY reviewer
) as waiting_table
ON waiting_table.developer == owner
LEFT JOIN( 
	SELECT 
		actual_review_spendings_member as developer,
		AVG(actual_review_spendings_total_hours) avg_spandings_for_review
	FROM (
		SELECT DISTINCT
			story_id,
			actual_review_spendings_member,
			actual_review_spendings_total_hours
		FROM stats
		WHERE actual_review_spendings_member != ""
	)
	GROUP BY actual_review_spendings_member
) as review_spandings_table
ON review_spandings_table.developer == owner
```

### QA
```sql
SELECT DISTINCT
	qa,
	count_table.story_count,
	deviation_table.avg_deviation,
	qa_part_table.avg_qa_part,
	reject_table.avg_pulls_qa_rejected_count
FROM stats
LEFT JOIN(
	SELECT 
		actual_qa_spendings_member as tester,
		COUNT(story_id) as story_count
	FROM (
		SELECT DISTINCT
			story_id,
			actual_qa_spendings_member
		FROM stats
		WHERE actual_qa_spendings_member != "" AND state_changes_to_in_development > 0
	)
	GROUP BY actual_qa_spendings_member
) as count_table
ON count_table.tester == qa
LEFT JOIN(
	SELECT 
		qa as tester,
		AVG(deviation) as avg_deviation
	FROM (
		SELECT DISTINCT
			story_id,
			qa,
			actual_qa - estimate_qa as deviation
		FROM stats
		WHERE qa != "" AND actual_qa > 0
	)
	GROUP BY qa
) as deviation_table
ON deviation_table.tester == qa
LEFT JOIN(
	SELECT
		qa as tester,
		AVG(IIF(actual_dev == 0, 1, CAST(actual_qa AS REAL) / CAST(actual_dev AS REAL))) as avg_qa_part
	FROM (
		SELECT DISTINCT
			story_id,
			qa,
			actual_qa - 0 as actual_qa,
			actual_dev - 0 as actual_dev
		FROM stats
		WHERE qa != "" AND actual_qa > 0
	)
	GROUP BY qa
) as qa_part_table
ON qa_part_table.tester == qa
LEFT JOIN(
	SELECT 
		qa as tester,
		AVG(pulls_qa_rejected_count) as avg_pulls_qa_rejected_count
	FROM (
		SELECT 
			story_id,
			qa,
			SUM(pulls_qa_rejected_count) as pulls_qa_rejected_count
		FROM (
			SELECT DISTINCT
				story_id,
				qa,
				pulls_qa_rejected_count - 0 as pulls_qa_rejected_count
			FROM stats
			WHERE qa != ""
		)
		GROUP BY story_id
	)
	GROUP BY qa
) as reject_table
ON reject_table.tester == qa
WHERE qa != ""
```
