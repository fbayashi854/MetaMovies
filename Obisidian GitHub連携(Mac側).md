Obsidian Git プラグインを使わない場合、Macの標準機能（ターミナル、シェルスクリプト）とOSの自動化ツール（**launchd** または **cron**）を組み合わせて、バックグラウンドで完全に自動同期させるシステムを構築します。
プラグインに依存しないため、Obsidianが重くならない、起動が速い、裏で静かに確実に動く、といったメリットがあります。

## 最初に一度だけ行う初期設定の手順

シェルスクリプトは、私たちが普段ターミナルで手動で打ち込んでいる git add や git commit、git push といったコマンドを「身代わりになって自動で実行しているだけ」だからです。そのため、土台となるGitの環境が整っていないとエラーになって動びません。
まだ初期設定が済んでいない場合は、自動化スクリプトを動かす前に、以下の手順を**一度だけ手動で**実行してください。

### 1. 保管庫のフォルダに移動する
ターミナルを開き、Obsidianの保管庫（Vault）があるフォルダに移動します。
```bash
cd /Users/あなたのユーザー名/Documents/ObsidianVault

```
### 2. Gitリポジトリを初期化する
フォルダ内にGitの管理用データを構築します。
```bash
git init

```
### 3. リモートリポジトリ（GitHub）と紐付ける
GitHub上で空のリポジトリを作成し、そのURL（HTTPSまたはSSH）を登録します。
```bash
git remote add origin https://github.com/あなたのユーザー名/リポジトリ名.git

```
### 4. 最初のコミットとプッシュを行う
一度手動でプッシュを通しておくことで、次回以降スクリプトが自動で処理を引き継げるようになります。
```bash
# 最初のファイルをステージング
git add .

# 最初の手動コミット
git commit -m "First commit"

# メインブランチの名前を「main」に設定
git branch -M main

# GitHubに最初のプッシュ（次回以降の同期先として固定）
git push -u origin main

```
## 💡 重要なチェックポイント
初期設定の最後に git push -u origin main を実行した際、**ユーザー名やパスワード（トークン）を求められずに、すんなりプッシュが完了すること**を確認してください。
もしここでパスワード入力を求められる状態のままだと、裏でシェルスクリプトが動いたときに「パスワード入力待ち」で処理が止まってしまい、自動更新が失敗してしまいます。
 * **HTTPS接続の場合**: Macの「キーチェーンアクセス」にGitHubのパーソナルアクセストークン（PAT）が記憶されている必要があります。
 * **SSH接続の場合**: ~/.ssh/ にある公開鍵がGitHub側に登録されている必要があります。
手動で1回プッシュが成功すれば、あとは準備したスクリプトと launchd（または cron）が、あなたがMacを触っていない間も10分ごとに裏でこの作業を繰り返してくれるようになります。

## 1. 自動同期シェルスクリプトの作成
まずは、Gitのプル・コミット・プッシュを一連の流れで行うスクリプトを作成します。
ターミナルを開くか、テキストエディタで以下の内容のファイルを作成します。ここでは例として ~/.local/bin/obsidian_sync.sh という名前で保存します（フォルダがない場合は作成してください）。
```bash
#!/bin/bash

# --- 設定 ---
# あなたのObsidian保管庫（Vault）の絶対パスに変更してください
VAULT_PATH="$HOME/Documents/ObsidianVault"
# ------------

cd "$VAULT_PATH" || exit

# 1. リモートの最新変更を取り込む
git pull origin main --rebase

# 2. 変更があった場合のみコミットしてプッシュ
if [[ -n $(git status -s) ]]; then
    # すべての変更（追加・削除・修正）をステージング
    git add -A
    
    # コミット（日時をメッセージに含める）
    CURRENT_DATE=$(date "+%Y-%m-%d %H:%M:%S")
    git commit -m "Mac auto sync: $CURRENT_DATE"
    
    # GitHubへプッシュ
    git push origin main
fi

```
### スクリプトへの実行権限の付与
作成したスクリプトを実行できるように、ターミナルで以下のコマンドを実行します。
```bash
chmod +x ~/.local/bin/obsidian_sync.sh

```
> 💡 **ポイント（事前認証）**:
> このスクリプトがパスワード入力なしで動く必要があります。事前にターミナルで一度手動で git push を行い、SSH鍵やMacのキーチェーン（Keychain Access）経由でGitHubへの認証が完了していることを確認しておいてください。
> 
## 2. macOSの機能で定期実行する（2つのアプローチ）
スクリプトを定期的に裏で実行させる方法は、macOS標準の **「launchd（推奨）」** と、シンプルな **「cron」** の2種類があります。管理のしやすさでお好きな方を選んでください。
### アプローチA：launchd を使う（macOS推奨・確実）
Macがスリープから復帰した際にもスケジュールを再調整してくれるため、ノートの同期にはこちらが向いています。
~/Library/LaunchAgents/com.user.obsidiansync.plist というファイルを新規作成し、以下の内容を貼り付けます。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.obsidiansync</string>
    <key>ProgramArguments</key>
    <array>
        <!-- あなたのユーザー名に合わせてパスを修正してください -->
        <string>/Users/あなたのユーザー名/.local/bin/obsidian_sync.sh</string>
    </array>
    <!-- 同期を行う間隔（秒単位）。以下は600秒（10分）の例 -->
    <key>StartInterval</key>
    <integer>600</integer>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>

```
**バックグラウンドジョブの有効化:**
ファイルを作成したら、ターミナルでシステムに登録します。
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.obsidiansync.plist

```
*(解除したい場合は、bootstrap を bootout に変えてコマンドを実行します)*
### アプローチB：cron を使う（設定がシンプル）
昔ながらのシンプルな定期実行ツールです。
 1. ターミナルで crontab -e を実行します（標準のエディタが開きます）。
 2. 以下の行を追加して保存します。
```cron
# 10分ごとにスクリプトをバックグラウンドで実行
*/10 * * * * /Users/あなたのユーザー名/.local/bin/obsidian_sync.sh >/dev/null 2>&1

```
> ⚠️ **注意**: macOSのセキュリティ機能（環境設定の「プライバシーとセキュリティ」＞「フルディスクアクセス」）で、/usr/sbin/cron やターミナルに適切な権限が与えられていないと、ファイル変更を検知できずエラーになることがあります。そのため、基本的には **launchd（アプローチA）** の方がMacでは安定して動作します。
> 
## プラグインなし運用の注意点
 * **競合の回避**: 他の端末（iPhoneやiPadなど）でも同じリポジトリを編集している場合、Mac側でスクリプトが走る直前にリモートとローカルの同じ行が書き換わっていると、git pull --rebase の段階でコンフリクト（衝突）が発生し、自動プッシュが止まります。
 * **コンフリクトが起きたら**: もし自動更新が止まっているなと感じたら、Obsidianの保管庫フォルダで一度ターミナルを開き、手動で git status や git pull を行って競合を解消（コードのクリーンアップなど）してください。
