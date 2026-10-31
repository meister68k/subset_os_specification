# sh0(1)

## 名前
**sh0** — 対話型の極小コマンドシェル

## 書式
`/bin/sh0 [script_path]`

## 説明
引数なしでは対話モードで標準入力から1行ずつ読み、script pathnameを1つ指定するとスクリプトモードでそのfileを実行します。スクリプトモードではプロンプトや入力echoを出しません。スクリプトは固定長bufferで1行ずつ読み、LFで行を区切ります。最大行長は126 byteです。pathnameに`-`を指定しても標準入力の別名にはなりません。

どちらのモードも空白で単語を分割し、組み込みまたは内蔵binaryを実行します。最大8引数（`argv[0]`を含む）、最大8コマンド単位/行です。`-c`などのoptionはありません。quote、escape、glob、環境変数、PATH探索もありません。スクリプトpathnameは標準入力fd 0とは別に開くため、スクリプト内で起動するコマンドの標準入力は継承元のままです。

`;`で最大8単位を左から順に実行します。各単位の終了を待ち、実行時失敗後も次へ進みます。先頭/末尾の`;`、`;;`、行中の構文不正は行全体を実行せず拒否します。`exit`は残りの単位を実行せず終了します。

単独コマンドには`<`と`>`によるredirectを各1つ指定できます。2段pipeline（`|` 1個）を実行できます。左stageの標準出力を右stageの標準入力へ接続し、両stageの終了を待ってから次のコマンド単位へ進みます。pipelineではsh0組み込みをstageとして使えません。

pipelineは外部内蔵コマンド2段までです。pipeline内のredirectは左stageの`<`と右stageの`>`に限ります。構文例は`echo left | cat`、`cat <in.txt | head`、`echo right | cat >out.txt`です。

スクリプトの構文errorやコマンド起動失敗は診断して次の行へ進みます。script open/read errorは異常終了します。行長超過の行は読み捨てて実行しません。スクリプト内の`exit`は残りを実行せずshellを終了します。`-c`、script引数の参照、option指定はありません。shebang行をスクリプトとして渡された場合は、先頭`#`のコメントとして読み飛ばします。script pathnameを直接コマンドとして実行するshebang解釈はkernel側の別機能です。

## 組み込み
- `exit`: shellを終了
- `cd path`: cwdを変更
- `pwd`: cwdを表示
- `PID`: sh0自身のPIDを表示

## 使用例
```sh
cd /tmp; pwd
cat <in.txt >out.txt
echo left | cat
echo hello | wc
sh0 /etc/rc
```

## 終了
対話モードは`exit`または入力EOFで0です。スクリプトモードは最後まで処理すると0、`exit`でも0で終了します。引数不正、script open/read error、shell自身の致命的な入力/出力errorでは1です。
