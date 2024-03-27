#### identify and handle null values
```sql
SELECT *
FROM `proyecto3-riesgorelativo.riesgorelativo.user_info` 
WHERE last_month_salary IS NULL
```
#### table join
```sql
SELECT *
FROM `proyecto3-riesgorelativo.riesgorelativo.user_info` AS a
LEFT JOIN `proyecto3-riesgorelativo.riesgorelativo.default` AS b
ON a.user_id = b.user_id
```
#### count null values
```sql
SELECT
COUNT(CASE WHEN default_flag = 0 THEN 1 END) AS cantidad_ceros,
COUNT(CASE WHEN default_flag = 1 THEN 1 END) AS cantidad_unos
FROM `proyecto3-riesgorelativo.riesgorelativo.user_flags` 
  WHERE last_month_salary IS NULL
```
```sql
SELECT
COUNT(CASE WHEN default_flag = 0 THEN 1 END) AS cantidad_ceros,
COUNT(CASE WHEN default_flag = 1 THEN 1 END) AS cantidad_unos
FROM `proyecto3-riesgorelativo.riesgorelativo.user_flags` 
  WHERE number_dependents IS NULL
```
#### replace null values with 0
```sql
SELECT
  * EXCEPT (last_month_salary, number_dependents),
  IFNULL(last_month_salary, 0) AS last_month_salary,
  IFNULL(number_dependents, 0) AS number_dependents
FROM
  `proyecto3-riesgorelativo.riesgorelativo.user_info`;
```
#### duplicate values
```sql
SELECT COUNT(*)
FROM `proyecto3-riesgorelativo.riesgorelativo.user_info` 
GROUP BY user_id
HAVING COUNT(*) > 1
```
#### correlation between two variables
```sql
SELECT
CORR(more_90_days_overdue, number_times_delayed_payment_loan_30_59_days) AS correlacion,
CORR(more_90_days_overdue, number_times_delayed_payment_loan_60_89_days) AS correlacion_2,
CORR(number_times_delayed_payment_loan_30_59_days, number_times_delayed_payment_loan_60_89_days) AS correlacion_3
FROM `proyecto3-riesgorelativo.riesgorelativo.loans_ detail`
```
#### change of values
```sql
SELECT
  loan_id, user_id,
  CASE
    WHEN LOWER(loan_type) IN ('other', 'others') THEN 'other'
    ELSE LOWER(loan_type)
  END AS loan_types
FROM
  `proyecto3-riesgorelativo.riesgorelativo.loans_outstanding
```
#### tukey's range test
```sql
WITH TukeyOutliers AS (
  SELECT
    last_month_salary,
    IF(last_month_salary < Q1 - 1.5 * RIC OR last_month_salary > Q3 + 1.5 * RIC, true, false) AS es_outlier
  FROM (
    SELECT
      last_month_salary,
      PERCENTILE_CONT(last_month_salary, 0.25) OVER () AS Q1,
      PERCENTILE_CONT(last_month_salary, 0.75) OVER () AS Q3,
      (PERCENTILE_CONT(last_month_salary, 0.75) OVER () - PERCENTILE_CONT(last_month_salary, 0.25) OVER ()) * 1.5 AS RIC
    FROM
      `proyecto3-riesgorelativo.riesgorelativo.user_flags`
  )
)

SELECT
  *
FROM
  TukeyOutliers
WHERE
  es_outlier = true;
