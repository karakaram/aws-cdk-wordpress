# ec2-wordpress

## EFS

1. [Creating an Amazon Elastic File System \- Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug/creating-using-create-fs.html)
2. [Creating Access Points \- Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug/create-access-point.html)
3. [Creating Security Groups \- Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug/accessing-fs-create-security-groups.html)

## EC2

User data

```
#!/usr/bin/bash
yum update -y
rpm -Uvh https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm

curl -fsL -o /tmp/amazon-cloudwatch-agent.rpm https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
rpm -U /tmp/amazon-cloudwatch-agent.rpm
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-wp -s

groupadd www
gpasswd -a ec2-user www

amazon-linux-extras install nginx1 -y
gpasswd -a nginx www

cat << 'EOS' >/etc/nginx/conf.d/security.conf
add_header X-XSS-Protection "1; mode=block";
add_header X-Frame-Options SAMEORIGIN;
add_header X-Content-Type-Options nosniff;
EOS

cat << 'EOS' >/etc/nginx/nginx.conf
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx www;
worker_processes auto;
worker_rlimit_nofile  10000;

pcre_jit on;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 2048;
    multi_accept on;
}

http {
    server_tokens off;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    charset UTF-8;
    server_names_hash_bucket_size 128;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    client_max_body_size     20M;
    client_body_buffer_size 768k;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   5;
    connection_pool_size 512;

    gzip on;
    gzip_http_version 1.0;
    gzip_proxied any;
    gzip_min_length 1024;
    gzip_comp_level 2;
    gzip_types text/plain text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript application/json;

    open_file_cache max=1000 inactive=60s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 1;
    open_file_cache_errors on;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;
}
EOS

cat << 'EOS' >/etc/nginx/conf.d/default_http.conf
server {
    listen 80 default_server;
    access_log  /var/log/nginx/access-default.log main;
    error_log   /var/log/nginx/error-default.log warn;

    location / {
        access_log off;
        return 200;
    }
}
EOS

cat << 'EOS' >/etc/nginx/conf.d/wordpress_http.conf
server {
    listen 80;
    server_name example.com;
    rewrite ^(.*)$ http://www.example.com$1 last;
}

server {
    listen 80;
    server_name www.example.com;
    access_log  /var/log/nginx/access-wordpress.log main;
    error_log   /var/log/nginx/error-wordpress.log warn;

    charset UTF-8;
    client_max_body_size 16M;
    root  /var/www/html;
    index index.php index.html index.htm;

    location = /50x.html {
        return 403;
    }


    rewrite /wp-admin$ $scheme://$host$uri/ permanent;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location = /nginx-health {
        access_log off;
        return 200 "healthy\n";
    }

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location ~* /\.well-known {
        allow all;
    }

    location ~* /\. {
        deny all;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location = /xmlrpc.php {
        deny all;
    }

    location ~* \.(jpg|jpeg|gif|png|css|js|swf|ico|pdf|svg|eot|ttf|woff)$ {
        expires 60d;
        access_log off;
    }

    location ~* /wp-login\.php|/wp-admin/((?!(admin-ajax\.php|images/)).)*$ {

        #satisfy any;
        #allow 0.0.0.0/0;
        #allow 127.0.0.1;
        #deny all;
        auth_basic "basic authentication";
        auth_basic_user_file  "/var/www/.htpasswd";

        location ~ [^/]\.php(/|$) {

            fastcgi_split_path_info ^(.+?\.php)(/.*)$;
            if (!-f $document_root$fastcgi_script_name) {
                return 404;
            }
            fastcgi_pass php-fpm;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_buffers 256 128k;
            fastcgi_buffer_size 128k;
            fastcgi_intercept_errors on;
            fastcgi_read_timeout 120s;
        }
        include conf.d/security.conf;
    }

    location ~ [^/]\.php(/|$) {

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) {
                return 404;
        }
        fastcgi_pass php-fpm;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_buffers 256 128k;
        fastcgi_buffer_size 128k;
        fastcgi_intercept_errors on;
        fastcgi_read_timeout 120s;

        include conf.d/security.conf;
    }
}
EOS


amazon-linux-extras install php7.3 -y
sed -i -r -e 's/^;?date.timezone =.*$/date.timezone = "UTC"/g' /etc/php.ini
#sed -i -r -e 's/^;?date.timezone =.*$/date.timezone = "Asia\/Tokyo"/g' /etc/php.ini
sed -i -r -e 's/^;?expose_php =.*$/expose_php = Off/g' /etc/php.ini
sed -i -r -e 's/^;?max_execution_time =.*$/max_execution_time = 120/g' /etc/php.ini
sed -i -r -e 's/^;?post_max_size =.*$/post_max_size = 16M/g' /etc/php.ini
sed -i -r -e 's/^;?upload_max_filesize =.*$/upload_max_filesize = 16M/g' /etc/php.ini
sed -i -r -e 's/^;?pdo_mysql\.default_socket=.*$/pdo_mysql\.default_socket=\/var\/lib\/mysql\/mysql\.sock/g' /etc/php.ini
sed -i -r -e 's/^;?mysqli\.default_socket =.*$/mysqli\.default_socket = \/var\/lib\/mysql\/mysql\.sock/g' /etc/php.ini
sed -i -r -e 's/^;?session.cookie_secure =.*$/session.cookie_secure = 1/g' /etc/php.ini
sed -i -r -e 's/^;?emergency_restart_threshold =.*$/emergency_restart_threshold = 10/g' /etc/php-fpm.conf
sed -i -r -e 's/^;?emergency_restart_interval =.*$/emergency_restart_interval = 1m/g' /etc/php-fpm.conf
sed -i -r -e 's/^;?process_control_timeout =.*$/process_control_timeout = 10/g' /etc/php-fpm.conf
sed -i -r -e 's/^;?user =.*$/user = nginx/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?group =.*$/group = www/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?pm.max_requests =.*$/pm.max_requests = 500/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?pm.max_children =.*$/pm.max_children = 10/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?pm.start_servers =.*$/pm.start_servers = 2/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?pm.min_spare_servers =.*$/pm.min_spare_servers = 1/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?pm.max_spare_servers =.*$/pm.max_spare_servers = 3/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?request_slowlog_timeout =.*$/request_slowlog_timeout = 10/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?request_terminate_timeout =.*$/request_terminate_timeout = 120/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?catch_workers_output =.*$/catch_workers_output = yes/g' /etc/php-fpm.d/www.conf

#sed -i -r -e 's/^;?listen =.*$/listen = 127.0.0.1:9000/g' /etc/php-fpm.d/www.conf
#sed -i -r -e 's/unix:\/run\/php-fpm\/www.sock;/127.0.0.1:9000/g' /etc/nginx/conf.d/php-fpm.conf

# php7.3
yum install php-bcmath php-gd php-mysqlnd php-mbstring php-opcache php-pecl-imagick php-xml -y

opcache.memory_consumption
sed -i -r -e 's/^;?opcache\.memory_consumption=.*$/opcache.memory_consumption=64/g' /etc/php.d/10-opcache.ini
sed -i -r -e 's/^;?opcache\.revalidate_freq=.*$/opcache.revalidate_freq=900/g' /etc/php.d/10-opcache.ini

yum install mysql -y

systemctl start php-fpm
systemctl enable php-fpm

systemctl start nginx
systemctl enable nginx

# reboot on kernel version up
grubby --default-kernel | grep `uname -r` || (sleep 10; reboot)&
```

