## インストール
- brewを使う brew install aider 
- pipxに「新しいnumpy」を使わせる 　　CFLAGS="-O0" pipx install aider-chat または pipx install aider-chat --pip-args="--prefer-binary“
## ollamaで実行する

- xcodebuild
   プロジェクトファイルの作成でループするのでうまくいかなかった
   aider --test-cmd "xcodebuild -scheme あなたのアプリ名 -destination 'platform=iOS Simulator,name=iPhone 15' clean build" --auto-test
- swiftbuild[[]]
  aider --test-cmd "swift build" --auto-test
   