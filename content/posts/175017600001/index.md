---
title: "MySQL数据库定时全量与增量备份"
date: "2025-06-18"
tags: ['MySQL']
categories: ['动手实践']
---

数据是无价的，对于数据库来说，由于人为或自然等等各种各样的突发事件，很容易导致数据错误或数据丢失等情况，甚至是数据库崩溃，所以为了保障数据的安全，我们需要对数据库进行有效的定时全量和增量备份来保护数据库。

通过`全量备份`和`增量备份`两种方式配合能够更高效的进行数据备份。

## 全量备份

```bash
mysqldump --lock-all-tables --flush-logs --master-data=2 -u root -p test > xxx.sql
```

- 参数 `--lock-all-tables`

  对于InnoDB将替换为 `--single-transaction`。

  该选项在导出数据之前提交一个 BEGIN SQL 语句，BEGIN 不会阻塞任何应用程序且能保证导出时数据库的一致性状态。它只适用于事务表，例如 InnoDB 和 BDB。本选项和 --lock-tables 选项是互斥的，因为 LOCK TABLES 会使任何挂起的事务隐含提交。要想导出大表的话，应结合使用 --quick 选项。

- 参数 `--flush-logs`

  结束当前日志，生成并使用新日志文件。

- 参数 `--master-data=2`

  该选项将会在输出SQL中记录下完全备份后新日志文件的名称，用于日后恢复时参考，例如输出的备份SQL文件中含有：`CHANGE MASTER TO MASTER_LOG_FILE='MySQL-bin.000002'`, `MASTER_LOG_POS=106;`。

- 参数 `test`

  该处的test表示数据库test，如果想要将所有的数据库备份，可以换成参数 `--all-databases`。

- 参数 `--quick` 或 `-q`

  该选项在导出大表时很有用，它强制 MySQLdump 从服务器查询取得记录直接输出而不是取得所有记录后将它们缓存到内存中。

- 参数 `--ignore-table`

  忽略某个数据表，如 `--ignore-table test.user` 忽略数据库test里的user表