## Install MySQL Server

https://dev.mysql.com/doc/refman/8.0/en/linux-installation-yum-repo.html

```
curl -fsLO https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
yum install mysql80-community-release-el7-3.noarch.rpm
yum-config-manager --disable mysql57-community
yum-config-manager --enable mysql80-community
yum repolist enabled | grep mysql
yum module disable mysql
yum install mysql-community-server
```

Configure my.cnf

When you have 200MB of memory available for MySQL

```
cat << 'EOS' >>/etc/my.cnf

character-set-server = utf8mb4
collation-server = utf8mb4_ja_0900_as_cs_ks
max_connections = 14
innodb_buffer_pool_size = 134217728
innodb_flush_method = O_DIRECT
EOS
```

When you have 1GB of memory available for MySQL

```
max_connections = 84
innodb_buffer_pool_size = 805306368
```


Variables (RDS MySQL8 t3.micro default)

```
t3.micro's DBInstanceClassMemory is 536870912 or 805304320
show variables like 'max_connections'; {DBInstanceClassMemory/12582880} 64
show variables like 'thread_cache_size'; default 8
show variables like 'sort_buffer_size'; default 262144 256KB
show variables like 'read_buffer_size'; 262144
show variables like 'key_buffer_size'; 16777216 16MB
show variables like 'innodb_buffer_pool_size'; {DBInstanceClassMemory*3/4} 402653184 384MB
show variables like 'innodb_log_file_size'; 134217728 128MB
show variables like 'innodb_file_per_table'; 1
show variables like 'innodb_flush_log_at_trx_commit'; default 1
show variables like 'innodb_flush_method'; O_DIRECT
show variables like 'innodb_buffer_pool_instances'; default 1
show variables like 'innodb_io_capacity'; default 200
show variables like 'innodb_read_io_threads'; default 4
show variables like 'innodb_write_io_threads'; default 4
show variables like 'gtid_mode'; OFF_PERMISSIVE
```

