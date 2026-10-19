# 極小UNIX `head` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-HEAD-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの `/bin/head` |
| 位置づけ | fileの先頭行または先頭byte列を出力するコマンド |
| 依拠する仕様 | `open_close_implementation_spec.md`、`stat_implementation_spec.md`、syscall入口仕様 |

## 1. 目的と範囲

単一のregular fileから先頭の行、または先頭の指定byte数を標準出力へ出力する。既定動作では先頭10行を、`-c`指定時は先頭からcount byteを出力する。固定長bufferによるread/write、EOF、終了コードを検証できる小さなユーザーランドコマンドとする。

初版は1 pathnameのみを扱い、標準入力、複数file、pipe、directory、follow、行数指定、時間・サイズsuffix、ロケール依存処理を対象外とする。ファイル内容はテキストに限定せず、`-c`はbinary-safeとする。

## 2. 起動形式

```text
head [-c count] file
```

- `file`は必須のregular file pathnameである。絶対pathと相対pathを受け付ける。相対pathは呼出し元Processのcwdを基準とする。
- `-c`は先頭optionとしてのみ認識し、その直後に10進非負整数`count`、続けてpathnameを要求する。
- `-c`を省略した場合は先頭10行を出力する。
- 引数不足、余分な引数、不明な先頭option、またはcountの形式不正はusage errorとする。
- `--`、`-n`、`-q`、`-v`、`+count`、複合option、数値suffix（`k`、`m`等）は非対応。
- `file`が`-`でも標準入力を意味せず、通常のpathnameとして扱う。optionのように見えるfile名は`./-name`のように指定できる。
- pathname長はFilesystem pathname仕様に従い、途中で切り詰めない。

## 3. `-c count`の数値・出力規則

- `count`はASCII数字`0`〜`9`を1文字以上並べた10進非負整数とする。`+`/`-`、空白、suffixは認めず、先頭ゼロは許可する。
- 10進値の蓄積時に各桁のoverflowを検査する。`count`の最大値は`INT32_MAX`とし、overflowやnarrowingで受け入れない。
- `count == 0`なら出力は0 byteとする。対象確認とopenは行うが、データreadは不要とする。
- fileの先頭から`min(count, file size)` byteをそのまま出力する。file sizeを事前取得する必要はなく、count分を出力した時点またはEOFで停止する。
- NUL、CR、LF、非ASCIIを含め、内容の変換、行境界処理、改行追加をしない。

## 4. 既定の先頭10行

- 行区切りはLF (`0x0A`) 1 byteとする。CRを削除したり、出力に改行を補ったりしない。
- EOFまでにLFが10個以上あれば、10個目のLFまでを出力する。10個未満ならEOFまでを出力する。
- 空fileは何も出力しない。改行なしの最終行があれば、そのbyte列を出力する。
- 入力は固定長bufferで複数回readしてよい。各buffer内でLFを数え、10個目のLFの直後までを出力対象とする。bufferに後続byteが含まれていても10行目より後は書き出さない。
- 入力を全てメモリへ蓄積しない。作業領域は固定長read bufferと必要最小限の走査状態に限る。

## 5. I/O契約

1. `stat(file, &st)`で対象がregular fileであることを確認する。
2. `open(file, O_RDONLY, 0)`で読み取り専用に開く。
3. `-c`モードは残りbyte数を管理し、各read要求を`min(buffer_capacity, remaining)`以下にする。正のread数だけremainingを減らす。
4. 既定行モードはLFを走査し、10個目のLFまでのprefixだけを標準出力へ送る。
5. 正の部分writeは未送信分を再送する。readが0ならEOF、readが負値なら失敗とする。短い正のreadをEOFとみなさない。
6. コマンドが開いたFDを必ず`close()`する。close失敗も最終終了ステータスへ反映する。

`head`は`lseek()`を使用しない。`-c`は先頭から逐次readするためseek不能な入力にも内部的には対応できるが、初版のCLIではregular fileのpathnameだけを受け付ける。

## 6. エラー処理・終了ステータス

| 状況 | 動作 |
|---|---|
| 引数形式・count不正 | usage診断をfd=2へ出し、file処理前に`1`で終了 |
| `stat()`失敗、regular fileでない | 診断して`1`で終了 |
| `open()`失敗 | 診断して`1`で終了 |
| `read()`失敗 | 診断し、出力を中断して`1`で終了 |
| stdoutの`write()`が0または負値 | 出力を中断して`1`で終了 |
| stdoutへの正の部分write | 残りを繰り返し書く |
| `close()`失敗 | 診断し、最終終了ステータスを`1`にする |
| 全処理成功 | `0`で終了 |

- 診断は標準エラーfd=2へ`head:`を付けて出す。文言とerrno数値の表記は実装依存とする。
- 診断write失敗時に追加診断や無限再試行をしない。
- 失敗時にもopen済みFDはcloseを試みる。既存のfd 0/1/2をcloseしない。
- syscall失敗値`-1`をエラーとして扱い、0 byte成功と混同しない。

## 7. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `stat()`、`S_ISREG()` | 対象種別確認 |
| OSラッパ | `open()`、`read()`、`close()` | file入力 |
| OSラッパ | `write()` | stdoutと診断 |
| ilibc | `std::strcmp()`等 | `-c` option識別 |
| ilibc | `std::strlen()`または既存format API | 診断 |
| ランタイム | `main()`の戻り値 | Process終了status |

- 動的メモリ確保、`FILE*`、`printf()`、ホストOS固有API、Filesystem内部APIを使わない。
- I/O bufferは固定長（512 byteを目安）とし、read要求サイズは`INT32_MAX`以下にする。
- byte数、buffer index、出力範囲の演算は境界検査し、overflowや範囲外アクセスを起こさない。
- `-c`のremainingは非負の検証済み値で管理し、read結果を検査してから減算する。

## 8. 受入テスト

1. 既定動作で10行未満・ちょうど10行・10行超のfileを扱う。
2. 空file、LFで終わるfile、改行なしの最終行を確認する。
3. 10行目がbuffer境界をまたぐ場合も、10行目のLFまで正確に出力する。
4. `head -c 0 file`は0 byte、`head -c 1 file`は先頭1 byteを出力する。
5. countがfile長未満、同じ、超過の各場合に、先頭から正しいbyte列を出力する。
6. `-c`でNUL、CR、LF、非ASCIIを含む入力をbyte単位で保持する。
7. 先頭ゼロ、`INT32_MAX`、上限超過、空文字、符号付き値、非数字を検証する。
8. directory、`stat/open/read/close`失敗を検証し、終了値1となる。
9. 正の部分writeを再送し、writeの0/負値で出力を中断する。
10. 途中read失敗後もFDをcloseし、close失敗も終了値へ反映する。

## 9. 非対応機能

- `-n`による行数指定（既定の10行のみ）。
- 標準入力、複数file、`-`を標準入力とする規則。
- `-q`、`-v`、follow、逆順表示。
- count suffix、`+count`、`--`。
- 同時更新されるfileに対するsnapshot保証。
