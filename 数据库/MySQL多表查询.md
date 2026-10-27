# MySQL 多表查询

今天学完第 5 章多表查询，刷了 8 道百度/美团面试 SQL。

## 1. 多表关系

表结构之间的联系分三种：

- **一对多**：部门与员工。在多的一方（员工）建外键，指向一的一方主键。
- **多对多**：学生与课程。建第三张中间表，至少两个外键关联两方主键。
- **一对一**：用户与用户详情。任意一方加外键关联对方主键，且外键设为 UNIQUE，多用于单表拆分。

## 2. 笛卡尔积

直接 `select * from emp, dept;` 会得到两表所有行的组合（17×6=102 行），这就是笛卡尔积。多表查询必须消除无效笛卡尔积，加连接条件：

```sql
select * from emp, dept where emp.dept_id = dept.id;
```

emp 表 id=17 的员工没有 dept_id，连接后被过滤掉。

## 3. 内连接

查询两表交集部分。

```sql
-- 隐式
select e.name, d.name from emp e, dept d where e.dept_id = d.id;

-- 显式
select e.name, d.name from emp e inner join dept d on e.dept_id = d.id;
```

起别名后只能用别名访问字段，不能再用原表名。

## 4. 外连接

```sql
-- 左外连接：左表全部 + 交集
select e.*, d.name from emp e left join dept d on e.dept_id = d.id;

-- 右外连接：右表全部 + 交集
select d.*, e.* from emp e right join dept d on e.dept_id = d.id;
```

左外和右外可以互换，调换表顺序即可。日常开发统一用左外连接。

找"没有部门的员工"：左连接后 where 右表主键 is null。

## 5. 自连接

一张表连自己，必须起别名。

```sql
-- 员工及其领导
select a.name '员工', b.name '领导' from emp a left join emp b on a.managerid = b.id;
```

## 6. 联合查询 UNION

```sql
select * from emp where salary < 5000
union all
select * from emp where age > 50;
```

`union all` 直接合并不去重；`union` 合并后去重。多条查询列数、类型必须一致。

## 7. 子查询

SQL 中嵌套 SELECT。按结果形态分四类：

| 类型 | 返回 | 常用操作符 |
|---|---|---|
| 标量子查询 | 单个值 | = <> > >= < <= |
| 列子查询 | 一列多行 | in not in any some all |
| 行子查询 | 一行多列 | = <> in not in |
| 表子查询 | 多行多列 | in |

```sql
-- 标量子查询：销售部所有员工
select * from emp where dept_id = (select id from dept where name = '销售部');

-- 列子查询：销售部和市场部员工
select * from emp where dept_id in (select id from dept where name = '销售部' or name = '市场部');

-- all：比财务部所有人工资都高
select * from emp where salary > all (select salary from emp where dept_id = (select id from dept where name = '财务部'));

-- 行子查询：与张无忌同薪资同领导
select * from emp where (salary, managerid) = (select salary, managerid from emp where name = '张无忌');

-- 表子查询：2006 年后入职的员工及部门
select e.*, d.* from (select * from emp where entrydate > '2006-01-01') e left join dept d on e.dept_id = d.id;
```

## 8. 面试 SQL 真题

**Q1. 第二高的薪水（不存在返回 NULL）**

```sql
select (select distinct salary from employee order by salary desc limit 1 offset 1) as SecondHighestSalary;
```

必须 distinct，否则并列第一会污染第二高；子查询包一层，查不到返回 NULL。

**Q2. 每个部门工资最高的员工**

```sql
select d.name, e.name, e.salary
from employee e join department d on e.dep_id = d.id
where (e.dep_id, e.salary) in (select dep_id, max(salary) from employee group by dep_id);
```

窗口函数写法：

```sql
select department, employee, salary from (
  select d.name department, e.name employee, e.salary,
         rank() over (partition by e.dep_id order by e.salary desc) rk
  from employee e join department d on e.dep_id = d.id
) t where rk = 1;
```

**Q3. 联查学生成绩，去掉 2023、2024 年入学的学生**

```sql
select s.student_name, c.course_name, sc.score
from students s
join scores sc on s.student_id = sc.student_id
join courses c on sc.course_id = c.course_id
where s.enrollment_year not in (2023, 2024);
```

年份字段在哪张表要先确认：入学年份在 students 就按 students 过滤，考试年份在 scores 就按 `year(sc.exam_date)` 过滤。

**Q4. 平均成绩大于 85 的学生**

```sql
select s.student_name, avg(sc.score) avg_score
from students s join scores sc on s.student_id = sc.student_id
group by s.student_id, s.student_name
having avg(sc.score) > 85;
```

分组后过滤用 having，不能用 where。

**Q5. 班级 1 中重名学生人数**

```sql
select sum(cnt) from (
  select student_name, count(*) cnt from students
  where class_id = 1 group by student_name having count(*) > 1
) t;
```

**Q6. 数学成绩排名第二**

```sql
select student_name, score from (
  select s.student_name, sc.score,
         dense_rank() over (order by sc.score desc) rk
  from students s
  join scores sc on s.student_id = sc.student_id
  join courses c on sc.course_id = c.course_id
  where c.course_name = '数学'
) t where rk = 2;
```

`rank()` 并列后跳号；`dense_rank()` 并列不跳号；`row_number()` 强行排序。

**Q7. 查询学号、姓名、平均成绩**

```sql
select s.student_id, s.student_name, avg(sc.score) avg_score
from students s join scores sc on s.student_id = sc.student_id
group by s.student_id, s.student_name;
```

**Q8. 查询每个学生的总成绩**

```sql
select s.student_id, s.student_name, sum(sc.score) total_score
from students s join scores sc on s.student_id = sc.student_id
group by s.student_id, s.student_name;
```

总成绩用 sum，平均成绩才用 avg。

## 9. 易错点（敲/理解的过程遇到的坑）

1. **标点与特殊字符**：复制代码时极易混入零宽空格（Zero-width space），导致 MySQL 解析报 `1064` 语法错误，需纯手打替换。（这个好诡异，我没有找出到底什么东西在我粘贴到Navicat的时候混进去了，最后只能删了重新敲）
2. **主键自增约束**：`AUTO_INCREMENT` 必须配合 `PRIMARY KEY` 或 `UNIQUE` 使用，且一张表仅能有一个自增列。主键默认不可为 `NULL`，无需显式写 `NOT NULL`。
3. **外键约束字段**：`ON DELETE SET NULL` 要求外键列本身允许为 `NULL`；建表时 `CONSTRAINT` 命名规范，便于后续 `DROP FOREIGN KEY`。
4. **外连接过滤条件**：左外连接若在 `WHERE` 中对右表字段做过滤，易导致外连接退化为内连接。若需保留主表全部记录，应将右表条件写在 `ON` 中。
5. **字段类型陷阱**：金额、成绩禁用 `FLOAT/DOUBLE`，使用 `DECIMAL(M,D)` 保证精度。`INT` 与 `VARCHAR` 隐式转换会导致索引失效。
6. **命名规范**：SQL 字段统一使用 `snake_case`（如 `student_id`），避免大小写敏感及连字符（`-`）引发的引号转义问题。