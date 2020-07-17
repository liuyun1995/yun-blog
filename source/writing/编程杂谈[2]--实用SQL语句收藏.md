1. 寻找表中重复数据(根据多个字段判断重复)

   ```sql
   select 字段1, 字段2, 字段3, count(1) as count
   from 表
   group by 字段1, 字段2, 字段3
   having count > 1
   ```

2. 对重复数据进行去重(取更大的ID)

   ```sql
   elete t1
   from 表1 t1, 表1 t2
   where t1.字段1 = t2.字段1
   and t1.字段2 = t2.字段2
   and t1.字段3 = t2.字段3
   and t1.id < t2.id
   ```

3. 查询表中某字段不同值的重复数

   ```sql
   select 字段1，count(1) as count 
   from 表
   where 字段1 in (值1, 值2，值3...)
   group by 字段1
   ```

4. 选取同一个字段的部分值进行排序

   ```sql
   select *
   from 表
   order by
   	case 字段
   		when 条件1 then 1
   		when 条件2 then 1
   		when 条件3 then 2
   		when 条件4 then 2
   		else 3
   	end asc
   ```

5. 选取同一个字段的部分值进行统计

   ```sql
   select
   	case 字段1
   		when 条件1 then 值1
   		when 条件2 then 值1
   		when 条件3 then 值2
   		when 条件4 then 值2
   		else 值3
   	end as 别名1,
   	SUM(字段2)
   from　表
   group by　别名1
   ```

6. 查询某条记录是否存在

   ```sql
   select 1 from 表 where 条件 limit 1
   ```

