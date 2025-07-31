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
### 1.3 HTTP 방화벽 열기
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

## 3. Web 서버의 WordPress conf 파일 수정
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

## 4. SELinux 설정
```
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_use_nfs 1
```
웹서버가 외부 DB와 연결될 수 있도록 SELinux의 보안 설정을 변경한다.
웹서버가 NFS 마운트 디렉터리에 접근을 허용할 수 있도록한다.

## 5. MySQL
### 5.1 MySQL 설치
```
sudo dnf install -y mysql
```
### 5.2 MySQL 접속
```
mysql -h "DB 서버 주소" -u "mysql user 이름" -p
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
          server web서버1_주소:80 weight=100 max_fails=3 fail_timeout=3s;
          server web서버2_주소:80 weight=100 max_fails=3 fail_timeout=3s;
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

## 2. SELinux 설정
```
sudo setsebool -P httpd_can_network_connect 1
```
nginx가 외부 네트워크로 연결할 수 있게 허용한다.
### 3. HTTP 방화벽 열기
```
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```

# NFS 공유 서버 구축

## 1. NFS 서버
### 1.1 NFS 
#### 1.1.1 NFS utils 설치
```
sudo dnf install -y nfs-utils
```
#### 1.1.2 NFS 공유 디렉토리 생성
```
sudo mkdir -p /var/www/html/wordpress
```
#### 1.1.3 NFS 서비스 시작 및 재부팅 시 자동 시작
```
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
```
#### 1.1.4 NFS 경로 파일 작성
```
sudo vi /etc/exports
/srv/share      web서버 대역/24(rw,sync,no_root_squash)
```
/etc/exports  : NSF Server가 NFS Client들에게 export하는 모든 경로들을 지정하는 파일 <br>
여러 web 서버를 연결해야 하기 때문에 특정 ip 주소가 아닌 대역으로 작성한다.

#### 1.1.5 NFS 서버 공유 확인
```
sudo exportfs -r
sudo exportfs -v
```
exportfs 명령어는 nfs서버를 다시 시작하지 않고도 공유목록을 수정할 수 있다. <br>
출처: https://wdy0705.tistory.com/40 [지극히 개인적인 IT 노하우:티스토리] <br>
-r : Reexport  all  directories /etc/exports 파일 다시 읽기 <br>
-v : Be verbose. When exporting or unexporting, show what's going on. 현재 공유 목록 확인 <br>

#### 1.1.6 NFS 서버 방화벽 열기
```
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --add-service=rpc-bind --permanent
sudo firewall-cmd --add-service=mountd --permanent
sudo firewall-cmd --reload
```

### 1.2 WordPress
#### 1.2.1 WordPress 압축 파일 가져오기
```
curl -o wordpress.tar.gz https://wordpress.org/latest.tar.gz
```
#### 1.2.2 WordPress 파일 추출
```
sudo tar xvf wordpress.tar.gz -C /var/www/html
```
-C 옵션을 통해서 web에 적용하기 위한 웹서버 디렉토리에 압축을 직접 푼다.
#### 1.2.3 WordPress config 파일 수정
```bash
sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
sudo vi /var/www/html/wordpress/wp-config.php

/** The name of the database for WordPress */
define( 'DB_NAME', 'database 이름' );

/** Database username */
define( 'DB_USER', 'mysql user 이름' );

/** Database password */
define( 'DB_PASSWORD', 'mysql user 비밀번호' );

/** Database hostname */
define( 'DB_HOST', 'DB 서버(primary) 주소' );
```
WordPress config 파일을 작성하기 위해 기존에 있는 sample 파일을 복사 후 config 파일에 db에 대한 내용을 작성한다.
## 2. NFS 클라이언트 서버 (Web)
### 2.1 NFS utils 설치
```
sudo dnf install -y nfs-utils
```
### 2.2 NFS 마운트 디렉토리 생성
```
sudo mkdir -p /var/www/html/wordpress
```
### 2.3 NFS 마운트 위치 확인
```
showmount -e NFS 서버 주소
```
### 2.4 NFS 마운트 수행
#### 2.4.1 수동 마운트
```bash
sudo mount -t nfs NFS 서버 주소:/var/www/html/wordpress /var/www/html/wordpress

