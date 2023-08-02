### Linux 集群监控部署：prometheus  + Grafana

### 一、前言

之前我们有用到top、free、iostat等等命令，去监控[服务器](https://activity.huaweicloud.com/discount_area_v5/index.html?utm_source=hwc-csdn&utm_medium=share-op&utm_campaign=&utm_content=&utm_term=&utm_adplace=AdPlace070851)的性能，但是这些命令，我们只针对单台[服务器](https://activity.huaweicloud.com/discount_area_v5/index.html?utm_source=hwc-csdn&utm_medium=share-op&utm_campaign=&utm_content=&utm_term=&utm_adplace=AdPlace070851)进行监控，通常我们线上都是一个集群的项目，难道我们需要每一台[服务器](https://activity.huaweicloud.com/discount_area_v5/index.html?utm_source=hwc-csdn&utm_medium=share-op&utm_campaign=&utm_content=&utm_term=&utm_adplace=AdPlace070851)都去敲命令监控吗？这样显然不是符合逻辑的，Linux中就提供了一个集群监控工具 – prometheus。

### 二、搭建被监测节点 node_exporter 

#### 2.1 查看Linux系统版本

```bash
#该命令仅适合Redhat系列的Linux系统，显示的版本信息也比较简单
cat /etc/redhat-release
```

![image-20230802141855624](images/image-20230802141855624.png)

#### 2.2 部署前的准备

1. 关闭所有Linux机器的防火墙：systemctl stop firewalld.service。
2. 保证所有Linux机器的时间是准确的，执行date命令检查；如果不准确，建议使用。
3. 如果你Linux上的时间不准确，可以使用ntp命令同步网络时间。

```bash
#首先 ntp 需要安装
yun intall -y ntp

#安装成功之后，输入如下命令
ntpdate pool.ntp.org
```

#### 2.3 部署Linux操作系统监控组件

1. 下载监控Linux的exporter（注意选择自己的操作系统，我的操作系统是 Linux centos7.9)，下载链接：https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

2. 将node_export 包上传到需要被监控的Linux服务器上，任意的目录下，执行解压命令。

   ```bash
   #1.解压命令
   tar -zxvf node_exporter-1.6.1.linux-amd64.tar.gz
   #2.使用复制命令复制到 node_exporter 文件夹
   mv node_exporter-1.6.1.linux-amd64 node_exporter
   ```

3. 进入解压后的文件夹中，执行启动脚本。

   ```bash
   #1.进入 node_exporter 文件夹
   cd node_exporter
   #2.执行启动脚本
   nohup ./node_exporter
   #3.查看nohup日志，tail -100 nohup.out，出现如下日志，代表启动成功
   ```

   **注意**：极有可能发生如下，报错信息如下：显示 listen tcp :9100: bind: address already in use 9100[端口被占用](https://so.csdn.net/so/search?q=端口被占用&spm=1001.2101.3001.7020)，那么如何杀掉9100端口的进程呢？

   ```log
   [root@mysql node_exporter-1.3.1.linux-amd64]# systemctl status  node_exporter
   ● node_exporter.service - node_exporter
      Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: disabled)
      Active: failed (Result: exit-code) since Wed 2023-02-08 14:21:40 CST; 8s ago
     Process: 32897 ExecStart=/opt/module/node_exporter-1.3.1.linux-amd64/node_exporter (code=exited, status=1/FAILURE)
    Main PID: 32897 (code=exited, status=1/FAILURE)
   
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:115 level=info collector=timex
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:115 level=info collector=udp_queues
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:115 level=info collector=uname
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:115 level=info collector=vmstat
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:115 level=info collector=xfs
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:115 level=info collector=zfs
   Feb 08 14:21:40 mysql systemd[1]: Unit node_exporter.service entered failed state.
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:199 level=info msg="Listening on" address=:9100
   Feb 08 14:21:40 mysql node_exporter[32897]: ts=2023-02-08T06:21:40.435Z caller=node_exporter.go:202 level=error err="listen tcp :9100: bind: address already in use"
   Feb 08 14:21:40 mysql systemd[1]: node_exporter.service failed.
   ```

   打开linux系统，在linux的桌面的空白处右击。

- 在弹出的下拉选项里，点击打开终端。
- 在终端窗口中输入 netstat -tln | grep 被占用的端口命令。
- 输入 lsof -i ：被占端口命令，回车后可查看端口被那个进程占用。
- 输入kill -9 进程 id 命令，回车后即可杀死占用的端口进程。（一般情况下不建议直接杀死进程）

我的端口号是被 gitlab-prometeus 占用的

![image-20230802173435694](images/image-20230802173435694.png)

第一种解决方案：kill -9 进程号

```bash
yum install lsof
lsof -i:9100
kill -9 pid进程号

#无法kill掉的时候，可以使用如下的命令
gitlab-ctl stop node_exporter

#查看状态
gitlab-ctl stop node_exporter
```

第二种解决方案：修改 node_exporter 端口号

```bash
#1.新增一个node_exporter服务
vi /usr/lib/systemd/system/node_exporter.service
#2.粘贴如下命令
[Service]
ExecStart=/usr/local/node_exporter/node_exporter --web.listen-address=:9111
[Install]
WantedBy=multi-user.target
[Unit]
Description=node_exporter
After=network.target

#3.执行如下的命令重新加载系统服务
systemctl daemon-reload
#4.启动服务
systemctl start node_exporter.service
#5.查看服务状态
systemctl status node_export.service
```

![image-20230802182202762](images/image-20230802182202762.png)

出现如上的结果表示 node_exporter.service 启动成功。

#### 2.4 启动成功之后，访问对应的接口

例如：http://192.168.xx.7:9111/metrics 

![image-20230802182547571](images/image-20230802182547571.png)

出现如上的结果表示结果正常。

































