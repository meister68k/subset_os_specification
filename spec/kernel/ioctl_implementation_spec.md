# `ioctl()` / `isatty()` 実装仕様書

## 1. 目的と範囲

本書は、ファイル記述子に対する限定的な `ioctl()` syscall と、それを用いる `isatty()` ユーザーランド関数を定める。初版は端末判定用 `BTX_TIOCISATTY` とConsoleDeviceの共有入力echoを取得・設定する要求を実装する。汎用の可変長引数、端末属性の取得・変更、`sgttyb` / `termios` 等の構造体は導入しない。

syscall ABI・番号管理は [`syscall_entry_spec.md`](syscall_entry_spec.md) と [`syscall_generator_spec.md`](syscall_generator_spec.md)、fdから対象への委譲は [`open_file_description_spec.md`](open_file_description_spec.md) に従う。

## 2. syscall ABI

syscall番号54を `ioctl` に割り当てる。これはV7の `ioctl` 番号を維持し、要求セットを本OS向けに限定した採用である。

```cpp
int32_t sys_ioctl(int32_t fd, uint32_t request, void* argp);
```

- 通常の成功は `0`、失敗は `-1`。`BTX_TIOCGETECHO` は例外としてecho ONなら`1`、OFFなら`0`を返し、失敗時は`-1`とする。errnoやグローバルエラー状態は追加しない。
- `fd` は現在のProcessのfd tableから解決する。
- 初版で有効な要求は `BTX_TIOCISATTY`、`BTX_TIOCGETECHO`、`BTX_TIOCECHOON`、`BTX_TIOCECHOOFF` のみである。
- `BTX_TIOCISATTY` の `argp` は `nullptr` でなければならない。非nullなら失敗する。カーネルはこの要求でポインタを参照しない。
- echo状態のGET/SET要求も `argp == nullptr` を要求し、echo値はrequestコードと戻り値で表す。ユーザーポインタは参照しない。
- 未知の要求、無効なfd、要求をサポートしない対象は失敗し、状態を変更しない。
- `request` は `uint32_t`、`argp` はポインタとしてABI境界を通し、ホストの整数幅や可変長引数に依存しない。

要求コードはユーザー向けヘッダーで次の `uint32_t` 定数として定義する。V7の要求番号との互換性は主張しない。

```cpp
constexpr uint32_t BTX_TIOCISATTY = 0x42540001u;
constexpr uint32_t BTX_TIOCGETECHO = 0x42540002u;
constexpr uint32_t BTX_TIOCECHOON = 0x42540003u;
constexpr uint32_t BTX_TIOCECHOOFF = 0x42540004u;
```

## 3. ユーザーランドAPI

システムコールABIの正本である `unistd.h` の `SYSTEMCALL` 範囲に、番号54の型付きラッパーを置く。

```cpp
SYSTEMCALL int32_t sys_ioctl(int32_t fd, uint32_t request, void* argp)
    {return system_call(54, fd, request, argp);}
```

同ヘッダー内で、`sys_ioctl()` の上位に固定引数のグローバル `ioctl()` と
`btx` 名前空間の `isatty()` を定義する。両方ともsyscall generatorの入力対象外とする。

```cpp
static inline int32_t ioctl(int32_t fd, uint32_t request, void* argp)
{
    return btx::sys_ioctl(fd, request, argp);
}

namespace btx {
static inline int isatty(int32_t fd)
{
    return ::ioctl(fd, BTX_TIOCISATTY, nullptr) == 0 ? 1 : 0;
}
}
```

- `ioctl()` は初版では固定3引数とし、C/POSIX形式の可変長引数にはしない。
- C++の `btx::isatty(fd)` は `ioctl(fd, BTX_TIOCISATTY, nullptr)` が成功すれば `1`、失敗すれば `0` を返す。
- C翻訳単位には名前空間がないため、同じ固定引数 `isatty()` wrapperをグローバル名で提供する。
- C++ではグローバル `isatty()` を定義せず、`btx::isatty()` のみを提供する。
- 無効fdと非端末の区別は行わず、失敗理由をerrnoへ保存しない。
- `isatty()` は `unistd.h` 内のユーザーランド関数として提供し、カーネル内に同名関数や独立syscallを設けない。

