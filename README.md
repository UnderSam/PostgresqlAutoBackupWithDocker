# PostgresqlAutoBackupWithDocker

這個專案是在研究
1. 如何在兩個 postgres 容器中達到互相備份並且利用 auto-login 達成自動備份儲存機制
2. 如何將 postgres 容器中的 data 達到 persistant 並且能夠利用 volume 跟其他 container 共享 data

每個 topic 會分開來 document
