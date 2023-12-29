
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
