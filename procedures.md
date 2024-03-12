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
        LISTAGG(''''''''
                || COLUMN_NAME
                || '''''''', '','')
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
        LISTAGG(''''''''
                || COLUMN_NAME
                || '''''''', '','')
    FROM
        ALL_TAB_COLUMNS
    WHERE
        TABLE_NAME = 'DM12_EXIST10'
        AND COLUMN_NAME LIKE '%/%';
-- execute example
EXEC wksp_odpdeals.p_consumption_date('DM12_EXIST10');
```
