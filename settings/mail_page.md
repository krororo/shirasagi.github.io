---
layout: default
title: メール取込
---

## 概要

メール取込はサーバーで受信したメールを SHIRASAGIのページ として公開する機能です。<br />
SHIRASAGI側のメール取込インターフェースとして、標準入力からメール（EML形式の文字列）を読み込んで、ページとして保存するタスクを備えています。<br />
取り込み時に緊急災害レイアウトの切り替えも可能です。

## メール取込フォルダー

受信したメールをページとして保存し、格納するフォルダーです。<br />
取り込まれたページは、このフォルダー配下に保存されます。<br />
メールの取り込みを行う場合はあらかじめ管理画面から作成して、以下の項目を設定する必要があります。<br />

### 基本情報
基本情報は通常のフォルダー設定と変わりません。

- ページレイアウト
  - 取り込んだページのレイアウトとして設定されます。

### メール取込設定
- 送信者メールアドレス
  - 受信したメールの From 条件を設定します。
  - メールアドレスもしくはホスト名を改行区切りで入力します。
  - 入力した値とメールの From ヘッダが一致した時のみ、取り込んだメールをページとして保存します。
- 宛先メールアドレス
  - 受信したメールの To 条件を設定します。
  - メールアドレスもしくはホスト名を改行区切りで入力します。
  - 入力した値とメールの To ヘッダが一致した時のみ、取り込んだメールをページとして保存します。
- 緊急情報表示期間
  - メール取込/緊急情報 パーツ（後述）に表示する期間を設定します。

### 緊急災害レイアウト設定
取り込み時、緊急災害レイアウトの切り替えを行う場合に設定します。

- 切り替え
  - 有効/無効を設定します。
- フォルダー
  - 切り替えの対象となる 緊急災害レイアウトフォルダー を設定します。

## メール取込ページ

取り込んだメールは「メール取込ページ」という種類のページとして保存されます。<br />
メールの本文がページの本文に設定され、公開状態で保存されます。<br />
緊急情報表示用の設定があります。

### 緊急情報表示

- 緊急情報表示開始日時
  - 取り込んだ日時が保存されます。
- 緊急情報表示終了日時
  - 取り込んだ日時から、フォルダーに設定した表示期間分を足した日時を保存します。
  - 例えばフォルダーの緊急情報表示期間を2日に設定した場合、取り込んだ日時から2日後が保存されます。

## 緊急情報パーツ

メール取込ページ一覧をリスト表示することができるパーツです。<br />
緊急情報表示開始〜終了内にあるページ一覧を表示します。<br />
トップページなどに配置して、一覧を表示する運用を想定していますが、パーツの動的表示を有効にすることで、即時更新とすることができます。<br />
パーツを動的にする方法は、他のパーツと同様に管理画面を編集し、「動的表示」を有効に設定ください。

## メール取込タスク

以下の rake コマンドを実行すると標準入力にメールを受け付けて、ページ保存を行います。

~~~
# rake mail_page:import site=www
~~~
※引数 site は対象のサイトのホスト名

入力は EML形式の文字列 に対応しています。<br />
Content-Type は一般的なメーラーで作成した UTF-8 と ISO-2022-JP の動作を確認しています。

### 動作確認

以下はメール取込タスクの動作確認の例です。<br />
（サーバー側のメール受信およびSHIRASAGIへの転送設定の動作確認ではありません）

１. SHIRASAGI をインストールしてサンプルサイトを立ち上げます。

２. 管理画面からメール取込フォルダーを作成します。<br />
送信者メールアドレスに sample@example.jp を設定します。<br />
宛先メールアドレスに sample@example.jp を設定します。

３. 以下のコマンドで、テスト用のメールを取り込みます。<br />

~~~
# cd /var/www/shirasagi
# cat spec/fixtures/mail_page/UTF-8.eml | rake mail_page:import site=www
~~~

４. 成功するとメール取込フォルダー配下にページが作成されます。<br />
サイト設定のジョブにも取り込んだログが残ります。

## サーバー側メール取り込み設定

サーバーでメールを受信し、SHIRASAGIのタスクに標準入力として引き渡す設定が必要になります。<br />
以下、取り込みメールのアドレスは sample@example.jp として説明します。

１．メールサーバを設定します。<br />
外部より sample@example.jp のメールが受信できるように設定します。

２．取り込みコマンドの実行ユーザを作成します。<br />
sudoがNOPASSWDで利用できるユーザを作成します。以下、mailinfo ユーザとします。


３．受信メールをSHIRASAGIに反映する為のシェルスクリプトを準備します。

~~~
# mkdir /var/www/mailhook/shirasagi -p
# cd /var/www/mailhook/shirasagi
# vi hook.sh
~~~

~~~
"| sudo /var/www/mailhook/shirasagi/script.sh"
~~~

~~~
# chown mailinfo. hook.sh
# vi script.sh
~~~

~~~
#!/bin/bash

export PATH=$PATH:/usr/local/rvm/wrappers/default;
umask 022

data=$( cat - )
data=$( echo "$data" | sed "s/$/\r/g" )

cd /var/www/shirasagi
echo "$data" | rake mail_page:import site=www
~~~

~~~
# chmod 777 script.sh
~~~

４．メール受信にて取り込みスクリプトを動作させるようにエイリアスを設定します。

~~~
# vi /etc/aliases
~~~

~~~
# Shirasagi Mail Hook
sample: mailinfo, :include:/var/www/mailhook/shirasagi/hook.sh
~~~

~~~
# newaliases
~~~