Status

```
show status like 'Max_used_connections'; 12
show status like 'Threads_created'; 24
show status like 'Sort_merge_passes'; 0
show engine innodb status;
select event_name, count_star, sum_timer_wait/1000000000 sum_timer_wait_ms from performance_schema.events_waits_summary_global_by_event_name where event_name like '%buf_pool_mutex%';
select * from sys.innodb_buffer_stats_by_schema;
```

Start MySQL

```
systemctl start mysqld
systemctl enable mysqld
```

Initialize users

```
grep 'temporary password' /var/log/mysqld.log

mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
CREATE DATABASE wordpress;
CREATE USER 'wordpress'@'127.0.0.1' IDENTIFIED WITH mysql_native_password BY 'MyNewPass4!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'127.0.0.1' WITH GRANT OPTION;
CREATE USER 'wordpress'@'%' IDENTIFIED WITH mysql_native_password BY 'MyNewPass4!';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'%' WITH GRANT OPTION;
```

Export Database

```
mysqldump -umyuser -p --default-character-set=utf8mb4 --host=myhost --single-transaction --set-gtid-purged=OFF --no-tablespaces wordpress > wordpress.sql
```

Import Database

```
mysql -umyuser -p --default-character-set=utf8mb4 --host=myhost wordpress < wordpress.sql
```

## Setup wordpress

Download wp-cli

```
curl -fsL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x /usr/local/bin/wp
```

Download wordpress

