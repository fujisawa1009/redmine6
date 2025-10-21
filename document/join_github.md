## Redmineアップデート対応

# 827 設計
■まだ未対応リスト
・【新規追加】# 　 ：Teams通知⇒以下選定プラグイン
・【新規追加】# 845：github連携⇒以下選定プラグイン
・【新規追加】# 　 ：バーンダウンチャート

■環境
Redmine6.0.6/Rails 7.2.2.1/ruby 3.3.9

■テーマ・プラグイン整理
【途中】# 844 テーマ適応
```
  # 最新版をGitHubからクローン
  git clone https://github.com/farend/redmine_theme_farend_bleuclair.git bleuclair

  # または特定バージョンをダウンロード
  wget https://github.com/farend/redmine_theme_farend_bleuclair/archive/refs/tags/v1.1.1.zip
  unzip v1.1.1.zip
  mv redmine_theme_farend_bleuclair-1.1.1 bleuclair

  手順2: Docker環境での適用

  現在のDocker環境に適用する場合：

  # ホスト側でテーマをダウンロード
  cd /project/redmine/docker/redmine/themes
  git clone https://github.com/farend/redmine_theme_farend_bleuclair.git bleuclair

  # Redmineコンテナを再起動
  docker compose restart redmine

  手順3: テーマの有効化

  1. Redmine管理画面にアクセス
    - http://localhost:3000/ にアクセス
    - 管理者でログイン
  2. テーマ設定
    - 管理 → 設定 → 表示 タブ
    - 「テーマ」で「Bleuclair」を選択
    - 「保存」をクリック
```

■プラグイン対応
【完了】# 826 kanban(移行前同様プラグイン)
バージョン: 0.0.11
https://github.com/happy-se-life/kanban

```
⇒設定について：
  ※管理者権限設定
    - 管理 → ロールと権限 → 各ロールにkanban権限を付与
    Kanban ⇒ Display menu link
```

【完了】# 839 redmine_issues_tree代替
⇒最新のredmine標準機能でカバー
```
1. チケット一覧で「親チケット」列を表示
2. 親チケットIDでソートしてインデント表示
3. ガントチャートでの階層表示
4. カスタムクエリでの親子フィルタ
```

■・【新規追加】# 845：github連携⇒以下選定プラグイン
以下対応参考
https://qiita.com/n_slender/items/54cd282c140fadbbb322

コンテナ内で実行
gem install json

ソースの配置してRedmineの再起動
cd /project/redmine/plugins
git clone https://github.com/koppen/redmine_github_hook.git
⇒【完了】管理ページ>プラグインに追加されていれば成功。Redmine Github Hook plugin(3.0.2)

Redmineサーバ上にリポジトリをclone
apacheユーザがgit pullなどをできるような形にする。
ここでつまずいたのですが、sudo -u apache をつけることで解決

【まだ】GitHubのdeploy key用の鍵作成
参照だけで良いので、デプロイキーに登録する公開鍵を作成します。
ssh-keygen -t rsa

/var/www/.ssh　に鍵を配置。(ownerをapacheに)
パスフレーズなし。
作成された公開鍵をGitHubリポジトリのSetting＞Deploy Keys　から登録
https://developer.github.com/guides/managing-deploy-keys/

git clone
--bare　をつけて、サーバモードにする必要がある。
cd /opt/repos
 git clone --bare git@github.com:XXXXXXXXXXX/hoge.git
リモートの設定をして、fetchできるようにします。
git remote add origin git@github.com:<username>/<project>.git
⇒【完了】fech確認までgit fetch

Redmineのリポジトリ設定
プロジェクトの画面の設定＞リポジトリ＞新しいリポジトリ　から設定
⇒バージョン管理システム：Git
識別子：
リポジトリのパス：/project/fm-ptn-analysis.git
エンコーディング：UTF-8

【まだ】GitHubのWebhook
GitHubにpushされたら、redmineのリポジトリにそれを通知させるために、webhookの設定をします。
手順はプラグインのreadmeに書いてあります。
下記のように、通知先のURLを設定します。

https://[your_redmine]/githook?project_id=[your_project_id]

チケットとコミット（リビジョン）の関連づけ
公式サイトにもある簡単なやり方まとめ。
http://redmine.jp/tech_note/subversion/

```
まず結論（やることの全体像）
ホスト側: /project/fm-ptn-analysis.git をコンテナ redmine-app に 同じパスで マウント（読み取り専用でOK）
コンテナ内: safe.directory を設定してから rake redmine:fetch_changesets を実行
（運用）ホストの cron で git fetch、ホストの cron で docker exec して Redmine に取り込み
```

