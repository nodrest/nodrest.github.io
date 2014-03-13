---
layout: post
category : 文档
title : GitLab安装
tagline : "版本库管理"
tags : [git, github, 版本管理]
---

> [安装GitLab](https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md)

__选择版本安装__

Make sure you view this installation guide from the branch (version) of GitLab you would like to install. In most cases this should be the highest numbered stable branch (example shown below).

![capture](http://i.imgur.com/d2AlIVj.png)

If this is unclear check the [GitLab Blog](https://www.gitlab.com/blog/) for installation guide links by version.

**重要注意事项**

This guide is long because it covers many cases and includes all commands you need, this is [one of the few installation scripts that actually works out of the box](https://twitter.com/robinvdvleuten/status/424163226532986880).

This installation guide was created for and tested on **Debian/Ubuntu** operating systems. Please read [doc/install/requirements.md](./requirements.md) for hardware and operating system requirements. An unofficial guide for RHEL/CentOS can be found in the [GitLab recipes repository](https://gitlab.com/gitlab-org/gitlab-recipes/tree/master/install/centos).

This is the official installation guide to set up a production server. To set up a **development installation** or for many other installation options please see [the installation section of the readme](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/README.md#installation).

The following steps have been known to work. Please **use caution when you deviate** from this guide. Make sure you don't violate any assumptions GitLab makes about its environment. For example many people run into permission problems because they changed the location of directories or run services as the wrong user.

If you find a bug/error in this guide please **submit a merge request** following the [contributing guide](../../CONTRIBUTING.md).

- - -

**概述**

GitLab安装由设置以下组建组成:

1. 包/依赖
2. Ruby
3. 系统用户
4. GitLab shell
5. 数据库
6. GitLab
7. Nginx


# 1. 包/依赖

Debian默认没有安装`sudo`. 确认系统更新并安装它.

{% highlight sh linenos %}
# run as root!
apt-get update -y
apt-get upgrade -y
apt-get install sudo -y
{% endhighlight %}

**注意:**
During this installation some files will need to be edited manually.
If you are familiar with vim set it as default editor with the commands below.
If you are not familiar with vim please skip this and keep using the default editor.

{% highlight sh linenos %}
# 安装vim并设置默认编辑器
sudo apt-get install -y vim
sudo update-alternatives --set editor /usr/bin/vim.basic
{% endhighlight %}

安装所需的软件包:

{% highlight sh %}
sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libffi-dev curl openssh-server redis-server checkinstall libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate
{% endhighlight %}

请确保你已经安装了正确版本的Git

{% highlight sh linenos %}
# 安装Git
sudo apt-get install -y git-core

# 确认Git是版本1.7.10或更高, 例如1.7.12 or 1.8.4
git --version
{% endhighlight %}

## 源码Source安装

是系统封装的Git太旧了？将其删除，并从源代码编译。

{% highlight sh linenos %}
# 删除包Git
sudo apt-get remove git-core

# 安装依赖
sudo apt-get install -y libcurl4-openssl-dev libexpat1-dev gettext libz-dev libssl-dev build-essential

# 从源码下载编译
cd /tmp
curl --progress https://git-core.googlecode.com/files/git-1.8.5.2.tar.gz | tar xz
cd git-1.9.0.0/
make prefix=/usr/local all

# 装入/usr/local/bin
sudo make prefix=/usr/local install

# When editing config/gitlab.yml (Step 6), change the git bin_path to /usr/local/bin/git
{% endhighlight %}

**注意:** In order to receive mail notifications, make sure to install a
mail server. By default, Debian is shipped with exim4 whereas Ubuntu
does not ship with one. The recommended mail server is postfix and you can install it with:

{% highlight sh %}
sudo apt-get install -y postfix
{% endhighlight %}

Then select 'Internet Site' and press enter to confirm the hostname.

## 使用ppa安装

{% highlight sh linenos %}
sudo add-apt-repository ppa:git-core/ppa
sudo apt-get update
sudo apt-get install git
{% endhighlight %}

# 2. Ruby

The use of ruby version managers such as [RVM](http://rvm.io/), [rbenv](https://github.com/sstephenson/rbenv) or [chruby](https://github.com/postmodern/chruby) with GitLab in production frequently leads to hard to diagnose problems. Version managers are not supported and we stronly advise everyone to follow the instructions below to use a system ruby.

删除旧的的Ruby1.8如果存在

{% highlight sh %}
sudo apt-get remove ruby1.8
{% endhighlight %}

下载Ruby和编译:
{% highlight sh linenos %}
mkdir /tmp/ruby && cd /tmp/ruby
curl --progress http://cache.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p451.tar.gz | tar xz
cd ruby-2.0.0-p451
./configure --disable-install-rdoc
make
sudo make install
{% endhighlight %}

安装Bundler Gem:
{% highlight sh %}
sudo gem install bundler --no-ri --no-rdoc
{% endhighlight %}

# 3. 系统用户

为Gitlab创建`git`用户:

{% highlight sh %}
sudo adduser --disabled-login --gecos 'GitLab' git
{% endhighlight %}

# 4. GitLab shell

GitLab Shell是GitLab专门开发一个ssh访问和版本库管理软件。
{% highlight sh linenos %}
# 去主目录
cd /home/git

# Clone gitlab shell
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-shell.git -b v1.8.1

cd gitlab-shell

sudo -u git -H cp config.yml.example config.yml

# Edit config and replace gitlab_url
# with something like 'http://domain.com/'
sudo -u git -H editor config.yml

# Do setup
sudo -u git -H ./bin/install
{% endhighlight %}

# 5. 数据库

我们推荐PostgreSQL数据库. 为MySQL查看[MySQL设置向导](doc/install/database_mysql.md).
{% highlight sh linenos %}
# 安装数据库包
sudo add-apt-repository ppa:chris-lea/postgresql-9.3
sudo apt-get update
sudo apt-get install -y postgresql-9.3 postgresql-client libpq-dev

# 登录PostgreSQL
sudo -u postgres psql -d template1

# 为GitLab创建用户.
template1=# CREATE USER git;

# 创建GitLab产品库 & 在数据库上授予的所有权限
template1=# CREATE DATABASE gitlabhq_production OWNER git;

# 退出数据库对哈
template1=# \q

# 试着用新用户连接库
sudo -u git -H psql -d gitlabhq_production
{% endhighlight %}

# 6. GitLab

{% highlight sh %}
# 我们将GitLab到用户"git"主目录
cd /home/git
{% endhighlight %}
## 克隆源码
{% highlight sh linenos %}
# 克隆GitLab版本库
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 6-6-stable gitlab

# 去gitlab目录
cd /home/git/gitlab
{% endhighlight %}
**注意:**
如果你想要*前沿*版本可以修改`6-6-stable`为`master`, 但是千万不能在生产服务器上安装最高版本!

## 配置
{% highlight sh linenos %}
cd /home/git/gitlab

# 拷贝样例GitLab配置
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml

# 请务必在必要时修改“localhost”为你服务GitLab主机的完整域名
#
# 如果你从源代码安装的Git，改变GIT bin_path到/usr/local/bin/git

sudo -u git -H editor config/gitlab.yml

# 确认GitLab能写 log/ 和 tmp/ 目录
sudo chown -R git log/
sudo chown -R git tmp/
sudo chmod -R u+rwX  log/
sudo chmod -R u+rwX  tmp/

# 为satellites创建目录
sudo -u git -H mkdir /home/git/gitlab-satellites

# 为sockets/pids创建目录，并确认GitLab能写如它们
sudo -u git -H mkdir tmp/pids/
sudo -u git -H mkdir tmp/sockets/
sudo chmod -R u+rwX  tmp/pids/
sudo chmod -R u+rwX  tmp/sockets/

# 创建public/uploads目录，否则备份将失败
sudo -u git -H mkdir public/uploads
sudo chmod -R u+rwX  public/uploads

# 拷贝样例Unicorn配置
sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb

# 如果你期望有一个高负荷的实例启用群集模式
# 例. 为2G内存服务器修改工作数为3
sudo -u git -H editor config/unicorn.rb

# 拷贝样例Rack attack配置
sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

# Configure Git global settings for git user, useful when editing via web
# Edit user.email according to what is set in gitlab.yml
sudo -u git -H git config --global user.name "GitLab"
sudo -u git -H git config --global user.email "gitlab@localhost"
sudo -u git -H git config --global core.autocrlf input
{% endhighlight %}
**Important 注意:**
Make sure to edit both `gitlab.yml` and `unicorn.rb` to match your setup.

## 配置GitLab数据库设置
{% highlight sh linenos %}
# PostgreSQL
sudo -u git cp config/database.yml.postgresql config/database.yml

# Make sure to update username/password in config/database.yml.
# You only need to adapt the production settings (first part).
# If you followed the database guide then please do as follows:
# Change 'secure password' with the value you have given to $password
# You can keep the double quotes around the password
sudo -u git -H editor config/database.yml
{% endhighlight %}

或者
{% highlight sh linenos %}
# Mysql
sudo -u git cp config/database.yml.mysql config/database.yml

# Make config/database.yml readable to git only
sudo -u git -H chmod o-rwx config/database.yml
{% endhighlight %}

## 安装Gems

修改Gemfile：

{% highlight sh%}
vim Gemfile
{% endhighlight %}

{% highlight ruby linenos%}
#修改source
#或者使用清华的镜像（未验证）
#指定 ipv4 的話用 http://mirrors.4.tuna.tsinghua.edu.cn/rubygems/
#指定 ipv6 的話用 http://mirrors.6.tuna.tsinghua.edu.cn/rubygems/
source "http://ruby.taobao.org/"
{% endhighlight %}

{% highlight sh linenos %}
cd /home/git/gitlab

# 为PostgreSQL (注意, 选项​​说 "without ... mysql")
sudo -u git -H bundle install --deployment --without development test mysql aws

# 或者如果你使用MySQL (注意, 选项​​说 "without ... postgres")
sudo -u git -H bundle install --deployment --without development test postgres aws
{% endhighlight %}

## 初始化数据库并激活高级功能
{% highlight sh linenos %}
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production

# 输入'yes'创建数据库表.

# 做完你能看到'Administrator account created:'
{% endhighlight %}

## 安装初始化脚本

Download the init script (will be /etc/init.d/gitlab):
{% highlight sh %}
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
{% endhighlight %}

And if you are installing with a non-default folder or user copy and edit the defaults file:
{% highlight sh %}
sudo cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
{% endhighlight %}

If you installed gitlab in another directory or as a user other than the default you should change these settings in /etc/default/gitlab. Do not edit /etc/init.d/gitlab as it will be changed on upgrade.

Make GitLab start on boot:
{% highlight sh %}
sudo update-rc.d gitlab defaults 21
{% endhighlight %}

## 设置logrotate
{% highlight sh %}
sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
{% endhighlight %}

## 检查应用现状

Check if GitLab and its environment are configured correctly:
{% highlight sh %}
sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
{% endhighlight %}

## 启动您的GitLab实例
{% highlight sh linenos %}
sudo service gitlab start
# or
sudo /etc/init.d/gitlab restart
{% endhighlight %}

## 编译assets
{% highlight sh %}
sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production
{% endhighlight %}

# 7. Nginx

**注意:**
Nginx is the officially supported web server for GitLab. If you cannot or do not want to use Nginx as your web server, have a look at the
[GitLab recipes](https://gitlab.com/gitlab-org/gitlab-recipes/).

## 安装
{% highlight sh %}
sudo apt-get install -y nginx
{% endhighlight %}

## 站点配置

Download an example site config:
{% highlight sh %}
sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
{% endhighlight %}

Make sure to edit the config file to match your setup:
{% highlight sh linenos %}
# Change YOUR_SERVER_FQDN to the fully-qualified
# domain name of your host serving GitLab.
sudo editor /etc/nginx/sites-available/gitlab
{% endhighlight %}
## 重启
{% highlight sh %}
sudo service nginx restart
{% endhighlight %}

# 完成了！

## 仔细检查应用现状

To make sure you didn't miss anything run a more thorough check with:
{% highlight sh %}
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
{% endhighlight %}
If all items are green, then congratulations on successfully installing GitLab!

## 初始登录

Visit YOUR_SERVER in your web browser for your first GitLab login.
The setup has created an admin account for you. You can use it to log in:
{% highlight sh %}
root
5iveL!fe
{% endhighlight %}
**Important 注意:**
Please go over to your profile page and immediately change the password, so
nobody can access your GitLab by using this login information later on.

**Enjoy!**


- - -


# 高级设置技巧

## 额外的标记样式

Apart from the always supported markdown style there are other rich text files that GitLab can display.
But you might have to install a depency to do so.
Please see the [github-markup gem readme](https://github.com/gitlabhq/markup#markups) for more information.
For example, reStructuredText markup language support requires python-docutils:
{% highlight sh%}
sudo apt-get install -y python-docutils
{% endhighlight %}
## 自Redis的连接

If you'd like Resque to connect to a Redis server on a non-standard port or on
a different host, you can configure its connection string via the
`config/resque.yml` file.
{% highlight sh linenos %}
# example
production: redis://redis.example.tld:6379
{% endhighlight %}

If you want to connect the Redis server via socket, then use the "unix:" URL scheme
and the path to the Redis socket file in the `config/resque.yml` file.
{% highlight sh linenos %}
# example
production: unix:/path/to/redis/socket
{% endhighlight %}

## 自定义SSH连接

If you are running SSH on a non-standard port, you must change the gitlab user's SSH config.
{% highlight ini linenos %}
# Add to /home/git/.ssh/config
host localhost  # Give your setup a name (here: override localhost)
user git# Your remote git user
port 2222   # Your port number
hostname 127.0.0.1; # Your server name or IP
{% endhighlight %}

You also need to change the corresponding options (e.g. ssh_user, ssh_host, admin_uri) in the `config\gitlab.yml` file.

## LDAP身份验证

You can configure LDAP authentication in config/gitlab.yml. Please restart GitLab after editing this file.

## 使用自定义Omniauth提供商

GitLab uses [Omniauth](http://www.omniauth.org/) for authentication and already ships with a few providers preinstalled (e.g. LDAP, GitHub, Twitter). But sometimes that is not enough and you need to integrate with other authentication solutions. For these cases you can use the Omniauth provider.

### 设置

These steps are fairly general and you will need to figure out the exact details from the Omniauth provider's documentation.

* Stop GitLab
{% highlight sh%}
`sudo service gitlab stop`
{% endhighlight %}
* Add provider specific configuration options to your `config/gitlab.yml` (you can use the [auth providers section of the example config](https://gitlab.com/gitlab-org/gitlab-ce/blob/masterconfig/gitlab.yml.example) as a reference)

* Add the gem to your [Gemfile](https://gitlab.com/gitlab-org/gitlab-ce/blob/masterGemfile)
{% highlight sh%} 
`gem "omniauth-your-auth-provider"`
{% endhighlight %}

* If you're using MySQL, install the new Omniauth provider gem by running the following command:
{% highlight sh%}
`sudo -u git -H bundle install --without development test postgres --path vendor/bundle --no-deployment`
{% endhighlight %}

* If you're using PostgreSQL, install the new Omniauth provider gem by running the following command:
{% highlight sh %}
`sudo -u git -H bundle install --without development test mysql --path vendor/bundle --no-deployment`
{% endhighlight %}

> These are the same commands you used in the [Install Gems section](#install-gems) with `--path vendor/bundle --no-deployment` instead of `--deployment`.

* Start GitLab
{% highlight sh %}
`sudo service gitlab start`
{% endhighlight %}

### 实例

If you have successfully set up a provider that is not shipped with GitLab itself, please let us know.
You can help others by reporting successful configurations and probably share a few insights or provide warnings for common errors or pitfalls by sharing your experience [in the public Wiki](https://github.com/gitlabhq/gitlab-public-wiki/wiki/Custom-omniauth-provider-configurations).
While we can't officially support every possible auth mechanism out there, we'd like to at least help those with special needs.