sudo vi /etc/fstab
NFS 서버 주소:/var/www/html/wordpress /var/www/html/wordpress nfs defaults 0 0

sudo mount -a
```
-t : 마운트할 파일 시스템 유형 명시
#### 2.4.2 자동 마운트
```bash
sudo dnf install -y autofs

sudo vi /etc/auto.master.d/nfs.autofs
/-      /etc/auto.direct

sudo vi /etc/auto.direct
/var/www/html/wordpress        -rw,sync NFS 서버 주소:/var/www/html/wordpress

sudo systemctl start autofs
```
### 2.5.1  NFS 마운트 확인
```
mount | grep wordpress
```
# iSCSI 서버 구축
## 1. iSCSI 서버
### 1.1 targetcli 설치
```
sudo dnf install -y targetcli
```
targetcli : 리눅스에서 iSCSI 타겟을 구성하고 관리하는 데 사용되는 대화형 쉘

### 1.2 targetcli 서비스 시작 및 재부팅 시 자동 시작
```
sudo systemctl enable --now target
```
### 1.3 targetcli 실행
```
sudo targetcli
```
### 1.4 iSCSI 구성
```
/> /backstores/block create name=disk-p dev=/dev/sdb
/> /backstores/block create name=disk-s dev=/dev/sdc
/> /iscsi create wwn=iqn.2025-07.com.example:storage-p
/> /iscsi create wwn=iqn.2025-07.com.example:storage-s
/> /iscsi/iqn.2025-07.com.example:storage-p/tpg1/luns create /backstores/block/disk-p
/> /iscsi/iqn.2025-07.com.example:storage-s/tpg1/luns create /backstores/block/disk-s
/> /iscsi/iqn.2025-07.com.example:storage-p/tpg1/acls create iqn.2025-07.com.example:db-p
/> /iscsi/iqn.2025-07.com.example:storage-s/tpg1/acls create iqn.2025-07.com.example:db-s
/> saveconfig
/> exit
```
각 명령에 대한 설명은... 나중에...
### 1.5 방화벽 열기
```
sudo firewall-cmd --add-port=3260/tcp --permanent
sudo firewall-cmd --reload
```
3260/tcp = iSCSI(Target) 서버와 Initiator(클라이언트)가 통신하기 위해 사용하는 표준 포트

## 2. DB 서버
### 2.1 iSCSI 패키지 설치
```
sudo dnf install -y iscsi-initiator-utils
```
### 2.2  InitiatorName 파일 작성
```bash
sudo vi /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2025-07.com.example:db-p
```
### 2.3 iSCSI 서비스 시작 및 재부팅 시 자동 시작
```
sudo systemctl start iscsi
sudo systemctl enable iscsi
```
### 2.4 타겟 검색 및 로그인
```
sudo iscsiadm -m discovery -t st -p iSCSI 서버 주소
sudo iscsiadm -m node -T iqn.2025-07.com.example:storage1 -p iSCSI 서버 주소 -l
```
-T 옵션은 iqn을 지정하는 것으로 위의 명령어를 실행했을 때 -t st를 통해 iqn이 출력된다.

### 2.5 연결 확인
```
sudo iscsiadm -m session -P 3
lsblk
```
### 2.6 마운트 설정
```
sudo mkdir /mnt/mysql
sudo mkfs.xfs /dev/sdb
sudo mount /dev/sdb /mnt/mysql
sudo vi /etc/fstab
/dev/sdb /mnt/mysql xfs defaults,_netdev 0 0

