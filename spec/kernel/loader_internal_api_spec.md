# ホスト環境 guest実行イメージ ローダ内部API仕様

| 項目 | 内容 |
|---|---|
| 文書ID | HOST-EXE-LOADER-API-001 |
| 版 | 0.1（初版） |
| 対象 | ホスト環境におけるFilesystem実行イメージのロードとcaller Process内実行を行うカーネル内部API（実装想定名`guest_loader`） |
| 関連仕様 | [`host_elf32_subset_spec.md`](host_elf32_subset_spec.md)、[`../command/command_load_spec.md`](../command/command_load_spec.md)、[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、[`shebang_execution_spec.md`](shebang_execution_spec.md)、[`process_context_implementation_spec.md`](process_context_implementation_spec.md)、[`process_management_spec.md`](process_management_spec.md) |
| 状態 | 提案 |

## 1. 目的と範囲

本書は、Filesystem上の実行イメージをロードして、現在実行中のProcess（`ProcessManager::current()`）の実行文脈上でguestを実行するカーネル内部APIの契約を定める。現段階の利用者はデバッグコマンド`load`（[`command_load_spec.md`](../command/command_load_spec.md)）ただ1つであり、`spawn()`のpathname経路（[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、[`shebang_execution_spec.md`](shebang_execution_spec.md)）の実装に先立ち、ロード機構をsyscall面から独立に検証するために設ける。

- 本APIはsyscallではない。syscall番号・syscall ABIの追加・変更を行わず、userlandから見えない。
- 実行イメージの形式・検査規則は[`host_elf32_subset_spec.md`](host_elf32_subset_spec.md)を規範とし、本書はその規則を繰り返さない。本書はAPIの形状、caller Process内での実行方式、failure分類、rollback契約を定める。
- 実行対象の解決（内蔵名照合、シバン、PATH探索）は本APIの責務ではない。本APIは純粋なFilesystem pathnameのみを受理する。
- 実装想定名`guest_loader`とデバッグコマンド`load`は音が似ているが共通の概念ではない。文書・コードで両者を区別して扱う。

## 2. APIの形状（概念例）

名称・シグネチャは未確定であり、確定しているのは以下の契約面のみである。

```cpp
// 概念例：名称・型は未確定。契約の意図を示すための概念
namespace kernel {
namespace guest_loader {

// ロード失敗の段階分類。guest実行開始後の障害ではこの値を返さない
enum class Stage : uint8_t {
    OPEN,       // pathname対象のopenが失敗
    ARGUMENT,   // pathname引数・guest argvの検査に失敗
    FORM,       // host_elf32_subset_spec.md §3〜§5の検査に違反
    RESOURCE,   // mapping・読込・protect・close等の資源操作に失敗
};

// 成功時：succeededが真でstatusにguest entryの戻り値。stageは無効
// 失敗時：succeededが偽でstageにfailure段階。statusは無効
struct RunResult {
    int32_t status;
    bool succeeded;
    Stage stage;
};

// pathnameをELF32 subset実行イメージとしてロードし、current()の実行文脈上でguestを実行する
RunResult load_and_run(const char* pathname, const char* const guest_argv[]);

}
}
```

- `pathname`と`guest_argv`はcallerが保持する文字列・配列であり、guest実行中もcaller側で生存させる。検査は本API側でも行う（§6）。
- 戻り契約は「成功時＝guest entryの戻り値をそのまま保持」「失敗時＝存在しないguest終了値を返さずfailure段階を保持」の2点とする。`RunResult`の表現（bool併用・optional相当表現・戻り値分離等）は実装時に確定する。
- 内部利用者およびunit testから、syscall経由を経ずに直接呼べる（`process_management_spec.md` §10.1の「カーネル起動側からもsyscallを経由せずに内部APIを利用する」に準じる層の考え方）。failure段階を保持するのは、どの段階で失敗したかの診断性のためである。

## 3. 呼出し条件

- 本APIは、`current()`のProcessが実行文脈を持ちsyscall dispatchが動く文脈で呼ぶ。`kernel::startup()`直後のようにProcessがまだ存在しない文脈からは呼ばない。`NO_CONTEXT`互換モードでは、通常のProcessの実行スタック上の呼出しがこれに当たる。
- 1回の呼出しは1つのguestをロードし実行して返る。本API自身が並行・重複ロードを行う機構を持たない。guest内から`spawn("load", ...)`等で生じる連鎖呼出しは独立のProcess・独立の呼出しとして扱う。
- 本APIはshebang・内蔵プログラム名を解決しない。callerのpathnameを純粋なFilesystem対象として扱い、解決は呼出し元Processのcwdを基準とする（[`command_load_spec.md`](../command/command_load_spec.md) §2に一致）。

## 4. ロード処理

ロード手順は[`host_elf32_subset_spec.md`](host_elf32_subset_spec.md) §7を規範とし、その対象がchild Processではなくcaller Processである点だけが差異である。

1. Filesystem APIでpathname対象を開き、実行形式検査に必要なheader・program headerを読み込む。読取には固定長bufferを用い、動的メモリを必要としない。
2. 形式・範囲・上限の検査を行う（elf spec §3〜§5、`MAX_GUEST_IMAGE_SIZE`）。offset加算・乗算・narrowingの前にoverflowを検査し、ELFはbyte列からfieldをdecodeして読む（ELF構造体をnative structとして直接読むことはelf spec §3が禁止している）。
3. `p_memsz` byte以上のmappingを確保する。mapping addressはhost OSに選ばせ、`MAP_FIXED`相当で既存addressを上書きしない。mappingはRW・非実行で作成し、`p_offset`から`p_filesz` byteを`image_base + p_vaddr`へ読み込む。
4. 読込完了後、mapping全体をR-Xへ変更する。W+X状態を作らない（elf spec §7と同一の契約）。
5. mapping・読込・protectが成功した後に限り、対象ファイルをcloseする。close失敗はguestを起動せずRESOURCE段階のfailureとする。
6. entry・guest argv・API vector（`kvec`）等の実行準備がすべて成功した後にguestを実行する。準備で1つでも失敗した場合は、確保済み資源を§7に従いrollbackする。

page rounding・mapping byte数・file offset・entry計算の算術はelf spec §7と同様、overflowを事前検査する。Filesystem short read・mapping/protection失敗・close失敗では、確保済みの資源を残さない。

## 5. caller Process内での実行

guest entryはcaller Process（`current()`）の実行文脈上で、通常の関数呼び出しとして実行する。新しいProcess・Context・スタックを用意しない。実行条件のcaller側の契約は[`command_load_spec.md`](../command/command_load_spec.md) §5と一致する。

- `NO_CONTEXT`では、本APIを呼び出したProcessの実行文脈（wait駆動実行者のスタック）上でguest entryを直接呼ぶ。
- `HOST_UCONTEXT`では、caller Process自身のContext・独立スタック（`HOST_PROCESS_STACK_SIZE`）上に、guest entryが1フレーム深く載る形で実行する。
- guest内のsyscallはentryへ渡すAPI vector（`kvec`）を経由して行われ、既存のsyscallディスパッチ規約に従う。dispatchは`current()`＝caller ProcessのFD table・cwdを参照する。プロセス管理層・syscall層への変更は行わない。
- guest entryの戻り値をguest終了値として保持し、この値を本APIの呼出し元へ返す。guestをScheduler等の別主体から実行させる機構を導入しない。
- guest実行中もcaller Processの表示名・FD table・cwd参照等のProcess状態を変化させない。Processのdebug出力上でguest実行中の表示名が`load`のままになる点は、caller仕様（[`command_load_spec.md`](../command/command_load_spec.md) §5.1）が受容する方針に従う。
- guestからの`sys_exit()`による非復帰終了は本段階の契約外である（`command_load_spec.md` §9.1参照）。本APIはguestをentry returnで終了させる前提であり、非復帰の実行切替を契約しない。

### 5.1 guestのスタック

- guestのスタック消費はcaller Processの実行スタック上で行われる。guestに使える実質領域はcallerのスタック容量（`HOST_PROCESS_STACK_SIZE`）からcaller自身のフレーム分を差し引いた大きさであり、guest側に個別の保証を設けない。
- 超過時の扱いは`HOST_UCONTEXT`のガードページ・`NO_CONTEXT`の保護なしを含めprocess_context仕様 §7に従い、本API内に独自のスタック検査を設けない。

## 6. 引数の検査

- `pathname`はNULLでなく、空文字列でなく、NUL終端されていなければならない。failure分類は§11.2を参照する。
- `guest_argv`はNULLでないNULL終端配列であり、`guest_argv[0]`はNULLでない。要素数は最大16、すべての文字列の終端NULを含む合計格納量は最大256 byteとし、[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md) §2と同値・同じboundaryとする。上限超過・NULL終端欠如・NULL `guest_argv[0]`はARGUMENT段階のfailureとする。
- 空文字列のguest引数は有効とする（spawn argv契約と同一の取り扱い）。

## 7. 失敗とrollback

- guest実行開始前に失敗した場合、本APIが確保・開いた資源はすべてrollbackする。mapping、file fd、ロード作業用の固定長領域を解放し、他の資源は残さない。rollbackは`NO_CONTEXT` / `HOST_UCONTEXT`のどちらのビルドでも同一の契約とする。
- close失敗もfailureとする。mapping・読込が完了していたとしても、guestを起動せずmappingを解放する。
- guest実行開始以降はfailure段階を返さない。guestがentry returnした時点で成功（succeeded）とし、戻り値をstatusとして保持する。実行中の障害・host例外は本APIの契約外であり、caller仕様（[`command_load_spec.md`](../command/command_load_spec.md) §4.3）およびprocess_context仕様の致命的扱いに従う。
- 失敗時にはcaller ProcessのFD table・cwd・fd参照数、他のProcessの状態を変化させない。cleanup中に他Processの資源を追加で確保・借用しない（失敗時のcleanupも新たな外部資源を残さない）。

## 8. spawn統合時の再利用

`spawn()`へのローダ統合時、本APIのロード部分（§4のopen・検査・mapping・close）をspawnのpathname経路と共通化する。その際、本書§4〜§6の契約（elf spec §7準拠のロード手順・overflow検査・byte列decode・固定長buffer・close失敗扱い）がspawn側でも保たれることを確認する。両経路の差分は次の表を今後の参照とする。

| 項目 | 本API（`load`経路） | spawn pathname経路 |
|---|---|---|
| 実行対象の解決 | 行わない（pathnameのみ） | 内蔵名照合→シバン→ELF（shebang spec §2〜§4） |
| 実行の紐付け | caller Processの実行文脈上で直接call | 子Processへimageを紐付け、Context/独立スタックを用意（elf spec §7・process_context仕様） |
| FD | `current()`のFD / cwdをそのまま使用 | spawn仕様のFD継承契約（`spawn_wait_implimentation_spec.md` §3.3） |
| 終了 | entry returnのstatusを本APIの戻り値として返す | 子Processの終了・wait経路（spawn/wait仕様） |
| rollback | 本書§7の契約（caller段階に限定） | spawn生成失敗契約（`CREATING`→`FREE`） |

本表は統合時の契約改変の有無（ロード部分が共通実装に対して本APIとspawnとで乖離していないか）を確認するための基準であり、本書は統合実装の設計を確定させない。

## 9. host依存の局所化

- `mmap()`・`mprotect()`等のhost OS機能と、その診断用のhost情報（`/proc/self/maps`等）はホストbackend内に閉じる。本APIの共通部分はこれらを直接呼ばない（elf spec §7「host固有`mmap()`/`mprotect()`はhost backend内に閉じる」に従う）。
- Cortex-M0 / MC68000等のtargetへの移植時は、guest imageの確保方式（物理address spaceへの直接読込、保護機構の有無）を含めてtarget側設計を別途定める。本書ではtarget側の実装方式を規定しない。

## 10. 受け入れ確認項目

1. 正常guestを本APIでロード・実行でき、guest entryの戻り値が成功statusとして保持される。
2. guest引数は要素数16ちょうど・合計256 byteちょうどで受理され、17要素・257 byte以上・NULL `guest_argv[0]`・NULL終端欠如がARGUMENT failureになる。
3. failure段階（OPEN / ARGUMENT / FORM / RESOURCE）が機能し、各failure段階でguest entryに進まない。
4. 実行中のmappingがR-XでありW+Xでないこと、連続複数回の成功・failureでmapping・fd残骸が残らないことを確認する。
5. `command_load_spec.md` §8で検証済みの項目を重複して再検証しなくてよい。本APIの受入テストは、guest引数上限・failure段階の境界を、caller経由で到達不能な値（guest引数16要素ちょうど等）を含めて直接検証する。
6. `NO_CONTEXT`ビルドと`HOST_UCONTEXT`ビルドの双方で上記が成立する。
7. 本APIのunit testがsyscall経由を経ずに、open・検査・rollbackの契約を検証できる。

## 11. 未決事項

1. `RunResult`の表現（bool併用・戻り値分離等）とfailure段階の保持方法。
2. 空文字列pathnameのfailure分類（OPEN段階 / ARGUMENT段階）。caller側のusage error扱いとの整合。
3. guestからの`sys_exit()`を含む非復帰終了の将来の契約。
4. target（Cortex-M0 / MC68000）環境での実装方式（image確保・保護機構）。
