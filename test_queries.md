# - find a new account from the Excel upload table

- query
```
select de.* FROM
WKSP_ODPDEALS.DM1_EXIST11 de
where not exists (select 1 from wksp_odpdeals.account a where de.reg_id = a.reg_id)
and reg_id is not null
union all 
select de.* FROM
WKSP_ODPDEALS.DM2_EXIST11 de
where not exists (select 1 from wksp_odpdeals.account a where de.reg_id = a.reg_id)
and reg_id is not null
```

# - find specific query from query history
```
SELECT sql_id, sql_text
FROM v$sql
where sql_text like '%PIVOT%'
ORDER BY last_active_time DESC
FETCH FIRST 10000 ROWS ONLY;
```

# - pivot query to show rows' value(consumption by date)
```
SELECT
    *
FROM
    (
        SELECT
            TO_CHAR(C.CONSUMPTION_DATE, 'YYYY-MM-DD') AS CONSUMPTION_DATE,
            A.ACCOUNT_NAME,
            A.ID,
            C.REG_ID,
            A.MAPPED_CA,
            A.MAPPED_SALES_REP,
            CONSUMPTIONS
        FROM
            WKSP_ODPDEALS.CONSUMPTIONS C,
            WKSP_ODPDEALS.ACCOUNT      A
        WHERE
            C.REG_ID = A.REG_ID
    ) PIVOT (
        SUM(CONSUMPTIONS)
        FOR CONSUMPTION_DATE
        IN ( '2024-01-05',
        '2024-01-06',
        '2024-01-07',
        '2024-01-08',
        '2024-01-09',
        '2024-01-10',
        '2024-01-11',
        '2024-01-12' )
    )
ORDER BY
    6 DESC NULLS LAST;
```

