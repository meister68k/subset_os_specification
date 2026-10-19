# 極小UNIX `rm` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-RM-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの `/bin/rm` |
| 位置づけ | 非directory pathnameを`unlink()`で削除するユーザーランドコマンド |
| 依拠する仕様 | `unlink_implementation_spec.md`、`filesystem_design_spec.md`、syscall入口仕様 |

## 1. 目的と範囲

指定された各pathnameのdirectory entryを削除する。初版は`unlink()` syscallの薄いユーザーランド利用例とし、recursive deleteや対話確認は提供しない。

`rm`は対象pathnameの解決、inode種別判定、link count更新、open中にunlinkされたfileの寿命管理を独自に実装しない。これらはOSの`unlink()`とFilesystem契約に委ねる。directoryは削除せず、directoryの削除には`rmdir`を使う。

## 2. 起動形式

```text
rm file ...
```

- `file`を1つ以上指定する。各pathnameを引数順に独立して処理する。
- optionは提供しない。`-r`、`-R`、`-f`、`-i`、`--`は特別扱いせずpathnameとして渡す。
- `-`で始まるpathnameも通常pathnameとして扱う。例えば`-data`という名前は`./-data`のように指定できる。
- 絶対pathと相対pathを受け付ける。相対pathは呼出し元Processのcwdを基準とする。
- 引数なしは使用方法エラーとする。
- 空pathnameは拒否され、OSの`EINVAL`相当の失敗となる。
- pathnameはFilesystem pathname仕様の上限内でなければならない。`PATH_MAX`は128バイト、`NAME_MAX`は27バイトであり、component上限は文字数ではなくbyte数で判定する。
- glob、クォート、環境変数は展開しない。shellが生成した`argv`をそのまま使う。

## 3. 削除契約

- 各引数について`unlink(path)` syscallを1回呼ぶ。
- `stat()`による事前検査、`open()`による一時FD作成、削除前の読み込みは行わない。対象がdirectoryかどうか等の検証はOS側が行う。
- 成功した場合はpathnameに対応するdirectory entryを1つ削除する。hard linkが複数あるfileでは、他の名前が残る限りfile内容とinodeは残る。
- open中のfileをunlinkしても、既存FDを閉じたり参照を変更したりしない。open参照が残る間は既存FDからの操作が継続できる。最後のnameがなくなった後のfinalize時期はOSのorphan lifetime契約に従う。
- directory（rootを含む）は`unlink()`で削除しない。OSが失敗を返し、空directoryも`rmdir`でのみ削除できる。
- `.`、`..`、root、末尾slashを含むpathnameの扱いは`unlink_implementation_spec.md`とpathname仕様に従う。
- ある引数の失敗で他の引数の処理を中断しない。成功した削除を後続の失敗に伴って復元しない。

## 4. 出力・エラー・終了ステータス

- 成功時は標準出力へ何も出さない。
- 失敗時は標準エラーfd=2へ`rm:`で始まる診断を出し、失敗したpathnameを示す。厳密な文言およびerrno数値の表示形式は実装依存とする。
- 診断writeが失敗しても追加診断や無限再試行をしない。
- 全pathnameを処理し、失敗が1件でもあった場合は最終終了ステータスを`1`とする。全件成功時は`0`とする。
- 終了は`main()`の戻り値をランタイムがProcess終了へ変換する既定経路を使う。

| 状況 | 動作 |
|---|---|
| 引数なし | 使用方法を診断し、`1`で終了 |
| pathname削除成功 | 無出力で次の引数へ |
| 対象なし、directory、path不正、I/O/Filesystem failure | 診断し、次の引数へ |
| 全引数成功 | `0`で終了 |
| 1件以上失敗 | 全引数処理後に`1`で終了 |

OS側の代表的な失敗には`ENOENT`、`ENOTDIR`、`EISDIR`、`EINVAL`、`ENAMETOOLONG`、Filesystem未mount時の`ENODEV`、破損時の`ECORRUPT`がある。syscall ABIは失敗を`-1`へ変換し、global errnoを追加しない。`rm`はFilesystemの`FsStatus`を直接扱わない。

## 5. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `unlink(path)` | pathname entry削除 |
| OSラッパ | `write()` | 標準エラーへの診断 |
| ilibc | `std::strlen()`または既存format API | pathname表示・診断 |
| ランタイム | `main()`の戻り値 | Process終了status |

- `argv`を順次処理し、動的メモリ確保を行わない。
- `FILE*`、`printf()`、Filesystem内部API、host OS固有の`unlink()`を使用しない。
- `write()`が正の部分書き込みを返した場合は残りを再送する。0または負値で停止する。
- `unlink()`失敗後に独自の`rmdir()`や別のFilesystem APIへfallbackしない。
- 既存FDをcloseせず、FileDescription、FD table、file positionを変更しない。

## 6. 受入テスト

1. 引数なしで診断し、終了ステータス`1`となる。
2. 通常fileを削除し、pathnameが解決できなくなることを確認する。
3. 複数pathnameを引数順に処理する。
4. 相対pathと絶対pathがcwd/root基準で解決される。
5. hard linkの一方を削除しても他方から同じ内容へ到達できる。
6. fileをopenしたままunlinkしても、既存FDからの読み書き・seekがOS契約どおり継続する。
7. 存在しないpathname、directory、root、`.`、`..`、空path、途中non-directoryを失敗として扱う。
8. `PATH_MAX` / `NAME_MAX`の境界と超過をOS契約どおり扱う。
9. 一つのpathnameが失敗しても後続を処理し、いずれかが失敗した場合は最終値`1`となる。
10. `-r`、`-R`、`-f`、`-i`、`--`をoptionとして処理せず、pathnameとして`unlink()`へ渡す。
11. 診断の部分writeを再送し、writeの0/負値で停止する。

## 7. 非対応機能

- directoryの削除、再帰削除（`-r`、`-R`）。空directoryも`rmdir`を使う。
- 強制削除・missing path無視（`-f`）。
- 確認プロンプト（`-i`）。
- `--`によるoption終端、help/version表示。
- glob、shell構文、環境変数、クォート処理。
- 複数pathnameをまとめたtransaction、rollback。
