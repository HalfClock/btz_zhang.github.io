---
layout:     post
title:      "Django 之自动化部署"
subtitle:   "将 Django 项目部署到云服务器上"
date:       2019-06-12 11:30:00
author:     "Btz"
header-img: "img/bg-django-deploy.jpg"
catalog: true
tags:
    - Django
---

# 前言

本博客是总结我**将 Django 项目部署到阿里云服务器上的流程**。重要的是，本博客不会将每一步的细节都事无巨细的列出，而是做一个概要性的总结与分析。

这个流程参照了[《Test-Driven Development With Python》](http://www.obeythetestinggoat.com/book/praise.harry.html)一书第九章至第十一章的内容。如果你想复现部署的流程，我建议你去看这本书的第九章 —— 第十一章。

用于部署的 Django 代码仓库在这里 ——> [software test repository](https://github.com/HalfClock/software_test)

# 在服务器上安装需要的服务

想要将 Django 项目部署在真实的服务器上，需要面临几个问题：

1. 项目暴露在了公网之下，**需要有安全策略。**
2. **服务器运行多个服务**，例如：Django、Flask、Spring **都需要监听默认的 80 端口。**
3. 网站的**高并发请求处理**，包括静态资源、Http 请求头的解析等。
4. 服务器如何进行**高效运维**。

此时，如果仅使用 Django 自带的 `manage.py runserver` 肯定是不能满足需求的，所以我们**需要一个高可靠的 WEB 服务器作为 WEB 应用程序的载体。**

现在，比较出名的 WEB 服务器有 Nginx 和 Apache，这里我选取了 Nginx。

#### 反向代理服务器 —— Nginx

正向代理可以理解为网站用户的代理服务，我们平常说的**搭梯子**，所谓的梯子就是正向代理，客户通过正向代理服务器访问到了平时访问不到的资源。

而反向代理是面向 WEB 服务器的，也就是说，当现实中有的**网站拥有成百上千台 WEB 服务器都运行着相同的服务**。然而**用户都是通过统一的域名来访问这个网站**，并不知道访问的具体是哪一台服务器，此时用户**访问的域名直接连接的服务器就是反向代理服务器。**反向代理的服务器负责帮我们把请求转发到真实的服务器那里。

![](/img/in-post/post-django-deploy/15602632166928.jpg)
>图片来源于[这里](https://www.jianshu.com/p/956debe2891d)

Nginx 是一款开源的、高性能的 **HTTP 服务器**和**反向代理服务器**，Nginx 可以作为一个 HTTP 服务器进行网站的发布处理，另外 Nginx 可以作为反向代理进行**负载均衡的实现。** 比如静态文件处理，网站安全，高并发处理等等。

Nginx 就扮演者反向代理服务器一角，其按**照请求数量按照一定的规则进行分发道不同的服务器处理的规则**，就是一种均衡规则，即我们平常说的**负载均衡。**

**Nginx 相关命令：**

```python
apt install nginx  # 安装 nginx
/etc/init.d/nginx start(stop/restart) # 启动/停止/重启 nginx 
nginx -t # 测试 nginx，浏览器输入 IP，若出现欢迎界面则成功。
```
#### 业务相关的服务
在安装 uWSGI 前需要先安装 python 3.6、pip3、virtualenv 等需要的服务。

此外此外还需要安装 git、需要的数据库服务等服务。

这里不再赘述，请自行搜索。
#### WSGI 服务 —— uWSGI

此外我们还需要一个 **WSGI 服务器**，例如 uWSGI/gunicorn ，用于与各种实现了 WSGI 接口的 WEB 框架协作，这里选取了 uWSGI 服务器。

* uWSGI 是一个 WEB 服务器，其实现了 WSGI 协议、uwsgi 协议、http 协议等。
* uwsgi 协议是一种线路协议，其用于在 uWSGI 服务器上与其他网络服务器（Nginx/Apacache）的数据通信。
* WSGI 是一种通信协议(规范)，用于规范 WEB 应用（Django）与 WEB 服务器（uWSGI）之间的通信。

**Nginx 和 uWSGI 及 Django 框架的关系如下图：**

![server-relation](/img/in-post/post-django-deploy/server-relation.png)

1. Nginx 负责接收 http 请求，**如果请求是静态文件**，那么直接根据 URI 返回静态文件，**如果是请求 WEB 应用数据**，那么将请求转发给 uWSGI 服务器。
2. uWSGI 接收 socket 消息后将信息转发给 WEB 应用程序(Django)。
3. Django 处理信息，返回 HTTP 响应，将之返回给 uWSGI，uWSGI 再返回给 Nginx，Nginx 进行打包将 HHTTP 响应发送给用户。

**uWSGI 相关命令**
> 注意，在安装 uWSGI 前需要先安装 python 3.6、pip3、virtualenv

```python
apt install python3-dev # 安装相关依赖包
apt install gcc # 安装相关依赖包
pip3 install uwsgi # 安装 uwsgi，在虚拟环境中
uwsgi --http  :8000 --module  项目.wsgi # 通过 Django 项目自带的 wsgi 直接运行，在项目目录下
pip3 uninstall uwsgi # 卸载 uwsgi
sudo apt-get remove uwsgi # 卸载 uwsgi
```


# 手动部署项目
正式部署前、你需要使用 root 账号创建一个**具有 sudo 权限的普通账号**，因为部署服务器不像安装各类服务，**不需要使用 root 账号**，并且若使用 root 账号部署和运行各个服务，**一旦系统被黑，那么将十分可怕。**

账号相关的一些命令：

```python
useradd -m 用户名 -s /bin/bash # 创建普通用户
passwd 用户名 # 为该用户设置密码
adduser 用户名 sudo # 将该用户添加至 sudo 用户组
su 用户名 # 切换用户
```

**接下来的所有操作都由这个非 root 账号完成。**

#### 建立站点的文件结构
首先，我们需要建立合理的文件目录，我们最好在用户目录（/home/username）下创建 “site/服务器域名或 IP” 这样的目录**作为项目的根目录。**

其次，应该根据不同类型的文件设置不同类型的文件夹：

1. database: 放置**数据库**文件的文件夹。
2. static: 放置网站**静态文件**的文件夹。
3. source: 放置**网站源码**的文件夹。
4. virtualenv: 放置 Django 项目的**虚拟环境**。

这样规划后的目录看起来像这样：

![15590561543622](/img/in-post/post-django-deploy/15590561543622.jpg)


#### 使用 GIT 服务器管理代码
若要实现自动化部署、那么使用 GIT 进行代码的版本控制是最方便、合理的。因为使用 GIT 能够很方便的将最新的代码拉至服务器，并当需要版本回溯时进行及时回溯。

使用 `git clone` 命令将 git 服务器中的代码拉至 source 文件夹下。

#### 创建需要的虚拟环境
使用 virtualenv **管理 python 项目的虚拟环境是很方便的**，尤其是在服务端，创建虚拟环境有助于隔离各个应用的不同环境需求，**避免配置冲突的尴尬**。

使用 `virtualenv -p /usr/bin/python3 virtualenv` 命令在 virtualenv 文件夹下创建以 python3 为基础的虚拟环境。

使用 `source virtualenv/bin/activate` 激活该虚拟环境。

>可使用 deactivate 退出该虚拟环境

**以后的代码都在该虚拟环境中完成了。**

然后在虚拟环境中安装 Django 等需要的依赖，若 python 项目源文件中**有 requirement.txt 文件**，那么可以直接使用:

`pip3 install -r requirement.txt` **安装 requirement.txt 文件记载的所有依赖。**

>`pip freeze > requirement.txt` 可以将该虚拟环境中安装的依赖导出到 requirement.txt 文件中

若没有，那么需要一个一个使用 pip 命令安装，例如：`pip3 install django`

#### 修改设置、构建静态文件、数据库迁移

在安装完所有需要的依赖以后，为了进一步保证冒烟测试能够成功。

我们首先需要**修改 Django 的设置**：最主要的是，**修改 Django 项目的 settings 文件，为 `ALLOWED` 添加服务器的 IP 地址**，其它的需要视不同的项目而定。

其次、我们要将静态文件构建至 static 文件夹下，此步**需要在 settings 文件中修改/添加**  `STATIC_ROOT = os.path.abspath(os.path.join(BASE_DIR, '../static'))` 然后，调用 `python manage.py collectstatic --noinput` **将静态文件项目的静态文件构建至 static 文件夹下。**

最后、我们要将必要的数据库文件迁移至 database 文件夹下，将 Django 的 settings 的 DATABASES 按照以下代码重新设置：

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, '../database/db.sqlite3'),
    }
}
```

使用 `python manage.py makemigrations` 和 `python manage.py migrate` 迁移数据库。

#### 冒烟测试
进行冒烟测试之前，还需要做一件事，到服务器厂商提供的管理页面**将服务器的 80 、8000 端口，以及需要开启的端口开启**，因为各个厂商的规则都不相同，所以这里不在赘述。

开启后、**使用 Django 自带的服务运行应用** -> `python manage.py runserver 0:8000` 

在浏览器访问服务器ip:8000，若正常访问项目，那么说明 Django 没有问题。

还可以**使用 uWSGI 进行冒烟测试**，首先关闭 Django 自带的服务，然后在项目目录下（**wsgi.py 的上级目录**）使用 `uwsgi --http  :8000 --module  项目.wsgi`。

若在浏览器访问服务器ip:8000，若正常访问项目，那么说明 uWSGI 服务器没有问题。

>停止 uwsgi 的命令：`pkill -f uwsgi -9`

#### 编写配置文件至能够正确运行
当冒烟测试没问题后，我们就要进行正式的服务器部署了，按照 [Nginx 和 uWSGI 及 Django 框架的关系图](https://halfclock.github.io/img/in-post/post-django-deploy/server-relation.png) 我们首先需要将 uWSGI 和 Django 连接，所以**需要编写 uWSGI 的配置文件**、然后我们需要**添加 systemd service** 让此服务能够开机自启/崩溃自动重启、最后需要**配置 nginx 让其和 uWSGI 连接**起来。

配置文件最好在项目文件夹中**单独设立一个 deploy_tools 目录存放。**

在 deploy_tools 添加 `uwsgi_conf.ini` uWSGI 配置文件：

>以下配置文件的的内容参考自：[这篇博客](https://www.jianshu.com/p/956debe2891d)

```python
#配置域应该是uwsgi，记住这个不能丢，否则会报错
[uwsgi]
# uwsgi 监听的 socket，可以为 socket 文件或ip地址+端口号，用 nginx 的时候就配socket , 直接运行的时候配 http, http-socket = 127.0.0.1:8080

