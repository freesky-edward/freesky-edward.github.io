---
layout: post
title: "在ubuntu上配置concourse"
date: 2018-08-06
type: tech
---

最近因参与Cloudfoundry社区的需要，涉及了CICD的concourse的环境搭建，涉足了一些concourse相关的安装及部署，这里就过程做简单记录。

### 环境要求
> linux ubuntu 16.0.4

### 安装步骤

1. 安装Postgres

```
sudo apt-get install
sudo apt-get install postgresql postgresql-contrib
```

2. 配置用户即数据库

```
sudo -u postgres createuser su  #su 是用户名，根据实际修改
sudo -u postgres createdb atc #atc 是concourse的默认数据库名
```

```
sudo -u postgres psql
psql=# alter user su with encrypted password '<password>'; 
psql=# grant all privileges on database atc to su ;
```
注：其中的<password>为你想设置的数据用户的密码

3. 安装concourse

在 [这里](https://concourse-ci.org/download.html) 获得程序包的下载地址

```
wget https://github.com/concourse/concourse/releases/download/v4.0.0/concourse_linux_amd64
```

```
chmod +x concourse_linux_amd64
cp concourse_linux_amd64 /usr/local/bin/concourse
```

测试安装情况：

```
concourse --version
```
4. 配置、启动concourse

```
nohup concourse quickstart --add-local-user user1:password@123 --main-team-local-user user1 
--external-url http://127.0.0.1:8080 --worker-work-dir /opt/concourse --postgres-host 127.0.0.1 
--postgres-port 5432 --postgres-user su --postgres-password <db-password> > /var/log/concourse.log &
```

其中<db-password>需与第二步中的password一致

5. 安装 fly

同上[地址](https://concourse-ci.org/download.html) 或者程序包地址

```
wget https://github.com/concourse/concourse/releases/download/v4.0.0/fly_linux_amd64 
chmod +x fly_linux_amd64
cp fly_linux_amd64 /usr/local/bin/fly
```
测试安装情况

```
fly --version
```

6. 配置fly

登录target

```
fly -t test login -c http://127.0.0.1:8080 -u user1 -p password@123 -n main
```
登录成功确认是否有名为test的target：

```
fly targets
```

设置pipeline

```
fly -t test sp -p pipeline1 -c pipeline.yml --load-vars-from vars.yml
```
