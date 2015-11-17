# 概述：Ansible 是如何工作的？
 
原文链接：[http://www.ansible.com/how-ansible-works](http://www.ansible.com/how-ansible-works)

目录:
- [什么是 Ansible？](#what)
- [有效的体系架构](#arch)
   - [ssh 密钥是您的朋友](#ssh)
   - [在简易的文本文件中管理您的主机清单](#inventory)
   - [PlayBooks：一个简洁但很强大的自动化语言](#playbooks)
   - [一个 app_config.yml 文件的示例](#yaml)
   - [扩展 Ansible：模块，插件和 API](#extension)

<h2 id="what">什么是 ANSIBLE？</h2>

Ansible 是一个十分简单的 IT 自动化工具，它可以自动实现[云资源配置](2015-10-22-Ansible 之云资源配置.md)，[配置管理](2015-10-22-ANSIBLE 之配置管理.md)，[应用程序配置](2015-10-22-ANSIBLE 之应用程序部署.md)，[内部服务编配](2015-10-22-Ansible 之编配.md) 还可以满足其它与 IT 有关的需求。

Ansible 从一开始就被设计为多层部署，它通过描述您系统的内容关系来模拟您的 IT 架构，而不仅仅是采用在某一时刻来管理某一个系统的方式。

Ansible 没有代理也没有额外的自定义安全架构，所以部署起来非常简单。最重要的一点就是 Ansible 使用了非常精炼的语言（YAML，以 Ansible Playbook 语言的形式），它允许您用通俗易懂的语言来描述您的自动化任务。

通过此文，我们将会让您快速的了解 Ansible，想了解更多的 Ansible 知识，请见：[http://docs.ansible.com/](http://docs.ansible.com/)。

<h2 id="arch">有效的体系架构</h2>

Ansible 是通过连接到您的 node 节点并推出叫作“Ansible 模块”的小程序来工作的。这些程序被写成了系统期望状态下的资源模型。然后 Ansible 执行了这些模块（默认情况下是通过 SSH）并在完成后删除这些模块。
您的模块库可以位于任何一台机器上，而且该服务器可以没有任何服务，后台程序和数据库。您可以使用您喜欢的终端程序来持续跟踪模块的变化，这里的终端程序既可以是文本编辑器也可以是一个版本控制系统。

<h3 id="ssh">ssh 密钥是您的朋友</h3>

Ansible 是支持使用密码的，但是最好的方式还是使用 ssh 代理的 SSH 密钥这种方式。尽管如此，您还是想使用 Kerberos，那也不错。登录的方式也有很多，除了 root 用户以外，您可以使用其它任何用户来登录，然后通过使用 `su` 或者 `sudo` 命令转换到其它的用户上。

Ansible 是通过“authorized_key” 模块来控制什么机器可以访问什么主机的最佳方式。当然像 kerberos 或认证管理系统虽然不是最好的方式但也可以使用。

~~~
ssh-agent bash
ssh-add ~/.ssh/id_rsa
~~~

<h3 id="inventory">在简易的文本文件中管理您的主机清单</h3>

默认情况下，Ansible 可以通过查看一个非常简单的 INI 文件（可以将您的管理机器按照自己的意愿进行分组）就可以知道它在管理着哪些机器。

为了添加新的机器，这里不再包含 ssl 签发证书的机构，所以如果一台机器没有连接上，很明显就是 ntp 或 dns 导致的问题了。

如果在您的系统架构中存在另一些真实的数据源（例如 EC2，Rackspace，OpenStack 等等中的数据），Ansible 可以用插件将这些数据进行可视化整理，描述并分组。

下面是纯文本格式的主机列表：

~~~
[webservers]
www1.example.com
www2.example.com

[dbservers]
db0.example.com
db1.example.com
~~~

一旦在文本文件中保存了主机的信息，Ansible 就会将分配给这些主机的变量保存到一个简单的文本文件中（该文件通常在 ansible 子目录中的 `group_vars/` 或 `host_vars/` 目录中或者直接保存在存放主机列表的文件中。）

或者，正如前面提到过的，使用一个动态的清单文件，以便从数据源（EC2，Rackspace，OpenStack）中下载您的主机清单。

### 基础：使用 Ansible 来运行临时的并行任务
一但您有了一个可用的实例，您就可以立即与它通信，而不需要额外的配置：

~~~
ansible all -m ping 
ansible foo.example.com -m yum "name=httpd state=installed"
ansible foo.example.com -a "/usr/sbin/reboot"
~~~

注意：我们这里分别使用到了 `ping`，`yum` 和默认的 `command` 模块，除了这些我们还可以运行其它的原始命令。这些模块写起来非常简单,Ansible 已经提供了[很多模块](http://docs.ansible.com/ansible/modules_by_category.html)，这可以让您的工作事半功倍。

Ansible 的模块里包含了大量的多达 200 多个构建工具集。

<h3 id="playbooks">PlayBooks：一个简洁但很强大的自动化语言</h3>

PlayBooks 可以将您的架构分成多个片段，在同一时间非常详细且全面的控制着亟待被处理的机器。同时 PlayBooks 也正是 Ansible 的魅力所在。

Ansible 的编配方法是非常精确的，我猜您在使用 ansible 以前写脚本的过程已经非常顺畅了，况且这里的特殊语法或特性并不复杂。

下面是 playbooks 的一个示例。

~~~ bash
---
- hosts: webservers
serial: 5 # update 5 machines at a time
roles:
- common
- webapp

- hosts: content_servers
roles:
- common
- content
~~~

<h3 id="yaml">一个 app_config.yml 文件的示例：</h3>

~~~ yaml
---
- yum: name={{contact.item}} state=installed
with_items:
- app_server
- acme_software


- service: name=app_server state=running enabled=yes


- template: src=/opt/code/templates/foo.j2 dest=/etc/foo.conf
notify: 
- restart app server
~~~

[Ansible 文档](http://docs.ansible.com/index.html)已经阐述的很详细了。当然除了以上这些，您还可以做更多的事情：

* 对被管理节点进行负载均衡处理并监控窗口。
* 使用配置文件的方式让服务器获取到其它服务器的 IP 地址。
* 设置一些变量和命令提示符，并为没被设置的变量设置默认值。
* 用命令来决定一台机器是否要运行。

当然 Ansible 还有很多让你意想不到的功能，但是该文主要是让你对 Ansible 有个[初步的认知](http://www.ansible.com/get-started)，所以请您继续关注后面的章节。

Playbooks 最重要的一点就是，该语言可读且易懂。您无需做像声明或用一种编程语言写代码这样的事情了。

<h3 id="extension">扩展 Ansible：模块，插件和 API</h3>

您可能想自己来写脚本，Ansible 模块可以用任何可以生成 JSON 的语言，例如 Ruby，Python，bash 等等。通过写一个程序（能够和数据源进行通信并返回 JSON）将主机列表插入到任何一种数据源中。为扩展 Ansible 的连接方式（SSH 并不是唯一的通信方式）和回调方式 Ansible 还提供了 Python 的各种 API 以供使用。