socket = 127.0.0.1:8000 # 配合 nginx 请使用这个，并注释掉下一行
# http-socket = 127.0.0.1:8080 # 单独运行请使用这个，并注释掉上一行
# socket = unix:/tmp/socket-name.socket # 可以使用 socket 文件监听，这其实是使用了进程间通信而不是端口监听，所以此时通信不需要走网络协议栈。

#指定项目的目录，在 app 加载前切换到当前目录
chdir = /home/site/IP 地址/source/项目目录

# Django 的 wsgi 文件，用来加载 wsgi.py 这个模块
module =  项目名.wsgi
# Python 虚拟环境的路径
home = /home/site/IP 地址/virtualenvs/
# master 启动主进程。
master = true
# 最大数量的工作进程数
processes = 10
# 指定工作进程中的线程数
threads = 2

# 设置socket的权限
chmod-socket = 664
# 退出的时候是否清理环境，自动移除 unix Socket 和 Pid 文件
vacuum = true
#日志文件路径
daemonize = ../uwsgi.log

```

内容填充完成后，还需要编写参数文件 `uwsgi_params` ,填充如下内容：

```python
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;
    
uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUSET_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;
    
uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
```

做完以后，使用 `uwsgi --ini qmblog_uwsgi.ini` 启动 uWSGI ，打开浏览器，若能够看到正常的项目页面，则配置成功。

>需要牢记配置文件中的 socket 和 http-socket 配置需要严格区别。

-------
为了使服务器能够稳定运行，我们需要**确保 uWSGI 在系统启动时运行，并保证在系统崩溃时重启。**

在 `/etc/systemed/system/` 下新建 “服务名.service” 文件。填充如下内容：

```shell
[Unit] 
Description = uWSGI server for 自己的服务名

