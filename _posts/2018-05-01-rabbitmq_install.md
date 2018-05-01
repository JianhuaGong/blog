---
title: RabbitMQ安装
date: 2018-05-01 23:30:09
categories:
- rabbitmq
---

下面详细描述如何在华为云安装RabbitMQ

# 1. 申请主机ubuntu 14.0.4 

在"弹性云服务器"上购买弹性云服务器ubuntu 14.0.4。创建完成主机后，为了方便使用，需把其安全组的规则添加出、入方向都为any；*RabbmitMQ支持在各种操作系统上安装，详情见官网: [http://www.rabbitmq.com/download.html][1]*
   
# 2. 安装erlang

RabbitMQ运行依赖erlang，所以先从[https://www.erlang-solutions.com/resources/download.html][2]选择对应的esl-erlang版本包下载，下载完成后，上传到创建的ECS上；再通过下面命令安装erlang。

    dpkg -i esl-erlang_20.3-1~ubuntu~trusty_amd64.deb
    apt-get -f install

# 3. 安装RabbitMQ

我们通过解压的形式来安装RabbitMQ，先从[http://www.rabbitmq.com/install-generic-unix.html][3]下载RabbitMQ unix通用安装包rabbitmq-server-generic-unix-3.7.4.tar.xz，再上传软件包并解压
   
       tar -xf rabbitmq-server-generic-unix-3.7.4.tar.xz

   
# 4. 启动RabbmitMQ

    cd rabbitmq_server-3.7.4/sbin
    nohup ./rabbitmq-server &
    rabbitmqctl status  #检查是否启动
    
# 5. 开启管理界面

       ./rabbitmq-plugins enable rabbitmq_management

   **增加管理用户并设置权限**

       ./rabbitmqctl add_user admin admin
       ./rabbitmqctl set_user_tags admin administrator
       ./rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"

   通过http://**服务器地址**:15672可访问管理RabbitMQ管理界面，使用上面创建的用户admin登录
   ![此处输入图片的描述][management_ui]
   
# 5. 常用命令
   
## 应用管理

    ./rabbitmqctl status //显示RabbitMQ状态
    ./rabbitmqctl stop //停止RabbitMQ应用，关闭节点
    ./rabbitmqctl stop_app //停止RabbitMQ应用
    ./rabbitmqctl reset //重置RabbmitMQ
    ./rabbitmqctl start_app //启动RabbitMQ应用
    ./rabbitmqctl restart //重置RabbitMQ节点
    ./rabbitmqctl force_restart //强制重置RabbitMQ节点
    ./rabbitmqctl list_queues   //查看队列
    ./rabbitmq-plugins list //查看开启的插件

## 用户管理

    ./rabbitmqctl add_user username password //添加用户
    ./rabbitmqctl delete_user username //删除用户
    ./rabbitmqctl change_password username newpassword //修改用户密码
    ./rabbitmqctl list_users //列出所有用户
    
## 权限管理

    ./rabbitmqctl add_vhost vhostpath //创建host
    ./rabbitmqctl delete_vhost vhostpath //删除host
    ./rabbitmqctl list_vhosts //列出所有host
    ./rabbitmqctl set_permissions [-p vhostpath] username <conf> <write> <read> //设置用户权限
    ./rabbitmqctl clear_permissions [-p vhostpath] username //删除用户权限
    ./rabbitmqctl list_permissions [-p vhostpath] //列出host上的所有权限
    ./rabbitmqctl list_user_permissions username //列出用户权限

  [management_ui]: https://jianhuagong.github.io/blog/images/management_ui.png


  [1]: http://www.rabbitmq.com/download.html
  [2]: https://www.erlang-solutions.com/resources/download.html
  [3]: http://www.rabbitmq.com/install-generic-unix.html