
### Developer  
```sql
=LAMBDA( 
     namesTable;
     count_table;
     completed_table;
     deviation_table;
     rejected_table;
     part_table;
     time_story_table;
     reviewed_table;
     waiting_table;
     review_spandings_table;
    {
     namesTable \
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( count_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'all_story_count' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( completed_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'completed_story_count' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( deviation_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_first_estimate_deviation' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( deviation_table ; "SELECT Col3 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_second_estimate_deviation' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( rejected_table ; "SELECT Col3 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_pulls_qa_rejected_count' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( rejected_table ; "SELECT Col3 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_pulls_reviewer_rejected_count' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( part_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_qa_part' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( part_table ; "SELECT Col3 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_review_part' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( time_story_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_time_on_story' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( reviewed_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'review_story_count' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( waiting_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_wait_review' ")\
     QUERY(MAP(namesTable; LAMBDA( name ; IFERROR(QUERY( review_spandings_table ; "SELECT Col2 WHERE Col1 = '" & name & "'"; 0); "") )); "SELECT * LABEL Col1 'avg_spandings_for_review' ")
    }
)(
LAMBDA( startTable; 
    LAMBDA( owner; actual_dev_spendings_member;
        UNIQUE({ 
            QUERY( owner; "SELECT * WHERE Col1 <> ''" ) ;
            QUERY( actual_dev_spendings_member ; "SELECT * WHERE Col1 <> '' OFFSET 1")
        })
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT D, L "))));

LAMBDA( startTable; 
    LAMBDA( story_id; spender; owner;
        QUERY( { story_id \ ARRAYFORMULA(IF( spender=""; owner; spender )) }; 
        "SELECT Col2, COUNT(Col1) WHERE Col2 <>'' GROUP BY Col2 LABEL Col2 'developer', COUNT(Col1) 'story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, L, D WHERE AF > 0 "))));

LAMBDA( startTable; 
    LAMBDA( story_id; spender; owner;
        QUERY( { story_id \ ARRAYFORMULA(IF( spender=""; owner; spender )) }; 
        "SELECT Col2, COUNT(Col1) WHERE Col2 <>'' GROUP BY Col2 LABEL Col2 'developer', COUNT(Col1) 'story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))    
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, L, D WHERE AF > 0 AND AE = 'Completed' "))));

LAMBDA( startTable; 
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
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, D, U, AA, AB WHERE D <> ''"))));

LAMBDA( table; 
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
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, D, G, E, J, K WHERE AF > 0 ")))));

LAMBDA( startTable;
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
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT D, A, U, T, V "))));

LAMBDA( startTable; 
    LAMBDA( story_id; owner; story_completed; first_move_to_in_development;
        QUERY({ story_id \ owner \ 
        ARRAYFORMULA(DATEDIF( LEFT(first_move_to_in_development; 10); LEFT(story_completed; 10); "D" ) )}
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_time_on_story'")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2));
    TRANSPOSE(INDEX(startTable;3));
    TRANSPOSE(INDEX(startTable;4)))  
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, D, AJ, AK WHERE D <> ''"))));

LAMBDA( startTable; 
    LAMBDA( story_id; reviewer; 
        QUERY({story_id \ reviewer}
        ; "SELECT Col2, COUNT(Col1) GROUP BY Col2 LABEL Col2 'developer', COUNT(Col1) 'reviewed_story_count' ")
    )( 
    TRANSPOSE(INDEX(startTable;1)); 
    TRANSPOSE(INDEX(startTable;2)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, R WHERE R <> '' "))));

LAMBDA( startTable; 
    LAMBDA( story_id; reviewer; total_days_ready_for_review;
        QUERY({story_id \ reviewer \ total_days_ready_for_review}
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_waiting_review' " )
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, R, AQ WHERE R <> '' "))));

LAMBDA( startTable; 
    LAMBDA( story_id; actual_review_spendings_member; actual_review_spendings_total_hours;
        QUERY({story_id \ actual_review_spendings_member \ actual_review_spendings_total_hours}
        ; "SELECT Col2, AVG(Col3) GROUP BY Col2 LABEL Col2 'developer', AVG(Col3) 'avg_spandings_for_review' ")
    )( 
    TRANSPOSE(INDEX(startTable;1));
    TRANSPOSE(INDEX(startTable;2)); 
    TRANSPOSE(INDEX(startTable;3)))   
)( TRANSPOSE(UNIQUE(QUERY(table!A:AX; "SELECT A, N, O WHERE N <> '' "))))
)
```

### QA
```sql
SELECT DISTINCT
	names.qa as qa,
	count_table.story_count,
	deviation_table.avg_deviation,
	qa_part_table.avg_qa_part,
	reject_table.avg_pulls_qa_rejected_count
FROM (
	SELECT qa
	FROM (
			SELECT qa FROM stats 
			UNION 
			SELECT actual_qa_spendings_member as qa FROM stats
	)
	WHERE qa != ""
) as names
LEFT JOIN (
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
ON count_table.tester = names.qa
LEFT JOIN (
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
ON deviation_table.tester = names.qa
LEFT JOIN (
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
ON qa_part_table.tester = names.qa
LEFT JOIN (
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
ON reject_table.tester = names.qa
WHERE names.qa != "";
```
