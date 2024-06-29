## 手動でのMisskey(隠れ家フォーク)のセットアップ方法

> [!TIP]
> 基本的な操作は[MisskeyHubのドキュメント](https://misskey-hub.net/ja/docs/for-admin/install/guides/manual/)に記載されている内容と同様です。  

### 1. 前提ソフトウェアのインストールとユーザーの作成
ドキュメントに記載されている手順に従って、Misskeyの動作に必要なソフトウェアのインストールおよびユーザーの作成を行います。

- [前提ソフトウェア](https://misskey-hub.net/ja/docs/for-admin/install/guides/manual/#%E4%BB%A5%E4%B8%8B%E3%81%AE%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E3%81%8C%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E8%A8%AD%E5%AE%9A%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%82%8B%E3%81%93%E3%81%A8)
- [ユーザーの作成](https://misskey-hub.net/ja/docs/for-admin/install/guides/manual/#%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%81%AE%E4%BD%9C%E6%88%90)

> [!IMPORTANT]
> ドキュメントに記載されている前提ソフトウェアに加え、**pgroongaのインストールが必須になります**  
> インストール方法は[こちら](https://pgroonga.github.io/ja/install/)を参照してください。
>
> #### マイグレーション時の注意点
> マイグレーションを行うユーザー (.config/default.yml のdbに記載されているユーザー) がスーパーユーザー権限を持っていないとコケる可能性があります。  
> マイグレーション時に `QueryFailedError: permission denied to create extension "pgroonga"` が表示された場合は、一時的にスーパーユーザー権限を付与してマイグレーションを行ってください。
>
> #### pgroongaを使用しない場合
> 手順「2. リポジトリのクローンとインストール」の後に以下のコマンドを実行し、関連するコミットおよびコードを取り除いてください
> ```bash
> git revert 923c9de5c5cb1ba2e3e9c924f5227aaa61bd7f00
> git revert 4b63224f3a53a5e38f9a901e40c7a4fadbbd42a8
> ```

### 2. リポジトリのクローンとインストール
以下のコマンドでリポジトリをクローンし、依存関係のインストールを行います。

```bashsudo -iu misskey
git clone --recursive https://github.com/hideki0403/kakurega.app.git
cd kakurega.app
git checkout master-kakurega
git submodule update --init
NODE_ENV=production pnpm install --frozen-lockfile
```
### 3. 設定およびビルドと初期化
ドキュメントに記載されている手順に従い、設定・ビルド・初期化を行います。

- [設定](https://misskey-hub.net/ja/docs/for-admin/install/guides/manual/#%E8%A8%AD%E5%AE%9A)
- [ビルドと初期化](https://misskey-hub.net/ja/docs/for-admin/install/guides/manual/#%E3%83%93%E3%83%AB%E3%83%89%E3%81%A8%E5%88%9D%E6%9C%9F%E5%8C%96)

### 4. 起動
以下のコマンドでMisskeyを起動できます。

```bash
NODE_ENV=production pnpm run start
```

### 5. サービスの設定 (任意)
ドキュメントに記載されている「[systemdを用いた管理](https://misskey-hub.net/ja/docs/for-admin/install/guides/manual/#%E3%83%93%E3%83%AB%E3%83%89%E3%81%A8%E5%88%9D%E6%9C%9F%E5%8C%96:~:text=GLHF%E2%9C%A8-,systemd%E3%82%92%E7%94%A8%E3%81%84%E3%81%9F%E7%AE%A1%E7%90%86,-Misskey%E3%81%AE%E3%82%A2%E3%83%83%E3%83%97%E3%83%87%E3%83%BC%E3%83%88)」に従って操作を行うことでサービスとして運用することが出来ますが、サービスファイルの作成時に以下の箇所を修正する必要があります。

```diff
--- a.service   2024-03-11 16:21:27.559208177 +0900
+++ b.service   2024-03-11 16:21:39.335118743 +0900
@@ -5,7 +5,7 @@
 Type=simple
 User=misskey
 ExecStart=/usr/bin/npm start
-WorkingDirectory=/home/misskey/misskey
+WorkingDirectory=/home/misskey/kakurega.app
 Environment="NODE_ENV=production"
 TimeoutSec=60
 StandardOutput=journal
 ```