- 更多 `mysqldump` 参数，请参考[网址](https://blog.csdn.net/helloxiaozhe/article/details/77680255)

### 全量备份脚本
```shell
#!/bin/bash

# 备份工具
tool=mysqldump
# 用户名
username=root
# 密码
password=QWE123456.
# 将要备份的数据库
database_name=oa

# 保存全量备份文件个数
number=15
# 备份保存路径
backup_dir=/logs/mysqlbackup
# 增量备份路径
backup_daliy_dir=/logs/bkdaliy
# 日志文件
logFile=/logs/mysqlbackup/bak.log
# 日期
dd=`date +%Y-%m-%d-%H-%M-%S`
# 备份文件
dumpFile="${database_name}-${dd}.sql"
gzDumpFile="${database_name}-${dd}.sql.gz"

beginTime=`date +"%Y年%m月%d日 %H:%M:%S"`
#如果文件夹不存在则创建
if [ ! -d $backup_dir ];
then
    mkdir -p $backup_dir;
fi
#简单写法 mysqldump -u root -p123456 users > /root/mysqlbackup/users-$filename.sql
$tool -u $username -p$password $database_name --ignore-table=$database_name.infra_api_access_log --quick --events --flush-logs --delete-master-logs --single-transaction --default-character-set=utf8mb4 | gzip > $backup_dir/$gzDumpFile


endTime=`date +"%Y年%m月%d日 %H:%M:%S"`
#写创建备份日志
echo 开始:$beginTime 结束:$endTime $gzDumpFile succ >> $logFile

#找出需要删除的备份
delfile=`ls -l -crt $backup_dir/*.sql.gz | awk '{print $9 }' | head -1`
#判断现在的备份数量是否大于$number
count=`ls -l -crt $backup_dir/*.sql.gz | awk '{print $9 }' | wc -l`
if [ $count -gt $number ]
then
  #删除最早生成的备份，只保留number数量的备份
  rm $delfile
fi

#删除所有增长备份数据
cd $backup_daliy_dir
rm -f *
```

### 还原备份

有两种方式还原，第一种是在 MySQL 命令行中，第二种是使用 SHELL 行完成还原

1. 在系统命令行中，输入如下实现还原：
`mysql -uroot -p123456 < /data/mysqlDump/mydb.sql`

2. 在登录进入mysql系统中,通过source指令找到对应系统中的文件进行还原：
`mysql> source /data/mysqlDump/mydb.sql`

## 增量备份

### 开启log_bin

- 检查状态

进入mysql命令行，执行 `show variables like '%log_bin%'` 查看log_bin开启状态

```bash
mysql> show variables like '%log_bin%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| log_bin                         | OFF   |
| log_bin_basename                |       |
| log_bin_index                   |       |
| log_bin_trust_function_creators | OFF   |
| log_bin_use_v1_row_events       | OFF   |
| sql_log_bin                     | ON    |
+---------------------------------+-------+
6 rows in set (0.01 sec)
```

- 编辑 mysql 的配置文件 `vim /etc/my.cnf`，在 mysqld 下面添加下面2条配置

```
[mysqld]
log-bin=/var/lib/mysql/mysql-bin
server_id=152
```

Tip1: 一定要加 `server_id`，否则会报错。至于server_id的值，随便设就可以。

Tip2: log_bin 中间可以下划线_相连，也可以-减号相连。同理server_id也一样。

- 重启mysql

`service mysqld restart`

- 再次在mysql命令行中执行 `show variables like '%log_bin%'`

```bash
mysql> show variables like '%log_bin%';
+---------------------------------+--------------------------------+
| Variable_name                   | Value                          |
+---------------------------------+--------------------------------+
| log_bin                         | ON                             |
| log_bin_basename                | /var/lib/mysql/mysql-bin       |
| log_bin_index                   | /var/lib/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                            |
| log_bin_use_v1_row_events       | OFF                            |
| sql_log_bin                     | ON                             |
+---------------------------------+--------------------------------+
6 rows in set (0.01 sec)
```

### 备份

- 进入mysql命令行，执行 `show master status;`

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      430 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

当前正在记录日志的文件名是 mysql-bin.000003

- 执行命令`mysqladmin -uroot -p密码 flush-logs`，生成并使用新的日志文件
再次查看当前使用的日志文件，已经变为 mysql-bin.000004 了。

### 增量备份脚本

```shell
#!/bin/bash

# 增量备份时复制mysql-bin.00000*的目标目录，提前手动创建这个目录
BakDir=/logs/bkdaliy
# 日志文件
LogFile=/logs/mysqlbackup/log.txt

# mysql的数据目录
BinDir=/var/lib/mysql
# mysql的index文件路径，放在数据目录下的
BinFile=/var/lib/mysql/mysql-bin.index

# 这个是用于产生新的mysql-bin.00000*文件
mysqladmin -uroot -pQWE123456. flush-logs

Counter=`wc -l $BinFile | awk '{print $1}'`
NextNum=0
# 这个for循环用于比对$Counter,$NextNum这两个值来确定文件是不是存在或最新的
for file in `cat $BinFile`
do
        base=`basename $file`
        NextNum=`expr $NextNum + 1`
        if [ $NextNum -eq $Counter ]
        then
                echo $base skip! >> $LogFile
        else
                dest=$BakDir/$base
                #test -e用于检测目标文件是否存在，存在就写exist!到$LogFile去
                if(test -e $dest)
                then
                        echo $base exist! >> $LogFile
                else
                        cp $BinDir/$base $BakDir
                        echo $base copying >> $LogFile
                fi
        fi
done

echo `date +"%Y年%m月%d日 %H:%M:%S"` $Next Bakup succ! >> $LogFile
```

### 还原备份

从备份的日志文件mysql-bin.000003中恢复数据

```bash
mysqlbinlog --no-defaults /var/lib/mysql/mysql-bin.000003 | mysql -uroot -p test
Enter password:
ERROR 1032 (HY000) at line 36: Can't find record in 'bk_user'
```

如果遇到上述问题的话，可以试着修改 /etc/my.cnf 配置。在server_id那一行下添加了 `slave_skip_errors=1032` ，然后再执行。

## crontab定时任务

在 Linux 中，周期执行的任务一般由cron这个守护进程来处理[ps -ef|grep cron]。cron读取一个或多个配置文件，这些配置文件中包含了命令行及其调用时间。cron的配置文件称为“crontab”，是“cron table”的简写。

### crontab语法

crontab命令用于安装、删除或者列出用于驱动cron后台进程的表格。用户把需要执行的命令序列放到crontab文件中以获得执行。每个用户都可以有自己的crontab文件。/var/spool/cron下的crontab文件不可以直接创建或者直接修改。该crontab文件是通过crontab命令创建的。

在crontab文件中如何输入需要执行的命令和时间。该文件中每行都包括六个域，其中前五个域是指定命令被执行的时间，最后一个域是要被执行的命令。每个域之间使用空格或者制表符分隔。

格式如下：

```
minute hour day-of-month month-of-year day-of-week commands
合法值 00-59 00-23 01-31 01-12 0-6 (0 is sunday)

除了数字还有几个个特殊的符号就是"*"、"/"和"-"、","，*代表所有的取值范围内的数字，"/"代表每的意思,"/5"表示每5个单位，"-"代表从某个数字到某个数字,","分开几个离散的数字。

-l 在标准输出上显示当前的crontab。
-r 删除当前的crontab文件。
-e 使用VISUAL或者EDITOR环境变量所指的编辑器编辑当前的crontab文件。当结束编辑离开时，编辑后的文件将自动安装。
```

### 添加执行命令

执行命令 `crontab -e`，添加如下配置

```bash
# 每个星期日凌晨3:00执行完全备份脚本
0 3 * * 0 /bin/bash -x /root/bash/Mysql-FullyBak.sh >/dev/null 2>&1

# 周一到周六凌晨3:00做增量备份
0 3 * * 1-6 /bin/bash -x /root/bash/Mysql-DailyBak.sh >/dev/null 2>&1
```

## 可能遇到的问题

### Can't connect to local MySQL server through socket '/tmp/mysql.sock'

```
mysqladmin: connect to server at 'localhost' failed
error: 'Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)'
Check that mysqld is running and that the socket: '/tmp/mysql.sock' exists
```

去修改mysql的配置文件，添加
```
[mysqladmin]
# 修改为相应的sock
socket=/var/lib/mysql/mysql.sock
```

### 执行mysqldump时遇到 `Unknown table 'column_statistics' in information_schema (1109)`

>如果使用MySQL 8.0+版本提供的命令行工具mysqldump来导出低于8.0版本的MySQL数据库到SQL文件，会出现Unknown table 'column_statistics' in information_schema的错误，因为早期版本的MySQL数据库的information_schema数据库中没有名为COLUMN_STATISTICS的数据表。

>解决问题的方法是，使用8.0以前版本MySQL附带的mysqldump工具，最好使用待备份的MySQL服务器版本对应版本号的mysqldump工具，mysqldump可以独立运行，并不依赖完整的MySQL安装包，比如在Windows中，可以直接从MySQL安装目录的bin目录中将mysqldump.exe复制到其他文件夹，甚至从一台电脑复制到另一台电脑，然后在CMD窗口中运行。

当使用是的MySQL 5.7.22。把5.7.20的 MYSQL_HOME/bin/mysqldump 替换掉 5.7.22的，接着就能顺利执行mysqldump了，也真是奇了怪了。
