# Vagrant + CentOS 6.6でnginxによるスマホ判別によるページ遷移制御

## 各モジュールなどのバージョン
* Vagrant 1.8.7
* CentOS 6.6
 *  <https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.0.0/centos-6.6-x86_64.box>
* nginx 1.10.3

## nginxの環境構築手順

1. `C:\vbox\nginx>vagrant up`

1. TeraTermなどを使用してアクセスし、nginxをインストールするためyumのリポジトリを追加する。<br>
※リポジトリを追加しない場合、かなり古いバージョンしか自動でインストールされない？<br>
`[root@localhost ~]# vi /etc/yum.repos.d/nginx.repo`
<pre>
<code>[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6/$basearch/
gpgcheck=0
enabled=1</code>
</pre>

1. 以下のコマンドでnginxをインストール<br>
`[root@localhost ~]# yum install --enablerepo=nginx nginx-1.9.4`

1. nginxを起動し、いったん動作確認する。<br>
`[root@localhost ~]# service nginx start`<br>
→<http://192.168.33.30/><br>
※Welcomeのページが表示されればnginxのインストールはOK

1. nginxの設定ファイルを編集する。<br>
/etc/nginx/conf.d/default.conf<br>
※/etc/nginx/nginx.confからincludeされている設定ファイル
<pre>
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

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
    set $dbg_v $sp_redirect_cookie;
    access_log /var/log/nginx/debug.log debug_log_fmt;

    # スマホ 且つ PC版のTopページへのアクセス 且つ Cookie「siteview」に「pc」と設定されていない場合のみ
    # スマホ専用のTopページへ強制リダイレクトする。
    if ($sp_redirect_cookie = "true_true") {
        rewrite ^ http://192.168.33.30/SP$request_uri? redirect;
        break;
    }

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
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
</pre>

1. /usr/share/nginx/htmlにサンプル用のHTMLを配置し、Chromeブラウザの「Toggle device toolbar」を使用して、nginxのTOPページやスマホのTOPページへ遷移し、deviceを切り替えながら動作確認する。<br>
※サンプルのHTMLはGitHubへアップしてあるやつが使える
