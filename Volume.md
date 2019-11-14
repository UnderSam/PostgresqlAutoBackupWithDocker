# PostgresqlAutoBackupWithDocker

Docker volume intro :
http://dockone.io/article/128
volumes 可以用作 data-only container, 並且也可以把 data 獨立出 container 來達成 persistant data.

Volumes use in postgres :
https://julianchu.net/2016/04/19-docker.html


## docker-compose to use volume : 
```
version: '3.1'

services:
 db:
  image: postgres
  restart: always
  environment:
   POSTGRES_PASSWORD: 1qaz2wsx
  volumes:
    - ./postgres-data:/var/lib/postgresql/data
  ports:
    - 5432:5432
```
after create yml file, just compose up with
```
docker-compose up -d
```
## use same volume without data container in two containers :

在 docker-compose.yml 中的 volumes 設定一樣即可, 但是 volume 的機制為 :
一次性讀取, 且不會在 running 中複寫外面的 volume, 需要 stop 才會覆寫, 在兩個共用一個 volume 的時候,
其中一方 stop 會把另一邊的 container 也 stop 並且更新最先 stop 的 data 到 volume, 所以 volume 就會
變成最早 stop 的 container.

流程這樣就會變為 :
開一個 pg1, 指定外在 volume, 每次 stop 都會更新, 當 pg1 shutdown, pg2 就可以馬上啟動接收 pg1 的 volume

## share data with data container

首先建立一個 volume 
```
docker volume create dbdata
```
然後在開 run 一個外掛 volume 的 data container
```
docker run -v dbdata:/var/lib/postgresql/data --name=dbdata postgres echo "Data-only container"
```
要使用這個 data 只需要在創建新的 db 指定 volumes-from 就好
```
docker run --rm -d --volumes-from dbdata --name db1 -e POSTGRES_PASSWORD=mypassword postgres
```
這樣就會得到一個沒有單純提供 data 持久化的 container, 注意 :
1. 不要 run data container，這樣會浪費資源。
2. 不要為了 data container 去跑最好的 image, 用 db 本身 image 即可


