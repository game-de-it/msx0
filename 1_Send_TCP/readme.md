# TCPパケットでデータを送信する

### はじめに
1. RG353VにArkOSを動作させています
2. RG353VはWiFi接続とToolsからEnable Remote Serviceを実行してSSH接続を可能にしておきます
  - SSHアカウントは　ユーザ名：ark、パスワード：ark
3. aptが利用できるディストリビューションなら、ArkOS以外のOSでもこの手順は参考になるかと思います

---

### mariaDBのインストール
```
$ sudo apt update
$ sudo apt -y install mariadb-server mariadb-client
```
### mariaDBの初期セットアップ
- rootのパスワードは「msx0root」
```
$ sudo mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

### 念の為、管理者ユーザの作成
- ユーザ名は「admin」、パスワードは「msx0root」
```
$ sudo mariadb
GRANT ALL ON *.* TO 'admin'@'localhost' IDENTIFIED BY 'msx0root' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit
```

### DBの作成とユーザの作成＆権限
- DB名は「msx0DB」
- ユーザ名は「msx0user」、パスワードは「msx0user」
- msx0userにmsx0DBのテーブル操作の全権限を付与
- msx0userはlocalhostのみ接続可能

```
$ sudo mariadb
CREATE DATABASE msx0DB;
show databases;
CREATE USER 'msx0user'@'localhost' IDENTIFIED BY 'msx0user';
SELECT user, host, password FROM mysql.user;
grant all privileges on msx0DB.* to 'msx0user'@'localhost';
SHOW GRANTS FOR 'msx0user'@'localhost';
exit
```

### テーブルの作成
- テーブル名は「msx0table」
- 「id」カラムはオートインクリメント
- 「name」と「value」カラムはVARCHAR( 50 )
- 「update_time」カラムはデータを受信したときの時刻が自動挿入される
```
$ sudo mariadb
use msx0DB;
create table msx0table ( 
`id` INT NOT NULL AUTO_INCREMENT ,
  `name` VARCHAR( 50 ) NULL DEFAULT NULL ,
  `value` VARCHAR( 50 ) NULL DEFAULT NULL ,
  `update_time` TIMESTAMP ON UPDATE CURRENT_TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ,
  PRIMARY KEY ( `ID` ) 
);
exit
```

### DB内部からテストデータの書き込み
```
$ sudo mariadb
use msx0DB;
insert into msx0table (name,value) value ('MSX0-01','test1');
select *from msx0table;
exit
```

### シェルからDBへテストデータの書き込み
```
$ mysql -umsx0user -pmsx0user -e "use msx0DB; insert into msx0table  (name,value) value ('MSX0-01','test2');"
$ mysql -umsx0user -pmsx0user -e "use msx0DB; select * from msx0table;"
```

### テーブルの削除とテーブルの再作成
```
$ sudo mariadb
use msx0DB;
drop tables msx0table;
create table msx0table ( 
`id` INT NOT NULL AUTO_INCREMENT ,
  `name` VARCHAR( 50 ) NULL DEFAULT NULL ,
  `value` VARCHAR( 50 ) NULL DEFAULT NULL ,
  `update_time` TIMESTAMP ON UPDATE CURRENT_TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ,
  PRIMARY KEY ( `ID` ) 
);
exit
```

### DBにデータを保存するためのシェルスクリプト作成
```
$ vi ./msx0DB_insert.sh

#!/bin/sh
mysql -umsx0user -pmsx0user -e "use msx0DB; insert into msx0table  (name,value) value ('$1','$2');"

:wqで保存終了

$ chmod 755 ./msx0DB_insert.sh
```

### 名前付きパイプファイルを作成
```
$ sudo mkfifo /tmp/namepipe
$ sudo chmod 777 /tmp/namepipe
```

### netcatを利用してポート番号9999でTCPデータの待受
```
$ while read LINE; do echo $LINE | xargs ./msx0DB_insert.sh ; done</tmp/namepipe | sudo nc -lk 9999 > /tmp/namepipe
```

### MSX0にプログラムを書き込む
- conf/addrのIPアドレスはLinuxのIPアドレスを記入()
- 送信されるデータは「MSX0-01 ランダム値」
  - MSX0ではランダム値にならない？
- 80行目でTCPパケットが送信される
  - msx/me/if/NET0/msg　にデータが書き込まれたタイミングで、TCPパケットが送信されるっぽい
  - TCPパケットは細かく分割されて送信される
  - 「細かく」という曖昧な表現なのは、決まった文字数でデータが分割されていないので・・・
```
10 CLEAR 8000
20 NL$=CHR$(10)
30 PA$="msx/me/if/NET0/"
40 _IOTPUT(PA$+ "conf/addr","192.168.10.140")
50 _IOTPUT(PA$+ "conf/port",9999)
60 _IOTPUT(PA$+ "connect",1)
70 SM$="MSX0-01 "+STR$(RND(1))
80 _IOTPUT(PA$+"msg",SM$+NL$)
90 _IOTPUT(PA$+ "connect",0)
```


以上