```
mkdir -p /var/www
wp core download --locale=ja --path=/var/www/wordpress
rm -f /var/www/wordpress/wp-config-sample.php
rm -f /var/www/wordpress/readme.html
wp config create --path=/var/www/wordpress --dbname=wordpress --dbuser=supervisor --dbpass=strongpassword --dbhost=host --dbprefix=wp_ --dbcharset=utf8mb4 --dbcollate=utf8mb4_ja_0900_as_cs_ks --extra-php <<'EOS'
define('DISABLE_WP_CRON', true);
define('WP_POST_REVISIONS', false);
if (isset($_SERVER['HTTP_CLOUDFRONT_FORWARDED_PROTO']) && $_SERVER['HTTP_CLOUDFRONT_FORWARDED_PROTO'] == 'https') {
    $_SERVER['HTTPS'] = 'on';
} elseif (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https') {
    $_SERVER['HTTPS'] = 'on';
}
EOS
#wp db drop --yes --path=/var/www/wordpress
wp db create --path=/var/www/wordpress
wp core install --path=/var/www/wordpress --url=https://www.xample.com --title=Example --admin_user=supervisor --admin_password=strongpassword --admin_email=info@example.com
wp plugin install advanced-database-cleaner --activate --path=/var/www/wordpress
wp plugin install amazonjs --activate --path=/var/www/wordpress
wp plugin install better-search-replace --activate --path=/var/www/wordpress
wp plugin install classic-editor --activate --path=/var/www/wordpress
wp plugin install disable-comments --activate --path=/var/www/wordpress
wp plugin install easy-wp-meta-description --activate --path=/var/www/wordpress
wp plugin install google-sitemap-generator --activate --path=/var/www/wordpress
wp plugin install health-check --activate --path=/var/www/wordpress
wp plugin install wp-mail-smtp --activate --path=/var/www/wordpress
wp plugin install flush-opcache --activate --path=/var/www/wordpress
chown -R nginx:www /var/www
find /var/www -type d | xargs chmod -R 775
find /var/www -type f | xargs chmod -R 664
```

Mount EFS File Systems and 

[Mounting EFS File Systems \- Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug/mounting-fs.html)

```
yum install -y amazon-efs-utils
mkdir -p /var/www/html
chown nginx:www /var/www/html
mount -t efs -o tls,accesspoint=fsap-1234567890xxxxxxx fs-12345678 /var/www/html
```

Move wordpress files to EFS

```
cp -a /var/www/wordpress/* /var/www/html
rm -rf /var/www/wordpress
```

[Mounting Your Amazon EFS File System Automatically \- Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug/mount-fs-auto-mount-onreboot.html)

```
grep -q fs-12345678 /etc/fstab || echo 'fs-12345678 /var/www/html efs _netdev,tls,accesspoint=fsap-1234567890xxxxxxx 0 0' >> /etc/fstab
```

Unmount EFS File System

```
umount /var/www/html
```

Configure log roaates

```
# log permission for nginx
chmod 775 /var/log/nginx

chown -R nginx:root /var/log/php-fpm
find /var/log/php-fpm -type d | xargs chmod -R 775
find /var/log/php-fpm -type f | xargs chmod -R 644
cat << 'EOS' | sed -i -e "\/var\/log\/php-fpm\/\*log {/r /dev/stdin" /etc/logrotate.d/php-fpm
    create 0644 nginx root
    daily
    rotate 10
EOS

logrotate -d /var/log/php-fpm

# wp-cron
cat << 'EOS' >/etc/cron.d/wp-cron
0-59/15 * * * * nginx /usr/local/bin/wp cron event run --due-now --path=/var/www/html >/dev/null 2>&1
EOS
```

## Import data

Before importing

```
sed -i -r -e 's/^max_execution_time =.*.*$/max_execution_time = 3600/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^request_terminate_timeout =.*.*$/request_terminate_timeout = 3600/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/fastcgi_read_timeout.*$/fastcgi_read_timeout 3600s;/g' /etc/nginx/conf.d/wordpress_http.conf
sed -i -r -e 's/^max_execution_time = .*$/max_execution_time = 3600/g' /etc/php.ini
systemctl restart nginx
systemctl restart php-fpm
```

After importing

```
sed -i -r -e 's/^max_execution_time =.*.*$/max_execution_time = 120/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/^;?request_terminate_timeout =.*.*$/request_terminate_timeout = 120/g' /etc/php-fpm.d/www.conf
sed -i -r -e 's/fastcgi_read_timeout.*$/fastcgi_read_timeout 120s;/g' /etc/nginx/conf.d/wordpress_http.conf
sed -i -r -e 's/^max_execution_time = .*$/max_execution_time = 120/g' /etc/php.ini
systemctl restart nginx
systemctl restart php-fpm
```

Update

```
wp option update home 'https://example.com' --path=/var/www/html
wp option update siteurl 'https://example.com' --path=/var/www/html
```

## Update wordpress