[Service] 
Restart = on-failure # 将在进程崩溃时自动重启
User = 能够访问项目文件的用户名 
WorkingDirectory = 项目源码的根目录 # /home/site/IP 地址/source/
ExecStart= 虚拟环境目录/uwsgi --ini /home/site/IP 地址/source/项目目录/deploy/uwsgi_conf.ini # 实际运行的命令

[Install] 
WantedBy=multi-user.target # 告诉 Systemd 我们想在系统启动 boot 时即运行此服务
```

填充完成后 .service 文件后需要重新加载，以便以 Systemd 启动 uWSGI：
* `sudo systemctl daemon-reload` 告诉 Systemd 加载刚才编写的 .service 文件
* `sudo systemctl enable 服务名` 告诉 Systemd 永远在 boot 时启动该服务
* `sudo systemctl start 服务名` 实际启动服务名的命令


-------

最后，我们需要**编写 nginx 的配置文件，以便让 nginx 和 uWSGI 能够连接**。

在 `/etc/nginx/sites-available ` 目录下添加 `IP 地址.conf` 配置文件，填充如下内容：

>以下配置文件的的内容参考自：[这篇博客](https://www.jianshu.com/p/956debe2891d)

```python
#配置文件内容：

# 转发给哪个服务器，可以通过 upstream 配置项让 nginx 实现负载均衡，列表中的每一个 IP 都是一个真实服务器的 IP，若有本地的 uWSGI 还可以写 http://socket 文件。
upstream django {    
    server   127.0.0.1:8000; 
    server   127.0.1.1:8000; 
}

