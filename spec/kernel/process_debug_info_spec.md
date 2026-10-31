# Processデバッグ情報取得関数仕様書

## 1. 目的と適用範囲

本書は、内蔵デバッグコマンド `ps` がProcess一覧と、取得可能な場合はContextスタック範囲を表示するためのカーネル内部関数を定める。ユーザー向けsyscallや公開ユーザーランドAPIは追加しない。内蔵バイナリからカーネル内部関数を呼ぶ例外は [`command_ps_debug_spec.md`](../command/command_ps_debug_spec.md) の範囲に限る。本APIはホストとマイコンの各ビルドで利用可能な内部APIとし、ターゲットABIには含めない。

Process状態・テーブル所有権は [`process_management_spec.md`](process_management_spec.md)、ホストContextとスタック境界は [`process_context_implementation_spec.md`](process_context_implementation_spec.md) に従う。本APIは診断専用であり、Process管理ポリシーやContext切替を行わない。

## 2. 内部API

```cpp
namespace kernel {

struct ProcessDebugInfo {
    int32_t pid;
    int32_t ppid;
    ProcessState state;
    char name[128];
    bool has_context;
    uintptr_t stack_begin;
    uintptr_t stack_end;
    uintptr_t guard_begin;
    uintptr_t guard_end;
};

bool debug_next_process_info(uint32_t& next_slot, ProcessDebugInfo& out);

}
```

- `next_slot`を0に初期化し、呼び出すたびにProcess table slotを昇順で走査する。
- 使用中スロットを見つけると、そのProcessの情報を`out`へ書き、次回走査位置を進めて`true`を返す。`FREE`スロットは飛ばす。slot加算の前に上限を検査し、整数wraparoundを利用しない。
- 残りのスロットに使用中Processがなければ`false`を返す。終了時に`out`を書き換えない。
- 無効なslot cursorが範囲外の場合も`false`を返す。これは内部診断APIであり、別のエラー値やerrnoを設けない。
- Process tableやProcessへの参照・ポインタを呼び出し元へ返さない。走査中にProcess状態やContextを変更しない。
- 配列全体のコピー、動的メモリ確保、Process tableの公開は行わない。
- `ProcessDebugInfo`は静的リンクされる内蔵バイナリとカーネル間の内部C++インターフェースであり、syscall ABI、永続形式、外部バイナリ形式ではない。
- 成功時は全フィールドを設定して`out`へ書き込む。失敗時は`out`を変更しない。

## 3. フィールド契約

| フィールド | 内容 |
|---|---|
| `pid` | ProcessのPID。Process管理仕様でsyscallに公開可能な`int32_t`範囲内 |
| `ppid` | 親PID。親なしの起動Processは既存仕様どおり0 |
| `state` | `ProcessState`の現在値 |
| `name` | Process生成時に指定された実行名。NUL終端された最大127 byteの文字列 |
| `has_context` | Processに初期化済みの実行Contextがある場合true |
| `stack_begin`, `stack_end` | 使用可能スタック領域の半開区間 `[stack_begin, stack_end)` |
| `guard_begin`, `guard_end` | ガード領域の半開区間 `[guard_begin, guard_end)` |

- stack/guardの境界を取得できない場合、対応するbegin/end両方に`uintptr_t`の最大値（`-1`を`uintptr_t`へ変換した値）を設定する。この値を本APIの「境界情報なし」sentinelとして予約し、実アドレス境界として返さない。
- `ps`を含む利用側はsentinelを特別扱いせず、通常のアドレス値と同じ書式で表示する。値の意味付けは表示を見る側に委ねる。
- ホストLinuxの`HOST_UCONTEXT`子Processでは、使用可能スタック範囲とLinuxガードページ範囲をContext backendから取得して設定する。
- `HOST_UCONTEXT`子Processでは`has_context`がtrueとなり、stack領域とguard領域は重ならず、境界値はContext backendの実際のmappingに一致する。
- マイコン向けContext backendは、所有境界を安全に取得できる場合に限って実値を返す。取得できない場合はsentinelを返し、推定値を返さない。Process列挙と`has_context`は境界情報の有無にかかわらず提供する。
- `NO_CONTEXT`、Process起動用の外部スタックを使うProcess、すでに終了後処理が済んだZOMBIEでは、Process所有stackがなければstack境界にsentinelを返す。guard領域がない場合もguard境界にsentinelを返す。
- Context backendは`ucontext_t`、CPUレジスタ構造体、ホスト固有メモリマッピングオブジェクトを`ProcessDebugInfo`へ格納しない。
- スタック使用量やhigh-water markはこのAPIの対象外である。スタック範囲から使用量を推定したり、スタック内容を走査したりしない。
- `name`はProcessが保持する名前をNUL終端込みで`out`へコピーする。Processの表示名の定義と上限は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。APIは名前を加工せず返す。

## 4. 所有権、整合性、呼出し制約

- `kernel::debug_next_process_info()`はsyscall handlerを経由せず、通常の内蔵バイナリ実行中に直接呼び出す。
- 関数はsyscall出力、ファイルI/O、ブロック、yield、Context切替を行わない。ProcessManagerの既存インスタンスとProcess tableを読み取る。
- 走査順はProcess table slot順であり、PID順・生成順を保証しない。
- 現行のwait駆動協調実行では、デバッグコマンドがProcess情報を走査している間、別のProcessは実行されない。将来プリエンプションまたは並列実行を導入した場合の原子的スナップショットやロック方式は本APIの保証外であり、その段階で再設計する。
- APIはデバッグ表示専用とし、権限判定やProcess情報の機密性境界として使用しない。

## 5. 確認項目

1. 全スロットが`FREE`なら最初の呼び出しが`false`を返し、`out`を変更しない。
2. 使用中Processをslot順に一件ずつ返し、`FREE`スロットを飛ばす。各Process名がNUL終端で完全にコピーされる。
3. 最後の使用中Processの後に`false`を返し、Process tableや`current()`を変更しない。
4. `HOST_UCONTEXT`子のスタック範囲とガード範囲がContext backendの実際の管理範囲と一致する。
5. ContextなしProcess、stack未所有Process、guard領域のないProcessでは、対応する境界値がsentinelとなる。
6. Process tableの内部参照、Contextの保存形式、ホストOSのmapping objectがAPI外へ露出しない。
7. マイコンContext backendで境界を取得できない場合、Process列挙を継続し、対応する境界値がsentinelとなる。
