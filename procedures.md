# - Getting date variables from columns

- procedure 
```
create or replace PROCEDURE wksp_odpdeals.p_consumption_date( p_table IN VARCHAR2  )
IS
    v_sql VARCHAR2(1000);
    TYPE ref_cursor IS REF CURSOR;
    v_cur ref_cursor;
    v_col1 VARCHAR2(1000);
BEGIN
    v_sql := '
    SELECT
        LISTAGG(''"''
                || COLUMN_NAME
                || ''"'', '','')
    FROM
        ALL_TAB_COLUMNS
    WHERE
        TABLE_NAME = ''' || p_table  || '''
        AND COLUMN_NAME LIKE ''%/%'' ';

    OPEN v_cur FOR v_sql;
    FETCH v_cur INTO v_col1;
    CLOSE v_cur;

    DBMS_OUTPUT.PUT_LINE('consumption date: ' || v_col1 );
    --EXECUTE IMMEDIATE v_sql;
END;
```

- usage, test
```
-- test query
    SELECT
        LISTAGG('"'
                || COLUMN_NAME
                || '"', ',')
    FROM
        ALL_TAB_COLUMNS
    WHERE
        TABLE_NAME = 'DM12_EXIST10'
        AND COLUMN_NAME LIKE '%/%';
-- execute example
EXEC wksp_odpdeals.p_consumption_date('DM12_EXIST10');
```

# - Insert consumption data by date

- procedure 
```
create or replace PROCEDURE wksp_odpdeals.p_consumption_insert( p_table IN VARCHAR2  )
IS
    v_sql VARCHAR2(1000);
    v_sql2 VARCHAR2(1000);
    TYPE ref_cursor IS REF CURSOR;
    v_cur ref_cursor;
    v_col1 VARCHAR2(1000);
BEGIN
    v_sql := '
    SELECT
        LISTAGG(''"''
                || COLUMN_NAME
                || ''"'', '','')
    FROM
        ALL_TAB_COLUMNS
    WHERE
        TABLE_NAME = ''' || p_table  || '''
        AND COLUMN_NAME LIKE ''%/%'' ';

    OPEN v_cur FOR v_sql;
    FETCH v_cur INTO v_col1;
    CLOSE v_cur;

    v_sql2 := '
    INSERT INTO WKSP_ODPDEALS.CONSUMPTIONS
        SELECT
            REG_ID,
            TO_DATE(CONSUMPTION_DATE, ''mm/dd/yyyy'') AS CONSUMPTION_DATE,
            TO_NUMBER(CONSUMPTIONS)                 AS CONSUMPTIONS
        FROM
            WKSP_ODPDEALS.' || p_table || ' UNPIVOT ( CONSUMPTIONS
                FOR CONSUMPTION_DATE
            IN ( ' || v_col1 || ' ) )
        ORDER BY
            "REG_ID",
            CONSUMPTION_DATE';

    EXECUTE IMMEDIATE v_sql2;
END;
```

- usage, test
```
-- test query
INSERT INTO WKSP_ODPDEALS.CONSUMPTIONS
 SELECT
        REG_ID,
        TO_DATE(CONSUMPTION_DATE, 'mm/dd/yyyy') AS CONSUMPTION_DATE,
        TO_NUMBER(replace(CONSUMPTIONS, '-', ''))                 AS CONSUMPTIONS
    FROM
        WKSP_ODPDEALS.DM12_EXIST10 UNPIVOT ( CONSUMPTIONS
            FOR CONSUMPTION_DATE
        IN ( 
            "2/24/2024","2/25/2024","2/26/2024","2/27/2024","2/28/2024","2/29/2024","3/1/2024"
            ) )
        --where not exists (select 1 from WKSP_ODPDEALS.consumptions cc where cc.REG_ID = reg_id and cc.CONSUMPTION_DATE = CONSUMPTION_DATE)
    ORDER BY
        "REG_ID",
        CONSUMPTION_DATE
;
-- execute example
EXEC wksp_odpdeals.p_consumption_insert('DM12_EXIST10');
```
