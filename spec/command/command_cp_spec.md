# 極小UNIX `cp` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-CP-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの `/bin/cp` |
| 位置づけ | 通常fileをbyte単位で複製するユーザーランドコマンド |
| 依拠する仕様 | `open_close_implementation_spec.md`、`stat_implementation_spec.md`、`filesystem_design_spec.md`、syscall入口仕様 |

## 1. 目的と範囲

1つの通常fileの内容を別pathnameへ複製する。固定長bufferを使ったread/write、`O_CREAT`、`O_TRUNC`、close、失敗処理をユーザーランドから確認できるようにする。

初版は単一のsourceとdestinationに限定する。directory、recursive copy、複数source、標準入力/標準出力、sparse file最適化、permissionや時刻の保存、transactional rollbackは対象外とする。内容はbinary-safeであり、NUL・CR・LF・非ASCII byteを変更しない。

## 2. 起動形式

```text
cp source destination
```

- pathnameをちょうど2つ指定する。余分な引数や不足はusage errorとする。
- optionは提供しない。`-r`、`-p`、`-f`、`--`は特別扱いせずpathnameとして解釈する。
- `-`も標準入出力の別名ではなく通常pathnameである。`-`で始まるファイル名は`./-name`のように指定できる。
- 絶対pathと相対pathを受け付ける。相対pathは呼出し元Processのcwdを基準とする。
- pathname長はFilesystem仕様の上限に従い、超過時に切り詰めない。
- sourceとdestinationは異なるinodeを指さなければならない。同一pathnameまたはhard link等で同じinodeを指す場合は、destinationをtruncateする前に失敗させる。

## 3. file種別とmode

- sourceは`stat()`で通常file（`S_ISREG(st_mode)`）であることを確認する。directoryなど通常fileでないsourceは拒否する。
- destinationが既存の場合も通常fileに限る。既存directoryや他の未対応種別は拒否する。
- destinationが存在しない場合は新規作成する。
- sourceのpermission、所有者、時刻、link数をdestinationへ複製しない。
- 新規destinationのmodeは`0666`を指定する。現行Filesystemはumaskを適用しないため、作成modeの許可bitは`0666`となる。
- 既存destinationは`O_TRUNC`で内容を空にして再利用する。元のpermission bitはopen/truncate契約に従って維持される。

## 4. コピー手順

1. 引数数と各pathname文字列を検査する。
2. `stat(source, &src_stat)`を行い、sourceが通常fileであることを確認する。
3. `stat(destination, &dst_stat)`が成功し、`S_ISREG(dst_stat.st_mode)`かつ`st_dev`と`st_ino`がsourceと一致する場合、self-copyとして拒否する。この検査が終わる前にdestinationを作成・truncateしてはならない。
4. sourceを`open(source, O_RDONLY, 0)`で開く。
5. destinationを`open(destination, O_WRONLY | O_CREAT | O_TRUNC, 0666)`で開く。destinationが存在しない場合は作成し、存在する場合はtruncateする。
6. 固定長512-byte bufferを使い、sourceをEOFまでreadする。正のread結果は全byteをdestinationへ書き切る。短い正のreadはEOFとみなさず、次のreadを続ける。
7. source EOF後にsourceとdestinationの両方をcloseする。
8. 全read/write/closeが成功した場合のみ終了status 0とする。

- sourceを先にopenする。source openが失敗した場合、destinationを作成・truncateしてはならない。
- self-copyはpathname文字列比較だけで判定しない。`stat`で得た`st_dev`と`st_ino`を比較する。
- destination open後にread/writeが失敗しても、destination内容をrollback・復元しない。既にtruncateまたは部分更新された状態を許容し、診断と失敗statusを返す。
- 複数processによる同時pathname差し替えや同時書き込みに対するatomicity/snapshot保証は設けない。

## 5. byte転送と算術

- `read(fd, buffer, 512)`を繰り返す。戻り値が正ならそのbyte数をすべてdestinationへ送る。0はEOF、負値はread failureとする。
- `write()`が正の部分書き込みを返した場合は未送信範囲を再送する。0または負値はwrite failureとする。
- `write()`に渡す残量は、buffer内の実際のread byte数以下であることを検証する。signed/unsigned変換やindex加算でoverflowを起こさない。
- 入力全体をメモリへ保持しない。コピー量に固定上限を設けない。
- 空fileはdestinationを空にし、成功時は0を返す。

