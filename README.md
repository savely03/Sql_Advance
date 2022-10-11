# SQL Advance
## Задачи:
* Расчет ARPU на основании продаж в марте 2022 года (только по клиентам бонусной системы)
* Расчет среднего Lifetime клиента (только по клиентам бонусной системы)
* Расчет LTV на основании рассчитанных ARPU и Lifetime
* Расчет показателя Rolling Retention в разрезе когорт по бонусной программе
* Проведение ABC-анализа
___ 
## Инофрмация о таблицах:
* BonusCheques - информация о бонусных чеках (выгрузка из бонусного процессинга)
* Sales - информация о продажах (выгрузка из кассовой программы)
* Employee - информация о сотрудниках
* Shops - информация об аптеках
* Discounts - информация о скидках
___  
### Данные для подключения:
* Хост – 89.208.197.76 
* Порт – 5432
* Логин – student 
* Пароль –  qweasd963 
* База данных – apteka 
___
```sql
--ARPU--
select sum(summ_with_disc) / count(distinct card) as arpu
    from bonuscheques b 
    where to_char(datetime, 'YYYY-MM') = '2022-03'
        and card like '2000%'

```
___
```sql
--Lifetime--
with lt as (
    select card,
           extract(days from (max(datetime) - min(datetime))) as diff
        from bonuscheques b 
        where card like '2000%' 
        group by card
        having min(datetime) < now() - interval '1 month'
)
select avg(diff)
    from lt

```
___
```sql
--LTV--
select 
	card,
	extract(days from (max(datetime) - min(datetime))) as diff
from bonuscheques b 
where card like '2000%' 
group by card
having min(datetime) < now() - interval '1 month'
), 
lifetime as (
	select avg(diff)/30 as lifetime
	from lt
),
arpu as (
	select sum(summ_with_disc)/count(distinct card) as arpu
	from bonuscheques b 
	where to_char(datetime, 'YYYY-MM') = '2022-03'
	and card like '2000%'
)
select lifetime*arpu as ltv
from lifetime, arpu

```
```sql
--Rolling Retention--

with view as (
	select card, 
		   min(datetime) as joined_dt
		from bonuscheques b 
		where card like '2000%'
		group by card
), transaction_type as (
	select to_char(joined_dt, 'YYYY-MM') as yw,
		   a.card,
		   date(entry_dt) - date(joined_dt) as diff
		from view a 
		join (select card, datetime as entry_dt
			      from bonuscheques) b
			on (a.card = b.card)
), transaction as (
	select a.yw, 
		  round(100.0 * count(distinct case when diff >= 1 then card end) / day0, 2) as day1,
		  round(100.0 * count(distinct case when diff >= 3 then card end) / day0, 2) as day3,
		  round(100.0 * count(distinct case when diff >= 7 then card end) / day0, 2) as day7,
		  round(100.0 * count(distinct case when diff >= 14 then card end) / day0, 2) as day14,
		  round(100.0 * count(distinct case when diff >= 30 then card end) / day0, 2) as day30,
		  round(100.0 * count(distinct case when diff >= 60 then card end) / day0, 2) as day60,
		  round(100.0 * count(distinct case when diff >= 90 then card end) / day0, 2) as day90
		from transaction_type a 
		join (select yw,
		             count(distinct case when diff = 0 then card end) as day0
			  	from transaction_type
			  	group by yw) b
			on (a.yw = b.yw)
		group by a.yw, day0
)	
select *
	from transaction
```

```sql
--ABC Analysis--
	
with view1 as (
	select dr_ndrugs, 
	   sum(dr_kol) as amount,
	   sum(dr_kol * dr_croz - dr_sdisc) as revenue
	from sales
	group by dr_ndrugs
)
select dr_ndrugs,
	   amount,
	   revenue,
	   case when sum(amount) over (order by amount desc) / sum(amount) over() <= 0.8 then 'A' 
	   when sum(amount) over (order by amount desc) / sum(amount) over() <= 0.95 then 'B'
	   else 'C' end as amount_abc, 
   	   case when sum(revenue) over (order by revenue desc) / sum(revenue) over() <= 0.8 then 'A' 
	   when sum(revenue) over (order by revenue desc) / sum(revenue) over() <= 0.95 then 'B'
	   else 'C' end as revenue_abc 
	from view1
```