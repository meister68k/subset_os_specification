# 極小UNIX `load` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-LOAD-CMD-001 |
| 版 | 0.1（初版） |
| 対象 | サブセットOSのデバッグ用コマンド `load` |
| 関連仕様 | [`../kernel/loader_internal_api_spec.md`](../kernel/loader_internal_api_spec.md)、[`../kernel/host_elf32_subset_spec.md`](../kernel/host_elf32_subset_spec.md)、[`../kernel/spawn_wait_implimentation_spec.md`](../kernel/spawn_wait_implimentation_spec.md)、[`../kernel/shebang_execution_spec.md`](../kernel/shebang_execution_spec.md)、[`../kernel/process_context_implementation_spec.md`](../kernel/process_context_implementation_spec.md)、[`../kernel/process_management_spec.md`](../kernel/process_management_spec.md)、[`../kernel/cooperative_scheduler_spec.md`](../kernel/cooperative_scheduler_spec.md) |
| 状態 | 提案 |

## 1. 目的と位置づけ

`load`は、Filesystem上の実行イメージをロードして自Process内で実行するデバッグ用コマンドである。`spawn()`へのローダ統合（実行対象の解決・外部イメージのロード・実行）に先立ち、[`host_elf32_subset_spec.md`](../kernel/host_elf32_subset_spec.md)のロード機構（実行形式検査、W^Xを満たすmapping、`kvec`経由のsyscall dispatch、実行中のimage護持、実行後の解放）をsyscall面から独立に検証するための入口とする。

- `load`はデバッグ用コマンドであり、Lv.0〜Lv.2のsyscall一覧・4分類（全体仕様書1.2節・3.6節）には含めない。syscall番号・引数順・戻り値のABIを一切変更しない。
- `load`は既存の内蔵ユーザープログラム機構（`process_management_spec.md` §7.2）に登録された入口関数であり、`spawn("load", ...)`で起動する。全体仕様書4.1節の「静的リンクによる疑似ユーザープログラム」と同系統の立場（カーネル内部機能を検証する特別入口）であり、spawn統合完了後もホスト実装でのロード不調の切り分け用に維持する。
- `load`は実行対象の起動のために、カーネル内部のローダAPI（[`loader_internal_api_spec.md`](../kernel/loader_internal_api_spec.md)、実装想定名`guest_loader`）を直接呼べる**特例**とする。疑似プログラムと同様、この特例はデバッグ検証経路に限り、通常のuserlandプログラムが内部APIを呼ぶ根拠にはならない。
- `load`自体をguest build契約（elf spec §6）に従う独立イメージとしてFilesystemに置いた場合、その`load`はカーネル内部APIを呼ぶことができずロード経路として機能しない。検証対象として規定する`load`は内蔵プログラムである。コマンド名は既存の登録名（`cat`、`cp`、`ls`等）と衝突しない。

## 2. 起動形式

```text
load <pathname> [args ...]
```

- 位置引数はオプションとして解釈しない。第1引数`pathname`はFilesystem上の実行イメージを指す純粋なpathnameであり、`-`で始まる引数もpathnameとして扱う。`-`や`--`に特別な意味はなく、`-`単独はpathname `-`として解決する。
- `pathname`は呼出し元Processのcwdから相対解決し、絶対pathnameはroot基準とする。`PATH`検索は行わない。
- 第2引数以降`args`は、順序と内容を変更せずguestへ引き渡すguest引数である（§3）。
- 引数が0個、または`pathname`として空文字列を指定した場合はusage errorとする（§6）。
- 引数の個数・文字列長は`load`自身のspawn起動時点でspawn仕様の上限検査を通過済みである。`load`側で引数個数の独自上限を新設しない。

## 3. guest引数の構成

`load`の起動argvからguest引数argvを次のように構成し、内部APIへ渡す。

```text
guest_argv[0] = <pathname>（loadに指定された文字列をそのまま。正規化・絶対化しない）
guest_argv[1...] = <args ...>
guest_argc = loadの起動argc - 1
```

