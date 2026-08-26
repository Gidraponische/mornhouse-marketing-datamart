# mornhouse-marketing-datamart
# Marketing Datamart & Campaign Profitability Analysis

## 1. Исходные данные

Для анализа прибыли использовались 4 таблицы за июнь–июль 2026 года:

* **`cost_table`** — расходы на рекламу.
* **`ad_revenue_raw`** — доход от показа рекламы в приложении.
* **`in_app_events_report`** — доход от встроенных покупок и подписок.
* **`non_org_installs_report`** — количество установок.

---

## 2. Как связывались данные
* **Единый уровень деталей:** Все данные сгруппированы по **дате, источнику (`media_source`) и кампании (`campaign_id`)**.
* **Очистка данных:** Названия рекламных сетей приведены к одному регистру, чтобы избежать расхождений в написании.
* **Предварительная группировка:** Сначала посчитали суммы расходов и доходов отдельно по каждой таблице, а затем объединили их.
* **Сопоставление:** Таблицы соединены так, чтобы соотнести затраты на рекламу с выручкой за каждый день.

---

## 3. Ключевые метрики

* **Выручка (`Total Revenue`)** = `Доход от рекламы + Доход от покупок`
* **Прибыль (`Profit`)** = `Выручка - Расходы`
* **Окупаемость (`ROAS`)** = `Выручка / Расходы`

---


## 4. SQL-код 

```sql
#1. Расходы
WITH costs AS (
  SELECT
    PARSE_DATE('%Y-%m-%d', CAST(date AS STRING)) AS date,
    LOWER(TRIM(media_source)) AS media_source,
    TRIM(CAST(campaign_id AS STRING)) AS campaign_id,
    SUM(cost_usd) AS cost_usd,
    SUM(clicks) AS clicks,
    SUM(impressions) AS impressions
  FROM `mornhouse-test-environment.test_app_dataset.cost_table`
  GROUP BY 1, 2, 3
),

#2. Агрегируем установки
installs AS (
  SELECT
    DATE(install_date) AS date,
    LOWER(TRIM(media_source)) AS media_source,
    TRIM(CAST(campaign_id AS STRING)) AS campaign_id,
    COUNT(install_date) AS installs
  FROM `mornhouse-test-environment.test_app_dataset.non_org_installs_report`
  WHERE install_date IS NOT NULL
  GROUP BY 1, 2, 3
),

#3. Доход от рекламы
ad_revenue AS (
  SELECT
    DATE(TIMESTAMP(event_date)) AS date,
    LOWER(TRIM(media_source)) AS media_source,
    TRIM(CAST(campaign_id AS STRING)) AS campaign_id,
    SUM(event_revenue_usd) AS ad_revenue_usd
  FROM `mornhouse-test-environment.test_app_dataset.ad_revenue_raw`
  GROUP BY 1, 2, 3
),

#4. Доход от покупок/подписок
in_app_revenue AS (
  SELECT
    DATE(TIMESTAMP(event_date)) AS date,
    LOWER(TRIM(media_source)) AS media_source,
    TRIM(CAST(campaign_id AS STRING)) AS campaign_id,
    SUM(event_revenue_usd) AS in_app_revenue_usd
  FROM `mornhouse-test-environment.test_app_dataset.in_app_events_report`
  GROUP BY 1, 2, 3
)

#5. Итоговое объединение
SELECT
  COALESCE(c.date, i.date, ar.date, iar.date) AS date,
  COALESCE(c.media_source, i.media_source, ar.media_source, iar.media_source) AS media_source,
  COALESCE(c.campaign_id, i.campaign_id, ar.campaign_id, iar.campaign_id) AS campaign_id,
  IFNULL(c.cost_usd, 0) AS cost_usd,
  IFNULL(c.clicks, 0) AS clicks,
  IFNULL(c.impressions, 0) AS impressions,
  IFNULL(i.installs, 0) AS installs,
  IFNULL(ar.ad_revenue_usd, 0) AS ad_revenue_usd,
  IFNULL(iar.in_app_revenue_usd, 0) AS in_app_revenue_usd,
  (IFNULL(ar.ad_revenue_usd, 0) + IFNULL(iar.in_app_revenue_usd, 0)) AS total_revenue_usd,
  ((IFNULL(ar.ad_revenue_usd, 0) + IFNULL(iar.in_app_revenue_usd, 0)) - IFNULL(c.cost_usd, 0)) AS profit_usd,
  SAFE_DIVIDE((IFNULL(ar.ad_revenue_usd, 0) + IFNULL(iar.in_app_revenue_usd, 0)), c.cost_usd) AS roas
FROM costs c
FULL OUTER JOIN installs i 
  ON c.date = i.date AND c.media_source = i.media_source AND c.campaign_id = i.campaign_id
FULL OUTER JOIN ad_revenue ar 
  ON COALESCE(c.date, i.date) = ar.date 
  AND COALESCE(c.media_source, i.media_source) = ar.media_source 
  AND COALESCE(c.campaign_id, i.campaign_id) = ar.campaign_id
FULL OUTER JOIN in_app_revenue iar 
  ON COALESCE(c.date, i.date, ar.date) = iar.date 
  AND COALESCE(c.media_source, i.media_source, ar.media_source) = iar.media_source 
  AND COALESCE(c.campaign_id, i.campaign_id, ar.campaign_id) = iar.campaign_id
ORDER BY date DESC, cost_usd DESC;
```

---
## 5. Дашборд
## [https://datastudio.google.com/s/rjnWCi8Gjis](url)


## 6. Ключевые выводы и рекомендации

* **Общая окупаемость трафика:**
  За период июнь–июль 2026 года закупка трафика показала отличную рентабельность. При общих затратах в **$20,575.41** совокупная выручка составила **$71,254.26**, чистая прибыль — **$50,678.85**

* **Топ 3 кампании:**
(ROAS = Total Revenue / Cost)
  ## 1. `998337322` — чистая прибыль **$5,921.97** (ROAS ~367%)
  ## 2. `140410898` — чистая прибыль **$4,892.54** (ROAS ~265%)
  ## 3. `161694703` — чистая прибыль **$3,809.89** (ROAS ~221%)

* **Масштаб привлечения:**
  За анализируемый период было привлечено **777,724 установок**. Ключевой источник конверсий — `googleadwords_int`.

---
## **Рекомендации:**
`Пересмотреть бюджет на кампании с окупаемостью около 100% и перенаправить высвободившиеся деньги на плавное масштабирование топ 3 кампаний лидеров.
На короткой дистанции в 2 месяца это единственное действие, которое увеличит профит без объемной истории данных`.
  

