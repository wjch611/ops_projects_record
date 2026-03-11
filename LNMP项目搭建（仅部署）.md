### 1、拿到一台服务器的root的ssh

### 2、对服务器进行初始化（执行init.sh）

```
wget https://raw.githubusercontent.com/wjch611/ops_script/main/sh/deb_init.sh

chmod +x deb_init.sh

./deb_init.sh
    1. 修改上海时间区域
    2. 安装基础软件：wget、vi、git....
    3. 添加devops,并且添加到sudo组
    4. 安装配置并且启用ufw
    5. 安装配置fail2ban
    .....

跳板机器:
    ssh-keygen -t ed25519
    ssh-copy-id devops@本机ip
    ssh-copy-id root@本机ip

    devops ssh回到本机
    ssh devops@ip
本机：
    sudo vi /etc/ssh/sshd_config
    把允许密码登陆改为no
    sudo systemctl restart ssh
```

### 3、安装部署软件

```
sudo apt update
```

nginx

```
sudo apt install -y nginx
```

mysql

```
sudo apt install -y mysql-server
```

php、php-fpm、php插件....

```
sudo apt install php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip -y
```

wp-cli

```
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp
```

### 4、配置数据库

初始化数据库安全配置

```
sudo mysql_secure_installation
```

创建网站的数据库和数据库用户

```
--登录 MySQL
sudo mysql -u root -p 
-- 执行以下 SQL
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 5、WP-CLI安装wordpress

```
sudo mkdir -p /var/www/wordpress
sudo chown $USER:$USER /var/www/wordpress
cd /var/www/wordpress
三步：
    1. 安装核心
    2. 安装配置文件
    3. 开始安装
# 下载核心文件
wp core download --locale=zh_CN

# 创建 wp-config.php
wp config create --dbname=wordpress_db --dbuser=wp_user --dbpass=your_password

# 执行正式安装
wp core install --url="http://your_domain.com" --title="我的网站" --admin_user="admin" --admin_password="strong_password" --admin_email="info@example.com"
```

恢复网站目录权限

> 使用find . type -f --exec
> 
> 目录755
> 
> 文件644
> 
> 配置文件600

```
 1. 统一所有者（必须）
sudo chown -R www-data:www-data /var/www/html

# 2. 先把所有文件设为 644
sudo find /var/www/html -type f -exec chmod 644 {} \;

# 3. 再把所有目录设为 755
sudo find /var/www/html -type d -exec chmod 755 {} \;

# 4. wp-config.php 特殊处理（最高权限）
sudo chmod 600 /var/www/html/wp-config.php
```

### 6、配置nginx

```
sudo rm /etc/nginx/site-available/default
sudo rm /etc/nginx/site-enabled/default


sudo vi /etc/nginx/site-available/wordpress
1. 监听端口
2. server_name
3. 根目录+index
4. 伪静态路由
5. php socket fastcgi路由
```

```
server {
    listen 80;
    server_name your_domain.com;
    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock; # 确认你的 PHP 版本对应路径
    }
}
```

```
sudo nginx -t
sudo systemctl restart nginx
```
