
1. Общее кол-во задач - Кол-во стори, которые хоть раз переводились в статус In dev + в поле Owner хоть раз стоял разработчик Х.
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

2. Кол-во выполненных задач - сумма всех стори, которые находятся в статусе Completed + в поле Owner хоть раз стоял разработчик Х.
```sql
SELECT developer, COUNT(story_id) AS completed_story_count
FROM(
    SELECT DISTINCT 
    story_id,
    IIF(actual_dev_spendings_member == "", owner ,actual_dev_spendings_member ) AS developer
    FROM stats
    WHERE state == "Completed"
)
GROUP BY developer
```

3. Отклонения первичной оценки трудозатрат от фактических трудозатрат - среднее значение по всем стори разницы между первично выставленным значением в поле Estimate и последним выставленным значением в поле Actual dev.
4. Отклонения вторичной оценки трудозатрат от фактических трудозатрат - среднее значение по всем стори разницы между вторым выставленным значением в поле Estimate и последним выставленным значением в поле Actual dev.
```sql
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
```

5. Кол-во итераций тестирования - среднее значение кол-ва переносов стори в статус In QA / кол-ва выставления лейбла QA Rejected в пулле.
6. Кол-во итераций ревью - среднее значение кол-ва переносов стори в статус Ready for review / кол-ва запросов изменений в пулле от ревьюеров.
```sql
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
```

7. Трудозатраты на тестирование относительно трудозатрат на разработку - среднее значение соотношения значений поля Actual QA к значениям поля Actual dev.
8. Трудозатраты на ревью относительно трудозатрат на разработку - среднее значение соотношения значений поля Actual review к значениям поля Actual dev.
```sql
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
```

9. Срок выполнения задач - среднее значение кол-ва дней между первым переносом задачи в In dev и переносом задачи в Completed.
```sql
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
```

10. Общее кол-во задач на ревью - Кол-во стори, в которых разработчик Х был проставлен в поле Reviewer.
```sql
SELECT 
    reviewer as developer,
    COUNT(story_id) as reviewed_story_count
FROM (
    SELECT DISTINCT
        story_id,
        reviewer
    FROM stats
    WHERE reviewer != ""
)
GROUP BY reviewer
```

11. Срок ожидания ревью в очереди - среднее значение кол-ва дней, которое стори находится в статусе Ready for review + в поле Reviewer выставлен разработчик Х.
```sql
SELECT 
	reviewer as developer,
	AVG(total_days_ready_for_review) as avg_waiting_review
FROM (
	SELECT DISTINCT
		story_id,
		total_days_ready_for_review,
		reviewer
	FROM stats
	WHERE reviewer != ""
)
GROUP BY reviewer
```

12. Трудозатраты на ревью - среднее значение списаний разработчика Х в поле Acrual review.
```sql
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
```

1. QA Общее кол-во задач - кол-во стори, которые хоть раз переводились в статус In dev + в поле QA хоть раз стоял тестировщик Х.
```sql
SELECT 
	actual_qa_spendings_member as qa,
	COUNT(story_id) as story_count
FROM (
	SELECT DISTINCT
		story_id,
		first_move_to_in_development,
		actual_qa_spendings_member
	FROM stats
	WHERE actual_qa_spendings_member != "" AND state_changes_to_in_development > 0
)
GROUP BY actual_qa_spendings_member
```

2. QA Отклонение первичной оценки трудозатрат от фактических трудозатрат -среднее значение по всем стори разницы между первично выставленным значением в поле Estimate QA и последним выставленным значением в поле Actual QA.
```sql
SELECT 
	qa,
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
```

3. QA Трудозатраты на тестирование относительно трудозатрат на разработку -  среднее значение соотношения значений поля Actual QA к значениям поля Actual dev.
```sql
SELECT
	qa,
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
```

4. QA Кол-во итераций тестирования - среднее значение кол-ва переносов стори в статус In QA / кол-ва выставления лейбла QA Rejected в пулле.
```sql
SELECT 
	qa,
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
```
