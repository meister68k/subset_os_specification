# システムコール入口仕様書

## 1. 目的

本書は「Ancient UNIX風 サブセットOS」における、ユーザープログラムからカーネル機能へ到達するシステムコール入口の仕様を定める。

対象はユーザー側の型付きシステムコールラッパー、アーキテクチャ依存のシステムコール入口、カーネル側ディスパッチ、および型付きカーネルハンドラとの境界までとする。各システムコールの機能そのもの、プロセス管理、FD管理、ファイルシステム等の内部仕様は各サブシステムの仕様書に委ねる。

初期実装はホストOS上で動作するエミュレーション版とし、実機ターゲットとしてMC68000およびCortex-M0を想定する。

---

## 2. 設計方針

### 2.1 入口の一元化

UNIX互換システムコールおよび本OS固有システムコールは、ユーザー側では共通の `system_call()` を経由する。

通常のユーザープログラムは `system_call()` を直接呼び出さず、`btx::sys_xxx()` の型付きラッパーを使用する。

概念構造は以下とする。

```text
ユーザープログラム
        |
        | 型付きraw syscall API
        v
  btx::sys_putstrk(), btx::sys_write(), ...
        |
        v
  system_call(call_no, ...)
        |
        +-- ホスト版 --> system_call_handler(call_no, ...)
        |
        +-- MC68000版 --> TRAP #7
        |
        +-- Cortex-M0版 --> 未実装
        |
        v
  kernel::sys_xxx()
        |
        v
  各カーネルサブシステム
```

`btx::sys_xxx()` はユーザー側の型付きraw syscallラッパー、`kernel::sys_xxx()` はカーネル側の型付きハンドラとする。同じ `sys_xxx` 名を使用するが、名前空間と役割は明確に異なる。

プロセス終了・待機の型付きラッパーとカーネルハンドラは、ABI幅を明示して次の形とする。

```cpp
int32_t btx::sys_exit(int32_t status); // 呼出し後は復帰しない
int32_t btx::sys_wait(int32_t* status); // 成功時は子PID、失敗時は-1
```

`sys_wait`の`status == nullptr`は終了値出力を省略する指定である。非nullの出力先の検証と安全な書き込みはsyscall境界の責務とし、ProcessManagerの子回収を確定する前に完了させる。出力に失敗した場合は`-1`とし、対象の子Processを回収済みにしてはならない。詳細な終了・回収順序は[`process_management_spec.md`](process_management_spec.md)に従う。

POSIX/UNIX風の `write()`、`read()` 等のライブラリAPIを別途提供する場合、それらは `btx::sys_xxx()` より上位のユーザーランド層とし、本書では規定しない。

### 2.2 ABI境界

システムコール番号は `uint32_t`、システムコールの共通戻り値は `int32_t` とする。

```cpp
int32_t system_call(uint32_t call_no, ...);
```

各システムコールの引数型はABIの一部として固定する。現在の基本的な整数引数には `int32_t` / `uint32_t` を使用し、ホスト環境固有の `int`、`long`、`size_t` の幅には依存させない。

ポインタ引数はC/C++のポインタ型として記述する。ポインタ幅そのものはターゲットのC ABIに従う。

エラー表現の詳細は各システムコールおよびエラーコード体系の仕様に委ねる。

### 2.3 アーキテクチャ依存部分の局所化

ユーザー側の `btx::sys_xxx()` のシグネチャとシステムコール番号はターゲット間で共通とする。

一方、`system_call()` からカーネルディスパッチへ制御を渡す方法はアーキテクチャ依存とする。

- ホスト版: C/C++可変長引数関数 `system_call_handler()` を直接呼び出す。
- MC68000版: Cの可変長引数呼出しで形成された引数列を利用し、`TRAP #7` でカーネルへ移行する。
- Cortex-M0版: 未実装。例外/SVCを含む具体的な入口ABIは別途定める。

カーネル側では、ディスパッチ後に `kernel::sys_xxx()` を型付き関数として呼び出す。各サブシステムは、ホスト版の `va_list` や実機のTRAP/SVC入口を意識しない。

---

## 3. ユーザー側インターフェース

### 3.1 `system_call()`

`system_call()` はアーキテクチャ依存の低レベル入口であり、通常のユーザープログラムから直接使用しない。

```cpp
int32_t system_call(uint32_t call_no, ...);
```

