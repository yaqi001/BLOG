# Ansible —— 安装篇
  
   目录:
   - [Prerequisites](#Prerequisites)
   - [开始安装](#startInstall)
      - [编译源码安装](#way1)
      - [软件安装](#way2)

最近在测试 oVirt 3.6 的 CinderGlance Docker Integration 这个特性，但是可以安装 ovirt-engine 的两种操作系统 CentOs 7 和 Fedora 22 上下载 Glance 和 Cinder 镜像的时候都有一些问题，还好目前还有一种手动安装镜像的方式，但是用到了 Ansible。于是就掉进了另一个“坑”中。如是说，什么事情从0到一都是艰难的。但我感觉 Ansible 的挑战更大了一点。好吧，就从安装开始吧。^_^

<h2 id="Prerequisites">Prerequisites</h2>

* 控制节点端

  在这里我们称安装了 Ansible 的机器为 Control Machine。既然是在学习 Ansible，那么系统一定是 Linux 吧，这个不用多说。必要的前提条件就是需要该台控制节点安装了 Python 2.6 或 2.7。（BTW：Ansible 就是用 Python 写的。）


* 被管理节点端

  好了，我们已经说完控制节点安装的前提条件，下面就是被管理节点的安装条件了。既然要被管理（也就是要和控制节点进行通信），那么必须有一种通信工具才可以。Ansible 里推荐使用 SSH；当然在被管理节点中也需要安装 Python，版本在 2.4 及以上既可。

<h2 id="startInstall">开始安装</h2>
   这里我用了两台机器进行测试的，分别是 CentOS 7 & Fedora 22，仅举一例进行说明：

<h3 id="way1">编译源码安装</h3>

1. 克隆源码：

   ~~~ bash
   $ git clone https://github.com/ansible/ansible.git --recursive
   ~~~

   `--recursive` 说的是在克隆完成后，使用默认设置对所有子模块进行初始化。相当于在项目克隆完成后立即运行 `git submodule update --init --recursive` 

2. 进入 ansible 目录中
   
   ~~~ bash
   $ cd ansible
   ~~~

3. 为了能够运行 ansible，我们需要执行 `env-setup` 脚本来设置环境变量。

   ~~~ bash
   $ source ./hacking/env-setup
   ~~~
   
   > **BTW**
   > 如果您不想看到太多没用的输出，请执行 `$source ./hacking/env-setup -q`
   下面是脚本中的一些片段，一看便知：

   ~~~ bash
   PREFIX_PYTHONPATH="$ANSIBLE_HOME/lib"  
   PREFIX_PATH="$ANSIBLE_HOME/bin"
   PREFIX_MANPATH="$ANSIBLE_HOME/docs/man"
   # $ANSIBLE 就是您克隆的 ANSIBLE 项目所在的绝对路径。
   ~~~

   ~~~ bash 
   # 下面是设置环境变量
   expr "$PYTHONPATH" : "${PREFIX_PYTHONPATH}.*" > /dev/null || export PYTHONPATH="$PREFIX_PYTHONPATH:$PYTHONPATH"
   expr "$PATH" : "${PREFIX_PATH}.*" > /dev/null || export PATH="$PREFIX_PATH:$PATH"
   expr "$MANPATH" : "${PREFIX_MANPATH}.*" > /dev/null || export MANPATH="$PREFIX_MANPATH:$MANPATH"
   ~~~

   ~~~ bash
   # 在这里还需要打包有个Python egg，不然 pkg_resources 就不能正常工作啦！
   (
        cd "$ANSIBLE_HOME"
        if [ "$verbosity" = silent ] ; then
            gen_egg_info > /dev/null 2>&1
            find . -type f -name "*.pyc" -delete > /dev/null 2>&1
        else
            gen_egg_info
            find . -type f -name "*.pyc" -delete
        fi
        cd "$current_dir"
   )
   # 下面是定义的 gen_egg_info 函数
   gen_egg_info()
   {
       if [ -e "$PREFIX_PYTHONPATH/ansible.egg-info" ] ; then
           \rm -r "$PREFIX_PYTHONPATH/ansible.egg-info"
       fi
       python setup.py egg_info #这个就是打包过程啦！打好的包就在 $ANSIBLE_HOME/lib/ansible.egg-info。这里使用打包格式是 egg_info，欲知更多格式请执行 python setup.py --help-commands。
   }
   ~~~

4. 现在还需要安装一些必备的软件包。如果您不想用系统的包管理器来安装的话，可以通过 pip 来安装。
   
   ~~~ bash
   $ easy_install pip
   $ pip install pyyaml jinja2 nose passlib pycrypto
   ~~~

5. 运行了 env-setup 脚本后，主机列表就默认存在 /etc/ansible/hosts 中了。当然主机列表文件所在的目录是可以自定义的，后面我们会提到。

   ~~~bash
   # 我的 hosts 文件中写的是：
   [fedora22] # 组名
   192.168.9.50  # 被管理机器的 IP 地址
   [centos7]
   192.168.9.58
   ~~~

6. 一切就绪，我们可以来测试一下 ansible 是否安装成功了。
   
   * 利用密码的方式测试：

     ~~~ bash
     $ ansible fedora22 -m ping --ask-pass`  
     SSH password: 
     192.168.9.50 | SUCCESS => {
         "changed": false, 
         "ping": "pong"
     }
     ~~~

   * 利用 SSH keys 的方式测试：

     ~~~ bash
     # 生成公钥 id_rsa.put 和私钥 id_rsa
     $ ssh-keygen
     Enter file in which to save the key (/home/helen/.ssh/id_rsa): 
     Enter passphrase (empty for no passphrase): # 如果您担心私钥的安全性，请为私钥设置口令。 
     Enter same passphrase again: 
     Your identification has been saved in /home/helen/.ssh/id_rsa.
     Your public key has been saved in /home/helen/.ssh/id_rsa.pub.
     The key fingerprint is:
     c8:2b:f7:f1:4f:95:84:73:bf:91:ac:a5:02:89:e4:3a helen@yingyun.36
     The key's randomart image is:
     +--[ RSA 2048]----+
     |                 |
     |             .   |
     |       .    o o  |
     |     .o.. .  +.o.|
     |      ooSo    o=.|
     |      ..  .  .+ o|
     |    .Eo .  ..o . |
     |     o.. o ..    |
     |        . ...    |
     +-----------------+
     ~~~
     
     ~~~ bash
     # 将本地的公钥传到目的机器（192.168.9.50）中。
     $ ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.9.50
     /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
     /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
     root@192.168.9.50's password: 

     Number of key(s) added: 1

     Now try logging into the machine, with:   "ssh 'root@192.168.9.50'"
     and check to make sure that only the key(s) you wanted were added.
     ~~~
   
     ~~~ bash
     # ssh 到目的机器
     [helen@yingyun ansible]$ ssh 'root@192.168.9.50' 
     Last login: Tue Nov  3 12:13:02 2015 from 192.168.9.56  # 无密码登录设置成功！
     [root@yingyun001 ~]# 
     ~~~ 

     ~~~ bash
     # 仍然是将本地公钥传送到目的机器，不过这次我们用 helen 用户，为什么用另一个用户，一会儿就知道啦。
     $ ssh-copy-id -i ~/.ssh/id_rsa.pub helen@192.168.9.58
     /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
     /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
     helen@192.168.9.50's password: 

     Number of key(s) added: 1

     Now try logging into the machine, with:   "ssh 'helen@192.168.9.50'"
     and check to make sure that only the key(s) you wanted were added.
     ~~~     
 
     ~~~ bash
     # 所有主机都用 root 用户来 ping
     $ ansible all -m command -a "uname -r" -u root
     192.168.9.58 | UNREACHABLE! => {
         "changed": false, 
         "msg": "ERROR! SSH encountered an unknown error during the connection. We recommend you re-run the command using -vvvv, which will enable SSH debugging output to help diagnose the issue", 
         "unreachable": true
     }
     192.168.9.50 | SUCCESS | rc=0 >>
     3.10.0-123.el7.x86_64
     ~~~

     ~~~ bash
     # 所有主机都用 helen 用户来 ping，和上面的对比后您应该明白 -u 的用法了吧！
     $ ansible all -m command -a "uname -r" -u helen
     192.168.9.58 | UNREACHABLE! => {
         "changed": false, 
         "msg": "ERROR! SSH encountered an unknown error during the connection. We recommend you re-run the command using -vvvv, which will enable SSH debugging output to help diagnose the issue", 
         "unreachable": true
     }
     192.168.9.50 | SUCCESS | rc=0 >>
     3.10.0-123.el7.x86_64
     ~~~          
     
     > **参数说明**
     >
     > `-m` —— 用于执行后面指定的模块，这里我们使用的是 `command`，当然也有其它的模块例如：`raw`，`shell` 以及 `script` 等等，您可以使用 `ansible-doc -l` 来查看当前 ansible 支持的所有模块。这里的 `command` 还可以替换成 `raw`，它俩唯一的区别就是 `raw` 支持管道，`command` 则不支持。

     > `-a` —— 指的是 Ad-Hoc 命令。官网对 Ad-Hoc 的定义是：Ad-Hoc 是一种可以快速执行的指令，且没有状态。
        
     > `-u` —— 使用 `-u` 后面指定的用户登录到被管理节点。需要注意的是：如果您使用的是 SSH keys 的方式登录的，请确保该用户接收到了来自控制节点的公钥。

###<h3 id="way2">软件安装</h3>
     
   * 确保您已经安装了 epel 源，然后进行 ansible 的安装。
    
     ~~~ bash
     $ sudo yum -y install ansible
     ~~~
      
   * 可以自己构建出 rpm 包，然后进行编译。
     1. `git clone git://github.com/ansible/ansible.git --recursive`

     2. `cd ./ansible`

     3. `make rpm`

     4. `sudo rpm -Uvh ./rpm-build/ansible-*.noarch.rpm` # 记得要安装依赖软件：`pyyaml` `jinja2` `nose` `passlib` `pycrypto`。

     5. 然后就可以用前面将到的方法来测试啦。
