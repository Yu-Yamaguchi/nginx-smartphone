# CentOS 6.9へのnginx1.14環境構築とnginxによるスマホ判別のページ遷移制御の実現方法

## 事前準備

以下のサイトを参考に、CentOS6.9の環境が事前に構築済みであること
[Vagrant2.0.3を使ったCentOS6.9の環境構築](https://qiita.com/You_name_is_YU/items/5f6e326bad13911a2c51)

## やっていること

- PCの場合`/index.html`にデフォルトでつながり、スマホの場合`/SP/index.html`につながるように設定
- スマホ利用者でもPC版の画面を利用するケースがあるため、`PCサイトへ`を選択した場合には、スマホユーザであってもPC側のサイトへアクセスできるようにする。（その逆も）

スマホに対応したWebアプリケーションを構築する方法として、サーバサイドの動的アプリケーションで、フィルター機能を実装してディスパッチする方法や、レスポンシブ対応するなど様々な実現方法があると思います。
今回は、nginxに頑張ってもらって、ディスパッチし、サーバーサイドでは利用者がPCだろうがスマホだろうが意識しなくて済むような仕組みにしたいなぁと思ってこんなことやってみました。

## nginxの環境構築手順

### 面倒なのでrootにスイッチして以下の操作を実施

```shell
su -
```

### TeraTermなどを使用してアクセスし、nginxをインストールするためyumのリポジトリを追加する。  
※リポジトリを追加しない場合、かなり古いバージョンしか自動でインストールされない？

```shell
vim /etc/yum.repos.d/nginx.repo

[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6/$basearch/
gpgcheck=0
enabled=1
```

### インストールできるnginxのバージョンを確認しておく

```shell
yum list --enablerepo=nginx | grep nginx

nginx.x86_64                                1.14.0-1.el6.ngx            nginx
nginx-debug.x86_64                          1.8.0-1.el6.ngx             nginx
nginx-debuginfo.x86_64                      1.14.0-1.el6.ngx            nginx
nginx-module-geoip.x86_64                   1.14.0-1.el6.ngx            nginx
nginx-module-geoip-debuginfo.x86_64         1.14.0-1.el6.ngx            nginx
nginx-module-image-filter.x86_64            1.14.0-1.el6.ngx            nginx
nginx-module-image-filter-debuginfo.x86_64  1.14.0-1.el6.ngx            nginx
nginx-module-njs.x86_64                     1.14.0.0.2.0-1.el6.ngx      nginx
nginx-module-njs-debuginfo.x86_64           1.14.0.0.2.0-1.el6.ngx      nginx
nginx-module-perl.x86_64                    1.14.0-1.el6.ngx            nginx
nginx-module-perl-debuginfo.x86_64          1.14.0-1.el6.ngx            nginx
nginx-module-xslt.x86_64                    1.14.0-1.el6.ngx            nginx
nginx-module-xslt-debuginfo.x86_64          1.14.0-1.el6.ngx            nginx
nginx-nr-agent.noarch                       2.0.0-12.el6.ngx            nginx
pcp-pmda-nginx.x86_64                       3.10.9-9.el6                base
```

### 以下のコマンドでnginxをインストール

```shell
yum -y install --enablerepo=nginx nginx-1.14.0
```

### nginxを起動し、いったん動作確認する。

```shell
service nginx start
```

以下のURL（Vagrantfileで定義しているIPアドレス）にアクセスして、Welcomのページが表示されればnginxのインストールは正常に完了している。

http://192.168.33.10/  

## nginxの定義を修正してスマホ判別させる

### nginxの設定ファイルを編集する。

※/etc/nginx/nginx.confからincludeされている設定ファイル

```shell
vim /etc/nginx/conf.d/default.conf

server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    # スマホの判定とredirectの判定結果を保持する変数の定義
    set $sp_redirect_cookie "";

    # スマホ判定
    if ($http_user_agent ~* "(android|iphone|ipod|ipad|windows phone|meego|KFAPWI)") {
        set $sp_redirect_cookie true;
    }

    # スマホのredirect判定
    if ($request_uri ~* "(^/$|^/index.html$)") {
        set $sp_redirect_cookie "${sp_redirect_cookie}_true";
    }

    # Cookieに保持されたサイト表示
    if ($cookie_siteview = 'pc') {
        set $sp_redirect_cookie "${sp_redirect_cookie}_pc";
    }

    # 判定結果をデバッグログ出力
    # set $dbg_v $sp_redirect_cookie;
    # access_log /var/log/nginx/debug.log debug_log_fmt;

    # スマホ 且つ PC版のTopページへのアクセス 且つ Cookie「siteview」に「pc」と設定されていない場合のみ
    # スマホ専用のTopページへ強制リダイレクトする。
    if ($sp_redirect_cookie = "true_true") {
        rewrite ^ http://192.168.33.10/SP$request_uri? redirect;
        break;
    }

    # 公開ディレクトリとしてHTMLリソースを配置する場所の定義
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
}
```

### `/usr/share/nginx/html`にサンプル用のHTMLを配置しスマホ判別が正しく動作するか検証

Chromeブラウザの「Toggle device toolbar」を使用して、nginxのTOPページやスマホのTOPページへ遷移し、deviceを切り替えながら動作確認する。  
※サンプルのHTMLはGitHubへにアップしているのでそれを利用する。

https://github.com/Yu-Yamaguchi/nginx-smartphone

#### `/usr/share/nginx/html`にサンプルHTMLをアップする

![キャプチャ.PNG](https://qiita-image-store.s3.amazonaws.com/0/246556/8df8dc18-cdc8-135e-1251-1ea27698ae06.png)

#### http://192.168.33.10/ にPCブラウザでアクセスする

![キャプチャ.PNG](https://qiita-image-store.s3.amazonaws.com/0/246556/f4f599d8-e31d-ba3d-55ef-686695a8efaa.png)

#### iPhoneとして `http://192.168.33.10/` にアクセス

nginxで判断する際、Cookieの情報も確認しているため、シークレットモードでChromeを起動し、iPhoneのユーザエージェントで `http://192.168.33.10/` にアクセスして動作検証しました。

![キャプチャ.PNG](https://qiita-image-store.s3.amazonaws.com/0/246556/dd16b031-3c20-534c-8b5a-79eea5cd94d7.png)