- guest引数の上限は[`spawn_wait_implimentation_spec.md`](../kernel/spawn_wait_implimentation_spec.md) §2と同値（要素数最大16、すべての文字列の終端NULを含む合計格納量最大256 byte）とする。要素数・合計格納量・終端NULの検査は内部API側の責務とする（[`loader_internal_api_spec.md`](../kernel/loader_internal_api_spec.md) §6）。
- `load`の起動argv自体がspawnの上限検査（最大16要素、256 byte）を済ませており、guest argvはその真部分集合である。したがって`load`経由で到達し得るguest引数は最大15要素であり、guest argvの上限ちょうど（16要素）・256 byteちょうど・超過の境界テストは`load`では実施できない。これらの境界は内部APIの受入テストで16要素/256 byteを直接渡して行う（[`loader_internal_api_spec.md`](../kernel/loader_internal_api_spec.md) §10）。
- 空文字列のguest引数は保持する（spawn argv契約と同一の取り扱い）。

## 4. 動作

### 4.1 処理順序

`load`の処理は内部API1回の呼出しで完結し、`load`自身はFilesystemのfdを一切開かない。open・読込・検査・closeはすべて内部API側で行い、それらで開いたfdはguestからも`load`からも見えない（shebang spec §6の「内部で開いた実行対象ファイルは継承しない」契約に準じる方針）。

1. 内部APIはpathname対象をFilesystemで開き、実行形式検査に必要なheader・program headerを読み込む。
2. 実行形式検査は[`host_elf32_subset_spec.md`](../kernel/host_elf32_subset_spec.md) §3〜§5およびサイズ上限（`MAX_GUEST_IMAGE_SIZE`）に従う。これに違反する対象はすべてロード失敗とする。
3. mapping確保（RW・非実行）→ 読込 → R-X化の手順、`MAP_FIXED`相当による既存address上書き禁止、W+X状態を作らないこと、算術のoverflow事前検査もelf spec §7に従う。
4. 読込完了後、対象ファイルをcloseする。close失敗は、それまでの手順が成功していてもロード失敗とする（guestを起動せず、mappingを解放する）。
5. guestを呼出し元Process（`current()`）の実行文脈上で実行する（§5）。guest entryへはargc、guest argv、API vector（`kvec`）を渡す。entry ABIはelf spec §6と[`loader_internal_api_spec.md`](../kernel/loader_internal_api_spec.md) §5に従う。
6. guest entryの戻り値を`load`の終了値として返す。guestが実行を終えて非実行になったことを確認した後にのみ、image mappingを解放する（elf spec §8のimage寿命契約）。

### 4.2 実行対象の解決スコープ

`load`は実行対象の解決を一切行わない。

- 内蔵プログラム名照合を行わない。第1引数に`!echo`等の内蔵登録名を指定してもpathnameとして扱う。
- シバンを解釈しない。対象の先頭2 byteが`#!`（shebang spec §3と同じ先頭位置の判定）であるファイルは、実行形式不正としてロード失敗とする。シバン処理（interpreter解決・argv変換）は`spawn()`側の責務であり、`load`は外部イメージのロード機構のみを検証する。
- 再帰解決は行わない。実行形式検査に合格しなければそこで失敗とし、interpreterを求めて再度開き直すことはない。

### 4.3 失敗とrollback

ロード失敗時（guest起動に至らない場合）は次を満たすこと。

- すでに確保したimage mappingは解放する。読込済みデータを保持する領域を残さない。
- 対象ファイルのfdはcloseし、fd残骸を外部から観測可能な形で残さない（OSのfdテーブル検査で確認する）。
- ProcessのFD状態・cwd・fd tableの参照数を変化させない。`load`自身が開いたfdは存在しない。
- 失敗の原因をfd=2へ診断出力する（§6）。診断文言は実装自由とするが、対象pathnameを含める。
- `load`は§6のステータス（126/127）で終了し、通常のProcess終了経路に従う。失敗したProcessを外部から観測可能な中途半端状態に残さない。