```
#### create new variables
```sql
SELECT
user_id,
COUNT(*) AS total_loans,
COUNT(CASE WHEN loan_types = 'real estate' THEN 1 END) AS real_loans,
COUNT(CASE WHEN loan_types = 'other' THEN 1 END) AS other_loans
FROM
proyecto3-riesgorelativo.riesgorelativo.loan_type_lower
GROUP BY
user_id;
```
```sql
SELECT
user_id,
age,
CAST(last_month_salary AS INT64) AS last_month_salary, -- Utilizando CAST para cambiar a INTEGER
number_dependents,
default_flag,
CASE
WHEN age BETWEEN 21 AND 30 THEN 'joven'
WHEN age BETWEEN 31 AND 60 THEN 'adulto'
WHEN age >= 61 THEN 'adulto mayor'
ELSE 'otro'
END AS grupo_edad
FROM
proyecto3-riesgorelativo.riesgorelativo.user_flags;
```
#### relative risk calculation
```sql
WITH cuartiles AS (
  SELECT
    age,
    last_month_salary,
    number_dependents,
    total_loans,
    real_loans,
    other_loans,
    more_90_days_overdue,
    using_lines_not_secured_personal_assets,
    debt_ratio,
    default_flag,
    NTILE(4) OVER (ORDER BY age) AS cuartil_age,
    NTILE(4) OVER (ORDER BY last_month_salary) AS cuartil_salary,
    NTILE(4) OVER (ORDER BY number_dependents) AS cuartil_dependents,
		NTILE(4) OVER (ORDER BY total_loans) AS cuartil_loans,
		NTILE(4) OVER (ORDER BY real_loans) AS cuartil_real,
		NTILE(4) OVER (ORDER BY other_loans) AS cuartil_other,
		NTILE(4) OVER (ORDER BY more_90_days_overdue) AS cuartil_overdue,
		NTILE(4) OVER (ORDER BY using_lines_not_secured_personal_assets) AS cuartil_lines,
		NTILE(4) OVER (ORDER BY debt_ratio) AS cuartil_debt,
    -- Agrega NTILE para otras variables según sea necesario
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_age AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_age,
    COUNTIF(default_flag = 0) AS flag_0_age
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_salary AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_salary,
    COUNTIF(default_flag = 0) AS flag_0_salary
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_dependents AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_dependents,
    COUNTIF(default_flag = 0) AS flag_0_dependents
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_total_loans AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_total_loans,
    COUNTIF(default_flag = 0) AS flag_0_total_loans
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_real AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_real,
    COUNTIF(default_flag = 0) AS flag_0_real
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_other AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_other,
    COUNTIF(default_flag = 0) AS flag_0_other
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_overdue AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_overdue,
    COUNTIF(default_flag = 0) AS flag_0_overdue
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_lines AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_lines,
    COUNTIF(default_flag = 0) AS flag_0_lines
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
calculo_debt AS (
  SELECT
    COUNTIF(default_flag = 1) AS flag_1_debt,
    COUNTIF(default_flag = 0) AS flag_0_debt
  FROM
    `proyecto3-riesgorelativo.riesgorelativo.tabla_final_limpia`
),
SELECT
  'age' AS variable,
  cuartil_age AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_age AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_age AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_age) / (COUNTIF(default_flag = 0) / flag_0_age)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_age
GROUP BY
  variable, cuartil_age, flag_1_age, flag_0_age

UNION ALL



SELECT
  'last_month_salary' AS variable,
  cuartil_salary AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_salary AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_salary AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_salary) / (COUNTIF(default_flag = 0) / flag_0_salary)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_salary
GROUP BY
  variable, cuartil_salary, flag_1_salary, flag_0_salary

UNION ALL



SELECT
  'total_loans' AS variable,
  cuartil_loans AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_total_loans AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_total_loans AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_total_loans) / (COUNTIF(default_flag = 0) / flag_0_total_loans)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_total_loans
GROUP BY
  variable, cuartil_loans, flag_1_total_loans, flag_0_total_loans

UNION ALL

SELECT
  'real_loans' AS variable,
  cuartil_real AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_real AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_real AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_real) / (COUNTIF(default_flag = 0) / flag_0_real)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_real
GROUP BY
  variable, cuartil_real, flag_1_real, flag_0_real

UNION ALL

SELECT
  'other_loans' AS variable,
  cuartil_other AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_other AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_other AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_other) / (COUNTIF(default_flag = 0) / flag_0_other)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_other
GROUP BY
  variable, cuartil_other, flag_1_other, flag_0_other

UNION ALL



SELECT
  'more_90_days_overdue' AS variable,
  cuartil_overdue AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_overdue AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_overdue AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_overdue) / (COUNTIF(default_flag = 0) / flag_0_overdue)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_overdue
GROUP BY
  variable, cuartil_overdue, flag_1_overdue, flag_0_overdue

UNION ALL


SELECT
  'using_lines_not_secured_personal_assets' AS variable,
  cuartil_lines AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_lines AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_lines AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_lines) / (COUNTIF(default_flag = 0) / flag_0_lines)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_lines
GROUP BY
  variable, cuartil_lines, flag_1_lines, flag_0_lines

UNION ALL


SELECT
  'number_dependents' AS variable,
  cuartil_dependents AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_dependents AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_dependents AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_dependents) / (COUNTIF(default_flag = 0) / flag_0_dependents)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_dependents
GROUP BY
  variable, cuartil_dependents, flag_1_dependents, flag_0_dependents

UNION ALL


SELECT
  'debt_ratio' AS variable,
  cuartil_debt AS cuartil,
  COUNTIF(default_flag = 1) / flag_1_debt AS tasa_1,
  COUNTIF(default_flag = 0) / flag_0_debt AS tasa_0,
  CASE
    WHEN COUNTIF(default_flag = 0) = 0 THEN NULL
    ELSE (COUNTIF(default_flag = 1) / flag_1_debt) / (COUNTIF(default_flag = 0) / flag_0_debt)
  END AS riesgo_relativo
FROM
  cuartiles
CROSS JOIN
  calculo_debt
GROUP BY
  variable, cuartil_debt, flag_1_debt, flag_0_debt
ORDER BY
  variable, cuartil;
```













