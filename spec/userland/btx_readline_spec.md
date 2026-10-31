# `btx_readline.h` 仕様書

## 1. 目的と範囲

本書は、BaTTaXユーザープログラムが標準入力から1行を読み取る共通関数 `btx::readline()` を定める。初版は固定長バッファを使い、TTY入力ではBSおよびDELによる末尾文字削除を提供する。ConsoleDeviceの共有echoは呼び出し中だけ抑制し、readline自身が編集表示と入力echoを行う。非TTY入力では入力バイトを編集せず、echoせず、LFまでを1行として返す。

実装は暫定的に新設する `btx_readline.h` のヘッダー内関数とし、`btx_unistd.h` は変更しない。syscallラッパー、`ioctl()`、`isatty()`の仕様は [`ioctl_implementation_spec.md`](../kernel/ioctl_implementation_spec.md) に従う。共通文字列関数やilibcの `gets()` の動作は変更しない。

## 2. ユーザーAPI

```cpp
namespace btx {

enum class ReadlineStatus : int32_t {
    error = -1,
    line = 0,
    eof = 1,
    too_long = 2,
};

static inline ReadlineStatus readline(char* buffer, uint32_t capacity, uint32_t* length);

}
```

- `buffer` は読み取り結果を書き込める領域、`capacity` はその領域のバイト数、`length` はLFを除く本文のバイト数の出力先である。
- `capacity` は終端NULを含む。成功時、本文の後ろにNULを1 byte書く。
- `buffer` が指す `capacity` byteの領域と、`length` が指す `uint32_t` objectは重なってはならない。これは呼び出し側の前提条件であり、実行時には検出しない。
- `length != nullptr` なら呼び出し開始時に `*length` を0にする。`buffer != nullptr && capacity > 0` なら `buffer[0]` をNULにする。各ポインタが指す出力先は書込み可能でなければならない。
- `buffer == nullptr`、`length == nullptr`、または `capacity == 0` は不正引数としてsyscallを呼ばず `ReadlineStatus::error` を返す。不正引数時も、上記条件を満たす個別の出力先はそれぞれ初期化する。`capacity == 0` の場合はbufferへ書き込まない。アクセス可能範囲や書込み可能性を実行時に検出する機能は持たない。
- `capacity == 1` は有効であり、空行は格納できる。本文バイトを1つでも受け取ると行長超過となる。

## 3. 戻り値

| 戻り値 | 意味 | `buffer` / `*length` |
|---|---|---|
| `line` | LFまたはEOFで行を取得した | NUL終端した本文と、本文バイト数 |
| `eof` | 本文を1 byteも受け取る前にEOF | 空文字列、長さ0 |
| `too_long` | 本文が `capacity - 1` byteを超えた | 空文字列、長さ0。超過行はLFまたはEOFまで消費済み |
| `error` | 不正引数またはread/write syscall失敗 | 有効な出力先は空文字列、長さ0 |

LF直前までに1 byte以上を受け取っていれば、LFなしEOFでも `line` とする。空行のLFは `line`、長さ0であり、EOFとは区別できる。失敗理由をerrnoへ保存しない。

## 4. 入出力先と依存

- 入力には `btx::sys_read(0, &byte, 1)` を使用する。
- 出力には `btx::sys_write(1, data, count)` を使用する。
- `btx::isatty(0)` が真の場合にTTY編集モードとする。偽の場合は非TTYモードとする。標準出力側のTTY状態は判定しない。
- TTYモードでは `BTX_TIOCGETECHO` でConsoleDeviceの共有echo状態を取得する。取得失敗時はsys_readを呼ばず `error` を返す。
- 取得状態がONなら `BTX_TIOCECHOOFF` でreadline中のdevice echoを止める。readlineは通常byteをechoし、BS/DELには `BS SP BS` を出力する。初期状態がOFFならdevice状態を変えず、readline自身のechoは同様に行う。
- 成功、EOF、too_long、入力/echo errorのいずれでも、初期状態がONだった場合は `BTX_TIOCECHOON` で復元する。復元要求失敗時は出力を空にして `error` を返す。初期状態がOFFならOFFのままにする。
- TTY判定に失敗したときも `btx::isatty()` の契約により非TTYモードとして読み取りを試みる。実際の入力エラーは `sys_read()` の失敗で報告する。
- ヘッダーは `btx_unistd.h` を取り込み、`btx::sys_read()`、`btx::sys_write()`、`btx::isatty()`、グローバル `ioctl()` を利用できることを前提とする。
- 動的メモリ確保、グローバル可変状態、先読み用の内部バッファは使用しない。入力状態は呼び出し間で保持しない。

