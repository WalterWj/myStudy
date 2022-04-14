
# MHA集群架构
- perl语言写的，一个开源的工具，日本出品
<img src=./MHA.jpg width="450px" />

> MHA 角色：
manager    管理人
node    节点

## 工作原理：
manger管理监控每一个一主两从的集群
manager是监控master的，默认每3秒会去监控master。

- 监控方式：select和connect
可以安装在独立的机器上，也可以安装在从机上。

- node包括master和slave

## master出现故障后，故障转移的大致流程：
1. 会检查所有的slave是否正常，服务器，SSH，mysql
2. 检查所有slave，获取最新binlog，找到latest slave
3. lastest slave和master对比，生成差异的binlog，并且保存到manager，会先在工作目录生成，然后传输。
4. latest slave和其他slave比较，生成他们之间的差异binlog，并且传输到slave，这是所有slave和binlog都是一致的了。
5. 选出new master，然后将第三步生成的差异binlog传输到new master。
6. 其他slave指定new master

## 修改主机名
```
/etc/sysconfig/network
hostname    命令
```

# MHA架构创建流程
## 一.搭建一主双从结构
看主从的笔记

## 二.安装MHA
- 1.在所有节点安装MHA node所需的perl模块（DND）
`[root@master ~]# yum install perl-DBD-MySQL -y`

- 2.在所有的节点安装mha node
```
[root@master lianxi]# cd /software/
[root@master software]# ls
mha4mysql-manager-0.56.tar.gz  perl-Config-Tiny-2.12-1.el6.rfx.noarch.rpm
mha4mysql-node-0.56.tar.gz
[root@master software]# scp -r * 172.16.12.111:/software/
[root@master software]# tar -xf mha4mysql-node-0.56.tar.gz 
[root@master software]# cd mha4mysql-node-0.56
[root@master mha4mysql-node-0.56]# perl Makefile.PL 
*** Module::AutoInstall version 1.03
*** Checking for Perl dependencies...
[Core Features]
- DBI        ...loaded. (1.636)
- DBD::mysql ...loaded. (4.036)
*** Module::AutoInstall configuration finished.
Checking if your kit is complete...
Looks good
Generating a Unix-style Makefile
Writing Makefile for mha4mysql::node
Writing MYMETA.yml and MYMETA.json
[root@master mha4mysql-node-0.56]# make && make install
[root@master mha4mysql-node-0.56]# ll /usr/local/bin/
总用量 80
-r-xr-xr-x. 1 root root 16363 9月  15 03:48 apply_diff_relay_logs
-r-xr-xr-x. 1 root root  1465 9月  14 16:47 dbilogstrip
-r-xr-xr-x. 1 root root  6291 9月  14 16:47 dbiprof
-r-xr-xr-x. 1 root root  5479 9月  14 16:47 dbiproxy
-r-xr-xr-x. 1 root root  4803 9月  15 03:48 filter_mysqlbinlog
-r-xr-xr-x. 1 root root  4194 9月  14 16:53 instmodsh
-r-xr-xr-x. 1 root root  1846 9月  14 16:55 l4p-tmpl
-r-xr-xr-x. 1 root root  8257 9月  15 03:48 purge_relay_logs
-r-xr-xr-x. 1 root root  7521 9月  15 03:48 save_binary_logs
-r-xr-xr-x. 1 root root  1026 9月  14 16:46 test-yaml
```

- 3.在monitor节点安装MA Mannager，注意MHA Manager的主机也是需要安装MHA Node。
通过cpan安装perl模块
```
[root@monitor software]# tar -xf cpan.tar -C /root
[root@master mha4mysql-node-0.56]# perl -MCPAN -e shell
o conf init
o conf commit
```
修改 /usr/share/perl5/CPAN/Config.pm文件，将urllist一行修改为：
`'urllist' => [q[file:/root/.cpan/sources/]],`

- 4.安装模块
```
perl -MCPAN -e shell
o conf init
o conf commit
> install Log::Dispatch
> install Parallel::ForkManager
> install Time::HiRes
rpm -ivh perl-Config-Tiny-2.12-1.el6.rfx.noarch.rpm  
```