guest実行開始後のrollback（guest実行中の障害、guest内でのSIGSEGV相当のhost例外）は対象外である。この領域の障害はprocess_context仕様 §7「スタック越境時の扱い」に準じる致命的扱いとし、`load`内での回復・保存・再実行の契約を設けない。

## 5. 実行モード・実行文脈

### 5.1 guestの実行文脈

guestは`load`を起動したProcess（`current()`）の実行文脈上で、新しいProcess・Context・スタックを作らずに動く。

- guestとProcessとで使うFD・cwdは同一である。guestがfd 0/1/2を経由する読み書きは通常どおりdispatchされる。
- guestは`load`と同一のProcessである。guest実行中もProcessの表示名は`load`を保つ（guestのpathnameへ置換しない）。debug出力上でguestが独立のProcessとしては現れないことを、`load`の既知の性質として受容する。guest実行内容とpathnameの対応が必要な場合は、`load`の起動argvのdebug出力で補う。
- guestが`spawn()`（内蔵名）を呼んだ場合も既存のspawn契約が適用される。guestから見える呼出し元は`load`のProcessである。
- guest内でのsyscall dispatchは`kvec`経由で既存のディスパッチ規約に従う。`.data`・`.bss`を持たないguest build契約（elf spec §6）は、`load`で起動する対象にもそのまま適用する。
- guestでの`sys_exit()`による非復帰終了は、本段階では保証しない。guestはentry returnで終了するguest ABIに従う（§9.1）。

### 5.2 `NO_CONTEXT`モード

`NO_CONTEXT`では、`load`のProcessを実行している親のスタック（wait駆動実行者）上でguest entryを通常の関数呼び出しとして実行する。疑似プログラムの実行方法と同様である。guestの暴走スタックは呼出し者を含むカーネル全体を破壊しうるが、このモードの既存の性質（子も親スタック上で実行する）と同等であり、`load`を中心に追加の保護契約を設けない。

### 5.3 `HOST_UCONTEXT`（Scheduler有効）モード

`HOST_UCONTEXT`では、`load`Process自身のContext・独立スタック（`HOST_PROCESS_STACK_SIZE`、既定128 KiB）上に、guest entryが1フレーム深く載る形で実行する。

- guest内で`sleep()`やpipe read/writeのblockを行った場合の待機は、Scheduler仕様の`WAITING(SLEEP)` / `WAITING(IO)`契約に従う。block時に`load`ProcessのContextを保存してSchedulerへ制御を戻し、起床後にguest実行を継続する。`load`を特別扱いしてblockを拒否しない。
- guestのスタック消費は`load`Processのスタック内で行われるため、guestに使える実質上限は`HOST_PROCESS_STACK_SIZE`から`load`自身のフレーム分を差し引いた量である。超過時はprocess_context仕様 §7のガードページにより致命的faultとなり、回収・再実行はしない。
- `load`自身もSchedulerからdispatchされる通常のProcessであることを前提とする。`NO_CONTEXT`ではこの節は適用されない。

## 6. 終了ステータス

guestの終了ステータスはmask・符号化・切詰めをせずそのまま透過する（spawn/wait仕様と同一の方針）。`load`自身の失敗は次の値で返す。

| 終了経路 | ステータス |
|---|---:|
| guest entryの戻り値 | その値（透過） |
| usage error（引数なし、`pathname`が空文字列） | `126` |
| ロード不能（open失敗、実行形式不正・シバン、guest引数上限超過、mapping/読込/close等の資源失敗） | `127` |

- 診断は標準エラーfd=2へ、接頭辞`load: `を付けた1行とする。診断自体のwrite失敗は追加処理せず、終了ステータスを変えない。
- guestの終了ステータスが`126`または`127`であった場合、それと`load`自身の失敗を`load`の戻り値だけから区別することはできない。デバッグ用コマンドとしてこの重複を受容する。区別が必要な場合はfd=2の診断と起動argvの記録等で補う。statusの再符号化・予約範囲の拡張は行わない。

