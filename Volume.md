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
## share volume in two containers :

在 docker-compose.yml 中的 volumes 設定一樣即可, 但是 volume 的機制為 :
一次性讀取, 且不會在 running 中複寫外面的 volume, 需要 stop 才會覆寫, 在兩個共用一個 volume 的時候,
其中一方 stop 會把另一邊的 container 也 stop 並且更新最先 stop 的 data 到 volume, 所以 volume 就會
變成最早 stop 的 container.

流程這樣就會變為 :
開一個 pg1, 指定外在 volume, 每次 stop 都會更新, 當 pg1 shutdown, pg2 就可以馬上啟動接收 pg1 的 volume
