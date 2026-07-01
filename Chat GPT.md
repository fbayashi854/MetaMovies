
---
aider ollamaの組み合わせでswiftでアプリ開発しようとしています  
うまくいかないケースとして  
シミュレータでは動くのに実機では動かない  
CLIベースの対話型のプログラムなのに何も聞かれずに終了するプログラムが正常終了してると勘違いして修正が行われない  
といった自動修正させる上での問題が出ています  
このような問題を出しにくい  
プロントやaiderに入れるtestコマンドの工夫点を教えてください

---

とても良いポイントです。  
Aider + Ollamaで自動修正ループを回す場合、一番重要なのは**「AIが判断する情報」を増やすこと**です。

AIは基本的に

testコマンドの終了コード  
標準出力  
標準エラー

しか見ていません。

そのため「正常終了=成功」と判断されるケースを極力減らす必要があります。

  

1. testコマンドは「期待する状態」を検証する

例えば

swift run MyApp

だけでは

exit code = 0

なので

成功

と判断されます。

代わりに

` bash
swift run MyApp <<EOF
quit
EOF | grep "Ready"
`


のように

期待する文字列が出るか

までチェックします。

さらに

swift run MyApp <<EOF

help

quit

EOF

  

grep "Available commands"

など

期待する出力を必ず検証

するようにします。

  

2. CLIはexpectを使う

対話型なら

expect

を使うのがかなりおすすめです。

例えば

#!/usr/bin/expect

  

spawn .build/debug/myapp

  

expect "command>"

  

send "help\r"

  

expect "Available commands"

  

send "quit\r"

  

expect eof

失敗すると

exit 1

になるので

Aiderが修正できます。

  

3. シミュレータだけでなく実機も確認する

例えば

xcodebuild \

destination='platform=iOS Simulator,name=iPhone 16'

だけでは

実機問題は絶対に見つかりません。

おすすめは

xcodebuild test ...

  

xcrun devicectl ...

  

idevicedebug ...

など実機で起動まで確認することです。

少なくとも

アプリが起動後5秒以内に落ちない

を確認します。

  

4. 実機ログを見る

一番重要です。

例えば

log stream \

--predicate 'process == "MyApp"'

や

xcrun simctl spawn booted log stream

をtestコマンドに含めます。

そうすると

Fatal error

や

permission denied

がAIに渡ります。

  

5. 「正常終了は禁止」にする

例えばCLIなら

if ! grep -q "Goodbye" output.log

then

    echo "Program exited unexpectedly"

    exit 1

fi

これだけでもかなり違います。

  

6. プロンプトでも明示する

毎回最初に

Never assume exit code 0 means success.

  

Always verify:

  

- expected output

- expected files

- expected UI state

- expected prompt

  

If these are absent, treat as failure.

と書いておくとかなり改善します。

  

7. SwiftならXCTestを利用する

CLIでも

XCTest

で

Process()

  

Pipe()

  

stdin

  

stdout

を使って

本当に対話できたか

までテストできます。

AIは

swift test

だけ回せば済みます。

  

8. Aiderに渡すtestコマンドを賢くする

例えば

./test.sh

だけ指定します。

中身は

#!/bin/bash

set -e

  

swift build

  

OUTPUT=$(printf "help\nquit\n" | .build/debug/myapp)

  

echo "$OUTPUT"

  

echo "$OUTPUT" | grep "Available commands"

  

echo "$OUTPUT" | grep "Bye"

これなら

どれか欠ければ

exit 1

になります。

  

9. AIへのシステムプロンプトを強化する

私なら最初にこれを書きます。

You are not allowed to assume a program works because it exited successfully.

  

A successful program must satisfy all of the following:

  

- produce the expected prompt

- accept the expected input

- produce the expected output

- keep running until the user exits

- pass every automated test

  

If any expectation is missing, treat it as a bug.

  

Never remove tests to make them pass.

  

If a test cannot verify behavior, improve the test before changing the code.

このように**「終了コードではなく期待される振る舞いで成功を判定する」**ことをAIに明確に指示します。

  

10. 実機とシミュレータの差異への対策

