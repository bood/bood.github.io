# 阿里云云数据库迁移到自建数据库

## 背景

项目中业务应用都跑在Kubernetes托管集群，而数据服务原先使用的是阿里云提供的MySQL RDS实例和Redis云数据库。最近考虑到下面几个因素，决定将其更换为集群中自建的版本。

* 可用性

  云数据库的一大优势是其可用性，MySQL和Redis都很容易配置成有双活保障的实例

  但项目可用性要求没有那么高的前提下并无太大优势，尤其是Kubernetes环境下服务器宕机后容器也能自动恢复运行

  尤其对于Redis而言，三节点的哨兵集群并不难配置

* 成本

  对于小项目而言，成本也是一个重要的考量

  一个入门级MySQL RDS实例大约每年3800元，而一个基于云盘的自建数据库仅需SSD云盘费用（1元/GB/月）以及集群中少许CPU内存资源。以20GB的MySQL为例存储费用仅为每年240元。

  需要注意的是高效云盘/SSD云盘最小容量是20GB。https://www.alibabacloud.com/help/zh/doc-detail/134767.htm

* 备份

  自建数据库的备份并不复杂，仅需配置对应云盘的快照即可

* 监控

  Kubernetes集群内的MySQL/Redis可以很方面的打开metrics，统一通过Prometheus进行监控

## MySQL

### 部署

阿里云购买的是MySQL 5.7.28的实例。

集群中采用 `bitnami/mysql --version 4.5.2` 的helm chart（这是最后一个5.7版本的chart，对应版本是5.7.26），并做如下修改：

1. 镜像版本修改为 `5.7.28-debian-9-r74`
2. sql-mode修改为空
3. 指定storage class，数据库由于性能很重要，这里直接使用ssd云盘
4. 不需要slave实例

完整values.yaml如下

```yaml
image:
  tag: 5.7.28-debian-9-r74 # 镜像版本
  pullPolicy: IfNotPresent

root:
  password: PASSWORD

replication:
  enable: false

master:
  persistence:
    storageClass: "alicloud-disk-ssd" # SSD性能较好
    size: 20Gi
  config: |-
    [mysqld]
    default_authentication_plugin=mysql_native_password
    skip-name-resolve
    explicit_defaults_for_timestamp
    basedir=/opt/bitnami/mysql
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    tmpdir=/opt/bitnami/mysql/tmp
    max_allowed_packet=16M
    bind-address=0.0.0.0
    pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
    log-error=/opt/bitnami/mysql/logs/mysqld.log
    character-set-server=UTF8
    collation-server=utf8_general_ci
    sql-mode="" # 删除sql-mode，与原数据库相同

    [client]
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    default-character-set=UTF8

    [manager]
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    pid-file=/opt/bitnami/mysql/tmp/mysqld.pid

slave:
  replicas: 0
```

不过事后看此版本chart还有待完善，如无法为service定制annotations。还是应该尝试使用最新的chart，然后更改image tag。

### 数据迁移

首先尝试的是 `mysqldump --all-databases`，由于配置了不少用户，希望将系统表也一并备份并恢复。然而出现了

`Storage engine 'InnoDB' does not support system tables`

很可能阿里已经对数据库做了魔改，只好自行重建用户，转而只导出数据（假设集群mysql已经forward到本机33060端口）

```bash
$ sqlfile=`date +%Y_%m%d_%H%M`.sql
$ mysqldump -h rm-abcdefg.mysql.rds.aliyuncs.com -u root -p --databases db1 db2 db3 --single-transaction --set-gtid-purged=OFF > $sqlfile
$ cat preimport.sql $sqlfile postimport.sql | mysql -uroot -p -h127.0.0.1 -P33060
```

其中 `preimort.sql` 和 `postimport.sql` 是可选的，为的是禁止唯一性和外键检查加速导入过程

```bash
$ cat preimport.sql
SET @OLD_AUTOCOMMIT=@@AUTOCOMMIT, AUTOCOMMIT = 0;
SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS = 0;
SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS = 0;
```

```sh
$ cat postimport.sql
SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS;
SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS;
SET AUTOCOMMIT = @OLD_AUTOCOMMIT;
COMMIT;
```

## Redis

### 部署

阿里云购买的是Redis 4.0的实例。

集群中采用 `bitnami/redis --version 7.1.1` 的helm chart。为与应用现有配置保持一致，本次迁移暂时没有采用哨兵模式。

配置做了如下修改：

1. 关闭了cluster模式
2. 指定storage class，这里对存储性能要求不高，使用高效云盘即可
3. 另外数据实时行要求没那么高，关闭了AOF使用RDB。请结合实际情况配置。

```yaml
image:
  pullPolicy: IfNotPresent

cluster:
  enabled: false

usePassword: true
password: PASSWORD

master:
  persistence:
    storageClass: "alicloud-disk-efficiency"
    size: 20Gi

configmap: |-
  appendonly no # 关闭AOF
  save 900 1 # 打开RDB
  save 300 10
  save 60 10000
```

### 数据迁移

注意迁移前需要关闭RDB并重启实例使之生效，否则redis容器退出前会覆盖传入的rdb文件

```yaml
configmap: |-
  appendonly no
  save "" # 关闭RDB
```

从阿里云Redis下载备份RDB文件，并传入redis pod

```bash
$ rdb=hins15788434_data_20210103183437.rdb
$ kubectl cp $rdb redis-master-0:/data/dump.rdb -n prod
```
重启Redis以后会自动从该RDB恢复

最后记得打开RDB，结合云盘快照应该可以避免大多数情况下的数据丢失

## VPN

最后聊一下一个实际的问题，即开发调试或者排查问题过程中需要连接到集群内的数据库或者Redis。

迁移前，连接阿里云数据库只需要连接到搭建的一个内网VPN即可。

最容易想到的是将集群内MySQL/Redis的Service类型设置成NodePort，这样VPN连接后只需要连接到任意一台集群节点IP的对应端口即可。不过这样如果集群内节点发生变化（比如节点被移除），连接方式就失效了。

事实上，还有一个更好的办法是利用集群配置的SLB，通过将Service类型设置成LoadBalancer，并且通过annotations指定要复用的SLB实例，并在SLB监听中增加相应端口即可实现暴露的IP不变。

配置如下：

```yaml
master:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/alibaba-cloud-loadbalancer-id: lb-id-here
```

效果如下：

```
$ kubectl get svc -n common mysql-mysql
NAME          TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)          AGE
mysql-mysql   LoadBalancer   172.23.9.17   172.19.197.170   3306:32181/TCP   2d1h
```

其中 `EXTERNAL-IP` 即为对应SLB的内网IP，假设SLB监听配置的也是3306端口，则之后只需要通过 `172.19.197.170:3306` 即可访问到该数据库。

