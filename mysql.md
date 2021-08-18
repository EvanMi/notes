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