## 三.配置SSH登录无密码验证
- 原理：
无密码ssh：生成公钥和公钥，然后传输给其他服务器公钥，那么别的服务器就可以无密码登陆ssh了。
`- ssh-keygen`创建公钥和密钥
`ssh-copy-id` 把本地主机的公钥复制到远程主机的`authorized_keys`文件上，`ssh-copy-id`也会给远程主机的用户主目录（home)和`~/.ssh`，`~/.ssh/authorized_keys`设置合适的权限

- 在所有的节点都需要操作：
执行以下命令：
```
ssh-keygen
ssh-copy-id root@172.16.50.55    这样的话是本机可以直接无密码连接，50.55的服务器还是需要密码的。
ssh-copy-id root@172.16.50.85
ssh-copy-id root@172.16.50.95
ssh-copy-id root@172.16.50.96
```

## 四.创建监控用户，在master上执行
```
create user 'monitor'@'172.16.12.%' identified by '123456';
grant all privileges on *.* to 'monitor'@'172.16.12.%';
flush  privileges;
```
**到这里整个集群环境已经搭建完毕，剩下的就是配置MHA软件了。**

## 五.配置MHA
### 1，创建MHA的工作目录，并且创建相关配置文件
- 在monitor上执行
```
mkdir -p /etc/masterha
cp mha4mysql-manager-0.56/samples/conf/app1.cnf /etc/masterha/
```

### 2，修改app1.cnf配置文件，修改后的文件内容如下
```
[server default]
manager_log=/var/log/masterha/app1/manager.log
manager_workdir=/var/log/masterha/app1
master_binlog_dir=/var/lib/mysql
master_ip_failover_script=/usr/local/bin/master_ip_failover
master_ip_online_change_script=/usr/local/bin/master_ip_online_change
password=123456
ping_interval=1
remote_workdir=/tmp
repl_password=rootroot
repl_user=root
ssh_user=root
user=monitor
[server1]
hostname=172.16.12.110    ##主机
port=3306

[server2]
candidate_master=1    ##备库
check_repl_delay=0
hostname=172.16.12.111
port=3306

[server3]
hostname=172.16.12.112    ##monitor
port=3306
```

### 3，设置relay log的清除方式（在每个slave节点上）：
- 关闭relay log自动清除设置
```
set global relay_log_purge=0;
```

- 注意：
   MHA在发生切换的过程中，从库的恢复过程中依赖于relay log的相关信息，所以这里要将relay log的自动清除设置为OFF，采用手动清除relay log的方式。在默认情况下，从服务器上的中继日志会在SQL线程执行完毕后被自动删除。但是在MHA环境中，这些中继日志在恢复其他从服务器时可能会被用到，因此需要禁用中继日志的自动删除功能。定期清除中继日志需要考虑到复制延时的问题。在ext3的文件系统下，删除大的文件需要一定的时间，会导致严重的复制延时。为了避免复制延时，需要暂时为中继日志创建硬链接，因为在linux系统中通过硬链接删除大文件速度会很快。（在mysql数据库中，删除大表时，通常也采用建立硬链接的方式）

### 4，pure_relay_logs.sh脚本使用
   MHA节点中包含了`pure_relay_logs`命令工具，它可以为中继日志创建硬链接，执行`SET GLOBAL relay_log_purge=1`,等待几秒钟以便SQL线程切换到新的中继日志，再执行`SET GLOBAL relay_log_purge=0`。
   
- pure_relay_logs脚本参数如下所示：
```
--user mysql                      用户名
--password mysql                  密码
--port                            端口号
--workdir                         指定创建relay log的硬链接的位置，默认是/var/tmp，由于系统不同分区创建硬链接文件会失败，故需要执行硬链接具体位置，成功执行脚本后，硬链接的中继日志文件被删除
--disable_relay_log_purge         默认情况下，如果relay_log_purge=1，脚本会什么都不清理，自动退出，通过设定这个参数，当relay_log_purge=1的情况下会将relay_log_purge设置为0。清理relay log之后，最后将参数设置为OFF。
```