ホスト版では次の `system_call_handler()` への呼び出しとして実装する。

```cpp
extern "C" int32_t system_call_handler(uint32_t call_no, ...);
```

### 3.2 型付きraw syscallラッパー

各システムコールには `btx` 名前空間内に `sys_xxx()` 形式の型付きラッパーを用意する。

例:

```cpp
namespace btx {

static inline int32_t sys_putstrk(const char* msg)
{
    return system_call(0, msg);
}

static inline int32_t sys_write(int32_t fd, const void* buf, uint32_t count)
{
    return system_call(4, fd, buf, count);
}

}
```

これにより、型安全でない可変長引数呼び出しを低レベル入口内部に限定する。

### 3.3 ラッパー生成

`btx::sys_xxx()` の定義は `unistd.h` 内の所定のマーカー間に配置し、システムコール定義からgeneratorで生成可能な構成とする。

システムコール番号、ラッパーの引数型、カーネル側ディスパッチで取得する型は、同一の定義から生成し、不一致を避ける。

---

## 4. ホスト版システムコール入口

### 4.1 `system_call_handler()`

ホスト版では実CPUのTRAPを使用せず、C/C++の可変長引数関数によってシステムコール入口を模擬する。

```cpp
extern "C" int32_t system_call_handler(uint32_t call_no, ...);
```

`system_call()` はマクロまたはinline関数により `system_call_handler()` へ接続する。

### 4.2 可変長引数の取得

引数は `<cstdarg>` / `<stdarg.h>` の以下の機構を用いて取得する。

```cpp
va_list
va_start
va_arg
va_end
```

ディスパッチ部はシステムコール番号によって引数の個数および型を決定し、本来の型で直接取得する。

例:

```cpp
case 4:
    {
        int32_t arg0 = va_arg(ap, int32_t);
        const void* arg1 = va_arg(ap, const void*);
        uint32_t arg2 = va_arg(ap, uint32_t);
        status = kernel::sys_write(arg0, arg1, arg2);
    }
    break;
```

複数の `va_arg()` を1つの関数呼び出し式に並べてはならない。関数呼び出しにおける引数式の評価順序はC++で未規定であり、複数の `va_arg()` を並べると取得順と引数順が対応しなくなるため、各 `va_arg()` を独立した文として宣言順に評価する。

取得した引数の一時変数名は `arg0`, `arg1`, ... と位置指定とし、dispatcher側の変数 (`status`, `ap`, `call_no`) と衝突させない。

生成対象は `case` ラベルとその中身のみとする。`switch` 文そのもの、`default:` 以降の処理、最終的な `return` 等はユーザー側コードで記述する。

各caseの戻り値はローカル変数 `status` に格納する。

### 4.3 可変長引数に関する制約

`va_arg()` で取得する型は、呼出し側から実際に渡された型と一致しなければならない。

したがって、次の3者の型を必ず一致させる。

1. `btx::sys_xxx()` の引数型
2. `system_call_handler()` の `va_arg()` 取得型
3. `kernel::sys_xxx()` の引数型

引数を一旦 `uintptr_t` 等へ統一してから復元する方式は採用しない。

整数定数等に対するdefault argument promotionsにも注意する。ABIで固定幅整数型を指定していても、C/C++の可変長引数規則上promotionが発生する型を追加する場合は、実際に渡される型と `va_arg()` の型を一致させること。

---

## 5. カーネル側インターフェース

### 5.1 `kernel::sys_xxx()`

ディスパッチ後は各システムコール固有の型付き関数を呼び出す。

例:

```cpp
namespace kernel {

int32_t sys_putstrk(const char* msg);
int32_t sys_read(int32_t fd, void* buf, uint32_t count);
int32_t sys_write(int32_t fd, const void* buf, uint32_t count);
int32_t sys_lseek(int32_t fd, int32_t offset, int32_t whence);
int32_t sys_unlink(const char* path);
int32_t sys_chdir(const char* path);
int32_t sys_getdents(int32_t fd, void* buf, uint32_t count);
int32_t sys_mkdir(const char* path, uint32_t mode);
int32_t sys_rmdir(const char* path);

}
```

`kernel::sys_xxx()` より下位のコードは、呼び出し元がホスト版 `system_call_handler()` であるか、実機のTRAP/SVC handlerであるかを意識しない。

### 5.2 未定義・未実装システムコール

未定義または未実装のシステムコール番号が指定された場合は、ディスパッチ層でエラーとして扱う。