Swift/iOS開発では、シミュレータと実機での違いが自動修正の妨げになることが多いため、テストを次の3段階に分けるのがおすすめです。

1. ユニットテスト (swift test や xcodebuild test)：ロジックの正しさを確認。
2. シミュレータでのUI・起動テスト：画面遷移や基本動作を確認。
3. 実機でのスモークテスト：アプリの起動、権限要求、クラッシュ有無、実機固有機能（カメラ、Bluetooth、USB、センサーなど）を確認。

Aiderの--test-cmdには、これらを順番に実行するスクリプトを指定し、どれか1つでも失敗したら非0終了コードを返すようにすると、自動修正の精度が大きく向上します。

  

私がAider + OllamaでSwift開発環境を構築するなら、**「テスト駆動」ではなく「振る舞い駆動（Behavior-Driven）」**に寄せます。つまり、--test-cmdは単なるビルドや終了コードの確認ではなく、「期待する入出力・ログ・UI状態・実機での起動確認」まで検証するスクリプトにし、AIには「期待した振る舞いが確認できなければ失敗」と判断させる設計にするのが最も効果的です。



expect は、対話型（インタラクティブ）プログラムを自動操作するためのツールです。

もともとはシェルスクリプトでは自動化しにくい「入力待ち」のあるプログラムを操作するために作られました。

例えば、次のようなCLIプログラムがあるとします。

$ myprogram

Enter your name:

普通のシェルスクリプトでは、この入力待ちに答えるのが苦手です。

しかし expect を使うと、

#!/usr/bin/expect

  

spawn myprogram

  

expect "Enter your name:"

send "Shigeki\r"

  

expect eof

というように、

- spawn : プログラムを起動
- expect : 指定した文字列が表示されるまで待つ
- send : キーボード入力を送る

という流れを書けます。

  

よく使われる用途

SSHの初回接続

Are you sure you want to continue connecting (yes/no)?

↓

yes

を自動入力。

  

パスワード入力

Password:

↓

mypassword

を自動入力。

※セキュリティ上は、公開鍵認証やトークンなどの利用が推奨されます。

  

メニュー操作

1) Create

2) Delete

3) Exit

↓

2

を自動入力。

  

Aiderとの相性

あなたが前に話していた

CLIベースなのに質問されず終了してしまう

というケースでは、expect はかなり役立ちます。

例えば

aider

を起動して

Model?

が出たら

ollama/deepseek-r1

を入力

というような自動化ができます。

逆に、

何も表示されず終了した

というケースも検知できます。

例えば

expect {

    "Model?" {

        send "ollama/deepseek-r1\r"

    }

    eof {

        puts "プログラムが途中で終了"

    }

    timeout {

        puts "入力待ちにならなかった"

    }

}

このように、

- 正常に質問が表示された
- すぐ終了した
- 一定時間何も起きない

を区別できます。

  

Aiderの自動修正との相性

以前相談されていた

- シミュレーターでは動くが実機では動かない
- CLIが何も聞かず終了したのに「正常終了」と誤認してしまう

といった問題では、expect をテストドライバーとして使う方法が有効です。

例えば

aider

    ↓

ビルド

    ↓

expectでCLI操作

    ↓

期待した文字列が出るか確認

    ↓

失敗ならログをAiderへ返す

というループにすれば、「終了コードが0だから成功」と判断するのではなく、実際の対話内容まで確認して自動修正に活かすことができます。

  

ただし注意点

expect は画面に表示される文字列をもとに動作するため、

- 表示メッセージが変わるとスクリプトの修正が必要
- 色付き出力や特殊なターミナル制御を使うプログラムでは扱いが難しい場合がある

という弱点があります。

  

あなたの Aider + Ollama + Swift の開発環境では、expect を「対話テストの自動化」に使うのは有力な選択肢です。ただし、より堅牢な方法としては、CLIアプリ自体に**非対話モード（コマンドライン引数や標準入力から設定を受け取るモード）**を用意し、expect はどうしても対話が必要な部分だけに使う設計のほうが、長期的には保守しやすくなります。