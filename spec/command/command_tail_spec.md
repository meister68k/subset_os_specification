# 極小UNIX `tail` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-TAIL-001 |
| 版 | 0.2（標準入力対応） |
| 対象 | サブセットOSの `/bin/tail` |
| 位置づけ | ファイル末尾の行またはbyte列を出力するコマンド。`lseek()`のユーザーランド動作試験を兼ねる |
| 依拠する仕様 | `lseek_implementation_spec.md`、`open_close_implementation_spec.md`、`stat_implementation_spec.md`、syscall入口仕様 |

## 1. 目的と範囲

標準入力または単一のregular fileから末尾の行、または末尾の指定byte数を標準出力へ出力する。pathname入力の`-c`動作では、`SEEK_END`でファイルサイズを取得し、`SEEK_SET`で出力開始位置へ移動してからreadする。これにより、通常ファイルのseek、read、共有file positionを保ちながらpipe入力にも対応する。

初版は1 pathnameまたは標準入力を扱う。複数file、directory、follow、出力行数指定、時間・サイズsuffix、ロケール依存処理を対象外とする。ファイル内容はテキストに限定せず、`-c`はbinary-safeとする。標準入力は固定長bufferに収まるサイズに制限する（§5）。

## 2. 起動形式

```text
tail [-c count] [file]
```

- `file`を省略した場合、標準入力fd=0から読む。
- `file`に`-`を指定した場合も標準入力fd=0から読む。それ以外はregular file pathnameとし、絶対pathと相対pathを受け付ける。相対pathは呼出し元Processのcwdを基準とする。
- 入力sourceは最大1つとする。`-c`は先頭optionとしてのみ認識し、その直後に10進非負整数`count`を要求し、pathnameは省略可能とする。
- `-c`を省略した場合は末尾10行を出力する。
- 余分な引数、不明な先頭オプション、または数値形式不正はusage errorとする。
- `--`、`-n`、`-f`、`+count`、複合option、数値suffix（`k`、`m`等）は非対応。
- `-`以外でoptionのように見えるfile名は`./-name`のように指定できる。
- pathnameの長さ上限はFilesystem pathname仕様に従い、途中で切り詰めない。

## 3. `-c count`の数値・出力規則

- `count`はASCII数字`0`〜`9`を1文字以上並べた10進非負整数とする。先頭の`+`/`-`、空白、suffixは認めない。先頭ゼロは許可する。
- 数値を蓄積する各桁でoverflowを検査する。`count`は`INT32_MAX`以下に制限し、超過値をwraparoundやnarrowingで受け入れない。
- `count == 0`なら出力byte数は0とする。pathname入力ではfileのopen、regular-file確認、`SEEK_END`および`SEEK_SET`の成功を確認する。標準入力ではEOFまで読み込むが出力はしない。
- pathname入力では`count > file_size`ならfile先頭から出力する。`count <= file_size`なら開始positionは`file_size - count`とする。標準入力では最後のcount byteを保持して出力する（§5）。
- `-c`は改行単位で処理しない。NUL、CR、LF、非ASCIIを含め、選択範囲のbyteを変更せずに出力する。
- `file_size`と開始positionは`lseek` syscallの`int32_t` position範囲内（0..`INT32_MAX`）でなければならない。これを超えるfileはseek失敗として扱う。

## 4. 通常の末尾10行

- 行はLF (`0x0A`) で区切る。出力時に改行を補ったり、CRを除去したりしない。
- fileが空なら何も出力しない。
- EOF直前がLFなら、そのLFは直前の行の終端であり、追加の空行として数えない。
- `end = lseek(fd, 0, SEEK_END)`を得た後、EOF直前のbyteがLFならそれを区切り数に含めず、残りの内容を後方へ走査する。LFを見つけるたびに行境界を数え、末尾LFありでは9個、末尾LFなしでは10個の境界を越えた位置を開始位置とする。必要数の境界より先にposition 0へ達した場合は開始位置を0とする。
- 各走査位置へ`lseek(fd, position, SEEK_SET)`して1 byte読み取る。seekはpositionを`int32_t`へ安全に変換できる範囲でのみ行う。end positionより前の走査中にreadが0または負値を返した場合は一貫した開始位置を決められないため失敗とする。
- 算出した開始位置へ`lseek(fd, start, SEEK_SET)`してからEOFまでreadし、取得したbyteをそのまま出力する。
- 10行に満たない場合はposition 0から出力する。
- 後方走査では、`SEEK_END`で得た位置から0までを境界確認し、位置計算で負値やsigned overflowを起こさない。1-byte read以外の結果（EOFまたはerror）は仕様に従って処理する。
- 巨大fileの全内容を固定長メモリへ保存しない。後方走査の作業領域は1 byteと固定長I/O bufferだけとする。

## 5. I/Oと`lseek()`契約

pathname入力:

1. `stat(file, &st)`で対象がregular fileであることを確認し、`open(file, O_RDONLY, 0)`で開く。
2. `lseek(fd, 0, SEEK_END)`を呼び、現在のEOF positionを取得する。
3. `-c`モードでは`start = max(0, end - count)`を検証済み整数演算で求める。行モードでは末尾10行の開始positionを後方走査で求める。
4. `lseek(fd, start, SEEK_SET)`を呼び、成功値が`start`と一致することを確認する。
5. 固定長bufferでreadを繰り返し、正のbyte数をstdoutへ全て書く。readが0ならEOF、負値ならerrorとする。短い正のreadはEOF扱いしない。
6. コマンドがopenしたFDを`close()`する。closeの成否も終了ステータスに反映する。