`sys_read()` は1 byte要求に対し、戻り値1を1 byte取得、0をEOF、負値をエラーとして扱う。要求を超える正の戻り値はsyscall契約外として `error` とする。`sys_write()` は要求範囲全体が出力されるまで部分書込みを繰り返す。0または負値、あるいは要求を超える戻り値があれば `error` とする。既に読み取った入力や一部出力は巻き戻せない。

## 5. 行の終端と通常文字

- LF (`0x0A`) だけを行終端とする。LFは本文に格納せず、TTYモードではLFを標準出力へ出力する。
- CR (`0x0D`) は通常の本文バイトであり、削除・変換しない。CRLFを1つの行終端として扱わない。
- NUL (`0x00`) はテキスト入力契約の対象外とし、出力文字列として扱わない。入力にNULが現れた場合は `error` とし、その行の残りをLFまたはEOFまでechoせず読み捨てる。LFで破棄を終えた場合、TTYではLFをechoする。
- BS (`0x08`) とDEL (`0x7F`) 以外の制御文字に特別な編集動作を割り当てない。通常の本文byteとして格納し、TTYモードでは同じbyteを出力する。Ctrl-Cによる中断、Ctrl-Uによる行全体削除、カーソル移動、挿入編集は提供しない。

## 6. TTY編集動作

TTYモードでは入力byteごとに次の処理を行う。

1. LFなら行を確定し、LFを標準出力へ出して `line` を返す。
2. BSまたはDELなら、本文が空でない場合に末尾byteを1つ削除し、画面上の削除表示として `BS`, `SP`, `BS` の3 byteを標準出力へ出す。本文が空なら何もしない。
3. その他の本文byteは容量内でバッファへ追加し、同じbyteを標準出力へ出す。

この方式はバイト単位の末尾削除であり、UTF-8等の文字コード境界を解釈しない。表示上の1文字が複数byteである場合でも、BS/DELはbyteを1つ削除する。

## 7. 非TTY入力

非TTYモードではBSおよびDELを含む入力byteを特別扱いせず、本文へそのまま格納する。標準出力へ入力内容や改行をechoしない。LFのみを行終端として除外する。

## 8. 容量超過

本文byte数が `capacity - 1` に達した状態でさらに本文byteを受け取った場合、行を超過扱いにする。

- `too_long` を返す前に、残りの入力をLFまたはEOFまで読み捨てる。次の呼び出しが同じ行の残部から始まらないようにする。
- 超過行ではBS/DELによる編集を受け付けない。TTYでは超過判定後の通常byteをechoしない。
- LFで行を破棄した場合、TTYでは終端LFをechoする。
- 入力エラーまたはecho出力エラーが読み捨て中に発生した場合は `error` を返す。
- 呼び出し元は `too_long` の内容を解析・実行してはならない。

## 9. エラーと出力の不変条件

- `sys_read()` が0を返した場合はEOFとして扱う。部分行がある場合はその行を返し、空のときは `eof` を返す。
- `sys_read()` またはecho用 `sys_write()` が失敗した場合は `error` を返す。
- `too_long`、`eof`、`error` の出力文字列は空とし、`*length` は0とする。通常の `line` 以外を呼び出し元がコマンドとして解析してはならない。
- syscall失敗時に既に消費・表示したbyteを復元しない。
- `readline()` は入力fd 0や出力fd 1をclose、dup、変更しない。

## 10. 確認項目

1. TTY上で通常byteを入力すると、入力順の本文、NUL終端、正しい長さが得られ、ConsoleDevice echoの二重出力なしで入力byteがechoされる。
2. TTY上でBSとDELがそれぞれ末尾1 byteを削除し、削除表示列 `BS SP BS` が出力される。空行上のBS/DELは無操作となる。
3. TTY上でLFは本文に含まれずechoされ、空行とEOFが戻り値で区別される。
4. CRは本文に保持され、CRLFはCRを残したままLFで行を終える。
5. 非TTY上ではBS/DELを含むbyteを保持し、入力byteをechoしない。
6. `capacity == 1`、ちょうど `capacity - 1` byteの本文、1 byte超過を確認する。
7. 超過行はLF/EOFまで消費され、次の呼び出しが次行の先頭から始まる。超過内容は返さない。
8. 部分行後のEOFは `line`、入力前EOFは `eof` となる。
9. NUL入力、readエラー、writeエラー、部分write、writeの0戻り値で規定の結果となる。
10. 不正引数はsyscallを呼ばず `error` となり、非nullの `length` は0、容量が正で非nullの `buffer` は空文字列に初期化される。`buffer` と `length` の重複は呼び出し前提違反として扱う。
11. 1 byte要求の `sys_read()` が2以上を返す場合は `error` となる。
12. echo初期状態ON/OFFの双方で、readline終了後に元のConsoleDevice echo状態が復元/維持される。