sudo mount -a
lsblk
```
마운트가 네트워크 연결보다 먼저 일어나기 때문에  <br> 
시스템 시작 시 자동 마운트되지 않기 때문에 접속이 되지 않는 현상이 발생한다.<br>
이를 해결하기 위해 _netdev 옵션을 적으면 네트워크 연결 후 마운트하게 된다.
### 2.7 db 저장 경로 변경
mysql에 접속해 select @@datadir를 통해서 /var/lib/mysql/에 데이터가 저장된다는 것을 확인할 수 있다. <br>
db 서버가 에러 나더라도 데이터 보존을 위해 <br>
iSCSI로 새로 마운트한 /mnt/mysql로 mysql의 데이터가 저장되도록 경로를 변경한다.
```bash
sudo systemctl stop mysqld
sudo chown -R mysql:mysql /mnt/mysql
sudo chmod 750 /mnt/mysql
sudo rsync -av /var/lib/mysql/ /mnt/mysql/

sudo vi /etc/my.cnf
[mysqld]
datadir=/mnt/mysql
socket=/mnt/mysql/mysql.sock

[client]
socket=/mnt/mysql/mysql.sock

sudo vi /etc/my.cnf.d/mysql-server.cnf
[mysqld]
datadir=/mnt/mysql
socket=/mnt/mysql/mysql.sock
log-error=/mnt/mysql/mysqld.log
pid-file=/run/mysqld/mysqld.pid
```

### 2.8 SELinux 설정
```
sudo semanage fcontext -a -t mysqld_db_t "/mnt/mysql(/.*)?"
sudo restorecon -R /mnt/mysql

ls -Zd /mnt/mysql # 적용여부 확인
system_u:object_r:mysqld_db_t:s0 /mnt/mysql
```
semanage fcontext -a -t mysqld_db_t "/mnt/mysql(/.*)?" <br>
: /mnt/mysql과 그 하위 경로들에 대해 SELinux의 mysqld_db_t 타입(context)을 부여
restorecon -R /mnt/mysql  <br>
: semanage로 등록한 정책을 하위 디렉토리와 파일까지 실제로 적용

# DB Replication
## 1. Primary 서버
### 1.1 Secondary 서버 유저 생성
```
mysql> CREATE DATABASE wp DEFAULT CHARACTER SET utf8;
mysql> CREATE USER 'repl'@'secondary 서버주소' IDENTIFIED BY '비밀번호';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'secondary 서버주소';
```
### 1.2 mysql conf 파일 작성
```
sudo vi /etc/my.cnf
[mysqld]
log-bin=mysql-bin
server-id=1
bind-address=0.0.0.0

sudo systemctl restart mysqld.service
```
### 1.3 
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      157 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
### 1.4
```
sudo mysqldump -u root -p wp > wp.sql
scp -P 22 wp.sql vagrant@secondary서버주소:/home/vagrant/
```
## 2. Secondary 서버
### 2.1
```
mysql> CREATE DATABASE wp DEFAULT CHARACTER SET utf8;
```
### 2.2 
```
sudo vi /etc/my.cnf
[mysqld]
server-id=2
read_only=1
super_read_only=1
```
### 2.3
```
sudo mysql -u root -p wp < wp.sql
```
### 2.4
```
mysql> CHANGE REPLICATION SOURCE TO
    -> SOURCE_HOST='Primary서버주소',
    -> SOURCE_USER='repl',
    -> SOURCE_PASSWORD='비밀번호',
    -> SOURCE_LOG_FILE='mysql-bin.000001',
    -> SOURCE_LOG_POS=157,
    -> SOURCE_PORT=3306,
    -> SOURCE_SSL=1;
mysql> START SLAVE;
```
### 2.5
```
sudo systemctl restart mysqld.service
show slave status\G;
```
show slave status의 출력 결과 중 아래처럼 나오면 성공적으로 replication이 된 것이다.
Slave_IO_Running: Yes
Slave_SQL_Running: Yes


