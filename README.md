# Web 서버 구축

## 1. HTTP

### 1.1 HTTP 설치
```
sudo dnf install -y httpd
```
### 1.2 HTTP 시작 및 재부팅 시 자동 시작
```
sudo systemctl start httpd
sudo systemctl enable httpd
```
### 1.3 HTTP 서비스 방화벽 열기
```
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```
## 2. PHP

### 2.1 PHP 설치
```
sudo dnf install -y php php-mysqlnd
```
wordpress는 php로 실행되므로, 이를 위해 php와 mysql 데이터베이스와 연결을 위한 `php-mysqlnd` 패키지를 설치한다.
## 3. WordPress

### 3.1 WordPress 압축 파일 가져오기
```
curl -o wordpress.tar.gz https://wordpress.org/latest.tar.gz
```
### 3.2 WordPress 파일 추출
```
sudo tar xvf wordpress.tar.gz -C /var/www/html
```
-C 옵션을 통해서 web에 적용하기 위한 웹서버 디렉토리에 압축을 직접 푼다.
### 3.3 WordPress config 파일 수정
```
sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
sudo vi /var/www/html/wordpress/wp-config.php

/** The name of the database for WordPress */
define( 'DB_NAME', 'database 이름' );

/** Database username */
define( 'DB_USER', 'mysql user 이름' );

/** Database password */
define( 'DB_PASSWORD', 'mysql user 비밀번호' );

/** Database hostname */
define( 'DB_HOST', 'DB 서버 ip 주소' );
```
WordPress config 파일을 작성하기 위해 기존에 있는 sample 파일을 복사 후 config 파일에 db에 대한 내용을 작성한다.

### 3.4 Apache 서버의 WordPress conf 파일 수정
```bash
sudo vi /etc/httpd/conf.d/wordpress.conf

<VirtualHost *:80>
        ServerName example.com
        DocumentRoot /var/www/html/wordpress
        <Directory "/var/www/html/wordpress">
                AllowOverride All
        </Directory>
</VirtualHost>

sudo systemctl restart httpd
```
해당 파일을 수정해 WordPress가 웹 서버에서 제대로 작동할 수 있도록 설정한다.

### 4. SELinux 설정
```
sudo setsebool -P httpd_can_network_connect_db 1
```
웹서버가 외부 DB와 연결될 수 있도록 SELinux의 보안 설정을 변경한다.
## 5. MySQL
### 5.1 MySQL 설치
```
sudo dnf install -y mysql
```
### 5.2 MySQL 접속
```
mysql -h "DB 서버 ip주소" -u "mysql user 이름" -p
```
외부 DB서버에서 생성된 데이터베이스에 접속한다.

# DB 서버 구축 
## 1. MySQL 

### 1.1 MySQL 설치
```
sudo dnf install -y mysql mysql-server
```
 서버에서는 DB 서버를 운영하기 위해 mysql-server 패키지를 추가적으로 설치한다. 
### 1.2 MySQL 시작 및 재부팅 시 자동 시작
```
sudo systemctl start mysqld
sudo systemctl enable mysqld
```
### 1.3 MySQL 서비스 방화벽 열기
```
sudo firewall-cmd --add-service=mysql --permanent
sudo firewall-cmd --reload
```
### 1.4 MySQL 접속
```
sudo mysql -u root
```
### 1.5 MySQL db 생성 및 user 생성, 권한 부여
```
mysql> create database "database 이름";

mysql> create user 'mysql user 이름'@'DB 서버의 게이트웨이' identified by 'mysql user 비밀번호';

mysql>  grant all privileges on "database 이름".* to 'mysql user 이름'@'DB 서버의 게이트웨이';
```
가상 머신이 동일한 서브넷에 있을 때는 직접 통신이 가능하지만, <br>
서로 다른 서브넷에 있으면 게이트웨이를 통해 라우팅해야 한다.
#### 1.5.1 (추가) bind-address로 접속 허용
```
sudo vi /etc/my.cnf
```
하지만 아직 시도 안 해봄

