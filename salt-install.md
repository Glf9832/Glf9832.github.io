# SaltStack 安装（ubuntu 18.04环境）

## 1、master安装

``` shell
apt-get update
apt-get install salt-master
```

修改/etc/salt/master文件

``` txt
interface: 0.0.0.0
hash_type: sha256
auto_accept: True
```

启动服务

``` shell
/etc/init.d/salt-master start
```

## 2、minion安装

``` shell
apt-get update
apt-get install salt-minion
```

修改/etc/salt/minion文件

``` txt
master: salt
hash_type: sha256
id: salt_minion_name
```

启动服务

``` shell
/etc/init.d/salt-minion start
```

## 3、验证

在master上执行

``` shell
salt-key -L
```