```
wp core update
wp core update-db
```

## CloudWatch Agent

1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/download-cloudwatch-agent-commandline.html
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file.html
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html
1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-on-EC2-Instance-fleet.html

```
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

On which OS are you planning to use the agent?
1. linux
2. windows
default choice: [1]:
1
Trying to fetch the default region based on ec2 metadata...
Are you using EC2 or On-Premises hosts?
1. EC2
2. On-Premises
default choice: [1]:
1
Which user are you planning to run the agent?
1. root
2. cwagent
3. others
default choice: [1]:
2
Do you want to turn on StatsD daemon?
1. yes
2. no
default choice: [1]:
1
Which port do you want StatsD daemon to listen to?
default choice: [8125]
8125
What is the collect interval for StatsD daemon?
1. 10s
2. 30s
3. 60s
default choice: [1]:
1
What is the aggregation interval for metrics collected by StatsD daemon?
1. Do not aggregate
2. 10s
3. 30s
4. 60s
default choice: [4]:
4
Do you want to monitor metrics from CollectD?
1. yes
2. no
default choice: [1]:
2
Do you want to monitor any host metrics? e.g. CPU, memory, etc.
1. yes
2. no
default choice: [1]:
1
Do you want to monitor cpu metrics per core? Additional CloudWatch charges may apply.
1. yes
2. no
default choice: [1]:
1
Do you want to add ec2 dimensions (ImageId, InstanceId, InstanceType, AutoScalingGroupName) into all of your metrics if the info is available?
1. yes
2. no
default choice: [1]:
2
Would you like to collect your metrics at high resolution (sub-minute resolution)? This enables sub-minute resolution for all metrics, but you can customize for specific metrics in the output json file.
1. 1s
2. 10s
3. 30s
4. 60s
default choice: [4]:
4
Which default metrics config do you want?
1. Basic
2. Standard
3. Advanced
4. None
default choice: [1]:
2
Are you satisfied with the above config? Note: it can be manually customized after the wizard completes to add additional items.
1. yes
2. no
default choice: [1]:
1
Do you have any existing CloudWatch Log Agent (http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AgentReference.html) configuration file to import for migration?
1. yes
2. no
default choice: [2]:
2
Do you want to monitor any log files?
1. yes
2. no
default choice: [1]:
1
Log file path:
/var/log/nginx/access-wordpress.log
Log group name:
default choice: [access-wordpress.log]
wp-logs
Log stream name:
default choice: [{instance_id}]
nginx-access-log
Do you want to specify any additional log files to monitor?
1. yes
2. no
default choice: [1]:
2
Saved config file to /opt/aws/amazon-cloudwatch-agent/bin/config.json successfully.
...
Please check the above content of the config.
The config file is also located at /opt/aws/amazon-cloudwatch-agent/bin/config.json.
Edit it manually if needed.
Do you want to store the config in the SSM parameter store?
1. yes
2. no
default choice: [1]:
1
What parameter store name do you want to use to store your config? (Use 'AmazonCloudWatch-' prefix if you use our managed AWS policy)
default choice: [AmazonCloudWatch-linux]
AmazonCloudWatch-wp
Trying to fetch the default region based on ec2 metadata...
Which region do you want to store the config in the parameter store?
default choice: [ap-northeast-1]
ap-northeast-1
Which AWS credential should be used to send json config to parameter store?
1. XXXXXXXXXXXXXXXXXXXXX(From SDK)
2. Other
default choice: [1]:
1
Successfully put config to parameter store AmazonCloudWatch-wp.
Program exits now.

Log group name:
wp-logs
/var/log/nginx/access-wordpress.log
nginx-access-log
/var/log/nginx/error-wordpress.log
nginx-error-log
/var/log/php-fpm/error.log
php-fpm-error-log
/var/log/php-fpm/www-slow.log
php-fpm-slow-log
```

## debug

```
curl -H 'Host:www.example.com' -H 'X-Forwarded-Proto: https' --head http://localhost
```
