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

SELECT sql_id, sql_text
FROM dba_hist_sqltext
WHERE sql_text like '%CONTRACT%' and sql_text like '%PAYGO%'
ORDER BY sql_id DESC;
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

# - insert contract values by AF, and PAYG
```
delete from WKSP_ODPDEALS.CONTRACT;

insert into WKSP_ODPDEALS.CONTRACT C
(reg_id, CONTRACT_TYPE, START_DATE, END_DATE, CONTRACT_AMOUNT, USED_AMOUNT)
SELECT
        T1.REG_ID,
        'AF' as contract_type,
        T1.START_DATE,
        END_DATE,
        replace(FLEX_CREDITS_PURCHASED, ',', '') as FLEX_CREDITS_PURCHASED,
        replace(USED_AMOUNT, ',', '') as USED_AMOUNT
    FROM
        WKSP_ODPDEALS.DM12_EXIST10 T1
    WHERE
        T1.START_DATE != 'PAYGO';

insert into WKSP_ODPDEALS.CONTRACT C
(reg_id, CONTRACT_TYPE)
SELECT
        T1.REG_ID,
        'PAYGO' as contract_type
    FROM
        WKSP_ODPDEALS.DM12_EXIST10 T1
    WHERE
        T1.START_DATE = 'PAYGO';

MERGE INTO WKSP_ODPDEALS.CONTRACT C
USING (
    SELECT
        T1.REG_ID,
        T1.START_DATE,
        END_DATE,
        replace(FLEX_CREDITS_PURCHASED, ',', '') as FLEX_CREDITS_PURCHASED,
        replace(USED_AMOUNT, ',', '') as USED_AMOUNT
    FROM
        WKSP_ODPDEALS.DM1_EXIST5 T1
    WHERE
        T1.START_DATE != 'PAYGO'
) SRC ON ( C.REG_ID = SRC.REG_ID )
WHEN MATCHED THEN UPDATE
SET C.START_DATE = SRC.START_DATE,
    C.END_DATE = SRC.END_DATE,
    C.CONTRACT_AMOUNT = to_number(replace(SRC.FLEX_CREDITS_PURCHASED, '-', '')),
    C.USED_AMOUNT = to_number(replace(SRC.USED_AMOUNT, '-', ''));

MERGE INTO WKSP_ODPDEALS.CONTRACT C
USING (
    SELECT
        T1.REG_ID,
        'PAYGO' as CONTRACT_TYPE
    FROM
        WKSP_ODPDEALS.DM1_EXIST5 T1
    WHERE
        T1.START_DATE = 'PAYGO'
) SRC ON ( C.REG_ID = SRC.REG_ID )
WHEN MATCHED THEN UPDATE
SET C.START_DATE = null,
    C.END_DATE = null,
    C.CONTRACT_AMOUNT = 0,
    C.USED_AMOUNT = 0
```

# - select 7 consumption dates from the last date
```
SELECT
    LISTAGG(''''
            || date_series
            || '''', ',')
FROM
    (SELECT 
to_char(start_date + LEVEL - 1, 'YYYY-MM-DD') AS date_series
FROM (SELECT min(consumption_date) AS start_date, 
             max(consumption_date) AS end_date
      FROM wksp_odpdeals.consumptions)
CONNECT BY LEVEL <= end_date - start_date + 1);
```

# - insert opportunity information to history table, and insert up to date ones
```

INSERT INTO WKSP_ODPDEALS.OPPORTUNITY_HISTORY
(oppty_id, OPPTY_NAME, FISCAL_QUARTER, Pipeline, REVENUE_TYPE, SALES_STAGE, FORECAST_TYPE_GROUP, WORKLOAD_NAME, PARTNER_NAME, REG_ID, UPDATE_DATE)
select oppty_id, OPPTY_NAME, FISCAL_QUARTER, Pipeline, REVENUE_TYPE, SALES_STAGE, FORECAST_TYPE_GROUP, WORKLOAD_NAME, PARTNER_NAME, REG_ID, sysdate as UPDATE_DATE
from WKSP_ODPDEALS.OPPORTUNITY;

delete WKSP_ODPDEALS.OPPORTUNITY;

INSERT INTO WKSP_ODPDEALS.OPPORTUNITY (
    OPPTY_ID,
    REG_ID,
    OPPTY_NAME,
    FISCAL_QUARTER,
    PIPELINE,
    REVENUE_TYPE,
    SALES_STAGE,
    FORECAST_TYPE_GROUP,
    WORKLOAD_NAME
    ,PARTNER_NAME
)
    SELECT
        DO.OPPORTUNITY_ID,
        DO.REGISTRY_ID,
        max(DO.OPPORTUNITY_NAME) as OPPORTUNITY_NAME,
        --max(DO.FISCAL_QUARTER) as FISCAL_QUARTER,
        max(DO.FISCAL_WEEK) as FISCAL_QUARTER,
        max(replace(DO."Pipeline (k$)", '-', '')) as pipeline,
        max(DO.REVENUE_TYPE) as REVENUE_TYPE,
        max(DO.SALES_STAGE) as SALES_STAGE,
        max(DO.FORECAST_TYPE_GROUP) as FORECAST_TYPE_GROUP,
        '' as WORKLOAD_NAME
        ,max(DO.PARTNER_NAME) as PARTNER_NAME
    FROM
        WKSP_ODPDEALS.DM1_OPPTY10 DO
     --where do.revenue_type != 'EXISTINGCOMMIT' and do.revenue_type != 'NEW' and do.revenue_type = 'EXPANDCOMMIT'
    --WHERE DO.FORECAST_TYPE_GROUP != 'Won'
    group by DO.OPPORTUNITY_ID,
        DO.REGISTRY_ID
   --     WHERE NOT EXISTS (SELECT 1 FROM WKSP_ODPDEALS.OPPORTUNITY o where do.opportunity_id = o.OPPTY_ID)

select DO.OPPORTUNITY_ID,
        DO.REGISTRY_ID from WKSP_ODPDEALS.DM1_OPPTY10 DO
        --where do.revenue_type != 'EXISTINGCOMMIT' and do.revenue_type != 'NEW' and do.revenue_type = 'EXPANDCOMMIT'
    group by DO.OPPORTUNITY_ID,
        DO.REGISTRY_ID HAVING COUNT (DO.OPPORTUNITY_ID) > 1 order by 2;
```


# - query current and history opportunities
```
select 'current' as status, o.oppty_id, oppty_name, fiscal_quarter, pipeline, revenue_type, sales_stage, FORECAST_TYPE_GROUP, workload_name, partner_name, null as update_date from wksp_odpdeals.OPPORTUNITY o, wksp_odpdeals.account a where a.reg_id = o.reg_id and o.id = 'A4RYC58'
union all
select 'history' as status, o.oppty_id, oppty_name, fiscal_quarter, pipeline, revenue_type, sales_stage, FORECAST_TYPE_GROUP, workload_name, partner_name, update_date from wksp_odpdeals.OPPORTUNITY_history o,  wksp_odpdeals.account a where a.reg_id = o.reg_id and o.id = 'A4RYC58' order by update_date
```

