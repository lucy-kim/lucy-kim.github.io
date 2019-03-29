---
layout: page
title: MySQL tutorial using SQL Cookbook (O'Reilly)
description: This tutorial offers key essential lessons on how to use MySQL taught in SQL Cookbook (O'Reilly), along with instructions on how to set up MySQL on personal computer (Mac) and the example data for practice.
---

In this tutorial, I compiled key lessons from the [SQL Cookbook](http://shop.oreilly.com/product/9780596009762.do) (by Anthony Molinaro and published in O'Reilly) from the perspective of someone with nonzero but minimal _SQL_ experience. I offer instructions on how to set up _MySQL_ on a personal computer (Mac) and work through selected examples from the book, also discussing alternative solutions I came up with for some problems.

## Setup

### Requirements
- Macbook
- SQL Cookbook. O'Reilly Media.
	- They share a [code](https://resources.oreilly.com/examples/9780596009762) to create example data used in the book. I slightly modified their code to add some new datasets which were not already included and share my code [here](https://www.dropbox.com/s/tpar4hdprxekm6h/crsqlbook_data.txt?dl=0). Copy and paste this code into your MySQL prompter after starting it.

### How to set up MySQL and example data

#### Download MySQL
1. Go to the [MySQL download](https://dev.mysql.com/downloads/mysql/) website:
2. For Mac, download `macOS 10.14 (x86, 64-bit), DMG Archive`

If still unclear, check out video tutorial [how to install MySQL on Mac](https://www.youtube.com/watch?v=Tq0TXcH6dAU) by Linda.com.

#### Set up example data from the book on computer:
1. Turn on the MySQL server on Preference Pane
2. On Terminal, enter:
```
cd /usr/local/mysql/bin
```
2. Then enter:
```
./mysql -u root -p
```
3. Enter the password you chose
	- If this is the first time you use MySQL on Mac, check out for more detailed instructions on password video tutorial on [how to start a MySQL](https://www.youtube.com/watch?v=q9S51sykd1A) by rudolfson.junior.
3. Run code in [crsqlbook_data.txt](https://www.dropbox.com/s/tpar4hdprxekm6h/crsqlbook_data.txt?dl=0) to create database “sqlbook_data” and 4 tables: “emp”, “dept”, “t10”, “t100”
	- Skip running this code if you've already created data. Instead just load the database by entering:
	```
	use sqlbook_data;
	```

## Key lessons from SQL Cookbook

### Retrieve records
1. The order in which the SQL queries are evaluated (p. 4): `FROM` clause ->  `WHERE` -> `SELECT` or `GROUP BY` -> `SELECT`

	This is why we need to have a query aliasing columns in an inline view, and then select * in the outer query, e.g.

	**WRONG:**
	```sql
	select sal as salary, comm as commission
	from emp
	where salary < 5000;
	```
	**RIGHT:**
	```sql
	select *
	from (
		select sal as salary, comm as commission
	    from emp
	        ) x
	where salary < 5000;
	```
2. Alias table created in the inline view, e.g. by *x*
3. `concat(col1, “ character “, col2)` to concatenate column values (even if col1, col2 are numeric)
4. Conditional logic variable syntax of the form; don't forget `end` at the end!!
```sql
case
		when col1 < 2000 then 1
		when col2 > 3000 then 2
		else 3
end as conditional
```
5. `limit 5` at the end of query: show the first 5 observations
6. Randomly select 5 rows with `order by rand() limit 5`
7. `where col1 is null` or `where col1 is not null` (never use col1!=null)
8. `coalesce(col1,0)` to fill in missing values with `0` or can use `case` clause
9. Search for patterns
	- col1 is 10 or 20: `where col1 in (10,20)`
	- col1 ends with 'ER': `where col1 regexp [ER]$` or `where col1 like '%ER'`
	- col1 starts with 'ER': `where col1 regexp '^[ER]'` or `where col1 like 'ER%'`
	- col1 contains 'I': `where col1 like '%I%'`

### Sorting query results
1. `order by col1, col2 desc`: sort by col1 in ascending, by col2 in descending order
2. Sort by the **last** two characters of job title: use `substr()`
```sql
select ename, job
from emp
order by substr(job,length(job)-1);
% this substring = all characters from the first to the last char
```
2. Sort by `comm` if job is salesman, and by `sal` for all other jobs -> use `case` expression in the `order by`
```sql
select ename, sal, job, comm
from emp
order by
   case
     when job='SALESMAN' then comm
     else sal
   end;
```

### Working with multiple tables
1. `union all` to stack rows from multiple tables; those tables must have same number and same data types of columns but not necessarily same names; if you want to remove duplicates (as in a real set concept), use `union`
2. `inner join` syntax: e.g.
```sql
select e.ename, d.loc
from emp e
inner join dept d
on e.deptno=d.deptno
where e.deptno=10;
```
3. **(Hard!!)** Select values in a column that don't exist in the other table (including null values)
```sql
select d.deptno
  from dept d
 where not exists (
     select 1
     from new_dept nd
     where d.deptno = nd.deptno);
```
4. Select rows that don't exist in another table
```sql
select d.*
from dept d
left outer join emp e
on (d.deptno=e.deptno)
where e.deptno is null;
```
5. Add new information in the left table by merging with other table
```sql
select e.ename, d.loc, eb.received
from emp e
join dept d
on (e.deptno=d.deptno)
left join emp_bonus eb
on (e.empno=eb.empno)
order by 2;
```
or
```sql
select e.ename, d.loc,
         (select eb.received
          from emp_bonus eb
          where eb.empno=e.empno) as received
from emp e, dept d
where e.deptno=d.deptno
order by 2;
```
6. Performing joins when using aggregates: e.g. sum salary and bonus for deptno=10 when bonus data contain bonuses for all employees in deptno=10

	1) Merge first and then sum (with distinct inside sum function)
```sql
select deptno, sum(distinct sal) as tot_salary, sum(bonus) as tot_bonus
from (
		select e.empno, e.ename, e.deptno, e.sal,
			   e.sal*case
						when eb.type=1 then 0.1
						when eb.type=2 then 0.2
						else 0.3
					 end as bonus
		from emp e, emp_bonus eb
		where e.empno=eb.empno
		and   e.deptno=10
        ) x
group by deptno, tot_salary;
```
or 2) Sum employee salary first and merge
```sql
select d.deptno, d.tot_salary,
	sum(e.sal*case when eb.type=1 then 0.1
							when eb.type=2 then 0.2
							else 0.3 end) as tot_bonus
from emp e,
	    emp_bonus eb,
	    (select deptno, sum(sal) as tot_salary
		 from emp
		 where deptno=10
		 group by deptno) d
where e.empno=eb.empno
and   e.deptno=d.deptno
group by deptno, tot_salary;
```
7. Performing outer joins when using aggregates: sum salary and bonus for deptno=10 when bonus data don't contain bonuses for all employees in deptno=10
```sql
select deptno, sum(distinct sal) as tot_salary, sum(bonus) as tot_bonus
from (select e.deptno, e.ename, e.sal,
					e.sal*case when eb.type is null then 0
										 else eb.type/10
								end as bonus
		 from emp e
		 left outer join emp_bonus eb
		 on (e.empno=eb.empno)
		 where deptno=10) x
group by deptno, tot_salary;
```
or same as the 2nd solution in \#6:
```sql
select d.deptno, d.tot_salary,
		  sum(e.sal*eb.type/10) as tot_bonus
from emp e,
        emp_bonus eb,
		(select deptno, sum(sal) as tot_salary
		 from emp
		 where deptno=10 group by deptno) d
where e.empno=eb.empno
and   e.deptno=d.deptno
group by deptno, tot_salary;
```
8. Returning missing data from both merging tables -> `union` the results of 2 different outer joins (can't be done soley by `left join` or `right join` one at a time)
```sql
select d.deptno, d.dname, e.ename
from dept d
left join emp e
on (d.deptno=e.deptno)
union
select d.deptno, d.dname, e.ename
from dept d
right join emp e
on (d.deptno=e.deptno);
```
9. Operation/comparison when a column contains null: e.g. find all employees whose commission is less than the commision of employee "WARD", including those with a null commission
```sql
select ename, comm
from emp
where coalesce(comm,0) < (select coalesce(comm,0) from emp where ename='WARD');
```
10. Extract numbers from alphanumeric var: e.g. get 123321 from 'paul123f321'

### Working with numbers
1. `avg(col)` to compute average; can be combined with `group by`
2. `max(col)`, `min(col)`, `sum(col)`
3. `count(*)` counts number of rows *vs* `count(col)` number of non-NULL values
4. Running total without using window function but with scalar subquery
```sql
select e.ename, e.sal,
		(select sum(d.sal)
		 from emp d
		 where d.empno <= e.empno) as run_total
from emp e
order by 3;
```
Note this can be done more concisely using `sum() over()`:
```sql
select ename, sal,
		  sum(sal) over(order by sal)
from emp;
```
5. Running product - take sum of logs `ln(sal)` and exponentiate
Note: this doesn't work if `sal` <=0 since you can't take log of non-positive values
```sql
select ename, sal, round(exp(log),0)
from (
		select ename, sal,
			sum(ln(sal)) over(order by sal) as log
		from emp
		) x;
```
or
```sql
select e.ename, e.sal,
		(select round(exp(sum(ln(d.sal))),0)
		 from emp d
	 	where d.empno <= e.empno
		 and   d.deptno=e.deptno) as run_product
from emp e
where e.deptno=10;
```
6. Running difference: only the first value in the group is conserved, multiply the rest by (-1) and sum them up
	- **Warning**: the MySQL solution on running difference on p.172 of the book is wrong.
	- **Don't know how to do this**
7. Finding a **mode** is a little tricky, involving `having count(*) >= all (count by value)`
```sql
select sal
from emp
where deptno=20
group by sal
having count(*) >= all (select count(*) from emp where deptno=20 group by sal);
```
8. **(Hard and didn't understand!!)** Finding a **median** is also tricky: do a self-join
```sql
select a.sal
from emp a, emp b
where a.deptno=b.deptno
and   a.deptno=20
group by a.sal
having sum(case when a.sal = b.sal then 1 else 0 end) >= abs(sum(sign(e.sal - d.sal)));
```
9. % salaries in deptno=10 out of total
```sql
select 100*sum(case when deptno=10 then sal
			  	end)/sum(sal) as pct_dept10
from emp;
```
10. Take average after excluding the highest and lowest values
```sql
select avg(sal)
from emp
where sal not in (
		(select min(sal) from emp),
		(select max(sal) from emp)
);
```
The above code may drop multiple employees, if they have highest or lowest salary. If you want to exclude only the highest and lowest value once each, just exclude them and divide by the total (N-2):
```sql
select (sum(sal)-min(sal)-max(sal))/(count(*)-2) as pct
from emp;
```
11. Update the total balance column (i.e. running total) after each transaction row, which can be either 'purchase' or 'payment'
```sql
select
		case when trx='PR' then 'Purchase'
			 else 'Payment'
		end as trx_type,
		amt,
		sum(case when trx='PR' then amt
			 	else -amt
			end) over(order by id) as balance
from V;
```

### Date arithmetic
1. Subtract 5 days/months/years from the date variable using `interval 5 day (month/year)` or `date_add` function
```sql
select ename, hiredate,
	    hiredate-interval 5 day as hd_minus_5d,
		hiredate+interval 5 month as hd_plus_5m,
		hiredate-interval 5 year as hd_minus_5y
from emp
where deptno=10;
```
or
```sql
select ename, hiredate,
		date_add(hiredate, interval -5 day) as hd_minus_5d,
		date_add(hiredate, interval +5 month) as hd_plus_5m,
		date_add(hiredate, interval +5 year) as hd_plus_5y
from emp
where deptno=10;
```
2. Find the number of days between 2 dates (e.g. hiring dates for Allen and Ward) using `datediff(d1,d2)`:=d1-d2; _put the earlier date last in the 2 arguments_
```sql
select datediff(ward_hd, allen_hd)
from (select hiredate as ward_hd
		from emp
		where ename='WARD') x,
		(select hiredate as allen_hd
		from emp
		where ename='ALLEN') y;
```
3. **(New trick!!)** Find the number of business days between 2 dates; Requires 2 steps:

	1) get two dates and create rows for each day between those dates
	2) count number of rows (i.e. days), excluding weekends
```sql
select sum(case when date_format(jones_hd+interval (id-1) day, '%a') in ('Sat','Sun') then 0
				   else 1
			  end) as biz_days,
		max(id) as days
from (
		select max(case when ename = 'BLAKE' then hiredate
				   end) as blake_hd,
			   max(case when ename='JONES' then hiredate
				   end) as jones_hd
		from emp
		where ename in ('BLAKE','JONES')
	    ) x,
		t100
where t100.id <= datediff(blake_hd,jones_hd)+1;
```
4. Find the number of months or years between 2 dates: use `year` and `month` functions
```sql
select n_month, round(n_month/12,0) as n_year
from (
		select (year(last_hd)-year(first_hd))*12 + (month(last_hd)-month(first_hd)) as n_month
		from (
			  select min(hiredate) as first_hd,
			  	   max(hiredate) as last_hd
			  from emp
			 ) x
		) y;
```
5. Find the number of seconds, minutes, or hours between 2 dates
```sql
select datediff(ward_hd, allen_hd)*24 as hr,
		  datediff(ward_hd, allen_hd)*24*60 as min,
		  datediff(ward_hd, allen_hd)*24*60*60 as sec
from
(select hiredate as ward_hd from emp where ename='WARD') a,
(select hiredate as allen_hd from emp where ename='ALLEN') b;
```
6. **(Involving many tricks!!)** Count the occurrences of each weekday in a year
	- Use `cast(string as date)` to convert string to a date variable
	- date_format(date, '%W') returns e.g. 'Monday'
```sql
select date_format(
		cast(concat(year(current_date),'-01-01') as date) + interval t500.id-1 day, '%W') day,
	count(*)
from t500
where t500.id <= datediff(
			cast(concat(year(current_date)+1,'-01-01') as date),
			cast(concat(year(current_date),'-01-01') as date))
group by date_format(
	cast(concat(year(current_date),'-01-01') as date) + interval t500.id-1 day, '%W');
```
7. **(Unfamiliar but important trick!!)** Find the date difference between the current record and the next one: Use the **scalar subquery**
```sql
select *, datediff(next_earliest_hd,hiredate) as gap
from (select e.deptno, e.ename, e.hiredate,
				(select min(d.hiredate)
				 from emp d
				 where d.deptno=e.deptno
				 and   d.hiredate > e.hiredate) as next_earliest_hd
		 from emp e) x
order by 1,3;
```

### Date manipulation
1. Determine if it's a leap year, i.e. February ends on the 29th
The solution in the book suggests using `lastday()` function but here's an alternative solution
```sql
select case when cast(concat(year(current_date)+1,'-02-28') as date) + interval 1 day=cast(concat(year(current_date)+1,'-03-01') as date) then 0
			   else 1
		  end as is_leapyear;
```
2. Determine the number of days in a year
```sql
select datediff(cast(concat(year(current_date),'-12-31') as date), cast(concat(year(current_date),'-01-01') as date))+1 as ndays;
```
or
```sql
select datediff(first_dayofyear+interval 1 year,first_dayofyear) as ndays
from (select current_date-interval (dayofyear(current_date)-1) day as first_dayofyear) x;
```
5. **Skipped 9.3-9.9**
6. Filling in missing dates: Find the number of employees hired in each month from 1980 to 1983

	1) First get boundary months: January 1st of the earliest hiring date year - Dec 1st of the last hiring date year
	```sql
	select adddate(min(hiredate),-dayofyear(min(hiredate))+1) as min_hd,
		cast(concat(year(max(hiredate)),'-12-01') as date) as max_hd
	from emp;
	```
	2) Add rows for each month in the boundary date range
	```sql
	select x.min_hd+interval t.id-1 month as month
	from (
		select adddate(min(hiredate),-dayofyear(min(hiredate))+1) as min_hd,
			cast(concat(year(max(hiredate)),'-12-01') as date) as max_hd
		from emp
	) x,
	t500 t
	where adddate(min_hd,interval t.id-1 month) <= max_hd;
	```
	3) Left outer join on the month of hiredate and count the number of non-NULL hiredates per month (this is final code)
	```sql
	select z.month, count(e.hiredate) num_hired
	from (
		select x.min_hd+interval t.id-1 month as month
		from (
			select adddate(min(hiredate),-dayofyear(min(hiredate))+1) as min_hd,
				cast(concat(year(max(hiredate)),'-12-01') as date) as max_hd
			from emp
		) x,
		t500 t
		where adddate(min_hd,interval t.id-1 month) <= max_hd
	) z
	left join emp e
	on (z.month=adddate(last_day(e.hiredate)-interval 1 month,1))
	group by z.month
	order by 1;
	```
7. Find all employees hired in February or December, or on a Tuesday
```sql
select sum(case when month(hiredate)=2 or month(hiredate)=12 then 1
		           else 0
		      end) as emp_feb_dec,
		  sum(case when dayname(hiredate)='Tuesday' then 1
				   else 0
			  end) as emp_tue
from emp;
select ename
from emp
where dayname(hiredate)='Tuesday' or monthname(hiredate) in ('February','December');
```
8. Comparing records using specific units of time: Which employees were hired on the same weekday and same month?
Do a self-join to compare dates but remove reciprocals
```sql
select concat(e.ename, ' was hired on the same month and same weekday as ', d.ename) as msg
from emp e, emp d
where date_format(e.hiredate,'%w%M')=date_format(d.hiredate,'%w%M')
and   e.empno < d.empno
order by e.ename;
```
9. Identifying overlapping date ranges
```sql
select b.empno, b.ename, concat('project ',b.proj_id,' overlaps project ',a.proj_id) as msg
from emp_project a, emp_project b
where a.empno=b.empno
and   b.proj_start >= a.proj_start
and   b.proj_start <= a.proj_end
and   a.proj_id < b.proj_id;
```

### Advanced searching
1. Skipped **11.1**
2. Skipping n rows in a table: e.g. show employees in odd rows
```sql
select x.rn, x.ename
from (select a.ename,
				(select count(*)
				from emp b
				where b.empno <= a.empno) as rn
		 from emp a) x
where mod(x.rn,2)!=0;
```
or
```sql
select x.rn, x.ename
from (select a.ename,
				count(*) over(order by empno
							  range between unbounded preceding and current row) as rn
		 from emp a) x
where mod(x.rn,2)!=0;
```
3. Selecting the top _n_ records: 1) rank, 2) limit to _n_ records; use scalar subquery!!
If you want to find the top 3 salary employees out of all employees:
```sql
select x.ename, x.sal
from (select a.ename, a.sal,
				(select count(*) from emp b where b.sal >= a.sal) as rn
		 from emp a) x
where x.rn <=3
order by 2 desc,1;
```
If you want to find the top 3 salary employees in each department:
```sql
select x.deptno, x.ename, x.sal
from (select a.deptno, a.ename, a.sal,
				(select count(*) from emp b where b.sal >= a.sal and b.deptno=a.deptno) as rn
		 from emp a) x
where x.rn <=3
order by 1,3 desc, 2;
```
In case, there are multiple employees with same salry within a department, and want to count the top 3 _distinct_ salary employees (i.e. there can be more than 3 employees per department in the output), restrict to distcint salary in each department in the scalar subquery:
```sql
select x.deptno, x.ename, x.sal
from (select a.deptno, a.ename, a.sal,
				(select count(*)
				 from (select distinct sal, deptno from emp) b
				 where b.sal >= a.sal and b.deptno=a.deptno) as rn
		 from emp a) x
where x.rn <=3
order by 1,3 desc, 2;
```

### Window function refresher
1. Number of employees in the department and job, respectively, for each employee's department/job
```sql
select ename, deptno, job,
		  count(*) over(partition by deptno) as dept_cnt,
		  count(*) over(partition by job) as job_cnt
from emp
order by 2;
```
2. Running total of salaries for employees in deptno=10
```sql
select deptno, ename, hiredate, sal,
		  sum(sal) over(partition by deptno) as total1,
	      sum(sal) over() as total2,
		  sum(sal) over(order by hiredate) as running_total
from emp
where deptno=10;
```
	You can use framing clauses like `range between unbounded preceding and current row` (default if unspecified), `rows between 1 preceding and 1 following`
```sql
select deptno, ename, hiredate, sal,
		  sum(sal) over(partition by deptno) as total1,
		  sum(sal) over() as total2,
	      sum(sal) over(order by hiredate
						range between unbounded preceding and current row) as running_total1,
	      sum(sal) over(order by hiredate
						rows between 1 preceding and current row) as running_total2,
		  sum(sal) over(order by hiredate
						range between current row and unbounded following) as running_total3,
		  sum(sal) over(order by hiredate
						rows between 1 preceding and 1 following) as running_total4
from emp
where deptno=10;
```
	Note: The framing clauses didn't work on MySQL version 8.0.15.
3. `max`/`min` over
4. To answer: "What is the number of employees in each dept, how many different types of employees in each dept, how many total employees in table?"
```sql
select deptno, job,
		count(*) over(partition by deptno) as nemp_dept,
		count(*) over(partition by deptno, job) as nemp_job_dept,
		count(*) over() as tot_employees
from emp;
```
```sql
select deptno,
		  emp_cnt as dept_total,
		  total,
		  max(case when job='CLERK' then job_cnt else 0) as clerks,
		  max(case when job='MANAGER' then job_cnt else 0) as managers,
		  max(case when job='PRESIDENT' then job_cnt else 0) as prez,
		  max(case when job='ANALYST' then job_cnt else 0) as anals,
		  max(case when job='SALESMAN' then job_cnt else 0) as smen
from (
	  select deptno, job,
	  	count(*) over() as total,
		  count(*) over(partition by deptno) as emp_cnt,
		  count(*) over(partition by deptno, job) as job_cnt
	  from emp
) x
group by deptno, emp_cnt, total;
```


### Errata I'd like to report:
- p.193 MySQL solution for using `datediff(d1,d2)` returns **d1-d2**, so should put **earlier** date **last** to return postiive difference.
- p.195 typo: "January 10th is a Tuesday and January 11th is a Monday" - since it's a hypothetical situation, dates don't have to matter but just for accuracy
- Ch 8.5 uses table "T500" but the provided code doesn't create this table; instead use the table "T100"
- p.253 typo 'Mysol'; should be 'MySQL'
