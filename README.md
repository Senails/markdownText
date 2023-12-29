
1.
```sql
    SELECT developer, COUNT(story_id) AS all_story_count
    FROM(
    	SELECT DISTINCT 
    	story_id,
    	IIF(actual_dev_spendings_member == "", owner ,actual_dev_spendings_member ) AS developer 
    	FROM stats
    	WHERE state_changes_to_in_development > 0
    )
    GROUP BY developer
```

2.
```sql
    SELECT developer, COUNT(story_id) AS story_count
    FROM(
    	SELECT DISTINCT 
    	story_id,
    	IIF(actual_dev_spendings_member == "", owner ,actual_dev_spendings_member ) AS developer
    	FROM stats
    	WHERE state=="Completed"
    )
    GROUP BY developer
```

3 и 4.

```sql
    SELECT 
    	AVG( first_estimate_delta ) avg_first_estimate_delta,
    	AVG( second_estimate_delta )avg_second_estimate_delta
    FROM ( -- большой разброс в значений
    	SELECT DISTINCT
    		story_id,
    		(estimate_first_value - actual_dev) as first_estimate_delta,
    		(estimate_second_value - actual_dev) as second_estimate_delta
    	FROM stats
    )
```

5 и 6.

```sql
    SELECT 
    	AVG(pulls_qa_rejected_count) AS avg_pulls_qa_rejected_count,
    	AVG(pulls_reviewer_rejected_count) AS avg_pulls_reviewer_rejected_count
    FROM (
    	SELECT 
    		story_id,
    		SUM(pulls_qa_rejected_count) AS pulls_qa_rejected_count,
    		SUM(pulls_reviewer_rejected_count) AS pulls_reviewer_rejected_count
    	FROM (
    		SELECT DISTINCT
    			story_id,
    			pulls_repository,
    			pulls_pull_id,
    			pulls_qa_rejected_count,
    			pulls_reviewer_rejected_count
    		FROM stats
    	)
    	GROUP BY story_id
    )
```

7 и 8.

```sql
    SELECT 
    	AVG(qa_part) AS avg_qa_part,
    	AVG(review_part) AS avg_review_part
    FROM (
    	SELECT 
    		story_id,
    		IIF(actual_qa>actual_dev, 1 , 
    			IIF(actual_qa == 0, 0, CAST(actual_qa AS REAL) / CAST(actual_dev AS REAL))
    		) as qa_part,
    		IIF(actual_review>actual_dev, 1 , 
    			IIF(actual_review == 0, 0, CAST(actual_review AS REAL) / CAST(actual_dev AS REAL))
    		) as review_part
    	FROM (
    		SELECT DISTINCT
    			story_id,
    			actual_dev - 0 as actual_dev,
    			actual_qa - 0 as actual_qa,
    			actual_review - 0 as actual_review
    		FROM stats
    	)
    )
```

9.
```
    SELECT 
    	AVG(story_completed_at - first_move_to_in_development) avg_time_in_dev
    FROM (
    	SELECT DISTINCT
    		story_id,
    		JULIANDAY(first_move_to_in_development) as first_move_to_in_development,
    		JULIANDAY(IIF( story_completed_at=="", datetime('now'), story_completed_at)) as story_completed_at
    	FROM stats
    )
```
