# リバースプロキシを利用して通信の暗号化、情報の秘匿性を高める

### はじめに
   <img src="https://github.com/game-de-it/msx0/blob/main/asset/SC03.png" width="800">  
   
※構成図の画像をクリックして拡大  

  
1. RG353VにArkOSを動作させています
2. RG353VはWiFi接続とToolsからEnable Remote Serviceを実行してSSH接続を可能にしておきます
  - ArkOSのSSHアカウントは　ユーザ名「ark」、パスワード「ark」
3. aptが利用できるディストリビューションなら、ArkOS以外のLinuxでもこの手順は参考になるかと思います

---

### nginxとluaとcjsonのインストール
- nginxは多機能なWEBサーバソフトウェアです
- extrasパッケージにluaモジュールが内包されています
  - luaはスクリプト言語です
- cjsonはjsonパラメータの操作・加工に利用します
```
$ sudo apt install -y nginx nginx-extras lua-cjson
```
### nginxの設定ファイルを削除
- 必要であればsites-enabledディレクトリ以外にdefaultファイルを移動させてもOK
```
$ sudo rm /etc/nginx/sites-enabled/default
```

### nginx.confの作成
- 既存のnginx.confの中身を全て削除して下記の設定をコピペしてください
  - 注意！！　この設定はローカルネットワーク内で動作検証する目的の内容なので、グローバルに足が出ているサーバでは利用しないでください
- 「XXXXX」にはnginxが動作しているLinuxのIPアドレスを入力してください
- 「YYYYY」にはAmbientのwriteKeyを入力してください
- 「ZZZZZ」にはAmbientのチャネルIDを入力してください
```
$ sudo vi /etc/nginx/nginx.conf

user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
events {
	worker_connections 768;
}
http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
	gzip on;
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
	server {
		listen       9999;
		server_name  XXXXX;
		access_log on;
		ignore_invalid_headers off;
		location / {
			limit_except POST {
			deny all;
			}
			access_by_lua_block {
				ngx.req.read_body()
				local body = ngx.req.get_body_data()
				local cjson = require "cjson.safe"
				local json = cjson.decode(body)
				local data = {writeKey='YYYYY',d1=json['d1'],d2=json['d2'],d3=json['d3']}
				local output = cjson.encode(data)
				ngx.req.set_body_data(output)
			}
			proxy_set_header Host  ambidata.io;
			proxy_pass https://ambidata.io/api/v2/channels/ZZZZZ/data;
		}
	}
}
```

### nginxの再起動
```
$ sudo systemctl restart nginx
```

### nginxの動作状態の確認
- Active: active (running)　になっていることを確認してください
  - 問題がある場合には/var/log/nginx/error.logファイルの中身を確認してください
```
$ sudo systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2023-10-18 10:45:36 JST; 6s ago
     Docs: man:nginx(8)
  Process: 23339 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 23340 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 23342 (nginx)
   CGroup: /system.slice/nginx.service
           ├─23342 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           ├─23343 nginx: worker process
           ├─23344 nginx: worker process
           ├─23345 nginx: worker process
           └─23346 nginx: worker process
```

### MSX0にプログラムを流し込む
- サンプルプログラムの行数を大きく変えたくなかったので、動作に不必要な行が残っています
- 変更箇所
  - 170行目のwait timeを10秒に変更してありますので、問題があればデフォルトの30秒へ変更してください
  - 190行目の処理はチャネルIDを必要としないのでコメントして無効化させてます
  - 210行目のaddrはnginxが動作しているLinuxのIPアドレスを入力してください
  - 330行目のwriteKeyをdummyという文字列に変更しています(そもそもこの要素自体、動作に必要ありません)
  - 380行目のPOSTメソッドのパスを変更しています
  - 390行目のHost:ヘッダーはnginxが動作しているLinuxのIPアドレスを入力してください
  - 290~310行目ではダミーの値を変数に代入していますが、IOTGETでセンサーの値を代入してもOKです
```
100 'SAVE"SEND2NET.BAS"
110   CLEAR 800
120   DQ$=CHR$(&H22):NL$=CHR$(13)+CHR$(10)
130 'user setting
140   CH$="xxxxx"            'channel ID
150   WK$="xxxxxxxxxxxxxxxx" 'write key
160   PA$="msx/me/if/NET0/"  'path
170   WT=10                  'wait time
180   RB=0                   'reboot or wait
190 '  IF CH$="xxxxx" THEN PRINT "please change the value CH$ and WK$ to your enviroment.":END
200 'connet
210   _IOTPUT(PA$+ "conf/addr","192.168.10.140")
220   _IOTPUT(PA$+ "conf/port",9999)
230   _IOTPUT(PA$+ "connect",1)
240 'check connect status
250   FOR I=0 TO 100:NEXT
260   _IOTGET(PA$+ "connect",S)
270   IF S<>1 THEN PRINT "connect fail":GOTO 580
280 'get sensor value
290   D1=1
300   D2=2
310   D3=3
320 'create message
330   CN$="{"+DQ$+"dummy"+DQ$+":"+DQ$+WK$+DQ$+","
340   CN$=CN$+DQ$+"d1"+DQ$+":"+DQ$+STR$(D1)+DQ$+","
350   CN$=CN$+DQ$+"d2"+DQ$+":"+DQ$+STR$(D2)+DQ$+","
360   CN$=CN$+DQ$+"d3"+DQ$+":"+DQ$+STR$(D3)+DQ$
370   CN$=CN$+"}"+NL$
380   SM$(0)="POST / HTTP/1.1"+NL$
390   SM$(1)="Host: 192.168.10.140"+NL$
400   SM$(2)="Content-Length:"+STR$(LEN(CN$))+NL$
410   SM$(3)="Content-Type: application/json"+NL$
420   SM$(4)=""+NL$
430   SM$(5)=CN$
440   SM$(6)=""
450 'send message
460   PRINT NL$+"---- Send Message ----"
470   I=0
480   IF SM$(I)<>"" THEN PRINT SM$(I);:_IOTPUT(PA$+ "msg",SM$(I)):I=I+1:GOTO 480
490   FOR I=0 TO 1000:NEXT
500 'receive message
510   PRINT NL$+"---- Receive Message ----"
520   FOR J=0 TO 10
530     _IOTGET(PA$+ "msg",RM$)
540     PRINTRM$;
550     FOR I=0 TO 100:NEXT
560   NEXT
570 'disconnet
580   _IOTPUT(PA$+ "connect",0)
590 'loop
600   IF RB=1 THEN 660 ELSE 610
610 'wait
620   PRINT NL$+"---- Wait (" + STR$(WT) + " sec ) ----"
630   TIME=0
640   IF TIME<WT*60 THEN GOTO 640
650   GOTO 200
660 'system reboot
670   PRINT NL$+"---- Sleep (" + STR$(WT) + " sec ) & Reboot ----"
680   _IOTPUT("host/power/wait", WT)
690   _IOTPUT("host/power/reboot",1)
```

### 動作確認
   <img src="https://github.com/game-de-it/msx0/blob/main/asset/SC04.png" width="800">  

1. MSX0でrunする
2. Ambient側でリストやグラフを作成して、MSX0から送られてきた値が入力されていることを確認します

以上
