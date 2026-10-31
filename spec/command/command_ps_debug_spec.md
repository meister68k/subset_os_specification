# デバッグ用 `ps` コマンド仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | BTX-DEBUG-PS-001 |
| 版 | 0.1（実装前） |
| 対象 | ホスト・マイコン共通の内蔵ユーザーバイナリ `ps` |
| 位置づけ | Process Contextと専用スタックを確認する開発・デバッグ用コマンド |
| 依拠する仕様 | [`process_debug_info_spec.md`](../kernel/process_debug_info_spec.md)、[`getpid_implementation_spec.md`](../kernel/getpid_implementation_spec.md)、[`spawn_wait_implimentation_spec.md`](../kernel/spawn_wait_implimentation_spec.md) |

## 1. 目的と範囲

Process tableの使用中Processを列挙し、PID・PPID・プロセス名・状態・Context有無・取得可能な所有スタック範囲・ガード範囲を標準出力へ表示する。ホストでは`HOST_UCONTEXT`の子Processが別のスタック領域を使うことを確認できる。マイコンではProcess一覧とContext有無を共通表示し、backendが境界を提供する場合に限って範囲も表示する。

`ps`は独立実行ファイルではなく、すべてのターゲットで`kernel::user_binary("ps")`から検索される静的リンク済み内蔵バイナリとする。Process一覧取得には [`process_debug_info_spec.md`](../kernel/process_debug_info_spec.md) のカーネル内部関数を直接呼び出す。このコマンドのためのsyscallは追加しない。ProcessDebugInfoのフィールド契約は全ターゲットで共通とし、取得できないContext境界はAPIのsentinel値をそのまま表示する。

## 2. 起動形式と利用条件

```text
ps
```

- 引数なしで実行する。余分な引数はusageをfd=2へ出し、終了値1を返す。
- 内蔵バイナリ登録名は`ps`とする。
- 入口は既存の内蔵バイナリ規約に従い、`int ps(int argc, const char** argv)`相当とする。
- `ps`はProcessがcurrentで、標準fd 1/2が初期化済みの実行文脈から呼ばれることを前提とする。
- デバッグ目的のため、Process内部情報を取得するkernel内部関数の呼び出しを許可する。一般ユーザーバイナリに同APIを公開するものではない。

## 3. 出力形式

最初に固定ヘッダーを1行出力し、続けてProcess table slot順に使用中Processを1行ずつ出力する。

```text
PID PPID NAME STATE CTX STACK GUARD
1 0 init WAITING none - -
2* 1 ps RUNNING context [0x00007f001000,0x00007f001200) [0x00007f000000,0x00007f001000)
3 1 echo READY context [0x00007f002000,0x00007f002200) [0x00007f001000,0x00007f002000)
```

- 実際の数値・process数は例示であり、固定しない。
- PIDとPPIDは10進符号なし表記とする。状態名は`FREE`を除く`ProcessState`の列挙名を表示する。
- `NAME`はProcessDebugInfoの`name`をNULまでそのまま出力する。Process名は`spawn()`の`name`引数から取得した値であり、`argv[0]`への変更やパスからのbasename抽出をしない。最大長は127 byte。
- `CTX`は初期化済みContextがあれば`context`、なければ`none`とする。ホスト・マイコンの実装方式名は表示形式に含めない。
- `STACK`と`GUARD`はAPIから受け取ったbegin/endを常に`[begin,end)`形式で表示する。境界情報がない場合もAPIのsentinel値を通常の数値として表示し、`ps`側で`-`等へ置き換えない。
- アドレスは`uintptr_t`幅（`2 * sizeof(uintptr_t)`桁）に合わせた`0x`付き固定幅16進表記とする。例の16桁は64-bitホストの場合であり、数値はビルド・実行ごとに変わり得る。
- stack使用量、high-water mark、saved stack pointerは表示しない。
- Process table slot番号は表示せず、PID・PPIDをProcess識別子とする。

## 4. 情報取得と表示手順

1. `btx::sys_getpid()`で`ps`自身のPIDを取得する。
2. `kernel::debug_next_process_info(cursor, info)`を`cursor=0`から繰り返し呼び、Process情報を1件ずつ取得する。Process table全体の配列コピーや動的確保は行わない。
3. 各レコードの`pid`が自身のPIDと一致する行には、PIDの直後に`*`を付ける（例：`2*`）。一致しない行は通常表示する。
4. 状態名とcontext文字列を選び、stack/guardの境界値を常に16進表記する。境界情報の有無を判定・変換しない。
5. ヘッダー・行をfd=1へ出力する。fd=2は引数エラーまたは出力失敗の診断に使う。

