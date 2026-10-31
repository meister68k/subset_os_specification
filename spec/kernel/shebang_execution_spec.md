# `spawn()` シバン実行仕様

| 項目 | 内容 |
|---|---|
| 文書ID | SPAWN-SHEBANG-SPEC-001 |
| 版 | 1.0 |
| 対象 | `spawn()`による内蔵プログラム・Filesystem実行対象の解決とシバン処理 |
| 関連仕様 | [`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、[`process_management_spec.md`](process_management_spec.md)、[`command_sh0_spec.md`](../command/command_sh0_spec.md) |
| 状態 | 仕様 |

## 1. 目的

本書は、内蔵ユーザープログラムとFilesystem上の実行ファイルを同じ`spawn()` APIから起動する規則、およびシバンスクリプトをインタープリターへ引き渡す規則を定める。`spawn()`のsyscall番号・引数順・FD継承ABIは変更しない。

実行イメージの形式、Filesystem pathnameの上限、host上の実行メモリ確保方式など、実行ローダ固有の事項はローダ仕様に従う。本書はそれらの形式を定義しない。

## 2. 実行対象の名前解決

`spawn(name, argv, inherit_fds, inherit_fd_count)`は次の順で実行対象を解決する。

1. `name`を内蔵ユーザープログラム表の登録名とバイト単位で完全一致照合する。
2. 一致した場合は、対応する内蔵プログラムを直接起動する。Filesystemを検索しない。
3. 一致しなければ`name`をFilesystem pathnameとして、呼出し元Processのcwd基準で解決する。絶対pathnameはroot基準とする。
4. `PATH`検索は行わない。コマンド名からpathnameへの探索は`init`、`sh0`等の呼出し側が行う。

内蔵プログラム名に`/`を含めない。`!echo`等は単なる登録名であり、接頭辞による名前空間や特殊な起動構文ではない。`!echo`と完全一致する登録があれば内蔵プログラムを起動し、なければ通常のFilesystem pathnameとして扱う。内蔵名と同じ文字列のFilesystemファイルがあっても内蔵プログラムが優先される。

## 3. シバンの認識

Filesystemから開いたpathname対象について、ファイル先頭の2 byteが`#!`（`0x23, 0x21`）と一致するとき、シバンスクリプトとして扱う。ファイル先頭以外にある`#!`はシバンではない。

シバン行は、先頭`#!`の後から最初のLFまたはEOFまでとする。実装は固定長バッファを用い、動的メモリを必要としない。シバン行の本文上限は127 byte（`#!`を除く）とし、超過した場合は`spawn()`失敗とする。NUL byte、空行、空のinterpreter名も不正として失敗する。

本文は`isspace()`に該当するbyteで分割する。先頭の空白類を読み飛ばし、最初の単語をinterpreter名とする。後続は空白類だけを許し、2個目の非空白単語があれば、任意引数形式としては扱わず不正として失敗する。引用符・エスケープ・変数展開はない。

`#!`直後に空白がなくてもよい。したがって次の行のinterpreter名は`!echo`となる。

```text
#!!echo
```

`isspace()`は共通Cライブラリの定義に従う。CRを個別に除去・正規化しない。CRが`isspace()`に該当する実装では、行末CRは区切り空白として扱われる。

## 4. interpreterの解決と再帰

抽出したinterpreter名を、§2と同じ規則で解決する。登録名と完全一致すれば内蔵プログラムを選び、それ以外はFilesystem pathnameとして扱う。`PATH`検索はしない。

初期仕様ではinterpreter自体が別のシバンスクリプトである場合、`spawn()`を失敗させる。シバンの再帰解釈・循環検出・最大再帰深度は実装しない。これにより、シバン処理は元の実行対象に対する一段だけとなる。

## 5. interpreterへ渡す引数

呼出し側の元の`argv`を`argv_in`、Filesystem上のスクリプトを指定した元の`name`を`script_name`とする。シバン対象を起動する場合、子へ渡すargvを次の順序で構成する。

```text
argv[0] = interpreter名
argv[1] = script_name
argv[2...] = argv_in[1...]
```

`argv_in[0]`は引き継がない。`argv_in[1...]`は内容と順序を維持する。スクリプトpathnameは解決後の絶対pathnameに置換せず、呼出し側が指定した文字列をそのまま渡す。相対pathnameは、子が継承するcwdを基準に再度解決できる。

生成されるargvも通常のspawn引数制限（要素数最大16、文字列のNULを含む合計格納量最大256 byte）を満たさなければならない。変換後に上限を超える場合は、子を公開せず`spawn()`を失敗させる。空文字列の元引数は保持する。

shebang行内の追加引数は初期仕様で未対応とする。元の呼出し側引数を転送する機構は備えるが、インタープリター側がそれらを受け付けるかは各コマンド仕様に従う。現行sh0はscript pathname以外の追加位置引数を受け付けない。

## 6. Process名、cwd、FD

- Process表示名は、常に呼出し側が`spawn()`へ渡した元の`name`とする。内蔵interpreter名や変換後の`argv[0]`で上書きしない。
- 子Processは通常のProcess管理仕様に従って呼出し元cwdを継承する。
- `inherit_fds`と`inherit_fd_count`の扱いは通常の`spawn()`と同一とする。シバン読取りのためにカーネルが開いたfdを子へ継承しない。
- スクリプトインタープリターはpathnameを自身で開く。呼出し元がスクリプトfdを明示的に継承リストへ含めた場合を除き、元のスクリプトfdはインタープリターから見えない。
- 内部で開いた実行対象ファイルは、実行対象解決と必要なイメージ準備後に閉じる。close失敗時も部分初期化された子Processを公開しない。

## 7. 失敗と資源処理

次の場合、`spawn()`は既存の失敗値で失敗を返し、子Processを公開しない。

- 内蔵名との照合に失敗し、pathnameを解決できない、または開けない。
- シバン本文が上限を超える、NULを含む、interpreter名が空、または追加の非空白単語を含む。
- interpreter名を内蔵名またはFilesystem pathnameとして解決できない。
- interpreterがさらにシバンスクリプトである。
- 元または変換後argvが通常の個数・格納量制限を満たさない。
- 読取り、seek、close、実行形式検査、イメージ準備、Process/Context/FD準備のいずれかが失敗する。

失敗時のProcess、FD参照、cwd参照、argv、Contextおよびイメージ資源のrollbackは通常のspawn生成失敗契約に従う。シバン解析のための一時領域はProcess公開前に解放する。

## 8. sh0との関係

シバン起動で`sh0`をinterpreterに指定する場合、たとえばFilesystem上のスクリプト先頭を次のようにできる。

```text
#! /bin/sh0
```

または、内蔵登録名が`!sh0`なら次のようにできる。

```text
#!!sh0
```

後者はシバン本文から`!sh0`を抽出し、内蔵名完全一致で起動する。シバンを使わず`spawn("!sh0", ...)`する場合も同じ内蔵プログラムが直接起動される。

どちらの場合もsh0には`argv[0]`にinterpreter名、`argv[1]`にスクリプトpathnameが渡るため、現行のsh0起動引数仕様でスクリプトモードとして動作する。シバン行自体はsh0のコメント規則でも読み飛ばされる。

## 9. 受け入れ確認項目

1. 登録済み内蔵名との完全一致でFilesystemを開かずに内蔵プログラムが選ばれる。
2. 内蔵名と一致しない名前がpathnameとして解決され、`PATH`探索が行われない。
3. 内蔵名と同名のFilesystem pathnameでは内蔵プログラムが優先される。
4. `#!!echo`から`!echo`が抽出され、登録済み内蔵プログラムがinterpreterとして選ばれる。
5. `#! /bin/sh0`のような空白ありのシバンでもinterpreter pathnameを抽出できる。
6. argvの変換順、`argv_in[0]`除外、元script pathnameの保持、後続引数と空引数の保持を確認する。
7. argv変換後の要素数・合計byte数超過は失敗し、子やFD参照を残さない。
8. 非先頭`#!`はシバン扱いされず、通常の実行形式判定へ進む。
9. 空interpreter、NUL、追加interpreter引数、本文長超過、未解決interpreter、シバンinterpreterの再帰が失敗する。
10. シバン読取用fdが子へ漏れず、明示的なFD継承リストの意味が維持される。
11. 読取り・seek・close・ローダ・Process準備の失敗で、ProcessをREADYとして公開せず資源を通常の失敗契約どおりに解放する。
