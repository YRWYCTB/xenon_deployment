# xenon_deployment
A doc to deploy xenon.

# xenon集群安装 for centos7
### 0、环境（三节点主从复制） 
* b1(master): 172.06.215.101

* b2(slave): 172.06.215.102

* b3(slave): 172.06.215.103

* 服务IP： 172.06.215.199


\#注：b1 b2 b3 分别为主机名

---
### 1、MySQL5.7增强半同步配置
1. 基于GTID增强半同步的主从复制架构

2. 参数设置：（b2、b3操作此步??）理论应该讲从库均设置为只读。
    * set global super_only=0;
    * set global read_only=0;
    
3. 安装插件：（b1、b2、b3操作此步）
    * INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
    * INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

    * set global rpl_semi_sync_master_enabled=1;
    * set global rpl_semi_sync_master_timeout=2147483648;
    * set global rpl_semi_sync_slave_enabled=1;

---
### 2、系统配置
1. 三台主机的mysql账号指定shell 为/bin/bash ，并改密码（三台机密码改成一样）：
    * chsh mysql
           

    * passwd mysql
    * mysql环境变量： 从其它普通账号复制 .bash* 到mysql 用户家目录
        * cd /usr/local/mysql
        * cp /home/xp/.bash* .
        * chown mysql:mysql .bash*

     * 添加mysql账号的sudo权限
         visudo 
        ```
        mysql   ALL=(ALL)       NOPASSWD: sbin/ip  
        ```

    
1. 三台主机做mysql账号ssh互信(db1 操作)
     * su - mysql
     * ssh-keygen
     * cd .ssh
     * cat id_rsa.pub >authorized_keys
     * chmod 600  authorized_keys
     * cd  ~
     * scp .ssh 172.16.215.102:~/
     * scp .ssh 172.16.215.103:~/
     

---
### 3、安装golang(db1 db2 db3操作)
1. 下载 wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz

2. 解压：tar -xzvf go1.14.2.linux-amd64.tar.gz /usr/local/

3. 设置环境变量：vim /etc/bashrc 添加以下变量   
```
    export GOPATH=/usr/local/go
    export GOBIN=$GOPATH/bin
    export PATH=$PATH:/usr/local/go/bin
```

---
### 4、安装xtrabackup
wget https://www.percona.com/downloads/Percona-XtraBackup-LATEST/Percona-XtraBackup-8.0-10/binary/redhat/7/x86_64/percona-xtrabackup-80-8.0.10-1.el7.x86_64.rpm
 
 yum localinstall percona-xtrabackup-80-8.0.4-1.el7.x86_64.rpm 
 
 yum install libdbi-dbd-mysql
 
---
### 5、安装xenon(b1 b2 b3操作)
1. git安装及下载xenon:
    yum install -y git
    mkdir /mygit
    cd /mygit
    git init
    git clone https://github.com/xenon/xenon
    
2. 安装：
    cd xenon
    make
    mkdir /data/xenon -p 
    cp -r bin xenon/
    cp -r conf/xenon-simple.conf.json xenon/xenon.json 
    echo "/etc/xenon/xenon.json" >xenon/bin/config.path 
    mv xenon/xenon.json /etc/xenon/
    
    chown mysql:mysql /data/xenon -R
    chown mysql:mysql /etc/xenon -R
    

---
### 6、所有机器配置xenon,下面以b1为例，其它机器，只需改成本机IP即可。
1. vim /etc/xenon/xenon.json 

```{
        "server":
        {
                "endpoint":"172.16.215.101:8801"
        },

        "raft":
        {
                "meta-datadir":"raft.meta",
                "heartbeat-timeout":1000,
                "election-timeout":3000,
                "leader-start-command":"sudo /usr/sbin/ip a a 172.16.215.199/16 dev ens33 && arping -c 3 -A 172.16.215.199 -I ens33",
                "leader-stop-command":"sudo /usr/sbin/ip a d 172.16.215.199/16 dev ens33"
        },

        "mysql":
        {
                "admin":"root",
                "passwd":"a",
                "host":"127.0.0.1",
                "port":3306,
                "basedir":"/usr/local/mysql",
                "defaults-file":"/data/mysql/mysql3306/mysql3306.cnf",
                "ping-timeout":1000,
                "master-sysvars":"super_read_only=0;read_only=0;sync_binlog=default;innodb_flush_log_at_trx_commit=default",
                "slave-sysvars": "super_read_only=1;read_only=1;sync_binlog=1000;innodb_flush_log_at_trx_commit=2"
        },

        "replication":
        {
                "user":"repl",
                "passwd":"a"
        },

        "backup":
        {
                "ssh-host":"172.16.215.101",
                "ssh-user":"mysql",
                "ssh-passwd":"mysql",
                "ssh-port":22,
                "backupdir":"/data/mysql/mysql3306/data",
                "xtrabackup-bindir":"/usr/bin",
                "backup-iops-limits":100000,
                "backup-use-memory": "2GB",
                "backup-parallel": 2
        },

        "rpc":
        {
                "request-timeout":500
        },

        "log":
        {
                "level":"INFO"
        }
}
```

---
### 7、开启xenon并配置集群（b1 b2 b3 操作）
1. 如果mysql 不是以mysqld_safe方式启动，关库；
 
2. 启动xenon:
    cd /data/xenon/bin
   ./xenon -c /etc/xenon/xenon.json  >./xenon.log 2>&1 &

3. 添加节点：
    ./xenoncli cluster add 172.16.215.101:8801,172.16.215.102:8801,172.16.215.101:8803
    
4. 查看集群状态
    ./xenoncli cluster status

---


本文参考知数堂吴老师公开课。     


