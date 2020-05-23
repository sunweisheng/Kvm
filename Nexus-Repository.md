# 搭建Nexus Repository包管理系统

## 下载安装程序

[下载Nexus Repository最新版本](https://www.sonatype.com/download-oss-sonatype)
[配置说明](https://help.sonatype.com/repomanager3)
将下载后的文件传输到服务器上

```shell
#修改配置文件
vi  /etc/security/limits.conf

#添加
* soft nofile 65536

#修改服务配置
vi /etc/systemd/system.conf

#修改
DefaultLimitNOFILE=65536

#重启
reboot

#查看结果
ulimit -n

#将当前目标下的文件上传到目标服务器的指定路径
scp nexus-3.23.0-03-unix.tar root@192.168.0.5:/opt/

#解压
tar -xvf nexus-3.23.0-03-unix.tar

#做一个软链接方便访问和更新
ln -s /opt/nexus-3.23.0-03/ /nexus

#修改运行用户
vi /nexus/bin/nexus.rc

#修改运行用户，除非自己个人使用否则不要用root用户
run_as_user="root"

ln -s /nexus/bin/nexus /etc/init.d/nexus

#init.d设置
cd /etc/init.d
chkconfig --add nexus
chkconfig --levels 345 nexus on
service nexus start

#创建Nexus服务
vi /etc/systemd/system/nexus.service
```

```conf
[Unit]
Description=nexus service
After=network.target
  
[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/nexus/bin/nexus start
ExecStop=/nexus/bin/nexus stop
User=root
Restart=on-abort
TimeoutSec=600
  
[Install]
WantedBy=multi-user.target
```

如果没有安装JAVA，请看：
[安装Java1.8](Install-Java-18.md)

```shell
#重新加载服务
systemctl daemon-reload

#开机启动
systemctl enable nexus.service

#运行服务
systemctl start nexus.service

#查看日志
tail -f /opt/sonatype-work/nexus3/log/nexus.log
```

访问http://192.168.0.5:8081 进入管理界面。