標準入力:

1. 標準入力fd=0からEOFまで逐次readし、pathname用の`stat()`、`open()`、`lseek()`は呼ばない。
2. 固定長の`TAIL_STDIN_CAPACITY` byte buffer（初版は512 byte）を使い、入力全体を保持する。容量を超えた入力はoverflowとし、以降のbyteをEOFまで読み捨ててから失敗する。容量超過時は不完全な末尾結果を出力しない。
3. EOF後、通常の行モードまたは`-c`規則で保持した入力の末尾部分を標準出力へ出力する。`count`が保持データ長以上なら保持データ全体を出力する。
4. `count == 0`でも標準入力をEOFまで読み取る。pipe writerを停滞させず、入力エラーや容量超過を検出するためである。
5. 標準入力fd=0はcloseしない。

- `lseek()`の成功値は新しいpositionであり、`-1`は失敗である。`-c`は最低限`SEEK_END`と`SEEK_SET`の双方を実行する。
- pathname入力に対し`SEEK_END`取得後にfileが他者から変更された場合のsnapshot保証はない。並行更新への特別な同期や再試行は行わない。
- `O_RDONLY`のregular fileでseekできることを検証する。seek結果・read位置の判定にFilesystem内部offsetやhost APIを使わない。

## 6. エラー処理・終了ステータス

| 状況 | 動作 |
|---|---|
| 引数形式・`count`不正 | usage診断をfd=2へ出し、file処理前に`1`で終了 |
| pathnameの`stat()`失敗、regular fileでない | 診断して`1`で終了 |
| pathnameの`open()`失敗 | 診断して`1`で終了 |
| pathname入力での`lseek()`失敗（position範囲外を含む） | 診断して`1`で終了 |
| 標準入力が`TAIL_STDIN_CAPACITY`を超過 | EOFまで読み捨て、不完全結果を出力せず`1`で終了 |
| `read()`失敗 | 診断して`1`で終了。標準入力の場合は末尾を確定できないため出力しない |
| stdoutの`write()`が0または負値 | 出力を中断し、`1`で終了 |
| stdoutへの正の部分write | 未送信分を繰り返し書く |
| `close()`失敗 | 診断し、最終終了ステータスを`1`にする |
| 全処理成功 | `0`で終了 |

- 診断は標準エラーfd=2へ`tail:`を付けて出す。厳密な文言・errno数値の表示形式は実装依存とする。
- 診断writeが失敗しても再帰的な診断や無限再試行をしない。
- 失敗時にも、open済みのfile descriptorは必ずcloseを試みる。既存のfd 0/1/2をcloseしない。
- syscallは`-1`をエラーとして返す現行ABIに従う。`lseek()`失敗をEOFやposition 0と誤認しない。

## 7. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `stat()`、`S_ISREG()` | pathname入力の対象種別確認 |
| OSラッパ | `open()`、`lseek()`、`read()`、`close()` | pathname操作とseek、入力読込み |
| OSラッパ | `write()` | stdoutおよび診断 |
| ilibc | `std::strcmp()`等 | `-c` option識別 |
| ilibc | `std::strlen()`または既存format API | 診断 |
| ランタイム | `main()`の戻り値 | Process終了status |

- 動的メモリ確保、`FILE*`、`printf()`、ホストOS固有API、Filesystem内部APIを使用しない。
- I/O bufferと標準入力保持領域は固定長とし、1回のread要求量は`INT32_MAX`以下とする。標準入力保持領域`TAIL_STDIN_CAPACITY`は512 byteである。
- byte数、position、出力範囲の加算・減算前に範囲を検査し、符号付きoverflowを起こさない。
- `-c`の計算で`end - count`を先に負値にしない。`end > count`を確認してから減算する。

## 8. 受入テスト

1. pathnameと標準入力の両方で、既定モードの10行未満・ちょうど10行・10行超を扱う。
2. EOFがLFで終わるfileと、改行なしで終わるfileの末尾行を確認する。
3. 空fileはstdoutを空にして成功する。
4. `tail -c 0`は出力0 byte、`tail -c 1`は入力末尾1 byteを出力する。stdin時にもEOFまでdrainする。
5. `count`がfile長未満、同じ、超過の各場合に開始位置と出力byte列が正しい。
6. NUL、CR、LF、非ASCII byteを含むfileで`-c`がbyte列を変更しない。
7. `-c`で`SEEK_END`後に計算した開始位置へ`SEEK_SET`し、そのpositionからreadすることを確認する。
8. `count`の先頭ゼロ、0、`INT32_MAX`、`INT32_MAX`超過、非数字、負数、空文字を検証する。
9. directory、pathnameの`stat/open/lseek/close`失敗、stdinを含む`read`失敗を検証し、終了値1となる。`lseek`失敗はEOFやposition 0と誤認しない。
10. 正の部分writeを再送し、writeの0/負値で出力を中断する。
11. 読み取り・seekによるfile position変化が同一open descriptionの規則に従い、close後にFD資源が残らない。
12. 標準入力が512 byte以内なら末尾を正しく出力し、513 byte以上ならEOFまでdrainした後に出力なし・status 1となる。

## 9. 非対応機能

- `-n`による行数指定（既定の10行のみ）。
- 複数file。
- `-f` follow、`-r` reverse、出力の書式加工。
- count suffix、`+count`、任意のseek origin。
- 同時更新されるfileのsnapshotやtail-follow同期。
