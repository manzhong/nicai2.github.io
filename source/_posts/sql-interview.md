---
title: sql_interview
abbrlink: 16917
date: 2018-04-23 20:32:22
tags:
categories:
summary_img:
encrypt:
enc_pwd:
---

### 一 日期连续问题

```sql
-- 求取连续登陆10天以上的用户和次数
-- table user_login user_id，up_date
select
	user_id
	,count(1) as up10_cnt
from
(
	select
		user_id,simple_date
		,count(1) as cnt
	from
	(
		select
				user_id
				,up_date
				,date_sub(up_date,rn) as simple_date
		from
		(
			select
				user_id
				,up_date
				,row_number() over (partition by user_id order by up_date asc) as rn
			from user_login a 
		) a 
	) a group by user_id,simple_date
) a where cnt>=10 group by user_id
;
```

### 二 连续正增长或负增长的天数

````sql
-- 标记每日gmv相对比前天是增长还是减少 并标记连续增长或减少的天数
-- trd_total date gmv
-- tag  date gmv dir date_cnt
-- a  1    23  1  null
-- a  2    24  1   1
-- a  3    25  1   2
-- a  4    22  0   1
-- a  5    21  0   2
create table wrk.table_gmv_20240703 as 
select
	tag
	,date
	,gmv
	,dir
	,row_number() over (partition by tag order by date) as rn
from table_gmv a 
;

select
		tag
		,date
		,gmv
		,dir
		,a.rn-coalesce(max(b.rn),0) as up_dir
from wrk.table_gmv_20240703 a 
left outer join
wrk.table_gmv_20240703 b
on a.tag=b.tag
  and a.rn>b.rn
  and a.dir <> b.dir
group by a.tag,a.date,a.gmv,a.dir,a.rn
;

````

### 三 日起重叠问题

````sql
-- 计算用户参加活动的天数 用户的日期范围是有可能重叠的
-- user_id begin_date end_date
-- 1       2017-03-01 2017-03-07
-- 1       2017-03-04 2017-03-09
-- 1       2017-03-07 2017-03-12
-- 1       2017-03-15 2017-03-17
select
	user_id
	,sum(dt_cnt) as fin_dt_cnt
from
(
	select
		user_id
		,begin_date
		,end_date
		,case when max_dt is null or begin_date>max_dt then datediff(end_date,begin_date)+1
		      when max_dt>=begin_date and max_dt<end_date then datediff(end_date,max_dt)
		      when max_dt>=end_date then 0 
		      end as dt_cnt
	from
	(
		select
			user_id
			,begin_date
			,end_date
			,max(end_date) over (partition by user_id order by begin_date rows between unbounded preceding and 1           precending) as max_dt
		from table_user a 
	) a 
) a group by user_id 
;
````

### 四 求直播间最大在线人数

```sql
-- 直播间访问数据：直播间ID，userid，上线时间，下线时间 求直播间最大在线人数
-- room_id，user_id，entry_time，leave_time
select
	room_id,max(cnt) as max_room_user_cnt
from
(
	select
		room_id,user_id,fin_time,tag
		,sum(tag) over (partition by room_id order by fin_time asc ) as cnt
	from
	(
		select
			room_id，user_id，entry_time as fin_time,1 as tag
		from room_table a 
		union all
		select
			room_id，user_id，leave_time as fin_time,-1 as tag
		from room_table a 
	) a
) a group by room_id
```

### 五 求两个人认识的组