# 设定虚拟主机配置，一个http中可以有多个server。
server {
    # 启动的nginx进程监听请求的端口
    listen      80;
    #定义使用域名访问，若没有域名可以注释掉
    server_name  域名;  
    charset     utf-8;

    # max upload size  
    client_max_body_size 75M;    # adjust to taste

    # location 配置请求静态文件多媒体文件，比如图片文件目录，若无可以注释掉。
    location /media  {
        alias  /home/site/IP 地址/media/;  
    }
    # 静态文件访问的 url
    location /static {
        # 指定静态文件存放的目录
        alias /home/site/IP 地址/static/;
    }

#  将所有非媒体请求转到 Django 服务器上
    location / {
        # 包含 uwsgi 的请求参数，路径为 uwsgi_params 绝对路径
        include  /home/site/IP 地址/source/项目目录/deploy/qmblog_uwsgi_params; 
        # 转交请求给uwsgi
        uwsgi_pass  django;  #这个 django 对应配置文件开头的 upstream django 配置项，对于动态请求，转发到本机的端口，也就是 uwsgi 监听的端口，uwsgi运行的主机和ip,后面我们会在本机的该端口上运行 uwsgi 进程
        # 下面两个配置意思是如果比如通过http://www.xxx.com直接访问的是static下的index.html或者index.htm页面，一般用于将首页静态化
        #root   /root/src/www/CainiaoBlog/static/;
        #index index.html index.htm; 
    }
    
    #精确匹配不同于上面 / ，这里 http://www.xxx.com 会匹配这个，根据这个配置将请求转发给另外 nginx 服务器，让另外服务器提供静态首页。同上面的访问 index.html 在另外同一台服务器上同一配置文件中结合。
    #location = / {
    #   proxy_pass  http://ip:port;
    #}
}
```

配置文件中的 root 和 alias 的主要区别**在于 nginx 如何解释 location 后面的 uri**，这会使两者分别以不同的方式将请求映射到服务器文件上：

* root 的处理结果是：root路径＋location路径
* alias 的处理结果是：使用 alias 路径替换location路径

填充完成以后，需要在 `/etc/nginx/sites-enabled ` 目录下软连接刚才在 `/etc/nginx/sites-available ` 目录下编写的 IP 地址.conf 文件。

在 `/etc/nginx/sites-enabled ` 目录下使用 `ln -s ../sites-available/IP 地址.conf`。

> `/etc/nginx/` 的两个文件夹 —— sites-enabled 和 sites-available 都可以存放 nginx 配置文件，但是，一般来讲 sites-available 存放着可以使用的 conf 文件，而 
sites-enabled 是 nginx 运行时加载的 conf 文件真正存放的目录，所以**通常 sites-enabled 里面放的都是 sites-available 里文件的软链接。**

>注意 `/etc/nginx/sites-enabled ` 中的其他文件最好删除掉，只剩刚才编写的配置文件

然后，重新加载 nginx 配置： `nginx -s reload` 或者 `sudo systemctl reload nginx`。

在浏览器中手动访问服务器，若能正常访问服务器则说明配置成功。

# 编写自动化部署脚本

手动部署是为了更好的自动化部署，因为各服务器可能在某些细节上会有一些差异，我们必须先手动部署一遍确保没有坑，再尝试自动部署。

为了在本地能够通过命令操纵服务器，和**保证过程的可重复性**。 我们需要在本地编写自动化部署的脚本，**在这里我选择使用 python3 编写的开源工具包：fabric3**进行脚本的编写。

Fabric 是一个工具包，可**自动执行要在服务器上运行的命令**。通常需要写一个名为fabfile.py 的文件，它将包含一个或多个函数，这些函数稍后可以从名为 fab 的命令行工具调用，如下所示：

`fab function_name：host = USERNAME@SERVER_ADDRESS`

这将调用 function_name，使用账号 USERNAME **建立与地址是 SERVER_ADDRESS 的服务器的连接。** 还有很多其他选项可用于指定用户名和密码，可以使用 fab --help 了解这些选项。

接下来我们编写 fabfile.py

>这个文件在本地编写，当然尽可能的放在 deploy_tools 目录下

```python
from fabric.contrib.files import append, exists, sed
from fabric.api import env, local, run
import random

