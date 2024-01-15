
1. Общее кол-во задач - Кол-во стори, которые хоть раз переводились в статус In dev + в поле Owner хоть раз стоял разработчик Х.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; spender; owner;
        QUERY( { story_id \ ARRAYFORMULA(IF( spender=""; owner; spender )) }; 
        "SELECT Col2, COUNT(Col1) WHERE Col2 <>'' GROUP BY Col2 LABEL Col2 'developer', COUNT(Col1) 'story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, L, D WHERE AF > 0 "))))
```

2. Кол-во выполненных задач - сумма всех стори, которые находятся в статусе Completed + в поле Owner хоть раз стоял разработчик Х.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; spender; owner;
        QUERY( { story_id \ ARRAYFORMULA(IF( spender=""; owner; spender )) }; 
        "SELECT Col2, COUNT(Col1) WHERE Col2 <>'' GROUP BY Col2 LABEL Col2 'developer', COUNT(Col1) 'story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, L, D WHERE AF > 0 AND AE = 'Completed' "))))
```

3. Отклонения первичной оценки трудозатрат от фактических трудозатрат - среднее значение по всем стори разницы между первично выставленным значением в поле Estimate и последним выставленным значением в поле Actual dev.
4. Отклонения вторичной оценки трудозатрат от фактических трудозатрат - среднее значение по всем стори разницы между вторым выставленным значением в поле Estimate и последним выставленным значением в поле Actual dev.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; owner; actual_dev; estimate_first_value; estimate_second_value;
        QUERY( {story_id \ owner \ actual_dev \ 
        ARRAYFORMULA(IF( estimate_first_value > 0; actual_dev - estimate_first_value; 0 ))\
        ARRAYFORMULA(IF( estimate_second_value > 0; actual_dev - estimate_second_value; 0 ))};
        "SELECT Col2, AVG(Col4), AVG(Col5) GROUP BY Col2 LABEL Col2 'developer', AVG(Col4) 'avg_first_estimate_deviation', AVG(Col5) 'avg_second_estimate_deviation' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3));
    TRANSPOSE(INDEX(startTable;4));
    TRANSPOSE(INDEX(startTable;5)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, D, U, AA, AB WHERE D <> ''"))))
```

5. Кол-во итераций тестирования - среднее значение кол-ва переносов стори в статус In QA / кол-ва выставления лейбла QA Rejected в пулле.
6. Кол-во итераций ревью - среднее значение кол-ва переносов стори в статус Ready for review / кол-ва запросов изменений в пулле от ревьюеров.
```
=LAMBDA( table; 
    QUERY( table; "SELECT Col2, AVG(Col3), AVG(Col4) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_pulls_qa_rejected_count', AVG(Col4) 'avg_pulls_reviewer_rejected_count' ")
)
(LAMBDA( startTable; 
    LAMBDA( story_id; owner; pulls_qa_rejected_count; pulls_reviewer_rejected_count;
        QUERY({ story_id \ owner 
         \ ARRAYFORMULA(IF(pulls_qa_rejected_count > 0; pulls_qa_rejected_count;0))
         \ ARRAYFORMULA(IF(pulls_reviewer_rejected_count > 0; pulls_reviewer_rejected_count;0))}
        ;"SELECT Col1, Col2, SUM(Col3), SUM(Col4) GROUP BY Col1, Col2")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;5));
    TRANSPOSE(INDEX(startTable;6)))  
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, D, G, E, J, K WHERE AF > 0 ")))))
```

7. Трудозатраты на тестирование относительно трудозатрат на разработку - среднее значение соотношения значений поля Actual QA к значениям поля Actual dev.
8. Трудозатраты на ревью относительно трудозатрат на разработку - среднее значение соотношения значений поля Actual review к значениям поля Actual dev.
```
=LAMBDA( startTable;
    LAMBDA( notNullTable;
        LAMBDA( owner; story_id; actual_dev ; actual_qa ; actual_review ;
            QUERY({ owner \ story_id \
            ARRAYFORMULA(IF(actual_qa > actual_dev; 1 ; 
                IF(actual_qa = 0; 0; actual_qa / actual_dev))) \
            ARRAYFORMULA(IF(actual_review > actual_dev; 1 ; 
                IF(actual_review = 0; 0; actual_review / actual_dev))) } 
            ; "SELECT Col1, AVG(Col3), AVG(Col4) GROUP BY Col1 LABEL Col1 'developer', AVG(Col3) 'avg_qa_part', AVG(Col4) 'avg_review_part' "; 1)
        )(
        TRANSPOSE(INDEX(notNullTable;1));
        TRANSPOSE(INDEX(notNullTable;2));
        TRANSPOSE(INDEX(notNullTable;3));
        TRANSPOSE(INDEX(notNullTable;4));
        TRANSPOSE(INDEX(notNullTable;5)))
    )(
    LAMBDA( owner; story_id; actual_dev ; actual_qa ; actual_review ; 
        TRANSPOSE(QUERY( { owner \ story_id \ 
        ARRAYFORMULA(IF(actual_dev > 0; actual_dev; 0)) \
        ARRAYFORMULA(IF(actual_qa > 0; actual_qa; 0)) \
        ARRAYFORMULA(IF(actual_review > 0; actual_review; 0)) }
        ; "SELECT * WHERE Col1 <> ''"))
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3));
    TRANSPOSE(INDEX(startTable;4));
    TRANSPOSE(INDEX(startTable;5)))) 
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT D, A, U, T, V "))))
```

9. Срок выполнения задач - среднее значение кол-ва дней между первым переносом задачи в In dev и переносом задачи в Completed.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; owner; story_completed; first_move_to_in_development;
        QUERY({ story_id \ owner \ 
        ARRAYFORMULA(IFERROR( TO_DATE(LEFT(story_completed; 19)) - TO_DATE(LEFT(first_move_to_in_development; 19)); "") )}
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_time_on_story'")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3));
    TRANSPOSE(INDEX(startTable;4)))  
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, D, AJ, AK WHERE D <> '' AND AJ <> '' "))))
```

10. Общее кол-во задач на ревью - Кол-во стори, в которых разработчик Х был проставлен в поле Reviewer.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; reviewer; 
        QUERY({story_id \ reviewer}
        ; "SELECT Col2, COUNT(Col1) GROUP BY Col2 LABEL Col2 'developer', COUNT(Col1) 'reviewed_story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, R WHERE R <> '' "))))