## 7. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| カーネル内部（特例） | `guest_loader`相当の内部ローダAPI（`loader_internal_api_spec.md`） | 実行対象のopen/検査/mapping/実行/解放の全処理 |
| OSラッパ | `write()` | fd=2への診断出力 |
| ilibc | 既存の文字列出力補助 | 診断の組立て |

- `open()` / `read()` / `close()` / `stat()`を`load`のコマンド本体からは呼ばない。実行対象へのFilesystemアクセスは内部API内で完結させる。
- 動的メモリ確保をしない。ロード作業用の固定長bufferと引数受け渡し用の固定長領域で賄う。
- 診断メッセージには`load`に指定されたpathname文字列をそのまま含める（解決後の絶対pathnameへ変換しない）。
- ilibcの公開API追加は不要。

## 8. 受入テスト

guestイメージはelf spec §6のbuild契約に従い、`readelf`等によるbuild時検査を事前に行った既存の検証guestを使う。

1. 正常guestを`load`で起動でき、guestがfd=1へ出力した文字列が`load`の標準出力に現れる。
2. guest entryの戻り値が`load`の戻り値としてmaskなしで親に届く。0・正の値・負の値で確認する。
3. guest引数が`guest_argv[0] = pathname、guest_argv[1..] = args`の順で届き、空文字列の引数も保持される。
4. guest内からsyscall（例: `getpid()`、`read()`/`write()`）を呼べ、結果が通常のsyscall契約と一致する。`getpid()`は`load`のProcessのPIDを返す。
5. `load`を連続複数回起動しても、mapping残骸・fd残骸・Process参照残りが残らない。
6. guest実行中のimage mappingはR-Xであり、W+Xでない（`/proc/self/maps`等のhost情報で実測）。
7. 不正実行形式をすべて127で拒否する: 存在しないpathname、directory、先頭`#!`のファイル、不正magic / class / data / machine / type / version、program header構成違反（複数`PT_LOAD`、`PT_GNU_STACK`欠如、RWEの`PT_LOAD`、未知権限bit）、`p_vaddr`非ゼロ、`p_filesz`≠`p_memsz`、entry範囲外、offset/size overflow、`p_memsz` > `MAX_GUEST_IMAGE_SIZE`、再配置・未定義symbol・`.data`/`.bss`・dynamic/TLSを含むimage。各失敗でfd・mapping・Process参照を残さない。
8. Filesystem short read、mapping/protect失敗、対象ファイルclose失敗を127で扱い、guestを起動しない。close失敗後にmappingまで解放されることを確認する。
9. 引数なし・空文字列pathnameを126で拒否する。`-`、`-x`及び`--`引数もpathname/引数としてそのまま渡す。
10. `NO_CONTEXT`ビルドと`HOST_UCONTEXT`ビルドの双方で項目1〜9が成立する。`HOST_UCONTEXT`ではguest内の`sleep()`およびpipe blockを含めて確認する。
11. `load`経由ではguest引数が16要素ちょうどに到達不能であること（最大15要素）を実装・テスト観点として確認する。

## 9. 残課題・他仕様との整合

1. guestからの`sys_exit()`（非復帰）は、spawn/wait仕様の`exit()`接続の進展に合わせて`load`にも適用するかを確定する。現段階ではguest entry returnを唯一の終了契約とし、`sys_exit()`呼出しの結果は保証対象外である。
2. `spawn()`へのローダ統合時、`load`が用いたロード部分（open・検査・mapping・close）をspawnのpathname経路と共通化する。ロード部分の契約はelf spec §7準拠のままとし、両経路の差分（実行の紐付け対象がcaller Processか子Processか）は[`loader_internal_api_spec.md`](../kernel/loader_internal_api_spec.md) §8「spawn統合時の再利用」で扱う。
3. guest crash・スタック越境を`load`の受入テストに含めるかは、process_context仕様の越境試験の進展と合わせて決める。
4. `NO_CONTEXT` / `HOST_UCONTEXT`の両ビルドを規定内とする。モード差の実行条件は各仕様（process_context仕様・cooperative_scheduler_spec.md）に従う。