C翻訳単位ではC++の名前空間を使わず、`ioctl()` は `sys_ioctl()` へ直接委譲し、`isatty()` はその結果を1/0へ変換するグローバルwrapperとする。

## 4. カーネル内の委譲

処理経路は次の通りとする。

```text
btx::sys_ioctl(fd, request, argp)
  -> syscall dispatcher
  -> kernel::sys_ioctl(fd, request, argp)
  -> 現在ProcessのFdTable
  -> FileDescription::ioctl(request, argp)
  -> FileInterface::ioctl(request, argp)
```

- `kernel::sys_ioctl()` はfdの取得とABI引数の検査を行い、fd tableや対象デバイスの私有状態を直接操作せず、`FileDescription` へ委譲する。
- `FileDescription::ioctl()` は対象 `FileInterface` の要求処理へ委譲する。ioctlはファイル位置やopen flagsを変更しない。
- `FileInterface` に限定virtual hook `ioctl(uint32_t request, void* argp)` を追加し、既定実装は `-1` とする。要求対応デバイスだけがoverrideする。対象種別をRTTIや `dynamic_cast` で判定しない。
- syscall番号54および引数列は、ユーザー側定義・生成dispatcher・`kernel::sys_ioctl()` で一致させる。

## 5. ConsoleDevice要求の動作

`BTX_TIOCISATTY` は「このfdのI/O対象がOSの端末入出力デバイスであるか」を問い合わせる読み取り専用要求である。

- 端末として登録された `ConsoleDevice` は、`BTX_TIOCISATTY` とnull引数を `0` で受理する。
- `ConsoleDevice`は共有入力echoを既定ONで開始する。`BTX_TIOCGETECHO` は現在状態を1/0で返し、ON/OFF要求は同じConsoleDevice実体の状態を変更する。fd 0/1/2およびその継承・複製はこのdevice状態を共有する。
- `ConsoleDevice::read()` は通常入力byteを、echoがONなら `HAL::putchar()` へ出す。EOT (`0x04`) はechoせずEOFとして扱う。同一readにEOTより前のbyteがあればその分を返し、次のreadでEOFを一度返す。
- echo状態のGETおよびOFF設定は入力echoだけを制御し、入力byte、I/O先、FD参照数、FileDescription状態は変更しない。ON/OFF設定が変更するのはConsoleDevice共有echo状態だけである。
- 通常ファイル、Pipe、`NullDevice`、`ZeroDevice`、`RedConsoleDevice` は初版のTTY判定・echo制御要求をサポートせず `-1` を返す。
- `BTX_TIOCISATTY` と `BTX_TIOCGETECHO` は状態を変更しない。ON/OFF要求はecho設定だけを変更し、他の端末状態やFD/FileDescription状態は変更しない。
- 将来端末として扱う別デバイスは、本要求を明示的にサポートすることで端末判定対象に加わる。

`ConsoleDevice` がホストI/Oへ接続される場合も、判定はホストのstdin/stdoutがTTYかどうかに依存させず、OS内の対象デバイス種別に基づかせる。

## 6. 確認項目

1. `ioctl()` がsyscall番号54、引数順 `fd, request, argp` でdispatchされる。
2. `ConsoleDevice`上の `BTX_TIOCISATTY` は0、echo GETはecho状態1/0、ON/OFFは0を返す。
3. echoは既定ONで入力byteごとに出力され、OFF中は出力されない。EOTはechoされずEOF契約に従う。
4. 通常ファイル、Pipe、Null/Zero/RedConsoleデバイスではTTY・echo要求が失敗し、状態を変更しない。
5. 無効fd、未知要求、各要求への非null引数は失敗する。
6. `btx::isatty(fd)` はTTY要求成功を1、失敗を0へ変換する。
7. syscall generatorが `sys_ioctl` を通常の型付きsyscallとして解析し、`ioctl()` / `isatty()` 本体は生成対象に含めない。
8. 処理が `FdTable -> FileDescription -> FileInterface` を通り、syscall handlerが対象種別を直接判定しない。
