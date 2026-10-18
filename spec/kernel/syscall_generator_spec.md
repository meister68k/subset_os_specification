# syscall generator 仕様書（改訂版）

## 1. 目的

OS利用者がincludeする `unistd.h` 相当ヘッダをシステムコールABIの正本とし、Ruby製generatorがカーネル側dispatcher等を生成する。

正本とする情報は、syscall番号、ユーザー向け関数名、戻り値型、引数型、引数名である。専用の `syscalls.def` 等は設けない。

## 2. 入力形式

generatorはヘッダ全体をC/C++として解析しない。次のマーカー間だけを対象とする。

```cpp
// SYSTEMCALL START
...
// SYSTEMCALL END
```

各実装済みsyscallは必ず1行で記述する。

```cpp
SYSTEMCALL <return-type> <name>(<arguments>) {return system_call(<number>[, <arguments>...]);}
```

例:

```cpp
SYSTEMCALL int32_t sys_putstrk(const char* str) {return system_call(0, str);}
SYSTEMCALL int32_t sys_exit(int32_t status) {return system_call(1, status);}
// No.2 未実装 (fork)
SYSTEMCALL int32_t sys_read(int32_t fd, void* buf, uint32_t count) {return system_call(3, fd, buf, count);}
SYSTEMCALL int32_t sys_write(int32_t fd, const void* buf, uint32_t count) {return system_call(4, fd, buf, count);}
```

ユーザー側inline wrapperそのものがgeneratorへの入力であり、generatorはユーザー側wrapperを生成しない。

## 3. 記述規約

初期仕様では以下を要求する。

- 1 syscallを1行に記述する。
- 行頭の最初の有効トークンを `SYSTEMCALL` とする。
- 戻り値型を明記する。
- 全引数に名前を付ける。
- 関数本体は `{return system_call(...);}` の単純形式とする。
- `system_call()` の第1引数は10進整数のsyscall番号とする。
- 第2引数以降はwrapperの仮引数を宣言順にそのまま列挙する。
- 引数の並べ替え、追加、削除、式変換を行わない。
- 可変長引数、関数ポインタ引数、配列形式の引数宣言、default argumentは使用しない。
- wrapper内に条件分岐等の処理を記述しない。

引数0個は `()` を標準形式とする。

未実装syscallは `SYSTEMCALL` 定義を置かず、必要ならコメントで欠番を示す。

```cpp
// No.2 未実装 (fork)
```

欠番は許容する。

## 4. 型

固定幅整数型、typedefされた単純な整数型、単純なポインタ型および `const` 修飾を使用可能とする。

例:

```cpp
int32_t
uint32_t
void*
const void*
char*
const char*
char**
```

空白は可能な範囲で正規化する。

以下のような複雑な宣言は初期仕様では対象外とする。

```cpp
int (*callback)(int)
int values[10]
const char name[]
...
```

必要になった時点で明示的に仕様を拡張する。

## 5. 解析結果

各 `SYSTEMCALL` 行から以下を取得する。

```text
syscall番号
戻り値型
関数名
引数数
各引数の型
各引数の名前
```

例:

```cpp
SYSTEMCALL int32_t sys_write(int32_t fd, const void* buf, uint32_t count) {return system_call(4, fd, buf, count);}
```

解析結果:

```text
number      = 4
return_type = int32_t
name        = sys_write

arg[0] = int32_t fd
arg[1] = const void* buf
arg[2] = uint32_t count
```

## 6. wrapper自己整合性検査

generatorはprototypeだけでなく `system_call()` 呼出しも解析し、仮引数と実引数が完全に一致することを確認する。

例えば、

```cpp
(int32_t fd, const void* buf, uint32_t count)
```

に対して、

```cpp
system_call(4, fd, buf, count)
```

でなければならない。

以下はエラーとする。

```cpp
system_call(4, buf, fd, count)
system_call(4, fd, buf)
system_call(4, fd, buf, count + 1)
```