`ps`はProcessDebugInfoのC++オブジェクト配置を独自に解釈せず、型付きkernel内部API経由で値を読む。`ucontext_t`やProcessテーブル配列には直接アクセスしない。

## 5. バッファ・出力制約

- 各行は固定長のローカルバッファで組み立てる。NUL終端も行う場合、容量は少なくとも `192 + 8 * sizeof(uintptr_t)` byteとする（最大長Process名、PID/PPIDの最大桁数、状態名、Context名、2個のアドレス範囲、区切り、LF、NULを含む）。書込み前に実際の長さを検査する。
- 16進変換は`uintptr_t`の全ビットを表示できる固定長処理とし、整数を狭い型へ切り詰めない。`printf()`、`%p`、動的メモリ確保は使用しない。
- `btx::sys_write(1, ...)`が部分書込みした場合は未送信部分を続けて書く。0または負値、要求を超える戻り値は出力失敗とする。
- 出力失敗時は可能ならfd=2へ診断し、終了値1を返す。診断出力失敗を再帰的に処理しない。
- Process tableを走査するためにプロセス管理の状態を変更しない。`spawn()`、`wait()`、`exit()`、`close()`等は呼ばない。

## 6. 実行モードごとの表示

| モード・Process | `CTX` | `STACK` / `GUARD` |
|---|---|---|
| Contextを初期化済み | `context` | stack/guardの実境界、またはAPIが返すsentinel値 |
| Contextなし（`NO_CONTEXT`を含む） | `none` | APIが返すsentinel値 |
| 起動時Processなど専用stackを所有しない | `none` | APIが返すsentinel値 |
| Context資源を解放済みの`ZOMBIE` | `none` | APIが返すsentinel値 |

`HOST_UCONTEXT`では親Processが`wait()`中に保存Contextで表示され、選択中の子Processが`RUNNING`として表示される。wait駆動方式のため、一覧は同時実行中のProcess群を意味しない。マイコンを含め、走査可能なProcess情報は同じ列形式で表示する。

## 7. 終了値

| 条件 | 終了値 |
|---|---:|
| Process一覧を最後まで出力 | 0 |
| 引数が余分 | 1 |
| Process情報取得または標準出力に失敗 | 1 |

カーネル内部関数はProcess状態を変更せず、走査終了をエラーと区別する。詳細エラー番号やerrnoは追加しない。

## 8. 確認項目

1. ホスト・マイコンの各ビルドで`user_binary("ps")`が同じ内蔵`ps`入口を返し、通常のProcessとして実行される。
2. PID 1、`ps`自身、未実行または終了待ちの子を含む使用中Processがslot順で表示され、`FREE`は表示されない。
3. `ps`自身の行に`*`が付き、`btx::sys_getpid()`の値と一致する。
4. `HOST_UCONTEXT`で複数の子を生成後に`ps`を実行すると、子ごとに異なるstack範囲が表示され、範囲が重ならない。guard範囲はstack範囲に隣接し、重ならない。
5. `NO_CONTEXT`や境界を提供しないマイコンbackendでは、Process一覧を表示し、stack/guard欄にAPIのsentinel値を出す。`ps`がsentinelを別表示へ変換しない。
6. Process return後にZOMBIEとして残る情報は表示されるが、回収済みContextのstack/guard範囲は表示されない。
7. 無効なProcessDebugCursorの終端処理、空のProcess table、最大Process数、全アドレス桁数を確認する。
8. 出力の部分writeとwrite失敗を確認する。Process table、current PID、FD参照数、Process状態が表示前後で変化しない。
9. 余分な引数でusageを出して終了値1を返し、Process table走査を行わない。
10. 最大長Process名を含む行が固定長buffer内で安全に生成され、名前が途中で切り詰められない。

## 9. 対象外

- ユーザー向け汎用`ps`、全Process情報取得syscall。
- scheduler statistics、CPU時間、実行時間、優先度、待機原因の詳細。
- stack使用量の推定、high-water mark計測、stack内容の走査。
- stack越境時のProcess回収やホスト致命faultからの復旧。
- アドレスのセキュリティ秘匿。これはデバッグ専用の内部コマンドである。
