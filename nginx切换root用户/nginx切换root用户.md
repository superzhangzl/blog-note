# Nginx切换root用户

应安全组的要求，提高服务器的安全性，需要将 Nginx切换为普通用户权限启动。但是会出现非root用户禁止绑定1024以下的端口。所以需要使用特殊命令`setcap`来设置绑定低端口的权限。同时由于之前写入日志的目录为root用户，当非root启动时会出现日志无法写入报错，可使用`chown`来设置权限。

脚本如下，按照实际需求更改参数：

```bash
#! /bin/bash

#原来nginx是由root用户启动服务，现在运维收回root用户权限，
#所有需要改为使用公共后端账户"ops"来启动nginx服务


############################【必读】############################################
#
# 注:执行该脚本前需要手动执行'sudo setcap'判断这个命令是否存在,若是不存在则不要执行该脚本
#
###########################################################################
nginx_root=/usr/local/nginx
echo 'nginx根目录为:' $nginx_root

nginx_conf=$nginx_root'/conf'
echo 'nginx配置文件目录为:' $nginx_conf

echo '支持setcap命令,正在更改'
#停止nginx
sudo /usr/local/nginx/sbin/nginx -s stop

#进入nginx项目根目录
cd $nginx_root

if [ -e 'run' ]
then
    echo 'nginx下run目录已存在'
else
    #新建run目录
    sudo mkdir run
fi
cd $nginx_conf
#备份一个更改前的配置文件，以防有问题可以直接恢复
sudo cp nginx.conf nginx.conf.backupR

#替换nginx.conf的的pid文件位置：因为使用ops用户无权访问/run目录下文件
sudo sed -i 's/pid \/run\/nginx.pid;/pid \/usr\/local\/nginx\/run\/nginx.pid;/g' /usr/local/nginx/conf/nginx.conf

#给ops赋权
sudo chown -R ops /usr/local/nginx/*
sudo chown -R ops /var/log/nginx/*

#非root用户禁止绑定1024以下的端口，使用该命令可以赋予普通用户绑定net service的端口，如80,443。
#否则在启动时会出现禁止绑定443端口而导致失败的问题。
sudo setcap cap_net_bind_service=+eip /usr/local/nginx/sbin/nginx


#重启nginx（不要sudo，以ops用户启动nginx，启动会提示有权限问题的警告，忽略即可>）
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf

#停1s再查看系统进程
sleep 1s

#查看关于nginx的进程及所属组
ps aux|grep nginx

```



参考文档：

- [setcap详解](https://www.cnblogs.com/nf01/articles/10418141.html)
