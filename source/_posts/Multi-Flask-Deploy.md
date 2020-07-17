---
title: Ubuntu服务器部署多个Flask应用
date: 2020-07-17 10:33:37
tags: Flask
categories: Python
keywords: Ubuntu，部署多个Flask
---
[TOC]
### 序
最近在学习Flask搞些小项目玩玩，故简单记录一下的学习之路。

#### 工具栈
Ubuntu+Flask+uWSGI+Nginx+Supervisor
适用情况：单个服务器、单个域名
#### 选取方案
部署多个Flask项目应用可以有两种情况吧：
* 在Nginx配置中为不同的项目使用不同的二级域名，需要配置好二级域名；
* 在Nginx配置中为不同的项目配置不同的端口号作转发入口；
使用哪种好见仁见智，根据所需罢了。

### Multi-Flask配置端口
#### 准备
默认你已安装好`Flask`,`uWSGI`,`Nginx`,`Supervisor`。
创建好放置多个Flask项目的根文件夹，⚠️注意：最好不要使用`sudo`去创建`mkdir`.
这里的是在创建了projects文件夹作使用。
#### 多个简单的Flask应用
创建多个Flask APP，简单即可。为不同项目配置好相应的虚拟环境。
* `virtualenv --no-site-packages -p /usr/bin/python3 env` 指定了Python3
* `virtualenv env`

#### uWSGI.ini
在Flask工程中配置uWSGI.ini文件：
```
[uwsgi]
socket = 127.0.0.1:7001     # 指定端口号，和Nginx、Flask启动app.run()中一致
plugins = python    # python版本
chdir = /home/ubuntu/projects/tattoo    # 项目所在路径
wsgi-file = manage.py   # 指定启动.py文件
callable = app  # 全局启动变量
home = /home/ubuntu/projects/tattoo/env     # 指定相应的环境变量
mount = /tattoo=manage.py   # 挂载项目启动文件
manage-script-name = true
```
可以在当前Flask目录下启动测试：`uwsgi --ini uwsgi.ini`

#### Supervisor配置
```
[program:tattoo]
command = uwsgi --ini /home/ubuntu/projects/tattoo/uwsgi.ini
stopsignal=QUIT
autostart=true
autorestart=true
stdout_logfile=/var/log/uwsgi/supervisor_tattoo.log
stderr_logfile=/var/log/uwsgi/supervisor_tattoo_error.log

```

#### Nginx配置
因为HTPPS原因，这里就配置了SSL，需要上传证书到服务器进行配置。如无需HTTPS，去除SSL相关的即可。
```
server {
        listen 443;
        server_name  [你的域名];
        charset      utf-8;
        client_max_body_size 5M;
        ssl_session_timeout 5m;
        ssl on;
        ssl_certificate     1_[你的域名]_bundle.crt;
        ssl_certificate_key 2_[你的域名].key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers on;

        location / {
                try_files $uri $uri/ = 404;
        }

        location = /favicon.ico {
                log_not_found off;
                access_log off;
        }

        location /paper {
            include uwsgi_params;
            uwsgi_pass 127.0.0.1:7000;
            uwsgi_param UWSGI_PYTHON /home/ubuntu/projects/paper/env;
            uwsgi_param UWSGI_CHDIR /home/ubuntu/projects/paper;
            uwsgi_param UWSGI_SCRIPT manage:app;

        }

        location /tattoo {
            include uwsgi_params;
            uwsgi_pass 127.0.0.1:7001;
            uwsgi_param UWSGI_PYTHON /home/ubuntu/projects/tattoo/env;
            uwsgi_param UWSGI_CHDIR /home/ubuntu/projects/tattoo;
            uwsgi_param UWSGI_SCRIPT manage:app;
        }
}
```

### 相关
#### virtualenv
使用`which python`或`which python3`查看Python版本安装位置。
创建虚拟环境指定Python版本：`virtualenv --no-site-packages -p /usr/bin/python3 env`或者`virtualenv env`
激活虚拟环境：`source env/bin/activate`
退出虚拟环境：`deactivate`
* 注意：创建虚拟环境的父文件夹`mkdir [文件夹]`时，最好不要使用sudo


#### requirements.txt
生成：pip freeze > requirements.txt
安装：pip install -r requirements.txt

#### 进程管理工具 supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart [项目配置名称]

#### Nginx
sudo nginx -s reload
sudo service nginx restart

#### 查看进程
ps -ef | grep uwsgi