- 设置定期清理relay脚本（两台slave服务器）
```
cat purge_relay_log.sh 
#!/bin/bash
user=root
passwd=123456
port=3306
log_dir='/var/log/masterha/app1/log'
work_dir='/var/log/masterha/app1'
purge='/usr/local/bin/purge_relay_logs'

if [ ! -d $log_dir ]
then
   mkdir $log_dir -p
fi

$purge --user=$user --password=$passwd --disable_relay_log_purge --port=$port --workdir=$work_dir >> $log_dir/purge_relay_logs.log 2>&1
```

- 添加到crontab定期执行
```
[root@monitor ~]# crontab -e
# crontab -l
0 4 * * * /bin/bash /root/purge_relay_log.sh
```

- `purge_relay_logs`脚本删除中继日志不会阻塞SQL线程。

### 5，检查SSH配置
- 检查MHA Manger到所有MHA Node的SSH连接状态：
```
masterha_check_ssh --conf=/etc/masterha/app1.cnf 
Fri Sep 16 11:07:34 2016 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Sep 16 11:07:34 2016 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Fri Sep 16 11:07:34 2016 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Fri Sep 16 11:07:34 2016 - [info] Starting SSH connection tests..
Fri Sep 16 11:07:35 2016 - [debug] 
Fri Sep 16 11:07:34 2016 - [debug]  Connecting via SSH from root@172.16.50.85(172.16.50.85:22) to root@172.16.50.95(172.16.50.95:22)..
Fri Sep 16 11:07:34 2016 - [debug]   ok.
Fri Sep 16 11:07:34 2016 - [debug]  Connecting via SSH from root@172.16.50.85(172.16.50.85:22) to root@172.16.50.96(172.16.50.96:22)..
Fri Sep 16 11:07:35 2016 - [debug]   ok.
Fri Sep 16 11:07:35 2016 - [debug] 
Fri Sep 16 11:07:35 2016 - [debug]  Connecting via SSH from root@172.16.50.95(172.16.50.95:22) to root@172.16.50.85(172.16.50.85:22)..
Fri Sep 16 11:07:35 2016 - [debug]   ok.
Fri Sep 16 11:07:35 2016 - [debug]  Connecting via SSH from root@172.16.50.95(172.16.50.95:22) to root@172.16.50.96(172.16.50.96:22)..
Fri Sep 16 11:07:35 2016 - [debug]   ok.
Fri Sep 16 11:07:36 2016 - [debug] 
Fri Sep 16 11:07:35 2016 - [debug]  Connecting via SSH from root@172.16.50.96(172.16.50.96:22) to root@172.16.50.85(172.16.50.85:22)..
Fri Sep 16 11:07:35 2016 - [debug]   ok.
Fri Sep 16 11:07:35 2016 - [debug]  Connecting via SSH from root@172.16.50.96(172.16.50.96:22) to root@172.16.50.95(172.16.50.95:22)..
Fri Sep 16 11:07:36 2016 - [debug]   ok.
Fri Sep 16 11:07:36 2016 - [info] All SSH connection tests passed successfully.
```

### 6，配置VIP和master_ip_failover文件
- 在master上执行

- vip：配置ip的子接口。
`#/sbin/ifconfig eth0:1 172.16.12.99/16`

- 在monitor上执行
 - 通过脚本的方式管理VIP。这里是修改`/usr/local/bin/master_ip_failover`，没有就创建(注意创建完成后给执行权限)
 
**注意脚本中的VIP，网口名和ssh_user用户名**

