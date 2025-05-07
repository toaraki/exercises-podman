# Podman 演習

## はじめに
このコンテンツは、こちらの[オリジナル](https://developers.redhat.com/learn/rhel/rhel-image-mode-podman-command-line)から内容を抜粋して作成しました。初学者に向けて必要な内容に絞りつつ、操作内容もより手軽に実行できるようにカスタマイズしています。

## 期待する演習環境について
このコンテンツは以下のことを前提として期待しています。

* Linux / Mac / Windows いずれかのOS環境上で、Podman が実行可能であること。
  * 引用もとのdevelopers.redhat.com の"Intractive Tutolial" タイプで、Podman に関連するコースもそのまま実行環境として利用できます。
* Linux CLI の基礎知識があること
* quay.io にアカウントがあること

### Podman 環境がない場合
本来的な使用方法からは逸脱しますが、簡易的にPodmanの実行環境を得たい場合、以下のコンテンツが有効です。
[様々なRed Hat テクノロジーをハンズオンできるサイト（全編英語）：https://developers.redhat.com/learn](https://developers.redhat.com/learn)

画面中段の検索フィールドに、"using podman" とすると、"Deploy containers using Podman" というコンテンツを見つけられます。

![alt text](image.png)

このカタログをクリックすると、Podman の実行環境（50分で自動的に消失する）が起動します。

![alt text](image-1.png)

こちらのコンテンツ（画面の右側のセクション）には、基本的なpodman コマンドの実行について説明されて、そこで学び、Terminalセクション（画面の左側のセクション）でコマンドを実行できます。この演習と合わせてトライしてみてください。
左側のセクションは、このコンテンツ用に仮想マシンが払い出されているため、コンテンツには無い一般的な操作も行えます。ただし、50分で環境が自動的消失する点と、パブリックにある環境であることから、扱うデータ等には注意してください。また、あまりに不適切な利用と判断された場合、サービスが中止となる可能性もあるので、この目的から逸脱した乱用は避けてください。


## コンテナイメージのbuild / run / stop / start / rm

podmanコマンドがセットアップされた環境で、以下の操作を行います。

### ContainerFIle ( Dockerfile ) の作成
以下のコマンドを実行し、containerfileを作成します。
ここでは、cat コマンドをヒアドキュメントでリダイレクトする方法を示していますが、直接エディタでcontainerfile を新規作成する方法でも問題ありません。

```
cat <<EOF >>containerfile

## いわゆるLAMPスタックを構築するcontainerfile

## registry.access.redhat.com　から、ubi9 のイメージを取得する
FROM registry.access.redhat.com/ubi9/ubi

## RUN命令により、取得したベースイメージ上で、dnfコマンドを実行する。
## httpd , mariadb , mariadb-server , php-fpm , php-mysqlnd をインストール
RUN dnf module enable -y php:8.2 nginx:1.22 && dnf install -y httpd mariadb mariadb-server php-fpm php-mysqlnd && dnf clean all
        
## コンテナ起動時に、httpd , mariadb , php-fpm が自動起動するように、systemd の起動定義を"enable"に設定
RUN systemctl enable httpd mariadb php-fpm
        
## htmlコンテンツの作成と配置
RUN echo '<h1 style="text-align:center;">Welcome to RHEL image mode</h1> <?php phpinfo(); ?>' > /var/www/html/index.html

EOF
```


### コンテナのビルド (build)

podman build コマンドを実行しコンテナをビルドします。前のステップで作成したcontainerファイルを"-f"オプションで指定することで、containerfile に指定した、lampスタックの構築処理を実行することができます。
また、作成されたコンテナイメージにタグ情報を付与するため、"-t"オプションを使用します(※)。コマンド実行時は、"quay.io/[my_account]/lamp-ubi9:latest"のmy_accountを、**quay.io のユーザアカウント名 に変更したうえで** 実行してください。

※ 最終的に、quay.io (パブリックスペースのコンテナイメージレジストリサービス)に、コンテナイメージをアップロードすることを想定し、"quay.io/[my_account]/lamp-ubi9:latest" を付与しています。

```
$ podman build -f containerfile -t quay.io/[my_account]/lamp-ubi9:latest
```

### コンテナの実行 (run)

作成したコンテナを起動します。コンテナの起動は以下のコマンドで行います。コマンド実行時は、"quay.io/[my_account]/lamp-ubi9:latest"のmy_accountを、**前のステップで指定した、quay.io のユーザアカウント名 に変更したうえで** 実行してください。

```
$ podman run -d --name lamp -p 8080:80 quay.io/[my_account]/lamp-ubi9:latest /sbin/init
```

#### 起動時に指定したオプションの説明
* -d : コンテナをデタッチモードで実行（バックグラウンドに起動し、新しいコンテナIDを付与）
* --name : 起動後のコンテナを識別するエリアス名を指定
* -p : ポートのマッピング (host port : container port) 今回は、podman を実行するホストの8080ポートに、コンテナの80ポートをバインドしています。
* 起動するコンテナイメージ（前ステップのビルド時に指定したタグを指定）
* コンテナ起動後に実行するコマンド

#### 起動した状態の確認

podmanを実行しているマシンのwebブラウザまたは、curlコマンドでlocalhost:8080 へアクセスします。以下は、起動した場合のcurl コマンドの実行例です。

![alt text](image-2.png)

続いて、podman ps コマンドで実行状態にあるコンテナの一覧を取得します。

```
$ podman ps
```
実行中（この画面は、ビルド時にタグ情報として、"quay.io/araki-tshr/lamp-ubi9:latest"を指定）
![alt text](image-3.png)


### コンテナの停止と再開

演習シナリオの中では、このタイミングでコンテナを停止する意味はありませんが、手段としてコンテナを停止・起動する方法を確認します。

コンテナの停止には、podman stop コマンド
```
$ podman stop コンテナ名(または、CONTAINER ID)
```

停止状態のコンテナを再度起動する場合は、podman start コマンドを使用します。
```
$ podman start コンテナ名(または、CONTAINER ID)
```

実行例
停止までの例
![alt text](image-4.png)

停止状態から起動までの例
![alt text](image-5.png)


### コンテナの削除
アプリケーションの変更等の理由で不要となったコンテナは、削除します。コンテナの削除には"podman rm "コマンドを使用します。尚、コンテナを削除する場合は、予めコンテナを停止しておく必要があります。

コンテナの削除
```
$ podman rm コンテナ名(または、CONTAINER ID)
```

実行例

![alt text](image-9.png)
### コンテナイメージのアップロード

ビルドしたコンテナが期待通り稼働することが確認できた（ここでは、という仮定をおいてます。）ので、イメージレジストリへアップロードします。

イメージのアップロード先をここでは、quay.io としています。quay.io には、Red Hat ネットワークのIDでアクセスすることが可能です。（無料の１つのプライベートリポジトリプラン）

```
$ podman login quay.io
$ podman push quay.io/[my_account]/lamp-ubi9:latest
```

コンテナのアップロードが完了すると、quay.io のリポジトリにアップロードしたイメージを確認するこができます。
![alt text](image-6.png)

### ローカル環境からのイメージの削除

podman build でビルドされたイメージは、podman を実行するホストのローカルストレージに保持されています。そのイメージが不要となったり、あるいはディスク容量の確保したい場合等は、イメージを削除します。尚、イメージを削除する場合は、該当のイメージから起動されているコンテナを停止しておく必要があります。

ホストのローカルに保持されているイメージの一覧を表示するには、podman image list コマンドを使用します。
```
$ podman image list
```

podman image rm コマンドに、削除対象のIMAGE ID を指定してイメージを削除します。
```
$ podman image rm [IMAGE ID]
```

実行例
![alt text](image-10.png)


以上で、contianerファイルを用いた、コンテナのbuild / run / stop / start / rm の演習を終了します。