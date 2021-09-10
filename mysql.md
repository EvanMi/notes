# Mysql

## Mysql安装



一、配置Mysql扩展源



```ruby
rpm -ivh http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql57-community-release-el7-10.noarch.rpm
```

二、yum安装mysql



```undefined
yum install mysql-community-server -y
```

三、启动Mysql，并加入开机自启



```bash
systemctl start mysqld
systemctl enable mysqld
```

四、使用Mysq初始密码登录数据库



```dart
mysql -uroot -p$(awk '/temporary password/{print $NF}' /var/log/mysqld.log)
```

五、修改数据库密码
 数据库默认密码规则必须携带大小写字母、特殊符号，字符长度大于8否则会报错。
 因此设定较为简单的密码时需要首先修改set global validate_password_policy和_length参数值。



```csharp
mysql> set global validate_password_policy=0;
Query OK, 0 rows affected (0.00 sec)
mysql> set global validate_password_length=1;
Query OK, 0 rows affected (0.00 sec)
```

六、修改密码



```css
mysql> set password for root@localhost = password('123456');
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

七、登录测试



```ruby
[root@http-server ~]# mysql -uroot -p123456
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
mysql> exit
```

八、其他机器可访问

```
登陆mysql,执行下列sql语句

    use mysql；

    update user set host='%' where user = 'root';

    flush privileges;
```

## 数据库事务

### 不可重复读和幻读的区别

不可重复读指的是当前事务受到了其他事务中的update的影响，而幻读指的是当前事务受到了其他事务中的insert和delete操作的影响。

### 丢失更新

(1) 由于提交事务而导致的丢失更新问题

![alt](imgs/mysql_update_lose.png)

(2) 由于事务回滚导致的丢失更新问题

![alt](imgs/mysql_rollback_lose.png)

### MVCC

{min_idx, [创建MVCC时活跃的事务id], current_idx, max_idx}

min_idx : 创建MVCC时活跃的事务id中最小的

[...] : 创建MVCC时所有的活跃的事务id

current_idx : 当前的事务id

max_idx: 创建MVCC时数据库会生成的下一个事务的id

如何使用MVCC？

MVCC是和undo log列表配合使用的，在读取一条数据的时候如果发现这个数据的事务id小于 min_idx，那么这条数据是可以查看的。如果大于等于max_idx，那么这条数据是无法查看的。如果这条数据的事务id在[min_idx, max_idx)的区间内，那么如果在[...]中，那么说明该事务还在活跃，所以数据无法访问。如果不在[...]中，那么说明该事务已经提交，这条数据就是可见的。

#### mysql如何使用MVCC

在RM级别的时候，事务中的每次select操作都会生成一个MVCC。

在RR级别的时候，事务中只有第一次select操作才会生成一个MVCC。 

### 数据库的锁

从类型上区分有读锁和写锁；

从力度上区分有表锁和行锁；

从持有时间上区分有临时锁和持久锁。

#### 表锁 VS. 行锁

表锁指的是对一整张表加锁，一般是DDL处理时使用，也可以在自己的SQL中指定；而行锁指的是锁定某一行或者某几行，或行和行之间的间隙。

表锁由MySQL服务器实现，行锁由存储引擎实现，常见的就是InnoDb，所以通常我们在讨论行锁时，隐含的一层意义就是数据库的引擎为InnoDb，而MyISAM存储引擎只使用表锁。

##### 1.1表锁

表锁由 MySQL 服务器实现，所以无论你的存储引擎是什么，都可以使用。一般在执行 DDL 语句时，譬如 **ALTER TABLE** 就会对整个表进行加锁。在执行 SQL 语句时，也可以明确对某个表加锁，譬如下面的例子：

```
# 加了一个读锁并进行查询操作
lock table products read;
select * from products where id = 100;
unlock tables;

# 加写锁
lock table products write;
```

表锁使用的**一次封锁**技术，也就是说，我们会在回话开始的地方使用lock命令将后面所有要用到的表加上锁，在锁释放之前，我们只能访问这些加锁的表，不能访问其他的表，最后通过unlock tables释放所有的锁。这样做的好处是不会发生死锁。

```
mysql> lock table products read, orders read;
Query OK, 0 rows affected (0.00 sec)
 
mysql> select * from products where id = 100;
 
mysql> select * from orders where id = 200;
 
mysql> select * from users where id = 300;
ERROR 1100 (HY000): Table 'users' was not locked with LOCK TABLES
 
mysql> update orders set price = 5000 where id = 200;
ERROR 1099 (HY000): Table 'orders' was locked with a READ lock and can't be updated
 
mysql> unlock tables;
Query OK, 0 rows affected (0.00 sec)

#可以看到由于没有对 users 表加锁，在持有表锁的情况下是不能读取的，另外，由于加的是读锁，所以后面也不能对 orders # 表进行更新。
```

MySQL表锁的加锁规则如下：

（1）对于读锁

​		持有读锁的会话可以读表，但不能写表；

​		允许多个会话同时持有读锁；

​		其他会话计算没有给表加读锁，也是可以读表的，但是不能写表；

​		其他会话申请该表写锁时会阻塞，知道锁释放。

（2）对于写锁

​		持有写锁的会话既可以读表也可以写表；

​		只有持有写锁的会话才可以访问该表，其他会话访问该表会被阻塞，知道锁释放；

​		其他会话无论申请该表的读锁或写锁，都会阻塞，直到锁释放。

表锁的释放规则：

​		使用unlock tables 语句可以显示释放所有表锁；

​		如果会话在持有表锁的情况下执行lock tables语句，将会释放该会话之前持有的锁；

​		如果会话在持有表锁的情况下执行start transaction 或 begin 开启一个事务，将会释放该会话之前持有的锁；

​		如果会话连接断开，将会释放该会话所有的锁。

##### 1.2行锁

行锁和表锁对比如下：

表锁：开销小，加锁快；不会出现死锁；锁粒度大，发生锁冲突的概率最高，并发度最低；

行锁：开销大，加锁慢；会出现死锁；锁粒度最小，发生锁冲突的概率最低，并发度也最高。