- 修改后内容为：
```
#!/usr/bin/env perl  
use strict;  
use warnings FATAL =>'all';  
  
use Getopt::Long;  
  
my (  
$command,          $orig_master_host, $orig_master_ip,  
$orig_master_port, $new_master_host, $new_master_ip,    $new_master_port  
);  
  
my $vip = '172.16.12.99/16';  # Virtual IP  
my $key = "1";  
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";  
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";
my $ssh_user="root";
my $exit_code = 0;  
  
GetOptions(  
'command=s'          => \$command,  
'ssh_user=s'         => \$ssh_user,  
'orig_master_host=s' => \$orig_master_host,  
'orig_master_ip=s'   => \$orig_master_ip,  
'orig_master_port=i' => \$orig_master_port,  
'new_master_host=s'  => \$new_master_host,  
'new_master_ip=s'    => \$new_master_ip,  
'new_master_port=i'  => \$new_master_port,  
);  
  
exit &main();  
  
sub main {  
  
#print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";  
  
if ( $command eq "stop" || $command eq "stopssh" ) {  
  
        # $orig_master_host, $orig_master_ip, $orig_master_port are passed.  
        # If you manage master ip address at global catalog database,  
        # invalidate orig_master_ip here.  
        my $exit_code = 1;  
        eval {  
            print "\n\n\n***************************************************************\n";  
            print "Disabling the VIP - $vip on old master: $orig_master_host\n";  
            print "***************************************************************\n\n\n\n";  
&stop_vip();  
            $exit_code = 0;  
        };  
        if ($@) {  
            warn "Got Error: $@\n";  
            exit $exit_code;  
        }  
        exit $exit_code;  
}  
elsif ( $command eq "start" ) {  
  
        # all arguments are passed.  
        # If you manage master ip address at global catalog database,  
        # activate new_master_ip here.  
        # You can also grant write access (create user, set read_only=0, etc) here.  
my $exit_code = 10;  
        eval {  
            print "\n\n\n***************************************************************\n";  
            print "Enabling the VIP - $vip on new master: $new_master_host \n";  
            print "***************************************************************\n\n\n\n";  
&start_vip();  
            $exit_code = 0;  
        };  
        if ($@) {  
            warn $@;  
            exit $exit_code;  
        }  
        exit $exit_code;  
}  
elsif ( $command eq "status" ) {  
        print "Checking the Status of the script.. OK \n";  
        `ssh $ssh_user\@$orig_master_host \" $ssh_start_vip \"`;  
        exit 0;  
}  
else {  
&usage();  
        exit 1;  
}  
}  
  
# A simple system call that enable the VIP on the new master  
sub start_vip() {  
`ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;  
}  
# A simple system call that disable the VIP on the old_master  
sub stop_vip() {  
`ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;  
}  
  
sub usage {  
print  
"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";  
}
```

### 7，配置`master_ip_online_change`文件
- 先修改`/usr/local/bin/master_ip_online_change`，没有就创建(创建之后注意给执行权限)
文件内容如下：
```
#/bin/bash  
source /root/.bash_profile  
  
vip=`echo '172.16.12.99/16'`  # Virtual IP  
key=`echo '1'`  
  
command=`echo "$1" | awk -F = '{print $2}'`  
orig_master_host=`echo "$2" | awk -F = '{print $2}'`  
new_master_host=`echo "$7" | awk -F = '{print $2}'`  
orig_master_ssh_user=`echo "${12}" | awk -F = '{print $2}'`  
new_master_ssh_user=`echo "${13}" | awk -F = '{print $2}'`  
  
stop_vip=`echo "ssh root@$orig_master_host /sbin/ifconfig  eth0:$key  down"`  
start_vip=`echo "ssh root@$new_master_host /sbin/ifconfig  eth0:$key  $vip"`  
  
if [ $command = 'stop' ]  
   then  
   echo -e "\n\n\n***************************************************************\n"  
   echo -e "Disabling the VIP - $vip on old master: $orig_master_host\n"  
   $stop_vip  
   if [ $? -eq 0 ]  
      then  
      echo "Disabled the VIP successfully"  
   else  
      echo "Disabled the VIP failed"  
   fi  
   echo -e "***************************************************************\n\n\n\n"  
fi  
  
if [ $command = 'start' -o $command = 'status' ]  
   then  
   echo -e "\n\n\n***************************************************************\n"  
   echo -e "Enabling the VIP - $vip on new master: $new_master_host \n"  
   $start_vip  
   if [ $? -eq 0 ]  
      then  
      echo "Enabled the VIP successfully"  
   else  
      echo "Enabled the VIP failed"  
   fi  
   echo -e "***************************************************************\n\n\n\n"  
