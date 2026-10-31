# Retail SQL Analytics
 
## SQL scripts for retail business analytics: cohort analysis, revenue dynamics, and margin optimization.


### 01_country_margin_drivers

* Denormalizing data into a 1NF flat table. Calculating gross profit and identifying the top-1 product category (key driver) by margin for each country.

```sql
with flat_table as (
	-- Denormalizing to 1NF flat table for metric calculations
	select 
		c."Country",
		s."Order Number",
		p."Category",
		(s."Quantity" * p."Unit Price USD") AS revenue,
    	(s."Quantity" * (p."Unit Price USD" - p."Unit Cost USD")) AS gr_profit
	from sales s
		left join products p 
			on s."ProductKey" = p."ProductKey"
		left join customers c  
			on s."CustomerKey" = c."CustomerKey"
),
	country_grain as (
	-- Aggregating metrics to Country grain
	select 
		"Country",
		count(distinct "Order Number") as orders,
		SUM(gr_profit) as gr_profit,
		SUM(gr_profit) / SUM(revenue) as marg_perc
	from flat_table 
	group  by "Country"
),
	top_Product as (
	-- Identifying the top-1 driver category by Gross Profit
	select 
		"Country",
		"Category",
		SUM(gr_profit),
		row_number () over (partition by "Country" order by SUM(gr_profit) DESC) as prod_rank
	from flat_table 
	GROUP BY "Country", "Category"
	)
	
	select 
		cg."Country",
		cg.orders,
		round(cg.gr_profit, 2) as "Gross Profit",
		round(cg.marg_perc * 100, 2) as "Margin %",
		tp."Category" as "Driver category"
	from country_grain cg
	left join top_Product tp 
		on cg."Country" = tp."Country"
	where cg.gr_profit > 500000 and prod_rank = 1
	order by cg.gr_profit desc;
```

### 02_mom_revenue_dynamics

* Business growth rate analysis. Calculating Month-over-Month (MoM) revenue dynamics using window functions.

```sql
with month_gr as (
	-- Aggregating daily revenue to monthly grain
	select 
		concat(extract(year from s."Order Date"), '-'  ,extract(month from s."Order Date")) as "Year-month", 
		(extract(year from s."Order Date") * 12) + extract(month from s."Order Date") as sort,
		SUM((s."Quantity" * p."Unit Price USD")) AS curr_rev
	from sales s 
	left join products p
		on s."ProductKey" = p."ProductKey"
	group by 1, 2
	order by 2
),
	prev as (
	-- Fetching previous month's revenue
	select 
	"Year-month",
	curr_rev,
	lag(curr_rev) over (order by sort) as lagged
	from month_gr
)

select 
	"Year-month",
	curr_rev as "Current revenue",
	lagged as "Previous revenue",
	ROUND(((curr_rev - lagged) / lagged) * 100, 2) AS "MoM%"
from prev;
```

### 03_cohort_retention_nrr

* Customer lifetime cohort analysis. Pivoting data into a wide matrix and calculating Net Retention Rate (NRR) as a percentage relative to the first purchase month.

```sql
with first_activity as (
	-- Identifying the first purchase month for each customer
		select
			"CustomerKey" as Cstmr_ID,
			date_trunc('month',min(s."Order Date")):: date as first_prch_M
		from sales s
		group by "CustomerKey"
),	
	lifetime_indx as (
	-- Calculating cohort lifetime index (months since first purchase)
		select
			fc.first_prch_M as Cohort_Month,
			(extract(year from s."Order Date") * 12 + extract(month from s."Order Date")) -
			(extract(year from fc.first_prch_M) * 12 + extract(month from fc.first_prch_M)) as Cohort_lifeme,
			SUM((s."Quantity" * p."Unit Price USD")) as revenue
		from sales s
		left join first_activity fc
			on s."CustomerKey" = fc.Cstmr_ID
		left join products p 
			on s."ProductKey" = p."ProductKey"
		group by fc.first_prch_M, 2
), 
	absolue_matrix as ( 
	-- Pivoting data into a wide matrix (Absolute values)
		select
			Cohort_Month,
			sum(case when Cohort_lifeme = 0 then revenue else 0 end) as M0,
			sum(case when Cohort_lifeme = 1 then revenue  else 0 end) as M1,
			sum(case when Cohort_lifeme = 2 then revenue  else 0 end) as M2,
			sum(case when Cohort_lifeme = 3 then revenue  else 0 end) as M3,
			sum(case when Cohort_lifeme = 6 then revenue  else 0 end) as M6,
			sum(case when Cohort_lifeme = 12 then revenue  else 0 end) as M12
		from lifetime_indx
		group by Cohort_Month
		order by Cohort_Month
)
	-- Calculating Net Retention Rate (NRR) % relative to Month 0
	select
		to_char(Cohort_Month, 'MM-YYYY') as Cohort_date,
		100.0 as "M0 %",
		ROUND((M1 / NULLIF(M0, 0)) * 100, 2) as "M1 %",
		ROUND((M2 / NULLIF(M0, 0)) * 100, 2) as "M2 %",
		ROUND((M3 / NULLIF(M0, 0)) * 100, 2) as "M3 %",
		ROUND((M6 / NULLIF(M0, 0)) * 100, 2) as "M6 %",
		ROUND((M12 / NULLIF(M0, 0)) * 100, 2) as "M12 %"
	from absolue_matrix
	order by Cohort_Month
```
## Data Source
The dataset used for this portfolio is the [Global Electronics Retailer](https://app.mavenanalytics.io/datasets?tag=Retail) provided by Maven Analytics.
