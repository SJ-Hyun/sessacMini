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

