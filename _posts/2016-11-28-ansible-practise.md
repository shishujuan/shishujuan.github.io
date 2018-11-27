---
layout:     post
title:      "Ansible(2) 实战"
subtitle:   ""
date:       2016-11-28 23:10:00 +0800
author:     "ssj"
header-img: "img/post/ansible-spec.png"
categories: 自动化运维
catalog: true
comments: true
tags:
    - Ansible
---


> 接上一篇总结了ansible的基本用法，这一次通过部署一个博客站点的例子来进行ansible实战。分为四个部分，第一部分是手动部署一个mezzanine站点；第二部分是通过ansible来部署mezzanine；第三部分是使用角色来重写第二部分的代码；第四部分则是ansible与docker一起使用的效果。（注: mezzanine是一个基于django的CMS系统，有点类似wordpress，[官网地址在这里](http://mezzanine.jupo.org/) ，不过我们的重点是ansible来部署它，而不是去深究它自身的运行机制）。本文内容和例子素材与上一篇《Ansible超详细使用指南》都是来自《Ansible_Up-And-Running》一书，我对书中各章节内容进行了翻译和整合。

# 1 手动搭建mezzanine
在ansible等配置管理和代码部署的工具出现之前，我们一般是要手动去部署一个系统的。mezzanine算是比较简单化的系统了，我们可以通过下面的步骤在自己的电脑上搭建一个博客系统（我这里的测试环境是macos10.12）。

- 先安装一下virtualenv。

	```
	pip install virtualenv
	```
- 接着创建一个环境venv并激活，然后安装mezzanine模块，接着创建工程，初始化数据库和工程。输入管理员用户名密码以及邮箱等信息，运行runserver命令就会默认监听在本地的8000端口了，打开浏览器输入`http://127.0.0.1:8000`即可访问了。

	```
	$ virtualenv venv
	$ source venv/bin/activate
	$ pip install mezzanine
	$ mezzanine-project myproject 
	$ cd myproject
	$ python manage.py createdb 
	$ python manage.py runserver
	
          _d^^^^^^^^^b_
       .d''           ``b.
     .p'                `q.
    .d'                   `b.
   .d'                     `b.   * Mezzanine 4.2.2
   ::                       ::   * Django 1.10.3
  ::    M E Z Z A N I N E    ::  * Python 2.7.10
   ::                       ::   * SQLite 3.14.0
   `p.                     .q'   * Darwin 16.3.0
    `p.                   .q'
     `b.                 .d'
       `q..          ..p'
          ^q........p^
              ''''

	Performing system checks...

	System check identified no issues (0 silenced).
	November 26, 2016 - 13:00:00
	Django version 1.10.3, using settings 'myproj.settings'
	Starting development server at http://127.0.0.1:8000/
	Quit the server with CONTROL-C.
	```
这是一个开发者模式运行的django应用，架构如图1所示:


![图1 mezzanine最简单架构](http://upload-images.jianshu.io/upload_images/286774-b415aec7fa0847dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当然如果要部署到正式环境，有以下几点要考虑：

- mezzanine默认使用的是sqlite数据库，在正式环境我们希望是一个功能更完善的数据库，比如postgresql或者mysql。
- 同时开发者模式并没有单独的web服务器，对于静态文件和动态内容都是通过django自带的http server来访问，在正式环境我们更希望通过分离静态动态内容，静态内容通过nginx直接访问，而动态内容通过一个http WSGI服务器如gunicorn或者uwsgi来实现访问。此外，正式环境可能还需要部署好https。
- 我们希望WSGI进程以守护进程的方式运行，同时能够很方便的控制启动，停止和重启等。使用一个服务管理工具是很方便的，在接下来的实例中我们采用supervisor作为服务管理工具。

# 2 ansible部署mezzanine
这一节用ansible来部署mezzanine，使用nginx做反向代理，gunicorn做应用服务器，基本架构如下：

![图2 采用nginx+gunicorn部署mezzanine](http://upload-images.jianshu.io/upload_images/286774-590dde3a9b71369a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.1 搭建测试环境
为了不影响自己的系统环境，同时也为了后面多服务器测试的方便，我这里使用virtualbox和vagrant搭建了几个虚拟机（测试环境macos10.12），步骤如下：

- 先下载virtualbox安装。[下载地址](https://www.virtualbox.org/wiki/Downloads)，当然，如果你用的虚拟机软件是parallel desktop，那么就不需要下载virtualbox了。
- 再下载vagrant。[下载地址](https://www.vagrantup.com/downloads.html)
- 然后下载一个vagrant支持的虚拟机文件xxx.box(注意你如果你的虚拟机软件用的是virtualbox，才跟我这里一样，否则请下载parallel desktop对应的虚拟机文件)，本来是可以直接通过vagrant命令下载的，不过速度较慢，我用的是 ubuntu/trusty64(14.04)，下载地址在[这里](https://atlas.hashicorp.com/ubuntu/boxes/trusty64/versions/20161111.0.0/providers/virtualbox.box)，用迅雷下载速度还不错，我下载后放置的目录为 `~/Downloads/virtualbox.box`。

安装好后，在virtualbox运行一个ubuntu/trusty64的虚拟机。在运行前，先下载我的[测试代码1](https://github.com/shishujuan/ansible-practice/tree/master/raw),然后进入playbooks目录，执行如下命令：

- 1) `vagrant box add ubuntu/trusty64 ~/Downloads/virtualbox.box` 
    添加之前下载的box文件。
- 2) `vagrant init ubuntu/trusty64`
 这个命令初始化vagrant，在当前目录也就是playbooks目录下会生成Vagrantfile文件。
在原来的Vagrantfile里面增加一行private ip配置，这里的ip设置为`192.168.56.18`是因为我的virtualbox那个网段为这个，你的virtualbox的网段如果不同设置为你自己的即可，注意这里如果设置错了可就没法访问了。最终除去注释后的Vagrantfile如下所示：

	```
	Vagrant.configure("2") do |config|
	  config.vm.box = "ubuntu/trusty64"
	  config.vm.network "private_network", ip: "192.168.56.18"
	end
	```
- 3）`vagrant up` 
  启动虚拟机，如果启动没有问题，接下来可以通过`vagrant`的命令来查看虚拟机的状态了。比如查看ssh配置:

	```
	ssj@ssj-mbp ~/mezzanine-example/raw/playbooks [master*] $ vagrant ssh-config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/ssj/mezzanine-example/raw/playbooks/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
	```
可以看到虚拟机的ssh端口为2222，私钥文件是当前创建目录下的 `.vagrant/machines/default/virtualbox/private_key`，虚拟机的名字和密钥都是vagrant默认配置好的。后面可以看到自己去编写Vagrantfile，可以指定创建虚拟机的ip以及是否创建私钥。如果设置`config.ssh.insert_key = false`，则不会在.vagrant目录创建一个单独的私钥，而是用我们的用户目录下面 `~/.vagrant.d/insecure_private_key`这个默认私钥。

- 4) 接下来可以连接虚拟机看看了。在当前目录执行 `vagrant ssh`，如无意外，你应该已经登录到虚拟机了。登录后默认用户名是vagrant，同时，虚拟机的vagrant用户已经被设置了可以无密码sudo（这都是vagrant的功劳）。

## 2.2 ansible部署
搭建好配置环境后，可以通过ansible来部署mezzanine了。这里我在`raw/playbooks`目录下面增加了一个`ansible.cfg`文件，其中内容如下:

```
[defaults]
hostfile = hosts
remote_user = vagrant
private_key_file = .vagrant/machines/default/virtualbox/private_key
host_key_checking = False
```
这几个配置项做的事情就是指定hostfile以及登录的用户名，私钥文件的位置以及不检查host的key。因为这个测试机器只有一台，所以hosts文件比较简单如下，只需要指定ssh的主机和端口即可：

```
web ansible_ssh_host=127.0.0.1 ansible_ssh_port=2222
```

接下来可以看下用来部署的playbook文件了，代码如下，只要运行 `ansible-playbook mezzanine.yml`就可以部署好一个mezzanine，数据库用的postgresql，web服务器用的nginx，WSGI用的是gunicorn，另外采用supervisor管理gunicorn进程。运行成功后，打开 <http://192.168.56.18.xip.io> 即可访问。

```
---
- name: Deploy mezzanine
  hosts: web
  
  ###定义变量###
  vars:
    user: "{{ ansible_ssh_user }}"
    proj_name: mezzanine-example
    venv_home: "{{ ansible_env.HOME }}"
    venv_path: "{{ venv_home }}/{{ proj_name }}"
    proj_dirname: project
    proj_path: "{{ venv_path }}/{{ proj_dirname }}"
    reqs_path: requirements.txt
    manage: "{{ python }} {{ proj_path }}/manage.py"
    live_hostname: 192.168.56.10.xip.io
    domains:
      - 192.168.56.18.xip.io
      - www.192.168.56.18.xip.io
    repo_url: https://github.com/shishujuan/mezzanine-example.git
    gunicorn_port: 8000
    locale: en_US.UTF-8
    conf_path: /etc/nginx/conf
    tls_enabled: True
    python: "{{ venv_path }}/bin/python"
    database_name: "{{ proj_name }}"
    database_user: "{{ proj_name }}"
    database_host: localhost
    database_port: 5432
    gunicorn_proc_name: mezzanine
  vars_files:
    - secrets.yml
  tasks:
    ##使用template模块替换sources.list文件，以加速安装软件包和python第三方模块，这是我添加的。####
    - name: set the apt source
      template: src=templates/sources.list.j2 dest=/etc/apt/sources.list
      become: True
      
    ##使用apt模块安装必要的软件包###
    - name: install apt packages
      apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
      become: True
      with_items:
        - git
        - libjpeg-dev
        - libpq-dev
        - memcached
        - nginx
        - postgresql
        - python-dev
        - python-pip
        - python-psycopg2
        - python-setuptools
        - python-virtualenv
        - supervisor

    ##启动supervisor##
    - name: ensure supervisord is running
      become: True
      service:
        name: supervisor
        state: running
        enabled: yes
        
    ##拉取mezzanine代码，这是我fork的书中的代码，pip安装的模块整合到了requirements.txt中，去除了pip部分。
    - name: check out the repository on the host
      git: repo={{ repo_url }} dest={{ proj_path }} accept_hostkey=yes
      
    ##pip安装requirements.txt中的python第三方模块##
    - name: install requirements.txt
      pip: requirements={{ proj_path }}/{{ reqs_path }} virtualenv={{ venv_path }}
      
    ##创建postgresql用户##
    - name: create a user
      postgresql_user:
        name: "{{ database_user }}"
        password: "{{ db_pass }}"
      become: True
      become_user: postgres
    - name: create the database
      postgresql_db:
        name: "{{ database_name }}"
        owner: "{{ database_user }}"
        encoding: UTF8
        lc_ctype: "{{ locale }}"
        lc_collate: "{{ locale }}"
        template: template0
      become: True
      become_user: postgres
      
    ##生成local_settings.py文件
    - name: generate the settings file
      template: src=templates/local_settings.py.j2 dest={{ proj_path }}/local_settings.py
      
    ##使用django_manage模块同步迁移django应用数据##
    - name: sync the database, apply migrations, collect static content
      django_manage:
        command: "{{ item }}"
        app_path: "{{ proj_path }}"
        virtualenv: "{{ venv_path }}"
      with_items:
        - syncdb
        - migrate
        - collectstatic

    ##使用script模块跑python代码设置站点和管理员密码。
    - name: set the site id
      script: scripts/setsite.py
      environment:
        PATH: "{{ venv_path }}/bin"
        PROJECT_DIR: "{{ proj_path }}"
        WEBSITE_DOMAIN: "{{ live_hostname }}"
    - name: set the admin password
      script: scripts/setadmin.py
      environment:
        PATH: "{{ venv_path }}/bin"
        PROJECT_DIR: "{{ proj_path }}"
        ADMIN_PASSWORD: "{{ admin_pass }}"
        
    ##使用template模块设置gunicorn配置文件
    - name: set the gunicorn config file
      template: src=templates/gunicorn.conf.py.j2 dest={{ proj_path }}/gunicorn.conf.py
    
    ##设置supervisor配置文件##
    - name: set the supervisor config file
      template: src=templates/supervisor.conf.j2 dest=/etc/supervisor/conf.d/mezzanine.conf
      become: True
      notify: restart supervisor
      
    ##设置nginx配置文件并notify后面的handler重启nginx##
    - name: set the nginx config file
      template: src=templates/nginx.conf.j2 dest=/etc/nginx/sites-available/mezzanine.conf
      notify: restart nginx
      become: True
      
    ##添加mezzanine.conf链接，并notify后面的handler重启nginx。##
    - name: enable the nginx config file
      file:
        src: /etc/nginx/sites-available/mezzanine.conf
        dest: /etc/nginx/sites-enabled/mezzanine.conf
        state: link
      notify: restart nginx
      become: True
      
    ##使用file模块移除nginx默认配置文件。##
    - name: remove the default nginx config file
      file: path=/etc/nginx/sites-enabled/default state=absent
      notify: restart nginx
      become: True
      
    ##添加ssl证书和key
    - name: ensure config path exists
      file: path={{ conf_path }} state=directory
      become: True
      when: tls_enabled
    - name: create ssl certificates
      command: >
        openssl req -new -x509 -nodes -out {{ proj_name }}.crt
        -keyout {{ proj_name }}.key -subj '/CN={{ domains[0] }}' -days 3650
        chdir={{ conf_path }}
        creates={{ conf_path }}/{{ proj_name }}.crt
      become: True
      when: tls_enabled
      notify: restart nginx
      
    ##安装poll twitter
    - name: install poll twitter cron job
      cron: name="poll twitter" minute="*/5" user={{ user }} job="{{ manage }} poll_twitter"

  ##重启nginx和重启supervisor的handlers##
  handlers:
    - name: restart nginx
      service: name=nginx state=restarted
      become: True
    - name: restart supervisor
      supervisorctl: name=gunicorn_mezzanine state=restarted
      become: True

```

用到的ansible模块由file，template，django_manage，supervisorctl, command, postgresql_db等，模块的参数详解可以见 <http://docs.ansible.com/ansible/modules_by_category.html>。

# 3 使用roles重写playbook
上一节是所有的功能都写到了一个playbook，这一节采用标准的role结构来实现相同功能，同时将db和web机器分开部署到两台虚拟机中。与上一节不同的是分开了db和web的play，另外将handler放到了role里面的handlers目录，代码内容基本一致。代码地址: <https://github.com/shishujuan/ansible-practice/tree/master/roles>。

Vagrantfile内容如下，定义了两个虚拟机。

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use the same key for each machine
  config.ssh.insert_key = false

  config.vm.define 'db' do |db|

    # Every Vagrant virtual environment requires a box to build off of.
    db.vm.box = "ubuntu/trusty64"

    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    db.vm.network "private_network", ip: "192.168.56.11"

    # If true, then any SSH connections made will enable agent forwarding.
    db.ssh.forward_agent = true

    db.vm.provider "virtualbox" do |vb|
      # Use VBoxManage to customize the VM. For example to change memory:
      vb.customize ["modifyvm", :id, "--memory", "512"]
    end
  end

  config.vm.define 'web' do |web|

    # Every Vagrant virtual environment requires a box to build off of.
    web.vm.box = "ubuntu/trusty64"

    # Create a private network, which allows host-only access to the machine
    # using a specific IP.
    web.vm.network "private_network", ip: "192.168.56.10"

    # If true, then any SSH connections made will enable agent forwarding.
    web.ssh.forward_agent = true

    web.vm.provider "virtualbox" do |vb|
      # Use VBoxManage to customize the VM. For example to change memory:
      vb.customize ["modifyvm", :id, "--memory", "512"]
    end
  end

end
```
然后更改了ansible.cfg的配置，设置了private key为我的用户目录下面的那个公用的key。roles的目录结果如下，一共3个role，其中`aptsource`是我自己加的，看名字也知道就是为了更改`sources.list`加快安装软件和python模块的速度。创建角色的目录层次结构可以用`ansible-galaxy`工具，非常方便。具体文件内容参见代码，应该不用过多注解了。

```
├── aptsource
│   ├── README.md
│   ├── defaults
│   │   └── main.yml
│   ├── files
│   │   └── sources.list
│   ├── handlers
│   │   └── main.yml
│   ├── meta
│   │   └── main.yml
│   ├── tasks
│   │   └── main.yml
│   ├── templates
│   ├── tests
│   │   ├── inventory
│   │   └── test.yml
│   └── vars
│       └── main.yml
├── database
│   ├── defaults
│   │   └── main.yml
│   ├── files
│   │   ├── pg_hba.conf
│   │   └── postgresql.conf
│   ├── handlers
│   │   └── main.yml
│   └── tasks
│       └── main.yml
└── mezzanine
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── tasks
    │   ├── django.yml
    │   ├── main.yml
    │   └── nginx.yml
    ├── templates
    │   ├── gunicorn.conf.py.j2
    │   ├── local_settings.py.filters.j2
    │   ├── local_settings.py.j2
    │   ├── nginx.conf.j2
    │   └── supervisor.conf.j2
    └── vars
        └── main.yml
```


# 4 ansible部署docker
由于docker只能在linux上运行，如果在mac上跑，需要另外安装一个linux的虚拟机。因此，我直接用第一节中的vagrant创建的`ubuntu/trusty64(14.04的64位版本)`做测试，需要安装的环境包括`docker.io, python-dev, ansible`。ansible版本为2.2.0，docker的版本为1.18。注意docker-py的版本，我这里安装的是1.2.1，其他的版本会跟docker API版本不兼容。另外我这里没有用书中自带代码中的作者自己写的docker模块，而是用的ansible自带的docker模块，有些语法点有所不同，我已经做了修改适配。如果你的系统不是ubuntu14.04，安装的docker版本不一样，那么需要安装的docker-py可能也会不一样。

另外要注意的是，docker模块在ansible新版本中已经不推荐使用了，取而代之的是`docker_container`， `docker_image`模块。

```
apt-get install docker.io python-dev python-pip libffi-dev
```
```
pip install jinja2 ansible docker-py==1.2.1
```
```
#查看docker版本
root@vagrant-ubuntu-trusty-64:~# docker version
Client version: 1.6.2
Client API version: 1.18
Go version (client): go1.2.1
Git commit (client): 7c8fca2
OS/Arch (client): linux/amd64
Server version: 1.6.2
Server API version: 1.18
Go version (server): go1.2.1
Git commit (server): 7c8fca2
OS/Arch (server): linux/amd64
```

完整的代码地址在这里： <https://github.com/shishujuan/ansible-practice/tree/master/docker>。分为两个目录，dockerfiles和playbooks。其中dockerfiles中的是Dockerfile，包括四个目录，用来创建镜像文件，启动容器在playbook中执行。进入对应的目录，运行`make image`即可创建好对应的镜像文件，运行`docker images`可以看到镜像文件。

```
ssj@ssj-mbp ~/mezzanine-example/docker/dockerfiles [master*] $ tree
.
├── certs
│   ├── Dockerfile
│   ├── Makefile
│   └── sources.list
├── memcached
│   ├── Dockerfile
│   ├── Makefile
│   └── sources.list
├── mezzanine
│   ├── Dockerfile
│   ├── Makefile
│   └── ansible
│       ├── files
│       │   ├── gunicorn.conf.py
│       │   ├── local_settings.py
│       │   ├── scripts
│       │   │   ├── setadmin.py
│       │   │   └── setsite.py
│       │   └── sources.list
│       └── mezzanine-container.yml
└── nginx
    ├── Dockerfile
    ├── Makefile
    └── nginx.conf

```
运行的playbook完整代码如下：

```
---
- name: run mezzanine from containers
  hosts: localhost
  vars_files:
    - secrets.yml
  vars:
    # The postgres container uses the same name for the database
    # and the user
    database_name: mezzanine
    database_user: mezzanine
    database_port: 5432
    gunicorn_port: 8000
    docker_host: "{{ lookup('env', 'DOCKER_HOST') | regex_replace('^tcp://(.*):\\d+$', '\\\\1') | default('localhost', true) }}"
    project_dir: /srv/project
    website_domain: "{{ docker_host }}.xip.io"
    mezzanine_env:
      SECRET_KEY: "{{ secret_key }}"
      NEVERCACHE_KEY: "{{ nevercache_key }}"
      ALLOWED_HOSTS: "*"
      DATABASE_NAME: "{{ database_name }}"
      DATABASE_USER: "{{ database_user }}"
      DATABASE_PASSWORD: "{{ database_password }}"
      DATABASE_HOST: "{{ database_host }}"
      DATABASE_PORT: "{{ database_port }}"
      GUNICORN_PORT: "{{ gunicorn_port }}"
    setadmin_env:
      PROJECT_DIR: "{{ project_dir }}"
      ADMIN_PASSWORD: "{{ admin_password }}"
    setsite_env:
      PROJECT_DIR: "{{ project_dir }}"
      WEBSITE_DOMAIN: "{{ website_domain }}"

  tasks:
    - name: start the postgres container
      docker:
        image: postgres:9.4
        name: postgres
        publish_all_ports: True
        env:
          POSTGRES_USER: "{{ database_user }}"
          POSTGRES_PASSWORD: "{{ database_password }}"
    - name: capture database ip address and mapped port
      set_fact:
        database_host: "{{ docker_containers[0].NetworkSettings.IPAddress }}"
        mapped_database_port: "{{ docker_containers[0].NetworkSettings.Ports['5432/tcp'][0].HostPort}}"
    - name: wait for database to come up
      wait_for: host={{ database_host }} port=5432
    - name: initialize database
      docker:
        image: lorin/mezzanine:latest
        command: python manage.py {{ item }} --noinput
        env: "{{ mezzanine_env }}"
        detach: False
      with_items:
        - syncdb
        - migrate
      register: django
    - name: debug manage result
      debug: msg="ret={{ django }}"
    - name: set the site id
      docker:
        image: lorin/mezzanine:latest
        command: /srv/scripts/setsite.py
        env: "{{ setsite_env.update(mezzanine_env) }}{{ setsite_env }}"
        detach: False
    - name: set the admin password
      docker:
        image: lorin/mezzanine:latest
        command: /srv/scripts/setadmin.py
        env: "{{ setadmin_env.update(mezzanine_env) }}{{ setadmin_env }}"
        detach: False
    - name: start the memcached container
      docker:
        image: lorin/memcached:latest
        name: memcached
    - name: start the mezzanine container
      docker:
        image: lorin/mezzanine:latest
        name: mezzanine
        env: "{{ mezzanine_env }}"
        links: memcached
    - name: start the mezzanine cron job
      docker:
        image: lorin/mezzanine:latest
        name: mezzanine
        env: "{{ mezzanine_env }}"
        command: cron -f
        detach: False
    - name: start the cert container
      docker:
        image: lorin/certs:latest
        name: certs
    - name: run nginx
      docker:
        image: lorin/nginx-mezzanine:latest
        ports:
          - 80:80
          - 443:443
        name: nginx
        volumes_from:
          - mezzanine
          - certs
        links: mezzanine
```
简单说明几点：

- 1）这里用到的docker模块主要是启动容器以及运行容器的一些初始化命令。如果要设置docker容器的端口映射，可以用`ports`参数，如nginx容器。
- 2）挂载数据卷可以直接用 `volumes_from` 指定数据卷的名字即可。
- 3) 要关联各个容器，可以用`links`参数。使用了links参数后，会在对应容器的`/etc/hosts`文件中加入一条ip和域名对应的记录，比如`mezzanine 172.17.0.12`这样。
- 4）有几个容器带有command的必须设置`detach=False`，因为detach参数默认为True，这样会导致容器在后台运行，这个时候去运行command里面的命令是会出错的。
- 5）postgres容器用到了`publish_all_ports: True`，而mezzanine并没有使用这个参数，是因为我们在mezzanine的Dockerfile里面已经有`EXPOSE 8000`指定了暴露的端口为8000，而postgres用的是一个官方的镜像，我们并没有设置端口，所以用了`publish_all_ports`去允许容器中的任意端口暴露。

要测试的话，先是在dockerfiles目录下面创建这几个镜像文件，然后运行 `ansible-playbook run-mezzanine.yml`即可启动容器和跑起来各个服务。查看容器的命令是 `docker ps`，进入容器的命令是 `docker exec -it xxx /bin/bash`，xxx是容器ID或者容器名。更多容器的基本操作请参考 <https://yeasy.gitbooks.io/docker_practice/content/>。

# 参考资料
- 《Ansible_Up-And-Running》
- [Ansible Modules](http://docs.ansible.com/ansible/list_of_all_modules.html)