# LoadBalancer 서버 구축
Web 서버 1. http 부분을 따라 실행한다.

## 1. Nginx
### 1.1 Nginx 설치
```
sudo dnf -y update
sudo dnf -y install nginx
nginx -v
```
### 1.2 Nginx 시작 및 재부팅 시 자동 시작
```
sudo systemctl start nginx
sudo systemctl enable nginx
```
### 1.3 Nginx conf 파일 수정 
```bash
sudo cp /etc/nginx/nginx.conf nginx.conf.backup
sudo vi /etc/nginx/nginx.conf

log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"'
                      'upstream: $upstream_addr, request_time: $request_time';
                        # 해당 부분으로 access_log에서 로드밸런싱을 확인할 수 있다.
        upstream mini-web {
          server web서버1_ip주소:80 weight=100 max_fails=3 fail_timeout=3s;
          server web서버2_ip주소:80 weight=100 max_fails=3 fail_timeout=3s;
        }

        server {
                location / {
                  proxy_pass http://mini-web;
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection 'upgrade';
                  proxy_set_header Host $host;
                  proxy_cache_bypass $http_upgrade;
                }
        }


sudo systemctl reload nginx
sudo nginx -t
```
로드밸런서 설정을 해주기 위해 nginx.conf 파일을 수정한다. <br>
수정하기 전 nginx.conf 파일을 백업해둔다. <br>
nginx -t 명령어를 통해서 설정 파일의 구문 오류를 확인한다. <br>

### 2. SELinux 설정
```
sudo setsebool -P httpd_can_network_connect 1
```
nginx가 외부 네트워크로 연결할 수 있게 허용한다.

# NFS 공유 서버 구축

## 1. NFS 서버
### 1.1 NFS utils 설치
```
sudo dnf install -y nfs-utils
```
### 1.2 NFS 공유 디렉토리 생성
```
sudo mkdir 공유경로
echo "NFS share Test File" | sudo tee 공유경로/test.txt
```
### 1.3 NFS 서비스 시작 및 재부팅 시 자동 시작
```
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
```
### 1.4 NFS 경로 파일 작성
```
sudo vi /etc/exports
/srv/share      web서버 대역/24(rw,sync,no_root_squash)
```
/etc/exports  : NSF Server가 NFS Client들에게 export하는 모든 경로들을 지정하는 파일
여러 web 서버를 연결해야 하기 때문에 특정 ip 주소가 아닌 대역으로 작성한다.

### 1.5 NFS 서버 공유 확인
```
sudo exportfs -r
sudo exportfs -v
```
exportfs 명령어는 nfs서버를 다시 시작하지 않고도 공유목록을 수정할 수 있다.
출처: https://wdy0705.tistory.com/40 [지극히 개인적인 IT 노하우:티스토리]
-r : Reexport  all  directories /etc/exports 파일 다시 읽기
-v : Be verbose. When exporting or unexporting, show what's going on. 현재 공유 목록 확인

### 1.6 NFS 서버 방화벽 열기
```
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --add-service=rpc-bind --permanent
sudo firewall-cmd --add-service=mountd --permanent
sudo firewall-cmd --reload
```
## 2. NFS 클라이언트 서버 
### 2.1 NFS utils 설치
```
sudo dnf install -y nfs-utils
```
### 2.2 NFS 마운트 디렉토리 생성
```
sudo mkdir 마운트경로
```
### 2.3 NFS 마운트 위치 확인
```
showmount -e NFS 서버 ip 주소
```
-t : 마운트할 파일 시스템 유형 명시
### 2.4.1 수동 마운트
```
sudo mount -t nfs NFS 서버 ip 주소:공유경로 마운트경로
```
### 2.4.2 자동 마운트
```
```
### 2.5.1  NFS 마운트 확인
```
cat 마운트경로/test.txt
echo "${HOSTNAME} Test File" | sudo tee 마운트경로/client.txt
```

