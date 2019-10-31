# PostgressAutoBackupWithDocker

今天要講的是利用 Docker 建立2個 Postgres Container(A and B), 並且能夠每天定時去備份 A 的資料到 B 的目錄中, 在 Docker 建立兩個一樣的 Container
可以想像是在分開的兩個系統(Server,Client)中進行備份動作

## 安裝 postgres

```
docker pull postgres
```

## 啟動兩個 postgres + 測試 Bridge 網路

```
docker run --name mypostgres1 -e POSTGRES_PASSWORD=1qaz2wsx -p 5432:5432 -d postgres
docker run --name mypostgres2 -e POSTGRES_PASSWORD=1qaz2wsx -p 5432:5432 -d postgres
```
這時候兩個 Container 的 Network 都會是 Bridge 的狀態, 並且能夠互相 Ping 到對方, 但是一開始建起來的 Postgres Container 不會有 Ping 這個指令, 因此
可以透過指令來安裝
```
apt-get update
apt-get install -y iputils-ping
```
現在安裝好了 Ping 指令, 需要先知道對方的 IP 是什麼, 這時候就可以先離開 Container, 在外面執行
```
docker container inspect mypostgres1
docker container inspect mypostgres2
```
並且找到最底下 Network 的 IPAdress, 記錄起來之後就可以回到 A(這邊設定A = postgres1, B = postgres2)去 Ping 看看 B, 你應該就會看到封包傳送成功的訊息

## 安裝 sshd 來讓兩邊能夠透過 ssh 來連線

因為要能自動備份並且傳送需要用到 ssh 跟 scp 的指令, 因次這邊需要安裝 sshd-server 的套件
```
apt-get install sshd-server
mkdir /var/run/sshd
echo 'root:screencast' |chpasswd
/usr/sbin/sshd  <--直接啟動
```
兩邊都安裝完成之後就可以來測試連線了, 但是在測試之前需要先改一下密碼, 不然之後會無法切回來 root, 系統這邊預設的 postgres 也可以順便改密碼
```
passwd root
passwd postgres
```
設定完之後就可以連線看看, 因為 root 預設是無法遠端的, 因此直接用 postgres 登入
```
ssh postgres@IPAdress
```
在輸入正確的密碼後就可以進入到系統之中, 也代表初步的連線是 OK 的, 不過因為需要用到自動備份加上傳送, 因此還需要設定 ssh key 來做到自動登入,
這邊因為不會使用 root 來當作操作 postgres, 因此另外建立一個 admin 在 A 系統中
```
adduser admin
```
在建立過程中, 基本上就一直 Enter 即可, 建立完後就可以開始設定自動登入的部分, 首先在 A 系統我們要先產生 ssh key
```
ssh-keygen -t rsa
```
建立完之後就可以把 rsa_pub 傳送到 B 的目錄中
```
scp /home/admin/.ssh/id_rsa.pub postgres@IPAdress:~
```
傳送完後就可以連線到 B 的系統來處理 key 的部分
```
ssh postgres@IPAdress
mkdir .ssh
chmod 700 .ssh
cat ./id_rsa.pub >> .ssh/authorized_keys
rm ./id_rsa.pub
exit
```
做完之後就可以直接在 A 免密碼自動登入囉 !

## 建立 postgres user, table, data

首先需要進行 psql 來把 admin 加入到使用者之中, 回到 A 系統中並且
```
su - postgres <-- 需要使用者 postgres 才能直使用 psql
createuser --interactive --pwprompt <-- 照著步驟完成創建 admin 角色
createdb -O admin testDB <-- 為 admin 建立 testDB
```
建立完之後就可以離開 postgres 改用 admin 登入 postgres 看看
```
su admin
psql -U admin testDB
```
若剛剛的步驟都正確現在就會進入到 postgres 的 shell 之中, 現在可以照著網路上的 tutorial 建立幾個 table 和 插入一些資料, 都做完之後就可以來處理備份的問題了

## 備份 script

備份的部分可以直接參考官網 : (PostgreSQL Documentation::Backup and Restore)[https://www.postgresql.org/docs/8.4/backup.html]
文件中列出了三種備份方法, 我們要考量我們的備份需求為何, 決定備份方案。本文需求在不停止 PostgreSQL 的狀態下進行備份。另一方面, 本文的資料庫內容異動少, 且不做資料庫的查詢分流動作。所以選擇採用 SQL Dump 的方式, 這種方式適用不需要即時性的同步, 而是每日定期備份的操作。

### 從 A 備份到 B
```
admin@IPAdress$ pg_dump -c services | gzip | \
     ssh postgres@IPAdress "gunzip | psql services"

# or

admin@IPAdress$ pg_dump -F tar services | gzip | \
     ssh postgres@IPAdress "gunzip | pg_restore -c -d services"
```
若只是要傳送壓縮則可以
```
admin@IPAdress$ pg_dump -F tar services | gzip | \
     ssh postgres@IPAdress "cat > ~/database_services.tar.gz"

# or simple way:
 admin@IPAdress$ pg_dump -F tar services | gzip ~/database_services.tar.gz
 admin@IPAdress$ scp ~/database_services.tar.gz postgres@IPAdress:~
```
### 從 B 還原備份
```
admin@IPAdress$ ssh postgres@IPAdress "pg_dump -F tar services | gzip " \
     | gunzip | pg_restore -c -d services

# or

admin@IPAdress$ ssh postgres@IPAdress "pg_dump -c services | gzip " \
     | gunzip | psql services
```
## crontab 排程

這邊的部分可以先參考 (鳥哥的 Crontab 教學)[http://linux.vbird.org/linux_basic/0430cron.php]
