---
layout:     post
title:      "Ansible(1) 超详细使用指南"
subtitle:   ""
date:       2016-11-19 16:49:00 +0800
author:     "ssj"
header-img: "img/post/ansible-spec.png"
categories: 自动化运维
catalog: true
comments: true
tags:
    - Ansible
---

本文最早发于 [简书](https://www.jianshu.com/p/f0cf027225df)。

> 在项目中有很多地方用到ansible。最初使用ansible只是为了方便代码部署和模板配置，毕竟手动去30+台机器手动部署代码是件很痛苦的事情。由于ansible基于ssh实现其功能，配置使用都比较简单，后来在线上的系统环境搭建，数据库权限统一配置中ansible都发挥了重要作用，极大的提高了生产力。这里整理了一份详尽的使用说明，有兴趣的可以瞅瞅。文章内容主要翻译整理自ansible官方网站推荐的`Ansible-Up and Running`一书。

# 1 为什么选择Ansible
来源：ansible一词源于科幻小说，是一种超光速通信设备。
Ansible is the simplest way to automate apps and IT infrastructure。
750+模块，24000+ github stars。

![图1 ansible的使用场景](http://upload-images.jianshu.io/upload_images/286774-0766d1bb5cdeb08f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

配置管理、应用部署等。配置管理工具有Chef, Puppet, Salt等，应用部署（将代码编译或打包然后传输到服务器部署并启动服务）工具有Capistrano，Fabric等，ansible集两者于一身，操作很简单但是功能强大。此外，还可以对多个服务器进行服务编排，支持openstack，amazon ec2， docker等。

ansible使用了一个DSL（domain-specific language）描述服务器状态。执行的文件称为playbook，文件格式为yaml。ansible简约而不简单。比起puppet的繁琐的配置和复杂语法( [Puppet基础篇4-安装、配置并使用Puppet | Puppet运维自动化经验分享](http://kisspuppet.com/2014/03/08/puppet_learning_base4/) )，简直是一股清流。  图2描述了ansible执行过程，执行了两个task和一个handler，先是使用了一个apt模块在web1，web2，web3上面执行了安装nginx的任务，再是用template模块拷贝了配置文件。另外，执行了一个notify nginx的handler重启了nginx。

![图2 ansible执行流程示意图](http://upload-images.jianshu.io/upload_images/286774-ae10632a49bcb37b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行流程：

```
1. 创建一个python脚本用于安装nginx包。
2. 拷贝python脚本到web1，web2，web3。
3. 分别在web1，web2，web3上执行该脚本。
4. 等待脚本在所有服务器上执行完毕。
5. 接着执行下一个task。
```
注意的几点：
- 1.在各个服务器执行脚本的过程是并行的，有个forks参数可以指定，默认是5，即一次可以在5个服务器上并行执行脚本。
- 2.要在所有的服务器都执行完第一个task后才会接着执行第二个task。（新版本新增了异步参数，一个服务器在执行完了它的任务后可以不等其他服务器执行完直接执行下一个task）。
- 3.ansible执行任务顺序与playbook中的顺序一致。

优势：
- 语法易读。yaml->json好比markdown->html。ansible的playbook可以被称之为可以执行的README。
- 远程主机不需要安装任何东西。（这有点夸大了，python2.5+或python2.4+simplejson模块和ssh是必须的，当然这些现在是Linux服务器标配）
- push-based。如chef和puppet是pull-based，先将文件修改推送到中心服务器，其他服务器的agent定期拉取新的配置管理脚本并在本机执行。而在ansible是push-based的，先在中心服务器修改playbook，执行该playbook，ansible会连接到各个服务器并执行模块改变服务器状态。push-based的好处就是只在需要的时候才操控去改变目标服务器状态。如果你更倾向于pull-based模式，可以用ansible-pull。
- ansible可以很方便的scaled down，单机跟多机没有什么区别。 Simple things should be simple, complex things should be possible 。
- 很轻量级的抽象。不像puppet之类的工具，有很高的抽象，比如有package这个概念，用于不用区分服务器版本来安装模块。但是在ansible中，提供的是apt和yum模块，由你自己采用，不要再额外学一些抽象的语法，简化你的学习成本。也有人觉得这是ansible的缺点，优缺点与否，各有评判。

# 2 安装配置
## 2.1 安装
`pip install ansible`
依赖环境：python

## 2.2 配置
配置ansible.cfg文件，ansible配置文件寻找路径：

```
1. File specified by the ANSIBLE_CONFIG environment variable 
2. ./ansible.cfg (ansible.cfg in the current directory)
3. ~/.ansible.cfg (.ansible.cfg in your home directory)
4. /etc/ansible/ansible.cfg
```

ansible.cfg配置文件实例

```
[defaults]
hostfile=/etc/ansible/hosts
private_key_file = /Users/ssj/.ssh/id_rsa_ansible
remote_user = ssj
remote_port = 22 
host_key_checking = False
```
注意，如果是在服务器上，不要放置private key，可以通过ssh forward。

## 2.3 测试
简单执行命令测试是否成功 ( -vvvv可以看到更多细节信息)，”changed”:false表示执行ping模块没有改变服务器状态，”ping”:pong表示模块执行后输出结果为pong。你也可以将ping模块改成command，加上参数执行指定命令。比如 `ansible testserver -m command -a uptime` ，当然，command是默认模块，因此还可以简化为 `ansible testserver -a uptime` 。

```
#hosts
[testserver]
127.0.0.1

#run command，-i hosts可以省去。
ssj@ssj-mbp ~/ansible $ ansible testserver -i hosts -m ping
127.0.0.1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

# 3 实体关系图

![图3 实体关系图](http://upload-images.jianshu.io/upload_images/286774-0c86751dcac8914e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- playbook包含很多个play
- play中包含name，tasks，hosts，vars，handles属性。
- tasks中包含各个真正执行的module，如apt，copy，file， git， svn，service，command，notify，mysql等。具体的模块参数和使用文档在[这里](http://docs.ansible.com/ansible/modules_by_category.html)

# 4 一个例子
```
---
- name: Configure webserver with nginx and tls
  hosts: webservers
  sudo: True
  vars:
    key_file: /etc/nginx/ssl/nginx.key
    cert_file: /etc/nginx/ssl/nginx.crt
    conf_file: /etc/nginx/sites-available/default
    server_name: localhost
  tasks:
    - name: Install nginx
      apt: name=nginx update_cache=yes cache_valid_time=3600
    - name: create directories for TLS certificates
      file: path=/etc/nginx/ssl state=directory
    - name: copy TLS key
      copy: src=files/nginx.key dest={{ key_file }} owner=root mode=0600
      notify: restart nginx
    - name: copy TLS certificate
      copy: src=files/nginx.crt dest={{ cert_file }}
      notify: restart nginx
    - name: copy nginx config file
      template: src=templates/nginx.conf.j2 dest={{ conf_file }}
      notify: restart nginx
    - name: enable configuration
      file: dest=/etc/nginx/sites-enabled/default src={{ conf_file }} state=link
      notify: restart nginx
    - name: copy index.html
      template: src=templates/index.html.j2 dest=/usr/share/nginx/html/index.html mode=0644
  handlers:
    - name: restart nginx
      service: name=nginx state=restarted
```

- 可以看到用到了apt，file，copy，template，notify，service等模块。
- 注意几个语法点：

```
YAML truthy
true, True, TRUE, yes, Yes, YES, on, On, ON, y, Y
 
YAML falsey
false, False, FALSE, no, No, NO, off, Off, OFF, n, N
```

- true和yes，on或者1都是一样的意思，一般在模块参数里面用yes和no，true和false在playbook中其他地方。
- 另外，比如下面的模块参数分行写，可以在第一行写 > ， 后面几行跟参数来实现。
- 注意notify是严格按照它在play中定义的顺序执行的，而不是notify调用的顺序执行的。比如下面的playbook，尽管先notify的是 `handler test2` ，实际执行时时按照play中handlers定义的顺序，也就是先执行 `handler test1`。


```
#!/usr/bin/env ansible-playbook
- name: test handlers
  hosts: webservers
  tasks:
    - name: assure file exist
      file: >
        path=/tmp/test.conf
        state=touch owner=ssj mode=0644

    - name: task1
      command: date
      notify: handler test2

    - name: task2
      command: echo 'task2'
      notify: handler test1

  handlers:
    - name: handler test1
      command: echo 'handler test1'
    - name: handler test2
      command: echo 'handler test2'
```

# 5 更多细节
## 5.1 inventory格式和配置
### inventory格式
```
[webserver]
127.0.0.1

dbserver1 ansible_ssh_host=127.0.0.1 ansible_ssh_port=22 color=red
dbserver2 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200 color=green

www.example.com

[dbserver] #group
dbserver1
dbserver2

[forum:children] #groups of groups
webserver
dbserver
```

可以用分组的方式，可以直接用域名(www.example.com)，也可以用别名(如testserver2)+变量指定ssh的ip地址和端口，比如ansible_ssh_host和color变量。命令 `ansible testserver2 -a date` ，通常我们要控制多台服务器，于是可以分组服务器，要在所有服务器执行可以用all。 `ansible all -a date`。

inventory除了可以指定主机的变量如上面的color之外，还可以将变量分组，也可以对主机变量单独存储到一个文件中，格式如下，注意如果host_vars中和group_vars中有相同变量，则以host_vars中的为准。host_vars变量只能本主机使用，group_vars是本group都可以使用。

```
############################
group_vars/dbserver
-----------------------
db:
   user: bbsdbuser
   password: bbsdbpasswd
   port: 3306
   name: bbs
   replica:
       host: slavedb
       port: 3307

host_vars/dbserver1
---------------------
db:
   user:server1dbuser
   password: server1password
   master: true
############################

ssj@ssj-mbp ~/ansible $ ansible dbserver1 -i hosts -a 'echo {{db.user}}' #host_vars优先级高
dbserver1 | SUCCESS | rc=0 >>
server1dbuser
----------------------------------------------------------
ssj@ssj-mbp ~/ansible $ ansible dbserver2 -i hosts -a 'echo {{db.master}}' #dbserver2所在的组变量文件没有db.master变量，报错。
dbserver2 | FAILED | rc=0 >>
the field 'args' has an invalid value, which appears to include a variable that is undefined. The error was: 'dict object' has no attribute 'master'
```
甚至支持：
```
[web]
web[1:20].example.com 
web-[a-t].example.com
```

inventory文件还支持动态的，通过 `-i inventory` 可以指定目录或者文件，这样目录下面可以放一个python脚本，用来动态获取主机列表。python脚本要可执行，同时实现下面两个命令：

```
• --host=<hostname> for showing host details
• --list for listing groups
```

最后，还可以通过add_hosts模块在运行时增加host配置，使用group_by模块在运行时创建group。比如通过 ansible_distribution来根据操作系统创建不同的组，再分别安装软件。

```
- name: group hosts by distribution
  hosts: myhosts
  gather_facts: True
  tasks:
    - name: create groups based on distro
      group_by: key={{ ansible_distribution }}

- name: do something to Ubuntu hosts
  hosts: Ubuntu
  tasks:
    - name: install htop
      apt: name=htop
# ...
- name: do something else to CentOS hosts
  hosts: CentOS
  tasks:
    - name: install htop
      yum: name=htop
# ...
```

### inventory默认配置
![图4 inventory参数配置](http://upload-images.jianshu.io/upload_images/286774-320d6cd7b762dd65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

几个参数解释下：

	- ansible_connection: ssh连接方式，默认是smart，也就是看本地机器是否安装了ssh客户端且支持ControlPersist特性。如果支持，则使用本地的ssh客户端，如果不支持，则使用一个基于python的ssh客户端库paramiko。
	- ansible_shell_type: ansible认为的远程服务器执行script的shell，默认认为是/bin/sh，当然也可以设置为csh，fish等，如果服务器支持的话。
	- ansible_python_interpreter: 服务器python解释器的路径。如果你的服务器python解释器不在这个目录，这要修改该配置。
	- ansible_*_interpreter: 如果用的是一个自定义的模块，不是python的，比如ruby，则设置该值指定解释器路径（比如/usr/bin/ruby）。

## 5.2 变量和Facts
### 变量
变量可以在play中通过vars来指定，也可以通过var_file指定一个文件，文件中存储变量。如之前的nginx的playbook可以改成这样：

```
vars_files:
     - nginx.yml

##nginx.yml文件内容
key_file: /etc/nginx/ssl/nginx.key
cert_file: /etc/nginx/ssl/nginx.crt
conf_file: /etc/nginx/sites-available/default
server_name: localhost
```

可以在play中使用debug模块打印变量的值，注意debug支持的参数有var，msg等，var中的变量不要使用 `{{}}` 包裹。

```
#!/usr/bin/env ansible-playbook
- name: test name
  hosts: webserver 
  vars:
    myvar: testmyvar
  tasks:
    - debug: var=myvar
    - name: capture output of id command
      command: id -un
      register: login
    - debug: var=login
```

使用register来注册一个变量后面使用，register注册的变量在这个playbook的其他play中也是可以使用的，不局限于这一个play。比如command模块的输出如下所示，可以通过login.stdout得到用户名。注意不同模块的输出可能是不一样的，同一个模块在不同情况下也不一样，比如apt模块安装nginx，如果机器已经安装了nginx，则输出里面change为false，而且不会有stdout，stderr和stdout_lines这些key。如果模块执行出错，则其他的host默认不会再执行，可以设置 `ignore_erros:True` 忽略模块的错误。

其他指定变量的方式如 host_vars目录，group_vars目录等。

```
{
        "changed": true, 
        "cmd": [
            "id", 
            "-un"
        ], 
        "delta": "0:00:00.007369", 
        "end": "2016-11-17 15:09:49.061725", 
        "rc": 0, 
        "start": "2016-11-17 15:09:49.054356", 
        "stderr": "", 
        "stdout": "ssj", 
        "stdout_lines": [
            "ssj"
        ], 
        "warnings": []
}
```

### Facts
如果在playbook中配置了 gather_facts:True，则会看到真正的任务开始前，会先执行一个[setup]的模块，用于收集服务器信息，包括cpu架构，操作系统类型，ip地址等信息。这些信息存储在特定的变量中，我们称之为facts。如果你的playbook中不需要这些信息，也可以设置gather_facts:False来加快playbook执行速度，收集服务器信息需要花费不少时间的。

通过命令 `ansible webserver -m setup` 可以看到ansible的gathter_facts的输出内容，软硬件信息都有。因为信息太多，还可以通过在setup模块加上参数filter来筛选你需要的内容，如果只需要网络信息，可以这样： `ansible webserver -m setup -a 'filter=ansible_eth*’` ，其中ansible_facts这个key是固定的。

```
127.0.0.1 | SUCCESS => {
    "ansible_facts": {
        "ansible_eth0": {
            "active": true, 
            "device": "eth0", 
            "ipv4": {
                "address": "xxx.xxx.xxx.xxx", 
                "broadcast": "xxx.xxx.xxx.xxx",
                "netmask": "xxx.xxx.xxx.xxx.xxx", 
                "network": "xxx.xxx.xxx.xxx"
            }, 
            "macaddress": "xx.xx.xx.xx.xx.xx", 
            "module": "bnx2", 
            "mtu": 1500, 
            "pciid": "0000:10:00.0", 
            "promisc": false, 
            "type": "ether"
        },
    }, 
    "changed": false
}
```

可以在定义本地facts，在 `/etc/ansible/facts.d/` 目录新建example.fact文件，内容如下：

```
[book]
title=Ansible: Up and Running
author=Lorin Hochstein
publisher=O'Reilly Media
```
然后在运行playbook的时候就可以通过 ansible_local读取这些变量了。

另外，还可以通过 `set_fact` 模块设置变量，比如之前得到了一个命令的输出，register到一个变量，然后把我们需要的变量提取出来用set_fact存储到另外一个变量中，简化了变量的引用。

```
- name: test name
  hosts: webserver 
  gather_facts: True
  tasks:
    - name: print ansible_local
      debug: var=ansible_local

    - name: capture output of id command
      command: id -un 
      register: login
      ignore_errors: True

    - set_fact: loginuser={{ login.stdout }}

    - name: show login user
      debug: var=loginuser
```

### 内置变量
![图5 内置变量](http://upload-images.jianshu.io/upload_images/286774-a140cfd7ca249470.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 命令行传递变量
还可以在运行playbook的时候在命令行传递变量。如果要传递一个包含变量的文件，可以用 `$ ansible-playbook greet.yml -e @greetvars.yml` 。

```
- name: pass a message on the command line
  hosts: localhost
  vars:
    greeting: "you didn't specify a message"
  tasks:
    - name: output a message
      debug: msg="{{ greeting }}"

$ ansible-playbook greet.yml -e greeting=hiya
```

### 变量优先级
ansible在很多地方可以设置变量，尽量不要重名。优先级由高到低如下：
- 命令行的参数， 上面的 `-e greeting=‘hello’` 。
- host_vars, group_vars中的变量，不管是在inventory中还是yaml文件中定义的。
- Facts变量
- role目录下的 `defaults/main.yml` 。

**对于role的特别说明一下，变量优先级最高的是命令行的参数变量，接着是playbook中调用role时传递的变量，再是`role/vars/main.yml`，然后再是play中的vars定义的变量，最后是` role/defaults/main.yml`中的变量。**如下面所示的playbook，除了运行时通过-e传递的命令行变量优先级最高外，余下的 `2 > 3 >1 > 4`。
```
#!/usr/bin/env ansible-playbook
- name: test name
  hosts: dbserver1
  vars:
    testvar: vars var #1
  roles:
    - role: database
      testvar: role var #2

#####database/vars/main.yml
testvar: vars.main.yml var #3

####database/defaults/main.yml
testvar: defaults.main.yml var #4
```
   

## 5.3 playbook要点
### 实用模块
- 如果想在控制机器而不是远程机器运行命令，可以用local_action。
- 如果机器没有启动起来，需要先等待机器启动再执行play，用wait_for模块。

```
- name: Deploy mezzanine
  hosts: web
  gather_facts: False
   # vars & vars_files section not shown here 
  tasks:
		- name: wait for ssh server to be running
		  local_action: wait_for port=22 host="{{ inventory_hostname }}" search_regex=OpenSSH
		- name: gather facts
		  setup:
```
- 如果不想一次在所有的hosts都执行，可以设置serial参数来设置每次执行几个host，比如升级服务器，我们不想影响服务，会一个一个跑。可以设置max_fail_percentage来指定最大失败的比率，比如设置为25%，则如果有4台机器，有2台任务执行失败则终止整个play，其他任务不再执行。

```
 - name: upgrade packages on servers behind load balancer
   hosts: myhosts
   serial: 1
   max_fail_percentage: 25
   tasks:
        # tasks go here
```

### 加密数据

一些数据如db密码等不能直接提交到代码库，可以提交一个加密的版本。加密文件可以用ansible-vault工具。运行playbook的时候加上参数

```
ansible-vault encrypt secrets.yml
ansible-vault create secrets.yml
ansible-vault view secrets.yml

$ ansible-playbook test.yml --ask-vault-pass
$ ansible-playbook mezzanine --vault-password-file ~/password.txt
```

### hosts格式

可以用冒号:表示合并服务器组，:& 求交集等。
指定运行的hosts可以在命令行加上 `—limit` 。
`ansible-playbook -l 'staging:&database' playbook.yml`
`ansible-playbook —limit 'staging:&database' playbook.yml`
![图6 hosts格式](http://upload-images.jianshu.io/upload_images/286774-7fc53bc92f9a3df3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Filters

- filter可以用在很多方面，比如默认值filter。如果database_host没有定义，则HOST的值设置为localhost。

	```
		"HOST": "{{ database_host | default('localhost') }}"
	```

- 针对任务返回值的filter。可以的取值有 failed，changed，skipped，success等。
	`failed_when: result|failed` 
- 文件路径的filter。
	
	```
	 vars:
    	homepage: /usr/share/nginx/html/index.html
 	 tasks:
  	 - 	name: copy home page
      	copy: src=files/{{ homepage | basename }} dest={{ homepage }}
	```
- 自定义filter。
	写一个自定义的filter，放在项目的 filter_plugins 目录下即可。下面是一个用于字符串分割的filter模块，使用时使用filter语法即可。
	
	```
	from ansible import errors
	def split_string(string, seperator=' '):
    	try:
        	return string.split(seperator)
    	except Exception, e:
        	raise errors.AnsibleFilterError('split plugin error: %s, string=%s' % str(e),str(string) )

	class FilterModule(object):
	    def filters(self):
	        return {
	            'split' : split_string,
	        }
	```

### lookups

查找变量可以通过lookup实现，支持file，redis，pipe，cvsfile等多种格式。（redis的需要安装python的redis模块）

### 复杂循环

- with_items
- with_lines
- with_fileglob
- with_dict
- ...

### debug你的playbook
```
检查语法：ansible-playbook --syntax-check playbook.yml 
查看host列表：ansible-playbook --list-hosts playbook.yml
查看task列表：ansible-playbook --list-tasks playbook.yml
检查模式(不会运行): ansible-playbook --check playbook.yml
diff模式(查看文件变化)： ansible-playbook --check --diff playbook.yml
从指定的task开始运行：ansible-playbook --start-at-task="install packages" playbook.yml
逐个task运行，运行前需要你确认：ansible-playbook --step playbook.yml
指定tags：ansible-playbook --tags=foo,bar playbook.yml
跳过tags：ansible-playbook --skip-tags=baz,quux playbook.yml
```

# 6 角色(Roles)
## 6.1 角色基本结构
roles可以简化playbook编写，让playbook更清晰和方便复用。一个名为database的role的目录结构如下，这些目录都是可选的，如果你的角色没有任何handler，则不需要handlers目录。roles的查找路径默认是`/etc/ansible/roles`，也可以在 `/etc/ansible/ansible.cfg`的`roles_path`中设置。

```
roles/database/tasks/main.yml
Tasks

roles/database/files/
Holds files to be uploaded to hosts

roles/database/templates/
Holds Jinja2 template files

roles/database/handlers/main.yml
Handlers

roles/database/vars/main.yml
Variables that shouldn’t be overridden

roles/database/defaults/main.yml
Default variables that can be overridden

roles/database/meta/main.yml
Dependency information about a role
```

## 6.2 pre\_tasks和post\_tasks
在角色执行任务前做一些前置工作，任务执行完后做一些后置处理。

```
- name: test 
  hosts: dbserver
  vars_files:
    - secrets.yml
  pre_tasks:
    - name: update the apt cache
      apt: update_cache=yes
  roles:
    - role: database
  post_tasks:
    - name: send email
      command: xxx
```

## 6.3 依赖角色
如果怕遗漏一些任务，比如设置ntp之类的，可以使用依赖角色的功能。这样在执行你的角色任务时会先执行依赖角色。

```
roles/database/meta/main.yml
dependencies:
    - { role: ntp, ntp_server=ntp.ubuntu.com }
```

## 6.4 Ansible Galaxy
可以通过ansible galaxy工具方便的创建一个角色目录。

```
ansible-galaxy init -p playbooks/roles database
```
如果不用-p指定路径，那么默认是会在当前目录创建角色的目录结构。创建后的目录结构如下：

```
playbook/roles/database
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```
ansible galaxy还是一个开源的角色库，你可以在其中下载到许多其他人写好的角色代码或者提交自己的角色代码。角色仓库的使用说明在[这里](https://galaxy.ansible.com/intro)。

##6.5 Ansible Tower
ansible tower是ansible公司提供的一套商用的web管理平台，也有试用版本，还没有试用过，后续使用了再补充。

# 7 加速你的ansible
## 7.1 SSH ControlPersist

```
ControlMaster auto
ControlPath $HOME/.ansible/cp/ansible-ssh-%h-%p-%r 
ControlPersist 60s
```

## 7.2 Pipelining（ansible的特性)
ansible通常执行的原理是在 ~/.ansible下面创建一个临时目录(通过ssh)，然后通过sftp或者scp拷贝python脚本到临时目录，然后执行这个脚本代码（再次通过ssh）。使用pipeline可以使得这些操作只要一个ssh连接去执行python脚本。即便是开启了ControlPersist，这个性能提升也很可观。在配置文件中加入`pipelining=true`即可开启。

需要注意的是，开启pipeling要关闭服务器的requiretty功能。增加文件`/etc/sudoers.d/disable-requiretty`，其中的内容为 `Defaults:ansibleuser !requiretty`，ansibleuser为你的用户名。

## 7.3 Fact Cache
如果你不需要用到服务器信息，可以关闭获取fact的功能，这样也可以加快playbook的执行效率。配置文件中加入 `gathering = explicit`即可，这样你要获取服务器信息，需要显示的在play中指定。

如果要用到fact信息，可以使用fact缓存，这样每个机器的fact信息只会获取一次而不是每次都去获取。fact缓存支持json，redis，memcached。如果用redis则需要在控制机上安装python的redis模块，自然redis也是要安装的。

```
[defaults]
gathering = smart
# 24-hour timeout, adjust if needed
fact_caching_timeout = 86400
# You must specify a fact caching implementation

# JSON file implementation
fact_caching = jsonfile //或者redis，memcached
fact_caching_connection = /tmp/ansible_fact_cache
```

## 7.4 Parallelism
可以设置`ANSIBLE_FORKS`环境变量或者在配置文件加上`forks=n`来指定并行执行的host的数目。

## 7.5 关于异步
ansible的1.7版本开始增加了异步参数 async，也就是说执行一个时间很长的任务时，可以不用等待它结束，而是直接先执行后面的任务，在后续的play中定时检查任务执行结果即可。

有几点注意一下，一个是async参数，是指任务执行的超时时间，如果这个时间设置的比任务执行时间短，则任务会超时失败。poll值为轮询任务状态的时间间隔，如果设置为0，表示启动并忽略，也就是说设置为0才是真正的开始异步执行，也就是直接执行后面的task，而为了知道异步任务执行的结果，可以用async_status来实现。如果poll设置为非0值，则还是阻塞执行的，并非异步。

```
- hosts: dbserver
  tasks:
  - name: simulate long running op (15 sec), wait for up to 45 sec, poll every 5 sec
    command: /bin/sleep 15
    async: 20
    poll: 0
    register: asynctest

  - name: check async status
    async_status: jid="{{ asynctest.ansible_job_id }}"
    register: job_result
    until: job_result.finished
    retries: 30
    delay: 2
```

# 8 创建自定义模块
在某些情况下，可能ansible自带的模块不能满足你的需求，需要自定义模块。可以通过python或者bash来写自定义模块，符合ansible的模块编写标准即可，[这里有很详细的文档](http://docs.ansible.com/ansible/dev_guide/developing_modules.html)。

# 9 Docker
docker是目前很火爆的技术，它提供了一套远程API供第三方程序调用，ansible的docker模块就是使用了这套API对docker操作。ansible用在docker上主要有两点：一是编排docker容器。通常一个系统需要很多个docker容器来支持，每个容器都运行一个服务。服务之间需要相互通信，同时你也要保证容器启动的顺序等，原生docker并没有这些工具支持，ansible则是非常合适的一个选择。二是创建镜像。官方的方式是通过Dockerfile来创建镜像，但是通过ansible来实现更加简单方便。

基于docker的应用的生命周期是这样的：

```
1. 在本地机器创建docker镜像。
2. 将docker镜像push到registry。
3. 远程机器上将镜像从registry上pull下来。
4. 在远程机器上启动容器。
```
使用ansible之后，则是下面这样的：

```
1. 写好用来创建docker镜像的playbook。
2. 运行playbook来创建镜像。
3. 将docker镜像推送到registry。
4. 写好一个拉取docker镜像并启动容器的playbook。
5. 执行playbook拉取和启动容器。
```
关于ansible搭建一个完整实例和docker化将在下一篇文章中详细描述，敬请期待。

# 参考资料
- 《Ansible_Up-And-Running》
