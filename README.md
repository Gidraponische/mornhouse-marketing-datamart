# mornhouse-marketing-datamart
Marketing Datamart &amp; Campaign Profitability Analysis 
# Marketing Datamart & Campaign Profitability Analysis

## 1. Анализ исходных данных (Data Profiling)

Для построения витрины использован датасет `mornhouse-test-environment.test_app_dataset` за июнь–июль 2026 года. Исходные данные распределены по 4 таблицам:

* **`cost_table`** — расходы на рекламу (`cost_usd`, `date`, `media_source`, `campaign_id`).
* **`ad_revenue_raw`** — доход от показа рекламы внутри приложения (`ad_revenue_usd`, `date`, `media_source`, `campaign_id`).
* **`in_app_events_report`** — доход от покупок и подписок внутри приложения (`in_app_revenue_usd`, `date`, `media_source`, `campaign_id`).
* **`non_org_installs_report`** — количество рекламных установок приложения (`installs_count`, `date`, `media_source`, `campaign_id`).

---

## 2. Логика объединения данных (Data Modeling)

Чтобы объединить данные из четырех независимых источников без искажений и дублей, применены следующие решения:

* **Зерно агрегации (Grain):** Все показатели сводятся к единому уровню детализации: **`date` + `media_source` + `campaign_id`**.
* **Стандартизация текстовых полей:** Названия источников `media_source` приведены к нижнему регистру и очищены от лишних пробелов с помощью `LOWER(TRIM(...)) AS media_source`.
* **Предварительная агрегация в CTE:** Каждая из 4 таблиц предварительно агрегируется по зерну в отдельных CTE-блоках. 
* **Тип соединения (FULL OUTER JOIN):** Использован `FULL OUTER JOIN` с конструкцией `USING (date, media_source, campaign_id)`. Это гарантирует сохранение данных, когда в определенный день есть расходы, но нет доходов (или наоборот).

---

## 3. Формулы ключевых метрик

* **Total Revenue** = `COALESCE(ad_revenue, 0) + COALESCE(in_app_revenue, 0)`
* **Profit** = `Total Revenue - COALESCE(cost, 0)`
* **ROAS** = `Total Revenue / NULLIF(cost, 0)`