具体的なエラー値（`ENOSYS`相当を含む）は、エラーコード体系の確定時に定める。

---

## 6. `putstrk`

### 6.1 目的

syscallディスパッチ機構の最初の疎通確認用として、本OS固有の `putstrk` を実装する。

`putstrk` はUNIX互換システムコールではなく、bootstrap/debug用途のカーネルサービスである。

FD管理が未実装の段階でも使用可能とする。

### 6.2 ユーザー側インターフェース

raw syscallラッパーは次の形式とする。

```cpp
btx::sys_putstrk(const char* msg);
```

`msg` が指すNUL終端文字列を機種依存の低レベル出力先へ出力する。

**末尾への改行追加は行わない。**

### 6.3 呼び出し経路

```text
btx::sys_putstrk(msg)
        |
        v
system_call(0, msg)
        |
        +-- host: system_call_handler(...)
        |
        +-- MC68000: TRAP #7
        |
        v
kernel::sys_putstrk(msg)
        |
        v
machine_putstr(msg)
```

`kernel::sys_putstrk()` は文字列出力処理そのものを実装せず、正式実装では機種依存層へ委譲する。

### 6.4 機種依存出力

低レベル出力先は機種依存とする。

| ターゲット | 出力方法 |
| --- | --- |
| ホスト版 | ホストOSの標準出力（`fputs()`等） |
| シリアルポート搭載マイコン | シリアルポート |
| X68000 | IOCSによる文字出力 |
| その他 | 各ターゲットで定義 |

共通層は具体的な出力デバイスを規定しない。

機種依存インターフェースは概念上、次のような最小APIとする。

```cpp
void machine_putstr(const char* msg);
```

テスト用stubでは `kernel::sys_putstrk()` からホスト標準出力へ直接出力してもよい。

### 6.5 `write()` との関係

`putstrk` は `write()` の暫定実装ではない。

`write()` は正式なUNIX互換システムコールとしてFD管理層を経由する。一方、`putstrk` はFDを使用せず機種依存の低レベル出力へ直接接続する。

```text
putstrk
   |
   +--> machine_putstr

write
   |
   +--> FD table
          |
          +--> open file / device / pipe
```

FD管理完成後も、`putstrk` はbootstrap/debug用の独立した機能として扱う。

---

## 7. MC68000実機版

### 7.1 TRAPによる入口

MC68000版ではシステムコール入口に **`TRAP #7`** を使用する。

```asm
trap    #7
```

### 7.2 呼び出し規約

ユーザー側 `system_call(uint32_t call_no, ...)` はCコンパイラの通常の可変長引数呼出しで形成された引数列を利用する。

TRAP発行前に、`system_call()` 呼び出し時の `call_no` を先頭とする引数列をカーネル側から参照できるようSPを設定し、`TRAP #7` を発行する。

現行実装の概念は以下の通りである。

```asm
move.l  %sp,%a1       ; system_call() 内の元SPを保存
move.l  argp,%sp      ; call_no を先頭とする引数列をSPに設定
trap    #7
move.l  %a1,%sp       ; 元SPを復元
move.l  %d0,status    ; 戻り値を取得
```

カーネル側TRAP handlerは、この引数列からシステムコール番号と各引数を取得し、対応する `kernel::sys_xxx()` を呼び出す。

### 7.3 戻り値

システムコールの戻り値はD0に格納する。

戻り値の型は `int32_t` とする。

### 7.4 レジスタ保存規約

ユーザー側 `system_call()` は元のSPをA1に保持してTRAPを発行するため、TRAP handlerは復帰時までA1の値を保存しなければならない。

その他のレジスタについて、ユーザー側inline assemblyは現時点で `d0`, `d1`, `a0`, `a1`, condition codeおよびmemoryを破壊され得るものとして扱う。

カーネルTRAP handler内部の詳細な保存・復帰規約はMC68000依存部の実装仕様に委ねるが、少なくとも本節のユーザー側契約を満たさなければならない。

### 7.5 ABI上の注意

MC68000版はCコンパイラの呼出し規約を利用するため、引数のスタック配置は使用するツールチェーンのABIと整合していなければならない。

特に固定幅整数型、ポインタ、および可変長引数の配置について、ユーザー側とカーネル側で同一ABIを前提とする。

---

## 8. Cortex-M0実機版

Cortex-M0版の `system_call()` は未実装とする。