fi 
```

### 8，检查整个复制环境状况。
- 通过masterha_check_repl脚本查看整个集群的状态
```
masterha_check_repl --conf=/etc/masterha/app1.cnf
```

- 在此之间，如果报错权限不够的话，那么就将对应文件夹于权限

### 9，检查MHA Manager的状态：
- 通过`master_check_status`脚本查看`Manager`的状态：
```
masterha_check_status --conf=/etc/masterha/app1.cnf
app1 is stopped(2:NOT_RUNNING).
```

- 注意：如果正常，会显示"PING_OK"，否则会显示"NOT_RUNNING"，这代表MHA监控没有开启。
```
app1 (pid:12597) is running(0:PING_OK), master:172.16.50.85
```

### 10，开启`MHA Manager`监控
`nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/masterha/app1/manager.log 2>&1 `

- 启动参数介绍：
```
--remove_dead_master_conf      该参数代表当发生主从切换后，老的主库的ip将会从配置文件中移除。

--manger_log                            日志存放位置

--ignore_last_failover                 在缺省情况下，如果MHA检测到连续发生宕机，且两次宕机间隔不足8小时的话，则不会进行Failover，之所以这样限制是为了避免ping-pong效应。该参数代表忽略上次MHA触发切换产生的文件，默认情况下，MHA发生切换后会在日志目录产生app1.failover.complete文件，下次再次切换的时候如果发现该目录下存在该文件将不允许触发切换，除非在第一次切换后收到删除该文件，为了方便，这里设置为--ignore_last_failover。
```

- 查看MHA Manager监控是否正常：
`masterha_check_status --conf=/etc/masterha/app1.cnf`

### 11，关闭`MHA Manage`监控
- 关闭很简单，使用`masterha_stop`命令完成。
`masterha_stop --conf=/etc/masterha/app1.cnf`

## 六.故障检查
### 1.出现问题在线切换
- 步骤如下：
a)首先，停掉`MHA`监控：
`masterha_stop --conf=/etc/masterha/app1.cnf`
b)其次，进行在线切换操作
`masterha_master_switch --conf=/etc/masterha/app1.cnf --master_state=alive --new_master_host=172.16.12.110 --new_master_port=3306 --orig_master_is_new_slave --running_updates_limit=10000`

- 其中参数的意思：
```
   --orig_master_is_new_slave 切换时加上此参数是将原 master 变为 slave 节点，如果不加此参数，原来的 master 将不启动
   --running_updates_limit=10000,故障切换时,候选master 如果有延迟的话， mha 切换不能成功，加上此参数表示延迟在此时间范围内都可切换（单位为s），但是切换的时间长短是由recover 时relay 日志的大小决定 
```

### 2.修复宕机的Master 
   通常情况下自动切换以后，原master可能已经废弃掉，待原master主机修复后，如果数据完整的情况下，可能想把原来master重新作为新主库的slave，这时我们可以借助当时自动切换时刻的MHA日志来完成对原master的修复。下面是提取相关日志的命令：
```
grep -i "All other slaves should start" /var/log/masterha/app1/manager.log 

Fri Sep 16 13:49:54 2016 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='172.16.50.95', MASTER_PORT=3306, MASTER_LOG_FILE='mysqlbinlog.000003', MASTER_LOG_POS=154, MASTER_USER='repl', MASTER_PASSWORD='xxx';
```
**获取上述信息以后，就可以直接在修复后的`master`上执行`change master to`相关操作，重新作为从库了。**

## 七.出现问题自动调整的相关变化
1. 这样就会只有一主一从
2. app1.cnf配置文件汇总server1的配置信息会自动删除
3. 从库的master to信息会被消除
4. 两个从库不在指向主库
5. 切换成主库的服务器会创建vip地址

# mha的限制
1. 不支持多级复制
2. 不支持日志为statment级别的load data infile
3. 不支持MySQL5.0以前的版本
