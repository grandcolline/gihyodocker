# 4. Swamによる実践的なアプリケーション構築

## 4.1 Webアプリケーションの構成

TODOアプリ作る環境構築する。

```bash
$ docker-compose up -d

$ docker exec -it manager docker swarm init
$ docker exec -it worker01 docker swarm join --token <<Token>>
$ docker exec -it worker02 docker swarm join --token <<Token>>
$ docker exec -it worker03 docker swarm join --token <<Token>>

$ docker exec -it manager docker node ls
$ docker exec -it manager docker network create --driver=overlay --attachable todoapp
```

## 4.2 MySQL Serviceの構築

```bash
$ git clone https://github.com/gihyodocker/tododb
$ cd tododb
$ tree .
```

ビルド＆プッシュ
```bash
$ docker build -t ch04/tododb:latest .
$ docker tag ch04/tododb:latest localhost:5000/ch04/tododb:latest
$ docker push localhost:5000/ch04/tododb:latest
```

デプロイ
```bash
$ docker exec -it manager docker stack deploy -c /stack/todo-mysql.yml todo_mysql
```

初期データ投入
```bash
$ docker container exec -it manager \
	docker service ps todo_mysql_master \
	--no-trunc \
	--filter "desired-state=running" \
	--format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"

#上で吐き出されるコマンドを実行し、コンテナ内に入る

$ init-data.sh
$ mysql -u gihyo -pgihyo tododb

mysql> SELECT * FROM todo \G
```

Slave確認
```bash
$ docker container exec -it manager \
	docker service ps todo_mysql_slave \
	--no-trunc \
	--filter "desired-state=running" \
	--format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"

#上で吐き出されるコマンドを実行し、コンテナ内に入る

$ mysql -u gihyo -pgihyo tododb

mysql> SELECT * FROM todo \G
```

## 4.3 API Serverの構築

ソース取得
```bash
$ git clone https://github.com/gihyodocker/todoapi
$ cd todoapi
$ tree -a -I '.git|.gitignore' .
```

ビルド
```
$ docker build -t ch04/todoapi:latest .
$ docker tag ch04/todoapi:latest localhost:5000/ch04/todoapi:latest
$ docker image push localhost:5000/ch04/todoapi:latest
```

デプロイ  
この段階では、`todo-app.yml`の中のnginxは使わないので一旦コメントアウトしておくこと！
```bash
$ docker exec -it manager docker stack deploy -c /stack/todo-app.yml todo_app
$ docker exec -it manager docker service logs -f todo_app_api
```

## 4.4 Nginxの構築

ソース取得
```
$ git clone https://github.com/gihyodocker/todonginx
$ cd todonginx
$ tree .
```

ビルド
```
$ docker build -t ch04/nginx:latest .
$ docker tag ch04/nginx:latest localhost:5000/ch04/nginx:latest
$ docker image push localhost:5000/ch04/nginx:latest
```

デプロイ
```bash
$ docker exec -it manager docker stack deploy -c /stack/todo-app.yml todo_app
```

## 4.5 Webの構築

ソース取得
```bash
$ git clone https://github.com/gihyodocker/todoweb
$ cd todoweb
$ tree .
```

ビルド＆プッシュ
```bash
$ docker build -t ch04/todoweb:latest .
$ docker tag ch04/todoweb:latest localhost:5000/ch04/todoweb:latest
$ docker push localhost:5000/ch04/todoweb:latest
```
※ package-lock.jsonがあると上手くいかないかも？

中身確認
```
$ docker run --rm -it ch04/todoweb:latest ls .nuxt/dist
```

nginx書き換え
```bash
$ cd todonginx
$ cp etc/nginx/conf.d/public.conf.tmpl etc/nginx/conf.d/nuxt.conf.tmpl
$ cp Dockerfile Dockerfile-nuxt
```
etc/nginx/conf.d/nuxt.conf.tmp
```diff
+    location /_nuxt/ {
+        alias /var/www/_nuxt/$1;
+        {{ if var "LOG_STDOUT" }}
+        access_log /dev/stdout json;
+        error_log /dev/stderr;
+        {{ else }}
+        access_log  /var/log/nginx/backend_access.log json;
+        error_log   /var/log/nginx/backend_error.log;
+        {{ end }}
+    }
```
Dockerfile-nuxt
```diff
       "/etc/nginx/conf.d/upstream.conf", \
       "--", \
   "render", \
-      "/etc/nginx/conf.d/public.conf", \
+      "/etc/nginx/conf.d/nuxt.conf", \
       "--" \
 ]
```

nginxのビルド＆プッシュ
```
$ docker build -f Dockerfile-nuxt -t ch04/nginx-nuxt:latest .
$ docker tag ch04/nginx-nuxt:latest localhost:5000/ch04/nginx-nuxt:latest
$ docker push localhost:5000/ch04/nginx-nuxt:latest
```

デプロイ
```
$ docker exec -it manager docker stack deploy -c /stack/todo-frontend.yml todo_frontend
```

Ingressで公開する
```
$ docker exec -it manager docker stack deploy -c /stack/todo-ingress.yml todo_ingress
```


## その他、使ったコマンド

```
# Stackの確認
$ docker exec -it manager docker stack ls

# Serviceの確認
$ docker exec -it manager docker service ls

# 立ち上がらないときログを見る
$ docker exec -it manager docker service logs -f <<Service>>
```

## 一緒に読んだページなど

- [dockerのENTRYPOINTとCMDの書き方と使い分け、さらに併用 - Qiita](https://qiita.com/hnakamur/items/afddaa3dbe48ad2b8b5c)
- [Entrykit のすすめ - Qiita](https://qiita.com/spesnova/items/bae6406bf69d2dc6f88b)
- [The Twelve-Factor App（日本語訳） - 設定](https://12factor.net/ja/config)


