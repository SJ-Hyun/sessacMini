# 구성
![구성도](https://github.com/SJ-Hyun/sessacMini/blob/main/infra.png?raw=true)

#### 서버 ip 주소 <br>
>LB : 192.168.55.10 <br>
Web-01 : 192.168.56.11 <br>
Web-02 : 192.168.56.12 <br>
NFS : 192.168.56.13 <br>
DB-01 : 192.168.57.14 | 192.168.56.14 <br>
DB-02 : 192.168.57.15 <br>
iSCSI : 192.168.57.16

# 목표
1. 로드밸런서(LB) 서버를 통해 두 대의 Web 서버에 대한 HTTP 트래픽을 분산 (Load Balancing)

2. NFS 서버를 활용하여, 두 Web 서버가 동일한 WordPress 디렉토리를 공유

3. Primary–Secondary 구조의 DB Replication을 통해 데이터 이중화 구현

4. DB 서버의 저장소를 iSCSI 서버에서 제공하는 디스크에 마운트하여, DB 데이터의 물리적 분리 및 안정성 확보
<br><br>
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
WorPress는 PHP로 실행되므로, 이를 위해 PHP와 MySQL의 연결을 위한 php-mysqlnd 패키지를 설치한다.

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
```
```
sudo systemctl restart httpd
```
해당 파일을 수정해 WordPress가 웹 서버에서 제대로 작동할 수 있도록 설정한다.

## 4. SELinux 설정
```
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_use_nfs 1
```
웹서버가 외부 DB와 연결될 수 있도록 SELinux의 보안 설정을 변경한다.<br>
웹서버가 NFS 마운트 디렉터리에 접근을 허용할 수 있도록한다. <br>
<br>
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
### 1.4 MySQL db 생성 및 user 생성, 권한 부여
```
sudo mysql -u root

mysql> create database database_이름;

mysql> create user 'user 이름'@'Web 서버 대역' identified by 'user 비밀번호';

mysql>  grant all privileges on database_이름.* to 'user 이름'@'Web 서버 대역';
```
<br>

# LoadBalancer 서버 구축

<!--  LoadBalancing를 위해 대표적으로 사용하는 프로그램은 HA Proxy와 Nginx 등등이 있다. <br>
HA Proxy도 해보고싶은데 너무너무 지쳐요.....-->

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
        upstream wp-web {
          server web서버1_주소:80;
          server web서버2_주소:80;
        }

        server {
                location / {
                  proxy_pass http://wp-web;
                  proxy_http_version 1.1;
                }
        }
```
```
sudo systemctl reload nginx
sudo nginx -t
```
로드밸런서 설정을 해주기 위해 nginx.conf 파일을 수정한다. <br>
nginx -t 명령어를 통해서 설정 파일의 구문 오류를 확인한다. <br>

## 2. SELinux 설정
```
sudo setsebool -P httpd_can_network_connect 1
```
nginx가 외부 네트워크로 연결할 수 있게 허용한다.
## 3. HTTP 방화벽 열기
```
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```
<br>

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
/var/www/html/wordpress   192.168.56.0/24(rw,sync,no_root_squash)
```
/etc/exports  : NSF Server가 NFS Client들에게 export하는 모든 경로들을 지정하는 파일 <br>
여러 web 서버를 연결해야 하기 때문에 특정 ip 주소가 아닌 대역으로 작성한다.

#### 1.1.5 NFS 서버 공유 확인
```
sudo exportfs -r
sudo exportfs -v
```
exportfs 명령어는 nfs서버를 다시 시작하지 않고도 공유목록을 수정할 수 있다. <br>
-r : /etc/exports 파일 다시 읽기 <br>
-v : 현재 공유 목록 확인 <br>

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
-C 옵션을 통해서 web 서버에 공유할 디렉토리에 압축을 직접 푼다.
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
showmount -e NFS서버_ip주소
```
### 2.4 NFS 마운트 수행
#### 2.4.1 수동 마운트
```
sudo mount -t nfs NFS서버_ip주소:/var/www/html/wordpress /var/www/html/wordpress
```
```bash
sudo vi /etc/fstab
NFS서버_ip주소:/var/www/html/wordpress /var/www/html/wordpress nfs defaults 0 0
```
```
sudo mount -a
```
-t : 마운트할 파일 시스템 유형 명시
#### 2.4.2 자동 마운트
```bash
sudo dnf install -y autofs
```
```bash
sudo vi /etc/auto.master.d/nfs.autofs
/-      /etc/auto.direct
```
```bash
sudo vi /etc/auto.direct
/var/www/html/wordpress        -rw,sync NFS서버_ip주소:/var/www/html/wordpress
```
```
sudo systemctl start autofs
```
### 2.5  NFS 마운트 확인
```
mount | grep wordpress
```
<br>

# iSCSI 서버 구축
## 1. iSCSI 서버
우선적으로 vbox 환경에서 20GB 디스크 2개를 추가한다. <br>
2개를 추가하면 /dev/sdb, /dev/sdc가 추가된 것을 확인할 수 있다.
### 1.1 targetcli 설치
```
sudo dnf install -y targetcli
```
targetcli : 리눅스에서 iSCSI 타겟을 구성하고 관리하는 데 사용되는 대화형 쉘

### 1.2 targetcli 서비스 시작 및 재부팅 시 자동 시작
```
sudo systemctl start target
sudo systemctl enable target
```
### 1.3 iSCSI 구성
```
sudo targetcli
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
iSCSI 서버의 sdb는 primary 서버에, sdc는 secondary 서버에 연결하고 <br>
구별을 위해 -p, -s로 구분하여 이름을 정해주었다.

### 1.4 방화벽 열기
```
sudo firewall-cmd --add-port=3260/tcp --permanent
sudo firewall-cmd --reload
```
iSCSI(Target) 서버와 클라이언트(Initiator)가 통신하기 위해 사용하는 표준 포트인 3260을 열어준다.

## 2. DB 서버
2개의 DB 서버 모두에서 진행하되 위 iSCCI 구성에서 작성한 db별 이름으로 적용한다. <br>
### 2.1 iSCSI initiator 패키지 설치
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
sudo iscsiadm -m discovery -t st -p iSCSI서버_ip주소
sudo iscsiadm -m node -T iqn.2025-07.com.example:storage-p -p iSCSI서버_ip주소 -l
```
-t st를 통해 iqn이 출력되면 로그인 시 -T 옵션으로 출력된 iqn을 입력한다. 

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
```
```bash
sudo vi /etc/fstab
/dev/sdb /mnt/mysql xfs defaults,_netdev 0 0
```
```
sudo mount -a
lsblk
```
마운트가 네트워크 연결보다 먼저 일어나기 때문에 시스템 시작 시 마운트되지 않아 접속이 되지 않는 현상이 발생한다.<br>
이를 해결하기 위해 _netdev 옵션을 적으면 네트워크 연결 후 마운트하게 된다.
### 2.7 db 저장 경로 변경
mysql에 접속해 select @@datadir 명령어로 /var/lib/mysql/에 데이터가 저장된다는 것을 확인할 수 있다. <br>
db 서버에 에러가 발생하더라도 데이터 손실 방지를 위해 iSCSI로 새로 마운트한 /mnt/mysql로 mysql의 데이터가 저장되도록 경로를 변경한다.
```bash
sudo systemctl stop mysqld
sudo chown -R mysql:mysql /mnt/mysql
sudo chmod 750 /mnt/mysql
sudo rsync -av /var/lib/mysql/ /mnt/mysql/
```
```bash
sudo vi /etc/my.cnf
[mysqld]
datadir=/mnt/mysql
socket=/mnt/mysql/mysql.sock

[client]
socket=/mnt/mysql/mysql.sock
```
```bash
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
# /mnt/mysql과 그 하위 경로들에 대해 SELinux의 mysqld_db_t 타입(context)을 부여

sudo restorecon -R /mnt/mysql
# semanage로 등록한 정책을 하위 디렉토리와 파일까지 실제로 적용

ls -Zd /mnt/mysql # 적용여부 확인
system_u:object_r:mysqld_db_t:s0 /mnt/mysql

sudo systemctl start mysqld
sudo mysql -u root
```
<br>

# DB Replication
## 1. Primary 서버
### 1.1 Secondary 서버 유저 생성
```
mysql> CREATE DATABASE db이름 DEFAULT CHARACTER SET utf8;
mysql> CREATE USER 'user 이름'@'secondary서버_ip주소' IDENTIFIED BY 'user 비밀번호';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'user 이름'@'secondary서버_ip주소';
```
Primary DB와 Secondary DB가 연결이 되어야하기 때문에 Secondary 서버에서 접속할 수 있도록 유저를 추가한다.
### 1.2 Primary 서버 MySQL conf 파일 작성
```
sudo vi /etc/my.cnf
[mysqld]
log-bin=mysql-bin
server-id=1
bind-address=0.0.0.0
```
```
sudo systemctl restart mysqld.service
```
### 1.3 Primary DB 상태 확인
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      157 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
Secondary DB에서 복제를 설정할 때 File과 Position에 대한 정보가 필요하다.
### 1.4 Primary DB 백업 및 전송
```
sudo mysqldump -u root -p db이름 > db이름.sql
scp -P 22 db이름.sql vagrant@secondaryDB_ip주소:/home/vagrant/
```
## 2. Secondary DB
### 2.1 Secondary DB에 백업할 db 생성
```
mysql> CREATE DATABASE db이름 DEFAULT CHARACTER SET utf8;
```
### 2.2 Secondary 서버 MySQL conf 설정
```
sudo vi /etc/my.cnf
[mysqld]
server-id=2
read_only=1
super_read_only=1
```
### 2.3 Primary 데이터 복원
```
sudo mysql -u root -p db이름.sql < db이름
```
### 2.4 복제 연결 설정
```
mysql> CHANGE REPLICATION SOURCE TO
    -> SOURCE_HOST='Primary서버_ip주소',
    -> SOURCE_USER='user 이름',
    -> SOURCE_PASSWORD='user 비밀번호',
    -> SOURCE_LOG_FILE='mysql-bin.000001',
    -> SOURCE_LOG_POS=157,
    -> SOURCE_PORT=3306,
    -> SOURCE_SSL=1;
mysql> START SLAVE;
```
1.3에서 확인한 File과 Position 값을 SOURCE_LOG_FILE와 SOURCE_LOG_POS에 입력해준다.
### 2.5 복제 시작 및 상태 확인
```
sudo systemctl restart mysqld.service
show slave status\G;
```
show slave status의 출력 결과 중 아래처럼 나오면 성공적으로 replication이 된 것이다. <br>
> Slave_IO_Running: Yes <br>
Slave_SQL_Running: Yes <br>