これにより、wrapperの型情報を実際のsyscall引数型として使用できる。

## 7. syscall番号

syscall番号はwrapper内の `system_call(<number>, ...)` から取得する。宣言順から自動割当しない。

generatorは少なくとも番号重複を検査する。V7由来の未実装番号等による欠番は許容する。

これにより、ヘッダ内でsyscallの並び順を変更してもABI番号は変化しない。

## 8. カーネル側handler

カーネル側はC++で実装し、システムコールhandlerは `kernel` namespaceに置く。

ユーザー側wrapperの関数名はそのまま維持し、カーネル側ではnamespaceによって区別する。

例:

```cpp
namespace kernel {

int32_t sys_write(int32_t fd, const void* buf, uint32_t count);

}
```

したがってユーザー側の、

```cpp
sys_write(fd, buf, count)
```

に対応するカーネル側handlerは、

```cpp
kernel::sys_write(fd, buf, count)
```

となる。

generatorが生成するdispatcherも `kernel::sys_xxx()` を呼び出す。

C++のnamespace、型検査等を積極的に利用し、C互換性のためだけのprefix追加は行わない。

## 9. ホスト版dispatcher

ホスト版では `system_call()` を可変長引数のOS側handlerへ接続する。

概念例:

```cpp
extern int32_t system_call_handler(uint32_t call_no, ...);
#define system_call(call_no, ...) system_call_handler(call_no, __VA_ARGS__)
```

generatorは番号ごとに `va_list` から引数を取得し、型付きカーネルhandlerを呼ぶコードを生成する。

入力:

```cpp
SYSTEMCALL int32_t sys_write(int32_t fd, const void* buf, uint32_t count) {return system_call(4, fd, buf, count);}
```

生成例:

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

複数の `va_arg()` を1つの関数呼び出し式に並べてはならない。関数呼び出しにおける引数式の評価順序はC++で未規定であり、複数の `va_arg()` を並べると取得順と引数順が対応しなくなるため、各 `va_arg()` を独立した文として宣言順に評価する (§18)。

## 10. default argument promotions

ホスト版はC/C++可変長引数を使用するため、default argument promotionsを考慮する。

`char`、`short` 等は `int` 等へ、`float` は `double` へpromotionされた値を `va_arg()` で取得し、必要なら元の論理型へ変換する。

`int32_t`、`uint32_t` 等のtypedefについて、ホスト処理系での基礎型によって厳密な `va_arg()` 型が変わり得る。generatorはサポートする型と取得型の対応規則を持つ。

安全な取得型を決定できない場合は推測せず生成エラーとする。

## 11. architecture固有 `system_call()`

`SYSTEMCALL` wrapperはarchitecture非依存とし、その下の `system_call(call_no, ...)` だけをarchitecture依存とする。

```text
sys_write(fd, buf, count)
        |
        v
system_call(4, fd, buf, count)
        |
        +-- host      -> system_call_handler(...)
        +-- m68k      -> TRAP
        +-- Cortex-M0 -> SVC等
```

generatorはwrapperから論理的なsyscall ABIを取得するが、architecture固有の物理的な引数渡し方式を推測しない。

## 12. 68000版との関係

現段階の68000版 `system_call()` はインターフェース検討用の参考実装とし、generator仕様はそのassembly実装に依存しない。

実機実装時に以下を確定する。

- TRAP番号
- Cコンパイラのcalling conventionとの接続
- `-fomit-frame-pointer` 等のコンパイルオプションに依存しない引数位置の取得
- user SP / supervisor SPの扱い
- syscall番号・引数列のカーネルへの受渡し
- 戻り値
- レジスタ保存・破壊規則

現段階では68000向けassemblyをgeneratorから生成しない。

## 13. 生成対象ファイル

generatorは、コマンドライン引数で指定された既存のカーネル側ヘッダファイルを更新する。

実行形式:

```text
ruby syscall_generator.rb <unistd.h> <syscall.cpp>
```

第1引数には、システムコール定義の正本となるユーザー向け `unistd.h` 相当ヘッダを指定する。

第2引数には、生成した `case` 群を挿入するカーネル側 `syscall.cpp` を指定する。

例:

```text
ruby syscall_generator.rb ./include/unistd.h ./kernel/syscall.cpp
```

この場合、

```text
入力元        ./include/unistd.h
更新対象      ./kernel/syscall.cpp
一時生成物    ./kernel/syscall.cpp.tmp
バックアップ  ./kernel/syscall.cpp.bak
```

とする。

generator自身は入力元および更新対象のファイル名・配置を固定しない。

## 14. 生成範囲

指定されたファイル中の、

```cpp
// SYSTEMCALL START
```

から、

```cpp
// SYSTEMCALL END
```

までの区間を生成対象とする。

両マーカー自身は保持する。

既存のマーカー間の記述はすべて破棄し、新たに生成した内容へ置換する。

例えば更新前が、

```cpp
namespace kernel {

// SYSTEMCALL START
古い生成内容
// SYSTEMCALL END

}
```

であれば、更新後は、

```cpp
namespace kernel {

// SYSTEMCALL START
新しい生成内容
// SYSTEMCALL END

}
```

となる。

マーカー外の内容は変更してはならない。

`SYSTEMCALL START` または `SYSTEMCALL END` が存在しない場合、複数存在する場合、順序が不正な場合はエラーとする。

## 15. 一時ファイル、更新、バックアップ

### 15.1 一時ファイル

generatorは更新対象ファイルを直接書き換えない。

まず、マーカー間を新しい生成内容に置換したファイル全体を、

```text
<output-file>.tmp
```

へ出力する。

例:

```text
syscall.cpp.tmp
```

### 15.2 正常終了時

生成および全検査に成功した場合、次の順序でファイルを更新する。

```text
1. syscall.cpp.tmp を完全に生成
2. 全検査が成功したことを確認
3. syscall.cpp     -> syscall.cpp.bak
4. syscall.cpp.tmp -> syscall.cpp
```

すなわち、元の `syscall.cpp` は `syscall.cpp.bak` として保存され、正常に生成された新しいファイルだけが `syscall.cpp` になる。

既存の `syscall.cpp.bak` が存在する場合は上書きしてよい。

正常終了後、`.tmp` はrenameされるため残らない。

### 15.3 エラー時

生成または検査中にエラーが発生した場合、

```text
syscall.cpp
```

および、

```text
syscall.cpp.bak
```

を変更してはならない。

ただし、

```text
syscall.cpp.tmp
```

は削除せず、その時点の生成結果を診断用として残す。

可能な限り、エラー発生箇所以前までの生成内容、および生成可能であった残りの内容を `.tmp` に反映する。

`.tmp` 自体を書き出せないI/Oエラーの場合は、この限りではない。

## 16. エラー検出

少なくとも以下をエラーとする。

- 入力となるユーザー向け `SYSTEMCALL` 定義の構文違反
- 戻り値型または引数名の欠落
- 未対応の複雑な型
- 可変長引数
- syscall番号が整数でない
- syscall番号の重複
- syscall名の重複
- 仮引数と `system_call()` 実引数の個数不一致
- 仮引数と実引数の順序・名前不一致
- `va_arg()` 型を安全に決定できない型
- 更新対象ファイルを読み込めない
- 更新対象ファイルの `SYSTEMCALL START` / `SYSTEMCALL END` が存在しない
- マーカーが複数存在する
- マーカーの順序が不正
- `.tmp` を生成できない
- バックアップまたはrenameに失敗する

構文・整合性エラー時は元の更新対象ファイルを変更せず、非0で終了する。

可能な限り以下を表示する。

```text
入力ファイル名
行番号
対象行
エラー理由
```

## 17. generator実装方針

