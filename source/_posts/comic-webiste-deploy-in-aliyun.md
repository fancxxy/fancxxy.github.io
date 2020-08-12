---
title: 配置阿里云以及部署ComicWeb
date: 2018-06-22 20:39:39
tags: Linux
---

新购买了阿里云ecs服务器，记录一下整个配置和部署过程，操作系统是CentOS 7。
<!--more-->

### 创建用户

``` shell
useradd -m -g users -G wheel -s /bin/bash username
passwd username
```

### 给用户添加root权限

``` shell
yum install sudo
EDITOR=vim visudo

```

添加如下内容


```
username hostname=(ALL) ALL
```

### 切换为普通用户

``` shell
su - username
```

## 切换shell为zsh并安装oh-my-zsh

``` shell
echo $SHELL
yum install zsh
cat /etc/shells
chsh -s /bin/zsh
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

修改.zshrc的主题

```
ZSH_THEME="gianu"
```

在.zshrc末尾添加下列配置，以修复warning: cannot set LC_CTYPE locale的提示

```
export LC_ALL=en_US.UTF-8
export LC_CTYPE=en_US.UTF-8
```

### 安装epel和ius仓库

``` shell
sudo yum install -y epel-release
sudo yum install -y https://centos7.iuscommunity.org/ius-release.rpm
sudo yum update
```

### 安装Python3.6

``` shell
sudo yum install -y python36u python36u-libs python36u-devel python36u-pip
```

### 修改Python的源

新增文件 ~/.pip/pip.conf

```
[global]
trusted-host=mirrors.aliyun.com
index-url=https://mirrors.aliyun.com/pypi/simple/
```


### 安装mariaDB

``` shell
sudo yum install -y mariadb-server mariadb-devel
sudo systemctl start mariadb
mysql_secure_installation
```

修改/etc/my.cnf

```
[client]
default-character-set = utf8mb4

[mysqld]
collation_server = utf8mb4_unicode_ci
character_set_server = utf8mb4
innodb_file_format = barracuda
innodb_file_per_table = 1
innodb_large_prefix = 1

[mysql]
default-character-set = utf8mb4
```

创建数据库用户

```
create user username@localhost identified by password;
grant all privileges on database.* to username@localhost;
flush privileges;
```



### 部署ComicWeb

``` shell
# 下载工程文件
git clone git://github.com/fancxxy/ComicWeb.git
cd ComicWeb/

# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate

# 安装依赖
pip install git+git://github.com/fancxxy/comicd.git
pip install -r requirements.txt

# 设置环境变量，把数据库真实连接串写到env.sh中
export $(cat env.sh | grep -v ^# | xargs)

# 升级数据库
flask db init
flask db migrate -m "initial commit"
flask db upgrade

# 创建web用户
flask shell
>>> user = User(email='email', username='username', password='password', confirmed=True)
>>> db.session.add(user)
>>> db.session.commit()

# 安装nginx
sudo yum install nginx

# 修改/etc/nginx/nginx.conf
location / {
    include uwsgi_params;
    uwsgi_pass 127.0.0.1:8001;
    uwsgi_param UWSGI_PYTHON /home/fancxxy/Works/ComicWeb/venv;
    uwsgi_param UWSGI_CHDIR /home/fancxxy/Works/ComicWeb;
    uwsgi_param UWSGI_SCRIPT web:app;
}

# 启动nginx
sudo systemctl enable nginx
sudo systemctl start nginx

# 安装以及配置uwsgi
pip install uwsgi
sudo cp uwsgi.service /etc/systemd/system/
sudo systemctl enable uwsgi
sudo systemctl start uwsgi
```

### 配置阿里云外部端口
在网络和安全>安全组中点击配置规则，添加安全组规则，配置http端口号，授权对象为0.0.0.0/0

