# 極小UNIX `man` コマンド仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-MAN-001 |
| 版 | 0.1（実装前） |
| 対象 | サブセットOSの内蔵ユーザーコマンド `man` |
| 位置づけ | Markdown形式のセクション1リファレンスを標準出力へ表示する |
| 関連仕様 | `command_*_spec.md`、ユーザーランド内蔵バイナリ仕様、`open_close_implementation_spec.md` |

## 1. 目的と範囲

`man`は、指定されたコマンドのリファレンスマニュアルを開き、Markdownの装飾を省いたプレーンテキストとして標準出力へ表示する。固定領域・逐次read/writeで実装し、ページャーや動的メモリ確保を必要としない。

マニュアルはすべてセクション1とし、Filesystem上の`/man/man1/<name>.1.md`に置く。セクション番号の指定や他セクションの探索は行わない。

## 2. 起動形式

```text
man name
```

- `name`をちょうど1つ指定する。引数なし、余分な引数、option形式はusage errorとする。
- `name`はページbasenameであり、section、`.1`、`.1.md`、directory prefixを付けずに指定する。例：`man cat`。
- `name`は空でない単一のbasenameとし、`/`を含む値は拒否する。ファイルパスはコマンドが`/man/man1/`を付けて構成する。
- page名の大文字小文字は区別する。例：sh0の組み込み`PID`は`man PID`で指定する。
- `-k`、`-f`、`-a`、`--`、section指定、複数page指定は提供しない。
- glob、quote、環境変数展開は行わず、受け取った`argv`を使う。

## 3. pageの検索

1. 引数検査後、`/man/man1/`、`name`、`.1.md`を連結してpathnameを作る。
2. pathnameを`O_RDONLY`でopenする。
3. 対応するpageが存在しない、openできない、または読み取りに失敗した場合は診断し、失敗終了する。
4. 終了前にopenしたfdをcloseする。標準入出力fdはcloseしない。

`man`はdirectoryを列挙して候補を検索しない。alias、section 1以外、別directory、basename推測、`.1.md`以外の拡張子は扱わない。pathnameの長さはFilesystemのpathname上限に従い、切り詰めない。

## 4. Markdown表示規則

完全なCommonMark実装は行わず、本リポジトリのリファレンスマニュアルに必要な構文だけを行単位で処理する。認識する構文記号を出力から除去し、本文と例の文字は保つ。