generatorはRubyで実装し、Clang等のC/C++ parserには依存しない。

基本処理:

```text
1. 第1引数で指定されたユーザー向けヘッダを読む
2. SYSTEMCALL定義を抽出・解析
3. syscall番号、型、名前、引数を内部表現へ格納
4. wrapperの自己整合性を検査
5. syscall間の重複等を検査
6. 第2引数で指定された更新対象ファイルを読む
7. SYSTEMCALL START / ENDを検査
8. マーカー間を生成内容に置換
9. ファイル全体を <output-file>.tmp へ出力
10. 全検査成功時のみ <output-file> を <output-file>.bak へrename
11. <output-file>.tmp を <output-file> へrename
```

正規表現と単純な文字列処理を基本とする。任意のC/C++を解析できるようparserを複雑化するより、入力書式を限定する。

## 18. 生成コード

generatorが生成するのは、dispatcherの `switch` 文に挿入する **`case` ラベルとその本体だけ** とする。

`switch` 文そのもの、`default:` 以降の処理、dispatcherからの `return` 等はユーザーが更新対象ファイル側に記述し、generatorは変更しない。

各 `case` では対応する `kernel::sys_xxx()` を呼び出し、その戻り値を既存の変数 `status` に代入した後、`break` する。

引数を取得する複数の `va_arg()` を1つの関数呼び出し式に並べてはならない。関数呼び出しにおける引数式の評価順序はC++で未規定であり、複数の `va_arg()` を並べると取得順と引数順が対応しなくなるため、各 `va_arg()` を独立した文として宣言順に評価した後、それらの一時変数を引数として `kernel::sys_xxx()` を呼ぶ。引数0個のsyscallはブロックを生成せず `status = kernel::sys_xxx();` とする。

一時変数名は `arg0`, `arg1`, ... と位置指定とし、dispatcher側の変数 (`status`, `ap`, `call_no`) と衝突させない。

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

更新対象ファイルは、例えば次のような構造を持つ。

```cpp
switch (call_no) {
    // SYSTEMCALL START
    // generatorがcase群をここへ生成
    // SYSTEMCALL END

    default:
        // ユーザー記述
        ...
}

return status;
```

`SYSTEMCALL START` / `SYSTEMCALL END` の外側はgeneratorの管理対象外であり、generatorは `switch`、`default:`、`return`、変数 `status` の宣言を生成しない。

## 19. 最初の成立条件

ユーザー向けヘッダに少なくとも、

```cpp
SYSTEMCALL int32_t sys_putstrk(const char* str) {return system_call(0, str);}
SYSTEMCALL int32_t sys_exit(int32_t status) {return system_call(1, status);}
// No.2 未実装 (fork)
SYSTEMCALL int32_t sys_read(int32_t fd, void* buf, uint32_t count) {return system_call(3, fd, buf, count);}
SYSTEMCALL int32_t sys_write(int32_t fd, const void* buf, uint32_t count) {return system_call(4, fd, buf, count);}
```

が存在する状態で、

```text
ruby syscall_generator.rb ./include/unistd.h ./kernel/syscall.cpp
```

を実行し、第1引数の `unistd.h` 相当ヘッダを正本として、第2引数の `syscall.cpp` の `SYSTEMCALL START` / `END` 間へ対応する `case` 群を生成できること。

生成されたdispatcherを介して、

```text
sys_putstrk(...)
    -> system_call(...)
    -> system_call_handler(...)
    -> generated dispatcher
    -> kernel::sys_putstrk(...)
    -> machine_putstr(...)
```

の疎通を確認する。

## 20. TODO

- generatorの正式な配置
- syscall番号の有効範囲
- syscall戻り値とエラー表現
- typedef型とdefault argument promotionsの対応方法
- `(void)` 引数リストを許容するか
- 予約syscall番号をgeneratorで検査するか
- 68000 / Cortex-M0用decoder生成方式
- ポインタ引数のユーザー空間検証との関係