## 6. エラー処理・終了ステータス

| 状況 | 動作 |
|---|---|
| 引数数不正 | usage診断をfd=2へ出し、fileを開く前に`1`で終了 |
| sourceの`stat()`失敗、sourceが通常fileでない | 診断し、destinationを変更せず`1`で終了 |
| destinationの`stat()`でsourceと同一inodeを確認 | destinationを変更せずself-copy errorとして`1`で終了 |
| sourceの`open()`失敗 | 診断し、destinationを開かず`1`で終了 |
| destinationの`open()`失敗 | 診断し、sourceをcloseして`1`で終了 |
| `read()`失敗 | 診断し、コピーを停止、両方をcloseして`1`で終了 |
| `write()`失敗 | 診断し、コピーを停止、両方をcloseして`1`で終了 |
| `close()`失敗 | 診断し、他方のcloseも試し、最終statusを`1`とする |
| 全操作成功 | `0`で終了 |

- 診断は標準エラーfd=2へ`cp:`を付けて出す。厳密な文言・errno数値の形式は実装依存とする。
- 最初に発生した意味のあるfailureを保持し、cleanup中のclose failureで上書きしない。ただしそれ以前にfailureがなくcloseだけ失敗した場合は失敗として記録する。
- 診断writeが失敗しても追加診断や無限再試行をしない。
- `stat()`の失敗は現行ABIでは`-1`のみで詳細errnoを得られない。destinationが存在しない場合の判定はopen/create syscallに委ね、Filesystem内部statusをコマンドから参照しない。

## 7. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `stat()`、`S_ISREG()` | source/destination種別およびinode識別 |
| OSラッパ | `open()` | source読取とdestination作成/truncate |
| OSラッパ | `read()`、`write()` | byte転送と診断 |
| OSラッパ | `close()` | openしたFDの解放 |
| ilibc | `std::strcmp()`等 | option/path文字列判定（必要な場合） |
| ilibc | `std::strlen()`または既存format API | 診断出力 |
| ランタイム | `main()`の戻り値 | Process終了status |

- 動的メモリ確保、`FILE*`、`printf()`、host OS固有API、Filesystem内部APIを使わない。
- 固定長bufferを用い、read/write要求量はbuffer容量以下にする。
- すべてのopen成功FDを、通常経路・error経路を問わずcloseする。fd 0/1/2などコマンドが所有しないFDはcloseしない。
- `open()`のflagsは共通ヘッダ定義を使い、数値を重複定義しない。

## 8. 受入テスト

1. 引数不足・超過でusage errorとなり、file状態が変化しない。
2. 空fileをコピーしてdestinationが空になる。
3. 通常の小fileと複数cluster相当のfileをbyte単位で複製する。
4. NUL、CR、LF、非ASCII byteを含むfileが完全一致で複製される。
5. 既存destinationがtruncateされ、新内容だけが残る。
6. destinationが存在しない場合に`0666`で作成され、umask相当の変更が適用されない。
7. 同一pathname、相対/絶対別名、hard link aliasによるself-copyを検出し、元内容を壊さない。
8. directory source、directory destination、source不在、親directory不在を失敗として扱う。
9. source open失敗時はdestinationをtruncateしない。
10. read/writeの部分転送を正しく継続し、0/負値のwriteで停止する。
11. destination write失敗時に部分destinationをrollbackせず、両FDをcloseし、終了status 1となる。
12. close failureを検出し、先行するread/write failureをcleanup errorで上書きしない。

## 9. 非対応機能

- 複数source、directory tree、recursive copy。
- `-r`、`-p`、`-f`、`-i`、`--`、mode/owner/time保存。
- sparse file保持、atomic temporary-file replacement、失敗時rollback。
- 標準入力・標準出力をsource/destinationとする規則。
- 同時更新・pathname raceに対するsnapshotまたはtransaction保証。