```

11. Срок ожидания ревью в очереди - среднее значение кол-ва дней, которое стори находится в статусе Ready for review + в поле Reviewer выставлен разработчик Х.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; reviewer; total_days_ready_for_review;
        QUERY({story_id \ reviewer \ ARRAYFORMULA(IF( total_days_ready_for_review = ""; 0; FLOOR(total_days_ready_for_review))) }
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_waiting_review' " )
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, R, AQ WHERE R <> '' "))))
```

12. Трудозатраты на ревью - среднее значение списаний разработчика Х в поле Acrual review.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; actual_review_spendings_member; actual_review_spendings_total_hours;
        QUERY({story_id \ actual_review_spendings_member \ actual_review_spendings_total_hours}
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_spandings_for_review' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, N, O WHERE N <> '' "))))
```

1. QA Общее кол-во задач - кол-во стори, которые хоть раз переводились в статус In dev + в поле QA хоть раз стоял тестировщик Х.
```
=LAMBDA( startTable; 
    LAMBDA( story_id; spender; 
        QUERY({ story_id \ spender }; "SELECT Col2, COUNT(Col1) GROUP BY Col2 LABEL Col2 'qa', COUNT(Col1) 'story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, AV WHERE AV <> '' "))))
```

2. QA Отклонение первичной оценки трудозатрат от фактических трудозатрат -среднее значение по всем стори разницы между первично выставленным значением в поле Estimate QA и последним выставленным значением в поле Actual QA.
```sql
=LAMBDA( startTable; 
    LAMBDA( story_id; qa; actual_qa; estimate_qa;
        QUERY({ story_id \ qa \ ARRAYFORMULA( actual_qa - estimate_qa ) }; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'qa', AVG(Col3) 'avg_deviation' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3));
    TRANSPOSE(INDEX(startTable;4)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, S, T, Z WHERE S <> '' "))))
```

3. QA Трудозатраты на тестирование относительно трудозатрат на разработку -  среднее значение соотношения значений поля Actual QA к значениям поля Actual dev.
```sql
=LAMBDA( startTable; 
    LAMBDA( story_id; qa; actual_qa; actual_dev;
        QUERY({ story_id \ qa \
        ARRAYFORMULA(IF(actual_dev = ""; 1; actual_qa / actual_dev ))}
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL AVG(Col3) 'avg_qa_part' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3));
    TRANSPOSE(INDEX(startTable;4)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, S, T, U WHERE S <> '' "))))
```

4. QA Кол-во итераций тестирования - среднее значение кол-ва переносов стори в статус In QA / кол-ва выставления лейбла QA Rejected в пулле.
```sql
=LAMBDA( startTable; 
    LAMBDA( story_id; qa; pulls_qa_rejected_count;
        QUERY(
            QUERY({ story_id \ qa \ ARRAYFORMULA(IF(pulls_qa_rejected_count =""; 0; pulls_qa_rejected_count))}
            ; "SELECT Col1, Col2, SUM(Col3) GROUP BY Col1, Col2")
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'qa', AVG(Col3) 'avg_pulls_qa_rejected_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, S, J WHERE S <> '' "))))
```