| Markdown構文 | 表示 |
|---|---|
| 行頭の`#`〜`######`と直後の空白 | 見出し文字だけを表示 |
| 行頭の`- `または`* ` | markerを除き項目本文を表示 |
| 行頭の`> ` | markerを除き引用本文を表示 |
| 行頭の番号付きlist marker（`N. `） | markerを除き項目本文を表示 |
| 行頭のfence（任意の言語名を伴う`` ``` ``） | fence行を表示せず、内側のコード行はそのまま表示 |
| 対応するbacktick pairで囲まれたinline code | backtick pairのみ除去 |
| 対応する`**`、`__`、`*`、`_` pair | marker pairのみ除去 |
| 単純なlink `[label](target)` | `label`のみ表示。入れ子のlabel/targetは非対応 |
| 行頭と行末が`|`のMarkdown table row | `|`で分割したcellを2つの空白で連結して表示 |
| table separator行（`---`等のみのcell） | 行全体を表示しない |

- 認識しないHTMLや拡張Markdown構文は解釈しない。危険な操作や外部参照も実行しない。
- emphasis/codeのmarkerは対応するpairがある場合だけ除去する。不一致の`*`、`_`、backtick等は通常文字として出力する。
- 変換は表示時だけに行い、マニュアルfileを書き換えない。
- 空行は読みやすさのため維持する。空でないpageの末尾に改行がない場合は、最後の出力行の後にLFを1つ出す。
- Markdown marker除去後のテキストを端末幅で再配置しない。長い行の折り返しは出力先に委ねる。
- 出力順はpageの記載順とし、見出し番号や例の行を並べ替えない。

### 4.1 固定長処理

- 1行の読取bufferは固定長512 byteとする。LFを行終端とし、CRは本文byteとして扱う。
- 512 byteを超える行は切り詰めず、page表示を失敗として中止する。
- 変換中の出力bufferも固定長とする。出力がbuffer容量を超えそうな場合は分割writeする。演算は範囲検査し、動的確保を行わない。
- page全体をRAMへ読み込まない。固定長bufferでread、変換、writeを繰り返す。

## 5. ページャーと入出力

- ページャー、画面クリア、画面位置制御、キー入力によるページ送りは使用しない。
- pageを先頭から末尾まで連続して標準出力fd=1へ出力する。
- 診断は標準エラーfd=2へ`man:`を付けて出す。
- `read()`の正の短い戻り値はEOFとみなさず、残りの入力を続けて読む。0はEOF、負値はerrorとする。
- `write()`の正の部分書込みは残りを再送する。0、負値、要求を超える戻り値では表示を中止する。
- 検索結果を対話的に選択したり、標準入力をマニュアル本文として読んだりしない。

## 6. 終了ステータスとエラー

| 状況 | 動作 |
|---|---|
| pageを最後まで表示し、closeも成功 | `0`で終了 |
| 引数不足・過剰、`name`不正 | usageを診断し、`1`で終了。pageをopenしない |
| pageが存在しない、open失敗 | 診断し、`1`で終了 |
| read、Markdown行長、変換、stdout write失敗 | 表示を中止し、fdをcloseして`1`で終了 |
| close失敗 | 診断可能なら出力し、`1`で終了 |

エラー後に別名や別sectionを探索してfallbackしない。診断出力自体が失敗した場合、再帰的に診断を行わない。

## 7. 使用API・実装制約

| API | 用途 |
|---|---|
| `open()` | page fileを読み取り専用で開く |
| `read()` | pageを固定長bufferへ逐次読取 |
| `write()` | 標準出力と診断の出力 |
| `close()` | page fdを解放 |
| `std::strcmp()`等 | basename検査・固定文字列判定 |
| `main()`の戻り値 | Process終了status |

- 内蔵バイナリ名とFilesystem pathnameは`man`、`/man/man1/<name>.1.md`とする。
- 動的メモリ、`FILE*`、`printf()`、host OS固有API、shell起動、カーネル内部Filesystem APIを使わない。
- `man`自身のpageも通常どおり`/man/man1/man.1.md`に置ける。自己参照表示を特別扱いしない。

## 8. 受入確認項目

1. `man cat`が`/man/man1/cat.1.md`のみを開き、section指定なしで表示する。
2. 大文字小文字を区別し、`man PID`が`PID.1.md`を選ぶ。
3. 引数なし、複数引数、slashを含むnameを拒否し、pageをopenしない。
4. 未知のpageを診断し、終了status 1とする。
5. 見出し、list、quote、code fence、inline code、強調、link、tableの装飾を除去し、本文を保つ。
6. code fence内の行はmarkerを除いて保持し、table separator行は表示しない。
7. 512 byte以下の通常行とbuffer境界で分割されたreadを正しく処理する。
8. 512 byte超の行は切り詰めず失敗する。
9. stdoutの部分writeを再送し、0/負値で中止する。
10. 全pageを連続出力し、TTYでもページャーやキー入力を使わない。
11. 成功・失敗いずれもopenしたfdをcloseし、close errorをstatusへ反映する。

## 9. 対象外

- セクション1以外のmanual、section指定、alias、複数directory検索。
- pager（`more`、`less`等）、対話的検索、page選択、画面制御。
- Markdown全仕様、HTML、画像、拡張構文、syntax highlighting。
- `-k` keyword search、`-f` whatis検索、`-a`全候補表示。
- `--help`、`--version`、locale変換、折り返しや端末幅検出。
