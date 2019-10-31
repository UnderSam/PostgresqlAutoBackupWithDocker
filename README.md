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

## 備份 script

## crontab 排程


