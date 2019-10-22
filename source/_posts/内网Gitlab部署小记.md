---
title: 一次内网Gitlab部署小记
date: 2019.10.20 10:27:30
tags:
  - Git
  - 运维
categories: 运维
keywords: Gitlab部署
description: 公司内网环境中，一次部署Gitlab企业版/社区版note
top_img: https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/gitlab_top_img.png
cover: https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/gitlab_cover.png
---

# 准备

## 环境



- OS：**CentOS 7.2** Linux version 3.10.0-327.4.5.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.3 20140911 (Red Hat 4.8.3-9) (GCC) ) #1 SMP Mon Jan 25 22:07:14 UTC 2016
- 配置：2核4g
- gitlab version：GitLab 12.3.5-ee





# 安装



## 安装配置依赖



### 配置防火墙HTTP&HTTPS

```bash
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld

```

这里根据想要配置的不同端口 灵活更改，因为我服务器并无其他web端口，所以这里默认还是使用80



> **PS**：如果firewall-cmd命令无法找到，请浏览这个[文章](https://www.tecmint.com/fix-firewall-cmd-command-not-found-error/)解决



###  Postfix （选择其他可以不装）

 install Postfix to send notification emails 安装Postfix 发送邮件消息通知，如果选用其他发送邮箱方式可以忽略。

```bash
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```



## 开始安装

这里官网中推荐安装ee也就是企业版本，因为如果没有订阅ee的 license ，默认和社区版功能相同，而如果想试用或者使用可以随时用企业版的功能。所以那就安装  Enterprise Edition 吧。[说明在此](https://about.gitlab.com/install/ce-or-ee/?distro=centos-7)

1. **添加仓库**

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
```

2. **设置URL并安装**

```bash
sudo EXTERNAL_URL="https://gitlab.example.com" yum install -y gitlab-ee
```

修改`https://gitlab.example.com`为自己本地的域名或者IP，此处我选择ip（如果要在线上访问，可以配置你的子域名，并设置解析）

![安装完成](https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/1571546466615.png)

安装成功。在`http://192.168.11.2/`web页面成功显示

![初次访问页面](https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/1571546966522.png)

默认第一次访问会重定向到设置密码页面。

![登录后主页](https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/1571547939027.png)



# 常用配置

问题文档： https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/README.md 

官方配置文档： https://docs.gitlab.com/omnibus/settings/configuration.html 



**注意！！！：**每次修改完配置要重载下

```
sudo gitlab-ctl reconfigure
```



配置文件路径：`/etc/gitlab/gitlab.rb`

(我这里就不完整贴出配置文件了，可长了...)



## external URL 外部访问地址

通俗点就是访问的URL。

```
##! For more details on configuring external_url see:
external_url 'http://192.168.11.2'
# registry_external_url 'https://registry.example.com'
# pages_external_url "http://pages.example.com/"
# gitlab_pages['artifacts_server_url'] = nil # Defaults to external_url + '/api/v4'
# gitlab_pages['auth_redirect_uri'] = nil # Defaults to projects subdomain of pages_external_url and + '/auth'
# gitlab_pages['gitlab_server'] = nil # Defaults to external_url
# mattermost_external_url 'http://mattermost.example.com'
# For example, if external_url is the same for two secondaries, you must specify
# If it is blank, it defaults to external_url.

# 外部访问地址
external_url 'http://192.168.11.2'
```



还有个**relative URL 相对URL**，从配置文档中了解到，这个配置可以设置为gitlab为目录访问，[具体查看](https://docs.gitlab.com/omnibus/settings/configuration.html#configuring-a-relative-url-for-gitlab)



## 邮件配置



最后我没有选择postfix，而且采用了aliyun邮箱发信



官方配置文档： https://docs.gitlab.com/omnibus/settings/smtp.html 

```
# 默认模板
### GitLab email server settings
###! Docs: https://docs.gitlab.com/omnibus/settings/smtp.html
###! **Use smtp instead of sendmail/postfix.**

# gitlab_rails['smtp_enable'] = true
# gitlab_rails['smtp_address'] = "smtp.server"
# gitlab_rails['smtp_port'] = 465
# gitlab_rails['smtp_user_name'] = "smtp user"
# gitlab_rails['smtp_password'] = "smtp password"
# gitlab_rails['smtp_domain'] = "example.com"
# gitlab_rails['smtp_authentication'] = "login"
# gitlab_rails['smtp_enable_starttls_auto'] = true
# gitlab_rails['smtp_tls'] = false

###! **Can be: 'none', 'peer', 'client_once', 'fail_if_no_peer_cert'**
###! Docs: http://api.rubyonrails.org/classes/ActionMailer/Base.html
# gitlab_rails['smtp_openssl_verify_mode'] = 'none'

# gitlab_rails['smtp_ca_path'] = "/etc/ssl/certs"
# gitlab_rails['smtp_ca_file'] = "/etc/ssl/certs/ca-certificates.crt"

### Email Settings
# gitlab_rails['gitlab_email_enabled'] = true
# gitlab_rails['gitlab_email_from'] = 'example@example.com'
# gitlab_rails['gitlab_email_display_name'] = 'Example'
# gitlab_rails['gitlab_email_reply_to'] = 'noreply@example.com'
# gitlab_rails['gitlab_email_subject_suffix'] = ''
# gitlab_rails['gitlab_email_smime_enabled'] = false
# gitlab_rails['gitlab_email_smime_key_file'] = '/etc/gitlab/ssl/gitlab_smime.key'
# gitlab_rails['gitlab_email_smime_cert_file'] = '/etc/gitlab/ssl/gitlab_smime.crt'


```





### postfix

可以通过上面安装的postfix来发送邮件，改动设置如下（去掉注释）

```
gitlab_rails['gitlab_email_enabled'] = true 
gitlab_rails['gitlab_email_from'] = 'gitlab@http://<服务器地址或域名>' 
gitlab_rails['gitlab_email_display_name'] = 'GitLab' 
```

没有完全尝试，后来考虑还是用其他邮箱较好



### Aliyun个人版

```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.aliyun.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "your username @aliyun.com"
gitlab_rails['smtp_password'] = "your password"
gitlab_rails['smtp_domain'] = "aliyun.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true

gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = 'your username @aliyun.com'
gitlab_rails['gitlab_email_display_name'] = 'Gitlab'
gitlab_rails['gitlab_email_reply_to'] = 'your username @aliyun.com'

```

这里多个文章提到如果安装了postfix后，再配置smtp服务器无效，但是当前gitlab版本中，我测试后是有效的。



**测试方法**

```
#进入控制台
gitlab-rails console
#发送测试邮件
Notify.test_email('收件人邮箱', '邮件标题', '邮件正文').deliver_now
```

![测试发信成功](https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/1571554772434.png)



## 禁用创建组权限

 GitLab默认所有的注册用户都可以创建组。  通过配置GitLab默认禁用创建组权限 。该权限在用户管理界面可以看到。

```
#开启gitlab_rails['gitlab_default_can_create_group'] 选项，并将值设置为false
### GitLab user privileges
gitlab_rails['gitlab_default_can_create_group'] = false

#保存后，重新配置并启动GitLab
sudo gitlab-ctl reconfigure
```



# 常用命令



## Service Management Commands

这类命令来管理服务

| command       | function         |
| :------------ | :--------------- |
| start         | 启动所有服务     |
| stop          | 关闭所有服务     |
| restart       | 重启所有服务     |
| status        | 查看所有服务状态 |
| tail          | 查看日志信息     |
| service-list  | 列举所有启动服务 |
| graceful-kill | 平稳停止一个服务 |

举例：

```bash
#启动所有服务
[root@localhost gitlab]# gitlab-ctl start
#启动单独一个服务
[root@localhost gitlab]# gitlab-ctl start nginx
#查看日志，查看所有日志
[root@localhost gitlab]# gitlab-ctl tail
#查看具体一个服务的日志,类似tail -f
[root@localhost gitlab]# gitlab-ctl tail nginx
```



## General Commands

全局命令

| command       | function                   |
| :------------ | :------------------------- |
| help          | 帮助                       |
| reconfigure   | 修改配置文件之后，重新加载 |
| show-config   | 查看所有服务配置文件信息   |
| uninstall     | 卸载这个软件               |
| cleanse       | 清空gitlab数据             |
| service-list  | 列举所有启动服务           |
| graceful-kill | 平稳停止一个服务           |

举例：

```bash
#显示所有服务配置文件
[root@localhost gitlab]#gitlab-ctl show-config
#卸载gitlab
[root@localhost gitlab]#gitlab-ctl uninstall
```



## DatabaseCommands

数据库命令

| command           | function                                           |
| :---------------- | :------------------------------------------------- |
| pg-upgrade        | 更新postgresql版本                                 |
| revert-pg-upgrade | 还远先前的(离现在正在使用靠近的版本)一个数据库版本 |
| show-config       | 查看所有服务配置文件信息                           |
| uninstall         | 卸载这个软件                                       |
| cleanse           | 清空gitlab数据                                     |
| service-list      | 列举所有启动服务                                   |
| graceful-kill     | 平稳停止一个服务                                   |

举例：

```bash
#升级数据库
[root@localhost gitlab]# gitlab-ctl pg-upgrade
Checking for an omnibus managed postgresql: OK
Checking if we already upgraded: OK
The latest version 9.6.1 is already running,nothing to do
#降级数据库版本
[root@localhost gitlab]# gitlab-ctl revert-pg-upgrade
```



# 过程中的问题

## 关于firewall

在firewall-cmd命令无法找到时，重新通过

```bash
sudo yum install firewalld

sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld
```

![1571540133060](https://cdn.jsdelivr.net/gh/iSk2y/CDN/blog/post/gitlab-deploy-note/1571540133060.png)

出现了报错

```
ERROR: Exception DBusException: org.freedesktop.DBus.Error.AccessDenied: Connection ":1.24" is not allowed to own the service "org.fedoraproject.FirewallD1" due to security policies in the configuration file
```

但是在启动时出现了问题，思考搜索一番答案后在，stackoverflow上找到了这个[解决方案](https://stackoverflow.com/questions/31397121/centos7-firewall-not-running)

>  More recently, [Bug 1575845](https://bugzilla.redhat.com/show_bug.cgi?id=1575845) in Red Hat's Bugzilla tracker shows a problem in RHEL/CentOS 7.3 or later that triggers this issue. Something with dbus policy not being passed correctly. The permanent fix (for now) could be upgrading your base image to a newer version of RHEL/CentOS. 
>
> However, these commands should also work, per this comment in the Bugzilla:
>
> ```bash
> sudo systemctl restart dbus
> sudo systemctl restart firewalld
> ```



## 发信失败

在使用smtp发信时，发信失败，

```
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
```

这两个属性关注下，是否到底用了ssl加密，同时注意端口配置。

如果不行，尝试看日志信息解决





## 文档&参考



- 官方安装： https://about.gitlab.com/install/#centos-7 
- 官方manual_install： https://docs.gitlab.com/omnibus/manual_install.html 
- [Centos 安装GitLab（从零开始配置）](https://www.jianshu.com/p/dbb4543bdd8e)  
- [Gitlab本地部署安装配置](https://www.jianshu.com/p/2ff7c00f3ab6)
- [CentOS 7 下 GitLab安装部署教程](https://ken.io/note/centos7-gitlab-install-tutorial)