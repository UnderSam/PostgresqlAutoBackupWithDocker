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