REPO_URL = 'https://github.com/HalfClock/software_test.git' # 项目的源码仓库

def deploy():
    site_folder = f'/home/{env.user}/sites/{env.host}' # 建立根目录
    source_folder = site_folder + '/source' 
    _create_directory_structure_if_necessary(site_folder) #创建站点的文件目录
    _get_latest_source(source_folder) # 从 GIT 服务器上拉取最新的代码，并设置到最新的版本
    _update_settings(source_folder, env.host) # 更改 Django 的 setting 文件
    _update_virtualenv(source_folder) # 创建虚拟环境
    _update_static_files(source_folder) # 构建静态文件
    _update_database(source_folder) # 迁移数据库

def _create_directory_structure_if_necessary(site_folder):
    for subfolder in ('database', 'static', 'virtualenv', 'source'):
        run(f'mkdir -p {site_folder}/{subfolder}')

def _get_latest_source(source_folder):
    if exists(source_folder + '/.git'):
        run(f'cd {source_folder} && git fetch')
    else:
        run(f'git clone {REPO_URL} {source_folder}')
    current_commit = local("git log -n 1 --format=%H", capture=True)
    run(f'cd {source_folder} && git reset --hard {current_commit}')

def _update_settings(source_folder, site_name):
    settings_path = source_folder + '/superlists/settings.py'
    sed(settings_path, "DEBUG = True", "DEBUG = False")
    sed(settings_path,
        'ALLOWED_HOSTS =.+$',
        f'ALLOWED_HOSTS = ["{site_name}"]'
    )
    secret_key_file = source_folder + '/superlists/secret_key.py'
    if not exists(secret_key_file):
        chars = 'abcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*(-_=+)'
        key = ''.join(random.SystemRandom().choice(chars) for _ in range(50))
        append(secret_key_file, f'SECRET_KEY = "{key}"')
    append(settings_path, '\nfrom .secret_key import SECRET_KEY')

def _update_virtualenv(source_folder):
    virtualenv_folder = source_folder + '/../virtualenv'
    if not exists(virtualenv_folder + '/bin/pip'):
        #run(f'python3.6 -m venv {virtualenv_folder}')
        run(f'virtualenv -p /usr/bin/python3 {virtualenv_folder}')
    run(f'{virtualenv_folder}/bin/pip install -r {source_folder}/requirements.txt')

def _update_static_files(source_folder):
    run(
        f'cd {source_folder}'
        ' && ../virtualenv/bin/python manage.py collectstatic --noinput'
    )

def _update_database(source_folder):
    run(
        f'cd {source_folder}'
        ' && ../virtualenv/bin/python manage.py migrate --noinput'
    )

```

实际上，**上面这个自动部署脚本做的事情和手动部署做的事情基本一样，只不过是将手动部署需要输入的命令统一起来，按顺序自动执行罢了。**就算换一种自动部署的方式，不用 fabric3 使用 shell 脚本，其流程也差不多。

上面的脚本还差配置 uWSGI、systemd server 、nginx 的配置文件没有自动化生成和重新加载，这部分也可以使用差不多的逻辑使用 run(命令) 来进行自动化生成和重新加载，**但是出于安全因素，此部分最好还是运维人员使用 root 账户手动配置比较好。**

最后使用 `fab deploy：host = USERNAME@SERVER_ADDRESS` 进行自动化部署。

# End

本篇博客总结了 Django 在真实的云服务器（Ubuntu）上部署的流程。首先，我们讨论了真实部署面临的一些问题，以及云服务器上安装对应的服务以避免这些问题。

然后，我们手动部署了一遍项目，这个过程包括设置一个合理的站点目录、用 GIT 管理代码、创建项目的虚拟环境、修改项目设置、构建静态文件和数据库迁移，并编写了 uWSGI、systemd server 、nginx 的配置文件使他们能够正常的通信。

最后，我们借用 fabric3 说明了自动化部署的原理及流程，并编写了一个自动化部署脚本。

希望这篇博客能够对你有所帮助。

> 本篇博文的很多内容均借鉴自这篇[博客](https://www.jianshu.com/p/956debe2891d)，如果你想要手把手的教程，这篇博客能够帮助到你。
