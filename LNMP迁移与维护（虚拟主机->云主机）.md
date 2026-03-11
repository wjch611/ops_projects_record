[TOC]

## 一、需求

##### 背景：

    由于PV变高，需要提升网站（性能、安全、可控）

    需要帮公司门户网站（部署在虚拟主机上的一个wordpress项目）迁移到云主机。

##### 原项目：

- 某云虚拟主机 + wordpress + mysql（...）

##### 目标：（单点LNMP架构）

阿里云主机 + ubuntu2204 + nginx + wrodpress + mysql

额外做了优化

- nginx：
  
  - https
  
  - 伪静态路由
  
  - 隐藏nginx版本
  
  - csp策略
  
  - 防敏感文件读取

- redis缓存加速

- logrotate日志切割

- 脚本实现网站自动备份

- 脚本实现资源监控报警

## 二、仅迁移

##### 1. 导出原网站数据

> 登陆虚拟主机控制面板导出（**wp数据库、/var/www/html**）

##### 2. ssh上vps

##### 3. 初始化vps（执行init.sh）

> 1. 设置上海时区
> 
> 2. 安装基础软件
> 
> 3. 创建一个devops用户（加入sudo组）
> 
> 4. 开启ufw，配置入站规则
> 
> 5. 配置fail2ban

6. 把跳板机的ssh公钥ssh-copy-id到devops用户和root用户

7. 修改ssh规则禁用密码登陆（**留下root密钥登陆是因为后面ansible自动化需要**）

##### 4. 安装基础部署软件（nginx、php(php-fpm)、mysql）

nginx：

```
sudo apt install -y nginx
```

mysql：

```
sudo apt install -y mysql-server

安全初始化：sudo mysql_secure_installation
1. 添加root密码
2. 禁止root远程登陆
3. 删除测试数据库、匿名用户
```

php+php-fpm+php插件：（还没安装php-redis）

```
sudo apt install -y php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc \
php-soap php-intl php-zip php-cli php-bcmath
```

##### 5. 迁移html

```
sudo rm /var/www/html -rf
tar -xzf wp-data.tar.gz
sudo mv html/ /var/www
```

修改权限与所有者

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

##### 6. 迁移数据库

创建wp数据库和用户

```
sudo mysql
```

```
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'Cww123#!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

导入数据

```
# 用 root 用户导入（输入 root 密码）
sudo mysql -u root -p wordpress < wordpress.sql
```

修改wp-config.php数据库配置

##### 7. Nginx配置

1、链接到enable

在site-available下添加配置/etc/nginx/sites-available/wordpress.conf

```
# 链接到 enabled
sudo ln -sf /etc/nginx/sites-available/wordpress.conf /etc/nginx/sites-enabled/
```

2、基础配置

> 1. 监听端口
> 2. server_name
> 3. 根目录+index
> 4. 伪静态路由
> 5. php socket fastcgi路由

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

3、配置优化（第四大点）

4、测试配置

```
sudo nginx -t
```

##### 8. 替换wp数据库的旧链接（wp search-replace）

安装WP-CLI

```
# 下载 WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

# 添加执行权限
chmod +x wp-cli.phar

# 移动到系统路径
sudo mv wp-cli.phar /usr/local/bin/wp

# 验证安装
wp --info
```

执行替换

```
sudo wp --path=/var/www/html search-replace '192.168.56.9' '192.168.56.10'
```

重启nginx

```
sudo systemctl restart nginx
```

##### 9. certbot配置https

安装certbot

```
sudo apt install certbot python3-certbot-nginx -y

certbot --version
```

执行交互

```
sudo certbot --nginx


Certbot 会自动：

1️⃣ 申请证书  
2️⃣ 修改 Nginx 配置  
3️⃣ 自动重载 Nginx

你的 Nginx 配置会变成类似：
```

```
server {  
 listen 443 ssl;  
 server_name example.com;

 ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;  
 ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

 root /var/www/html;
}
```

续期测试

```
sudo certbot renew --dry-run
```

## 三、数据一致性检查

> 基于：hosts灰度验证
> 
> 在一致性检查之前，先改本地/etc/hosts，不要直接修改dns解析

##### 文件层sumcheck验证

```
cd /var/www/html
find . -type f -exec sha256sum {} + > file_checksum.txt

