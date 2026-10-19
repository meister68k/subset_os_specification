# 極小UNIX `rmdir` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-RMDIR-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの `/bin/rmdir` |
| 位置づけ | 空directoryをpathnameで削除するユーザーランドコマンド |
| 依拠する仕様 | `mkdir_rmdir_implimentation_spec.md`、`filesystem_design_spec.md`、syscall入口仕様 |

## 1. 目的と範囲

指定された空directoryの名前を削除する。directory内容を再帰的に削除する機能、オプション解析、pathnameの独自解決は行わない。

このコマンドはユーザーランドから`rmdir(path)` syscallを呼ぶ。相対pathは呼出し元Processのcwd、絶対pathはrootを基準としてOS側で解決する。directoryの空判定、direntとlink countの更新、open/cwd参照があるdirectoryの寿命管理はOS/Filesystem側の責務とする。

## 2. 起動形式

```text
rmdir directory ...
```

- `directory`を1つ以上指定する。各pathnameを引数順に独立して処理する。
- オプションは提供しない。`-p`、`--`は特別扱いせず、pathnameとして渡す。
- `-`で始まるpathnameも通常のpathnameとして扱う。
- 絶対pathと相対pathを受け付ける。
- 引数なしは使用方法エラーとする。
- 空pathnameは拒否され、OSの`EINVAL`相当の失敗となる。
- 各pathnameはFilesystem仕様の`PATH_MAX`（128バイト未満のNUL終端文字列）以内でなければならない。name componentは`NAME_MAX`（27バイト）以内とする。上限は文字数ではなくバイト数で判定する。

## 3. 削除契約

- 各引数について`rmdir(path)` syscallを1回呼び出す。
- 指定対象が存在し、directoryであり、OSが定義する空directory条件を満たす場合に限り削除を試みる。
- `rmdir`は再帰削除をしない。対象directoryに通常fileまたは子directory等の有効entryがあれば失敗する。`.`と`..`は空判定に必要なdirectory entryとして扱われる。
- `.`または`..`を最終componentに指定した場合は失敗する。root directoryの削除も許可しない。
- 対象directoryがopen中またはProcessのcwdであっても、ユーザーランド側で参照状態を検査・変更しない。削除可否と削除後のinode寿命はOS側の`rmdir()`契約に従う。
- 一つのpathnameが失敗しても残りのpathnameを処理する。成功した削除は後続の失敗時に復元しない。

## 4. 出力・エラー・終了ステータス

- 成功時は標準出力へ何も出さない。
- 失敗時は標準エラーfd=2へ`rmdir:`で始まる診断を出す。pathnameと、利用可能ならエラー内容を含める。厳密な文言とerrno数値の表示形式は規定しない。
- 診断出力自体が失敗しても追加の再試行や無限ループを起こさない。
- 失敗したpathnameを記録し、後続の引数を処理する。
- 全引数成功時は終了ステータス`0`、引数不足または一件以上の失敗時は`1`とする。
- 終了は`main()`の戻り値をランタイムがProcess終了へ変換する既定経路を使う。

| 状況 | 動作 |
|---|---|
| 引数なし | 使用方法を診断し、`1`で終了 |
| directory削除成功 | 無出力で次のpathnameへ |
| 対象なし | 診断し、次のpathnameへ |
| 通常fileまたは非空directory | 診断し、次のpathnameへ |
| root、`.`、`..`などOSが拒否するpathname | 診断し、次のpathnameへ |
| path不正、I/OまたはFilesystemエラー | 診断し、次のpathnameへ |
| 全pathname成功 | `0`で終了 |
| 一件以上失敗 | 全pathname処理後に`1`で終了 |

OS側仕様で定める代表的な失敗は、存在しない対象の`ENOENT`、通常fileの`ENOTDIR`、非空directoryの`ENOTEMPTY`、rootの`EBUSY`、`.`/`..`の`EINVAL`である。途中component不正、長さ超過、破損、I/O失敗等の詳細は`mkdir_rmdir_implimentation_spec.md`とpathname仕様に従う。コマンドはFilesystemの`FsStatus`を直接扱わない。

## 5. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `rmdir(path)` | directory削除 |
| OSラッパ | `write()` | 標準エラーへの診断 |
| ilibc | `std::strlen()`または既存書式出力 | pathname長・診断出力 |
| ランタイム | `main()`の戻り値 | Process終了ステータス |

- `argv`を順次処理し、動的メモリ確保を行わない。
- `FILE*`や`printf()`、Filesystem内部API、ホストOS固有の`rmdir`を使用しない。
- `write()`が部分書き込みを返した場合は残りを再送する。0または負値で停止し、診断の失敗を再帰的に診断しない。
- 削除失敗後に別APIで対象名やinodeを変更しない。回復・rollbackはOS/Filesystemの契約に委ねる。

## 6. 受入テスト

1. 引数なしで診断し、終了ステータス`1`となる。
2. 空directoryを削除し、標準出力が空、終了ステータスが`0`となる。
3. 複数pathnameを指定し、全てを引数順に処理する。
4. 相対pathと絶対pathがcwd/root基準で処理される。
5. 存在しない対象、通常file、非空directory、root、`.`、`..`を失敗として扱う。
6. `PATH_MAX`と`NAME_MAX`境界、および一つ超過した値をOS契約どおり扱う。
7. 一つのpathnameが失敗しても後続pathnameを処理し、最終終了値を`1`とする。
8. `-p`、`--`、`-name`をオプションとして解釈せず、pathnameとしてOSへ渡す。
9. OSの`rmdir`仕様どおり、open/cwd参照がある対象の削除後も参照元からの利用と最終解放が整合する。
10. 標準エラーへの部分書き込みを完了まで再送し、書き込み失敗時に停止する。

## 7. 非対応機能

- `-p`による親directoryの連鎖削除。
- `-r`、`-R`その他による再帰削除。
- `--`によるオプション終端、ヘルプ・バージョン表示。
- glob、shell構文、環境変数、クォート処理。
- 複数pathnameに対する一括transactionまたはrollback。
