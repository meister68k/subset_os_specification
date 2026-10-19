# 極小UNIX `mkdir` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-MKDIR-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの `/bin/mkdir` |
| 位置づけ | pathnameを指定してdirectoryを作成するユーザーランドコマンド |
| 依拠する仕様 | `mkdir_rmdir_implimentation_spec.md`、`filesystem_design_spec.md`、syscall入口仕様 |

## 1. 目的と範囲

指定された各pathnameに新しいdirectoryを1つ作成する。親directoryが存在しない場合に親を再帰作成する機能、権限変更、umask、シンボリックリンク、オプション解析は対象外とする。

このコマンドはユーザーランドから`mkdir(path, mode)` syscallを呼ぶ。pathname解決、cwd基準の相対path、root基準の絶対path、directory構造の初期化はOS側の責務とし、コマンドからFilesystem内部APIやディスク表現を参照しない。

## 2. 起動形式

```text
mkdir directory ...
```

- `directory`を1つ以上指定する。各引数は独立に作成し、引数順に処理する。
- オプションは提供しない。`-p`、`-m`、`--`は特別扱いせず、pathnameとして渡す。
- `-`で始まるpathnameも通常のpathnameとして扱う。
- 絶対pathと相対pathを受け付ける。相対pathは呼出し元Processのcwdを基準とする。
- 引数なしは使用方法エラーとする。
- 空pathnameは拒否され、OSの`EINVAL`相当の失敗となる。
- 各pathnameはFilesystem仕様の`PATH_MAX`（128バイト未満のNUL終端文字列）以内でなければならない。name componentは`NAME_MAX`（27バイト）以内とする。文字数ではなくバイト数で上限判定する。

## 3. 作成契約

- 各引数について、`mkdir(path, 0777)`相当を1回呼び出す。
- `mode`は許可ビット`0777`を指定する。現行OSはumaskを適用しないため、成功したdirectoryの許可ビットは`0777`となる。
- `0777`はdirectory type bitsを含まない。OS/Filesystem側がdirectory種別を設定する。
- 既存のdirectoryまたは通常fileと同名の場合は作成失敗（`EEXIST`相当）。既存directoryを成功扱いにしない。
- pathnameに複数componentがある場合、中間componentはすでに存在する必要がある。`mkdir -p`相当の再帰作成はしない。
- `.`、`..`や末尾slashなどの詳細なpathname規則とエラーは`mkdir_rmdir_implimentation_spec.md`およびFilesystem pathname仕様に従う。
- 一つの引数が失敗しても残りの引数を処理する。作成済みdirectoryを後からrollbackしない。

## 4. 出力・エラー・終了ステータス

- 成功時は何も標準出力へ出さない。
- 失敗時は標準エラーfd=2へ`mkdir:`で始まる診断を出す。pathnameと、利用可能ならエラー内容を含める。厳密な文言とerrno数値の表示形式は規定しない。
- 診断出力自体が失敗しても追加の再試行や無限ループを起こさない。
- 失敗した引数を記録し、後続の引数を処理する。
- 全引数が成功した場合は終了ステータス`0`、引数不足または一件以上の失敗があった場合は`1`とする。
- 終了は`main()`の戻り値をランタイムがProcess終了へ変換する既定経路を使う。

| 状況 | 動作 |
|---|---|
| 引数なし | 使用方法を診断し、`1`で終了 |
| directory作成成功 | 無出力で次の引数へ |
| path不正、親なし、既存名、非directoryの親、資源/I/Oエラー | 診断し、次の引数へ |
| 全引数成功 | `0`で終了 |
| 一件以上失敗 | 全引数処理後に`1`で終了 |

OS側のstatus-to-errno対応はsyscall境界の仕様に従う。コマンドはFilesystemの`FsStatus`を直接扱わない。

## 5. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `mkdir(path, mode)` | directory作成 |
| OSラッパ | `write()` | 標準エラーへの診断 |
| ilibc | `std::strlen()`または既存書式出力 | pathname長・診断出力 |
| ランタイム | `main()`の戻り値 | Process終了ステータス |

- 固定上限内の`argv`を順次処理し、動的メモリ確保を行わない。
- `FILE*`や`printf()`、Filesystem内部API、ホストOS固有のmkdir呼び出しを使わない。
- `write()`が部分書き込みを返した場合は残りを再送する。0または負値で停止し、診断の失敗を再帰的に診断しない。
- `mkdir()`失敗時にOSが作成済みとしていないpathnameを成功扱いしたり、独自にrollbackしたりしない。

## 6. 受入テスト

1. 引数なしで診断し、終了ステータス`1`となる。
2. cwd直下への単一directory作成が成功し、標準出力が空、終了ステータスが`0`となる。
3. 複数pathnameを指定し、全てを引数順に作成する。
4. 相対pathと絶対pathがそれぞれcwd/root基準で作成される。
5. `mode`に`0777`を渡し、作成inodeのdirectory種別と許可ビットがOS契約どおりである。
6. 同名file/directory、存在しない親、非directoryの親、空path、`.`、`..`を失敗として扱う。
7. `PATH_MAX`と`NAME_MAX`の境界、および一つ超過した値をOS契約どおり扱う。
8. 一つの引数が失敗しても後続引数を処理し、最終終了値を`1`とする。
9. `-p`、`-m`、`--`、`-name`をオプション構文として解釈せず、通常のpathnameとしてOSへ渡す。
10. 標準エラーへの部分書き込みを完了まで再送し、書き込み失敗時に停止する。

## 7. 非対応機能

- `-p`による親directoryの再帰作成。
- `-m`によるmode指定、umask適用、権限検査。
- `--`などのオプション終端、ヘルプ・バージョン表示。
- glob、shell構文、環境変数、クォート処理。
- 失敗した一連の作成操作のtransactional rollback。