整个打包->新服务->解压->验证
sha256sum -c file_checksum.txt
```

##### 数据库验证

> old和new都执行这个脚本，结果进行对比
> 
> 查询目标数据库的所有表，表的行数

```
#!/bin/bash

DB="wordpress"

OUTPUT="db_verify_$(date +%F_%H-%M-%S).txt"

echo "===== Database Verification Report =====" | tee $OUTPUT
echo "Database: $DB" | tee -a $OUTPUT
echo "Time: $(date)" | tee -a $OUTPUT
echo "" | tee -a $OUTPUT

# 获取所有表
tables=$(mysql -N -e "
SELECT table_name
FROM information_schema.tables
WHERE table_schema='$DB';
")

echo "Table | Rows | Checksum" | tee -a $OUTPUT
echo "----------------------------------------" | tee -a $OUTPUT

for table in $tables
do
    # 获取行数
    rows=$(mysql -N -e "
    SELECT COUNT(*) FROM $DB.$table;
    ")

    # 获取 checksum
    checksum=$(mysql -N -e "
    CHECKSUM TABLE $DB.$table;
    " | awk '{print $2}')

    printf "%-30s %-10s %-20s\n" "$table" "$rows" "$checksum" | tee -a $OUTPUT

done

echo "" | tee -a $OUTPUT
echo "Verification finished." | tee -a $OUTPUT
echo "Report saved: $OUTPUT"
```

##### 业务层验证

> 通过访问前台页面、后台登录、图片资源等功能

## 四、优化nginx配置

> 配置的位置：不是nginx.conf，是site.conf，并且在server{}中

##### 隐藏 nginx 版本

```
server_tokens off;
```

##### 设置CSP策略，防止XSS/点击劫持...（最重要）

> - 前台严格+后台宽松（需要在nginx的conf.d/下单独写一个配置文件）
> 
> - 先严后宽，浏览器的console查看报错，一步一步添加必须允许的源

```
# 全局 CSP 策略映射
map $request_uri $csp_policy {
    # 【后台策略】
    "~^/wp-admin"   "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' data: https:; frame-ancestors 'self';";

    # 【登录页策略】
    "~^/wp-login"   "default-src 'self'; script-src 'self' 'unsafe-inline' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' data: https:; frame-ancestors 'self';";

    # 【前台策略】默认严格
    default         "default-src 'self'; script-src 'self' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' data: https:; frame-ancestors 'none'; object-src 'none'; base-uri 'self';";
}
```

```
server {
    listen 80 default_server;
    server_name _;
    root /var/www/html;
    index index.php index.html index.htm;

    # 隐藏 nginx 版本
    server_tokens off;

    # =============================
    # 核心安全响应头
    # =============================
    # 使用 map 定义的变量
    add_header Content-Security-Policy $csp_policy always;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # =============================
    # 路径与文件保护
    # =============================

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 禁止访问敏感文件及备份
    location ~* /(wp-config.php|readme.html|license.txt|\.bak|\.sql|\.env) {
        deny all;
    }

    # 禁止 xmlrpc
    location = /xmlrpc.php {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 禁止 uploads 目录执行 PHP
    location ~* /wp-content/uploads/.*\.php$ {
        deny all;
    }

    # WordPress 核心路由
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # =============================
    # PHP 处理 (优化部分)
    # =============================
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;

        # 调优缓冲区，防止大页面拉取失败
        fastcgi_buffers 16 16k; 
        fastcgi_buffer_size 32k;

        # 确保在 PHP 请求中也发送 CSP Header
        add_header Content-Security-Policy $csp_policy always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
    }

    # =============================
    # 静态文件缓存 (优化部分)
    # =============================
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot|otf)$ {
        expires 30d;
        access_log off;
        add_header Cache-Control "public, no-transform";

        # 静态资源通常不需要复杂的 CSP，保持默认同源即可
        add_header Access-Control-Allow-Origin "*"; 
    }

    client_max_body_size 64M;
}
```

##### 伪静态设置

> try_files uri uri/ /index.php?$args;

```
1. $uri 根据url去找具体文件，读到就直接返回
2. $uri/ 找不到这个文件，就找这个文件名的文件夹，如果存在就寻找这个文件夹下面的index
3. /index.php?$args; 全部找不到，就去根目录下的index.php
```

根目录下/index.php，一般会有一个逻辑：
即使搜索不到最后的关键词语也不会报NGINX的404，只会报wordpress的404
所以判断是否是伪静态就乱输入uri看看有没有nginx的404就行。

##### 阻止敏感文件读取

```
# 禁止访问敏感文件及备份
location ~* /(wp-config.php|readme.html|license.txt|\.bak|\.sql|\.env) {
    deny all;
}
```

## 五、添加redis缓存

安装redis、php-redis、object-cache插件

```
sudo apt install redis-server -y
```

```
sudo apt install php-redis -y

sudo systemctl restart php8.1-fpm
```

## 六、logrotate对nginx日志切割

添加规则

```
sudo vi /etc/logrotate.d/nginx-custom

# nginx 日志轮转
/var/log/nginx/*.log {
    # 每天切割
    daily
    # 日志不存在不报错
    missingok
    # 保留14个日志
    rotate 14
    # gzip压缩
    compress
    # 延迟压缩
    delaycompress
    # 空日志不轮转
    notifempty
    # 新日志权限
    create 0640 www-data adm
    # postrotate 只执行一次
    sharedscripts
    postrotate
        /usr/sbin/nginx -s reopen
    endscript
}
```

```
sudo logrotate -f /etc/logrotate.d/nginx-custom #手动切割测试
```

## 七、自动备份脚本

> 把html+wp数据库打包压缩，发送到远程机器，并且发送飞书通知。
> 
> 自动删除一周外的备份

前提：

1. 配置好.my.cnf

2. 密钥ssh->远程

```
#!/bin/bash

set -euo pipefail

# ====================== 配置 ======================
DB_NAME="wordpress"

BACKUP_DIR="/home/devops/wp_backup"
RETENTION_DAYS=7

REMOTE_USER="oem"
REMOTE_IP="192.168.56.1"
REMOTE_PATH="/home/oem/wp_backup_received"

FEISHU_WEBHOOK="https://open.feishu.cn/open-apis/bot/v2/hook/xxxxx"

DEBUG=true
# ==================================================

DATE=$(date +%Y-%m-%d_%H-%M-%S)
WORK_DIR="$BACKUP_DIR/tmp_$DATE"
FINAL_FILE="$BACKUP_DIR/backup-$DATE.tar.gz"
LOG_FILE="$BACKUP_DIR/backup.log"

mkdir -p "$BACKUP_DIR"

log() {
    MSG="[$(date '+%Y-%m-%d %H:%M:%S')] $1"
    echo "$MSG" | tee -a "$LOG_FILE"
}

debug() {
    if [ "$DEBUG" = true ]; then
        echo "[DEBUG] $1" | tee -a "$LOG_FILE"
    fi
}

run_cmd() {
    debug "$1"
    eval "$1" >> "$LOG_FILE" 2>&1
}

log "========== WordPress 备份开始 =========="

# 创建临时目录
run_cmd "mkdir -p $WORK_DIR"

# ---------------- 数据库备份 ----------------
log "备份数据库..."

run_cmd "mysqldump --single-transaction --quick $DB_NAME > $WORK_DIR/wordpress.sql"

log "数据库备份完成"

# ---------------- 复制网站目录 ----------------
log "复制网站目录..."

run_cmd "cp -a /var/www/html $WORK_DIR/"

log "网站文件复制完成"

# ---------------- 打包 ----------------
log "开始压缩备份..."

run_cmd "tar -czf $FINAL_FILE -C $WORK_DIR ."

log "压缩完成"

# 删除临时目录
run_cmd "rm -rf $WORK_DIR"

# ---------------- 同步远程 ----------------
log "准备远程备份目录..."

run_cmd "ssh $REMOTE_USER@$REMOTE_IP 'mkdir -p $REMOTE_PATH'"

log "开始 rsync..."

run_cmd "rsync -avz $FINAL_FILE $REMOTE_USER@$REMOTE_IP:$REMOTE_PATH/"

log "远程同步完成"

# ---------------- 清理旧备份 ----------------
log "清理旧备份..."

run_cmd "find $BACKUP_DIR -name 'backup-*.tar.gz' -mtime +$RETENTION_DAYS -delete"

log "清理完成"

# ---------------- 计算大小 ----------------
SIZE=$(du -sh "$FINAL_FILE" | awk '{print $1}')

log "备份文件大小: $SIZE"

# ---------------- 飞书通知 ----------------
log "发送飞书通知..."

curl -s -X POST -H "Content-Type: application/json" \
-d "{
  \"msg_type\": \"text\",
  \"content\": {
    \"text\": \"【WordPress备份】\n✅ 成功\n文件: backup-$DATE.tar.gz\n大小: $SIZE\n服务器: $(hostname)\n时间: $(date '+%Y-%m-%d %H:%M:%S')\"
  }
}" "$FEISHU_WEBHOOK" >> "$LOG_FILE" 2>&1

log "========== 备份完成 =========="

exit 0
```

设置root的crontab，每天晚上3点进行备份

```
0 3 * * * /home/devops/wp-back.sh >> /home/devops/wp-back.log 2>&1
```

## 八、自动监控脚本

cpu和内存超过阈值报警飞书

```
#!/bin/bash

# ====================== 配置区 ======================
FEISHU_WEBHOOK="https://open.feishu.cn/open-apis/bot/v2/hook/a8edcbf7-fac5-45b4-b6e1-da19d0122b8d"
CPU_THRESHOLD=80
MEM_THRESHOLD=80
LOG_FILE="/var/log/monitor.log"
STATUS_FILE="/tmp/monitor_last_status.txt"      # 记录上一次状态（0=正常，1=超标）
LAST_ALERT_TIME_FILE="/tmp/monitor_last_alert.txt"  # 上次告警时间（防重复）
# =================================================

# 获取当前时间（秒）
CURRENT_TIME=$(date +%s)

# 获取 CPU 使用率（整数部分）
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}' | cut -d. -f1)

# 获取内存使用率（整数部分）
MEM_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100.0}' | cut -d. -f1)

# 当前状态：0=正常，1=超标
CURRENT_STATUS=0
if [ "$CPU_USAGE" -gt "$CPU_THRESHOLD" ] || [ "$MEM_USAGE" -gt "$MEM_THRESHOLD" ]; then
    CURRENT_STATUS=1
fi

# 读取上次状态（不存在默认为 0）
LAST_STATUS=$(cat "$STATUS_FILE" 2>/dev/null || echo 0)

# 读取上次告警时间（不存在默认为 0）
LAST_ALERT_TIME=$(cat "$LAST_ALERT_TIME_FILE" 2>/dev/null || echo 0)

# 日志记录当前指标
echo "[$(date '+%Y-%m-%d %H:%M:%S')] CPU: $CPU_USAGE% | MEM: $MEM_USAGE% | Status: $CURRENT_STATUS" >> "$LOG_FILE"

# 情况1：有超标 → 发送告警（每5分钟一次）
if [ "$CURRENT_STATUS" -eq 1 ]; then
    if [ $((CURRENT_TIME - LAST_ALERT_TIME)) -ge 300 ]; then
        MSG="【资源超标告警】\nCPU: $CPU_USAGE% (阈值 $CPU_THRESHOLD%)\n内存: $MEM_USAGE% (阈值 $MEM_THRESHOLD%)\n服务器: $(hostname)\n时间: $(date '+%Y-%m-%d %H:%M:%S')"
        curl -X POST -H "Content-Type: application/json" \
            -d "{\"msg_type\":\"text\",\"content\":{\"text\":\"$MSG\"}}" "$FEISHU_WEBHOOK" > /dev/null 2>&1
        echo "$CURRENT_TIME" > "$LAST_ALERT_TIME_FILE"
        echo "告警已发送：$MSG" >> "$LOG_FILE"
    fi
fi

# 情况2：从超标 → 全部恢复 → 发送恢复通知（只发一次）
if [ "$LAST_STATUS" -eq 1 ] && [ "$CURRENT_STATUS" -eq 0 ]; then
    MSG="【资源恢复正常】\nCPU: $CPU_USAGE% | 内存: $MEM_USAGE%\n服务器: $(hostname)\n时间: $(date '+%Y-%m-%d %H:%M:%S')"
    curl -X POST -H "Content-Type: application/json" \
        -d "{\"msg_type\":\"text\",\"content\":{\"text\":\"$MSG\"}}" "$FEISHU_WEBHOOK" > /dev/null 2>&1
    echo "恢复通知已发送：$MSG" >> "$LOG_FILE"
fi

# 更新当前状态到文件（供下次对比）
echo "$CURRENT_STATUS" > "$STATUS_FILE"

exit 0
```

设置root的crontab，每分钟执行一次

```
* * * * * /home/devops/monitor.sh >> /home/devops/monitor.log 2>&1
```