ユーザー側の `btx::sys_xxx()` のシグネチャおよびシステムコール番号は他ターゲットと共通とするが、カーネルへの遷移方法、引数渡し、戻り値レジスタ、破壊レジスタ等の具体的ABIは実装時に別途定める。

---

## 9. syscall番号

システムコール番号はユーザーランドとカーネル間のABIであるため、確定後は安易に変更しない。

V7由来のシステムコールについては、可能な範囲でV7の番号を踏襲する。V7で廃止扱いとなっている0番を、本OS固有の `putstrk` に割り当てる。

現時点で確定している番号は以下の通り。

| 番号 | システムコール | 備考 |
| ---: | --- | --- |
| 0 | `putstrk` | 本OS固有 |
| 1 | `exit` | V7と同番号 |
| 2 | `fork` | 本OSでは未実装・不採用。番号は予約 |
| 3 | `read` | V7と同番号 |
| 4 | `write` | V7と同番号 |
| 5 | `open` | V7と同番号 |
| 6 | `close` | V7と同番号 |
| 7 | `wait` | V7と同番号 |
| 8 | `creat` | V7と同番号 |
| 18 | `stat` | V7と同番号 |
| 35 | `sleep` | V6と同番号、V7ではftime |
| 64 | `spawn` | 本OS固有（V7定義範囲の63を超える） |
| 66 | `getdents` | 本OS固有（V7定義範囲の63を超える） |
| 67 | `mkdir` | 本OS固有（V7定義範囲の63を超える） |
| 68 | `rmdir` | 本OS固有（V7定義範囲の63を超える） |

今後追加するシステムコールの番号は、全体仕様書で採用対象となっているシステムコールとV7番号との対応を踏まえて決定する。

`mkdir` / `rmdir`のphase 1 syscall経路とFilesystem連携は[`mkdir_rmdir_implimentation_spec.md`](mkdir_rmdir_implimentation_spec.md)に従う。
`getdents`の戻り値、wire record、FD offset連携は[`getdents_implementation_spec.md`](getdents_implementation_spec.md)に従う。
`unlink`のpathname解決、Filesystem連携、open中inodeの寿命は[`unlink_implementation_spec.md`](unlink_implementation_spec.md)に従う。
`lseek`の32-bit position演算、seek不能target、FD共有offsetは[`lseek_implementation_spec.md`](lseek_implementation_spec.md)に従う。
`chdir`のpathname検査、directory属性確認、Process cwd参照交換は[`chdir_implementation_spec.md`](chdir_implementation_spec.md)に従う。
`spawn()` / `wait()`の処理モデルは[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。

---

## 10. 実装・生成方針

システムコール入口の骨格は以下の順序で整備する。

1. `uint32_t` のcall numberと `int32_t` の共通戻り値を用いる `system_call()` を定義する。
2. `btx::sys_xxx()` の型付きraw syscallラッパーを用意する。
3. ホスト版 `system_call_handler(uint32_t, ...)` を実装する。
4. generatorにより、ユーザー側ラッパーとカーネル側dispatcherの `case` 本体を同一のシステムコール定義から生成する。
5. `putstrk` によりユーザー側ラッパーからカーネル側ハンドラまでの経路を確認する。
6. 全体仕様書の実装計画に従い、ファイル関連等のシステムコールを順次追加する。
7. MC68000版では `TRAP #7` 入口を使用する。
8. Cortex-M0版の入口ABIは実機移植時に定義・実装する。

プロセス管理・FD管理の完成を、すべてのUNIX互換システムコール追加の前提とはしない。ホスト版では疑似ユーザープログラム等を用いて、依存サブシステムが実装済みのシステムコールから先行して検証してよい。

---

## 11. 対象外

本仕様では以下を対象外とする。

- 個々のUNIX互換システムコールの詳細な機能仕様
- FD table / open-file objectの内部構造
- プロセス構造体
- ユーザーポインタの保護・検証方式
- MMUを用いたユーザー/カーネル空間分離
- シグナルによるsyscall中断・再開
- POSIX/UNIX風libc API (`read()`, `write()` 等) の詳細
- Cortex-M0の具体的な例外/SVC ABI
- 未追加システムコールの最終番号割当
- 各アーキテクチャのhandler内部実装詳細

これらは必要となる実装段階で各サブシステムまたはアーキテクチャ依存部の仕様として定める。
