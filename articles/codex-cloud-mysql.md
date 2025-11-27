---
title: "Codex Cloud環境にMySQL導入→APIの結合テストが実行可能に"
emoji: "🚅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Codex", "AI駆動開発", "AIエージェント", "AI", "開発生産性"]
published: true
publication_name: "digeon"
---
私は株式会社Digeonでインターンをしている大学生です。

この記事は、Codex Cloud環境にMySQLサーバを立ち上げた結果、APIの結合テストを実行してくれるようになり、生産性が上がったという話です。

MySQL導入前は、投げたタスクでAPIの結合テストが回らず、ローカルで手直しが必要でした。
しかし、導入後は、レビュー段階のプルリクエストまで自動で完結するようになりました。

↓オールグリーン！

![APIのCI/CDがすべて通っている](https://storage.googleapis.com/zenn-user-upload/52ca46952d41-20251126.png)

# Codex Cloudとは

Codex Cloudは、OpenAIのAIコーディングエージェントCodexをWeb上で使用できるサービスです。
タスクを投げると、クラウド上のサンドボックス環境でAIエージェントが開発をしてくれます。
GitHubと連携しており、プルリクエストを作るところまでやってくれます。
諸々の作業がGUIで完結したり、並行でタスクを投げられたりするのも便利です。

![Codex Cloudのトップ画面](https://storage.googleapis.com/zenn-user-upload/245c8fc2a3ca-20251126.png)

Codex Cloudではタスク実行用の環境をセットアップスクリプトでカスタマイズできます。
セットアップスクリプト実行後の状態をキャッシュする機能もあり、実行時間も気になりません。
ただし、サンドボックス環境には制約があり、ローカルと同じ構成を持ち込めない場合があります。

# 課題：APIの結合テストが実行できない

私は現在とあるサービスのWebアプリを開発しています。

開発環境としてDockerでGoのAPIサーバやMySQLサーバを立ち上げています。
APIの結合テストがあり、MySQLにテスト用のデータベースを立てて接続しています。

しかし、Codex Cloudの環境ではDocker in Dockerに非対応でコンテナを立ち上げられません。

https://community.openai.com/t/codex-docker-in-docker-in-environment-setup

Codex CloudにはデフォルトでGoのランタイムがありますが、MySQLは入っていません。
そのため、テスト用のデータベースが立ち上げられず、APIの結合テストを実行することができません。

↓タスク実行結果。`go test ./...`がデータベースに接続できずに失敗しています。

![Codex CloudがAPIの結合テストを実行できていない](https://storage.googleapis.com/zenn-user-upload/358981246fbf-20251127.png)

テストの修正・追加の必要があるタスクを投げた場合、テストが実行できないため、テストが失敗する状態でプルリクエストが作られます。
適宜、ローカルの開発環境でテストを実行して、人の手で修正する必要がありました。

↓このように、APIのCI/CDではテストの実行ワークフローだけ落ちることがしばしばでした。

![APIのCI/CDでテストだけが落ちている](https://storage.googleapis.com/zenn-user-upload/3a8ad3c53eeb-20251126.png)

# MySQLを導入
そこで、Codex Cloud環境にMySQLを導入すると、APIの結合テストが実行できるようになりました。
タスクを投げると、そのままレビューできる段階のプルリクエストが完成します。
雑多なバグ修正や煩雑な仕様変更のタスクはとりあえず投げておくという使い方ができます。

↓導入後のタスク実行結果。`make test-light`が成功しています。（※APIの結合テストには外部のオブジェクトストレージサーバに接続するテストもあるため、そのテストを除外したコマンドが`make test-light`です。）

![Codex CloudがAPIの結合テストを実行できている](https://storage.googleapis.com/zenn-user-upload/17ec3ad1361a-20251127.png)

↓APIのCI/CDがすべて通っています。気持ちいいですね。

![APIのCI/CDがすべて通っている](https://storage.googleapis.com/zenn-user-upload/52ca46952d41-20251126.png)

# 技術的な話

環境のセットアップスクリプトに次のスクリプトを追加しました。

MySQLのインストール処理は初回だけ時間がかかりますが、キャッシュのおかげで2回目以降は高速です。

## セットアップスクリプト

```sh
sudo apt-get update
# mysql-server ではなく自前でインストール
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server-core-8.0 mysql-client

sudo useradd -r -s /bin/false mysql || true
sudo mkdir -p /var/lib/mysql /var/lib/mysql-files
sudo chown -R mysql:mysql /var/lib/mysql /var/lib/mysql-files

sudo chown -R mysql:mysql /var/lib/mysql
sudo mkdir -p /var/run/mysqld
sudo chown mysql:mysql /var/run/mysqld

# --initialize-insecure は本番環境では非推奨
sudo -u mysql mysqld --initialize-insecure --datadir=/var/lib/mysql
# --secure-file-priv="" は本番環境では非推奨
sudo -u mysql mysqld --daemonize --datadir=/var/lib/mysql --secure-file-priv=""

for i in {1..30}; do
  if mysqladmin ping -uroot --silent; then
    echo "MySQL is ready!"
    break
  fi
  sleep 1
done

sudo mysql -u root <<EOF
CREATE USER IF NOT EXISTS '${DB_USER}'@'%' IDENTIFIED BY '${DB_PASSWORD}';
CREATE DATABASE IF NOT EXISTS ${DB_NAME};
GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'%';
FLUSH PRIVILEGES;
EOF
```

## 環境変数

- `DB_USER`
- `DB_PASSWORD`
- `DB_HOST`
- `DB_PORT`
- `DB_NAME`

`DB_HOST`は`127.0.0.1`にする必要があります。他の環境変数はローカルと同じで大丈夫です。

## スクリプトの解説

Codex Cloudのセットアップスクリプトは、単一プロセス、非対話コンテナという制約のもとで実行されます。
そのため、単に`apt-get install mysql-server`とすると、単一プロセス上でサービスが起動してしまい、他のセットアップができません。
また、`mysql-server`の`postinst`は`systemd`の無いCodex Cloud環境では失敗します。
そこで、自前で`mysql-server-core`と`mysql-client`をそれぞれ導入して初期化しています。

# おわりに

単にMySQLをインストールするだけではなかったため、少し大変でした。
それでも、その効果は絶大で、いつもCodex Cloudにはお世話になっています。

MySQL以外にも導入すると便利なものがあると思います。
皆さんもぜひ、Codex Cloudの環境を整備してみてください。

Codex Cloudを使ったことがない人は使ってみてはいかがでしょうか？

# 参考記事

https://community.openai.com/t/codex-docker-in-docker-in-environment-setup

https://zenn.dev/sunagaku/articles/codex-cloud-5-tips
