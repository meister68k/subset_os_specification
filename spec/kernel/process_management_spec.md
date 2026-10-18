# プロセス管理仕様書 (`process_management_spec.md`)

## 1. 目的と位置づけ

本書はAncient UNIX風サブセットOSにおけるプロセス管理の基本構造、および今後の設計に対する制約を定める。現時点までに合意した項目を仕様とし、結果型・エラー表現・Schedulerとの詳細な接続等の未確定事項は区別する。

- 上位仕様：`subset_os_overview_spec.md`
- 起動順序・内蔵プログラム：`kernel_core_spec.md`
- FD管理・参照数：`open_file_description_spec.md`
- 共通実装規約：`AGENTS.md`

UNIX V7程度の外部機能を目標とするが、その内部実装の再現は要求しない。Cortex-M0・MC68000への移植性、MMUなし、メモリ2MB未満、固定長資源管理、簡潔なC++実装を重視する。例外・RTTIは使用しない。本文中のコードは、明記しない限りAPI確定ではない概念例である。

## 2. 基本構成（決定事項）

- プロセスは`Process`クラスで表現し、その内部状態を隠蔽する。`ProcessManager`が生成、終了、回収、PID割当、状態遷移を統括する。
- プロセス実体は固定長のプロセステーブルに格納する。テーブルの最大エントリ数はコンパイル時定数とし、具体値は未定。
- カーネル起動コンテキストは**Processとして登録しない**（現段階のB案）。`kernel::startup()`はProcessの外で実行する。
- 最初の通常ユーザープロセスは`init`とし、通常PID 1を与える。
- PIDはテーブル添字とは独立した循環割当方式を採用する。通常の自動割当はPID 1以上とする。PID 0は将来の特別用途のために確保し、有効なPIDとして表現できる設計にする。現段階ではPID 0のProcessを作らない。
- 各Processが自身の`FdTable`を持ち、マルチタスク対応段階では実行コンテキストも持つ。`FileDescription`本体はシステム全体で共有する。
- 実行中のProcessを指す内部参照（概念的にはポインタ）を管理し、通常のsyscallはこれを経由してプロセス固有のFD等へアクセスする。
- プロセスの管理責務（生成・状態・回収）と、実行対象を選ぶスケジューラの責務を分離する。
- さらにコンテキスト保存・復元の機構を独立した**Context層**に分離する。

### 2.1 構成と所有関係

```text
kernel::startup() （Process外）
    |
    +-- プロセステーブル（固定長）
    |     +-- Process（PID 1: init）
    |     |     +-- FdTable --+-- FileDescription（全体共有）
    |     |     +-- Context  |           |
    |     |                 |       FileInterface
    |     +-- Process ...   |
    |                       +-- 別ProcessのFdTableからも参照可能
    |
    +-- Scheduler（後続段階で詳細決定）
            |
            +-- Context層（CPU/ホスト依存の保存・復帰）
```

Processが持つコンテキストと、起動時・スケジューラ側のコンテキストは別物として扱う。Context層は、PID、プロセス状態、FD、スケジューリング方針を管理しない。

## 3. Processクラス

### 3.1 構造・カプセル化

`Process`はコピーによって複製しない。PID、親PID、状態、終了状態などの内部値を外部から自由に書き換えさせない。既存の`FileDescription`と同様、状態遷移や割当・解放は管理操作を通して行う。

合意した構成に基づく概念例は次のとおり。

```cpp
// 概念例：名称・メンバ・シグネチャは未確定
class Process {
public:
    Process() = default;
    Process(const Process&) = delete;
    Process& operator=(const Process&) = delete;

    pid_t pid() const;
    pid_t ppid() const;
    ProcessState state() const;
    WaitReason wait_reason() const;
    FdTable& fd_table();
    const FdTable& fd_table() const;
    uint32_t cwd_inode() const;

private:
    pid_t pid_ = -1;
    pid_t ppid_ = -1;
    ProcessState state_ = ProcessState::FREE;
    WaitReason wait_reason_ = WaitReason::NONE;
    FdTable fd_table_;
    uint32_t cwd_inode_; // Filesystemのcwd参照をProcessが1つ所有する
    // Contextは後続のコンテキスト切り替え段階で追加する。
    int32_t exit_status_ = 0;

    friend class ProcessManager;
};
```

この例は実際のコンストラクタや型の確定を意味しない。`Context`を値として保持するか、独立管理領域への参照にするか、スタックを誰が所有するかは未確定。`FdTable&`を公開する場合も、そのクラス自身の操作契約を迂回した操作を許す趣旨ではない。

### 3.2 プロセス状態（確定）

```cpp
enum class ProcessState : uint8_t {
    FREE, CREATING, READY, RUNNING, WAITING, EXITING, ZOMBIE
};

enum class WaitReason : uint8_t {
    NONE, CHILD, SLEEP, IO
};
```

| 状態 | 意味 |
|---|---|
| `FREE` | 未使用スロット |
| `CREATING` | 生成途中、資源準備未完了で実行不可 |
| `READY` | 実行可能、実行待ち |
| `RUNNING` | 実行中 |
| `WAITING` | 子終了等を待つ。理由は別の`WaitReason`で保持 |
| `EXITING` | 終了処理中。資源解放が未完了 |
| `ZOMBIE` | 実行資源の解放完了、親の回収待ち |

`WaitReason`は`ProcessState`と独立して保持する。初期段階は`NONE`と`CHILD`を実装し、`SLEEP`と`IO`は後続段階で使用する。通常の非待機状態では`NONE`を用いる。

主な状態遷移は以下とする。

```text
FREE -> CREATING -> READY -> RUNNING
          |           ^        |  |
          +--失敗--> FREE      |  +--> EXITING -> ZOMBIE -> FREE
                      READY <--+             （waitで回収）
                         ^     |
                         +-- WAITING <-------+
                             （待機・起床）
```

- `spawn()`の全準備成功後に限り`CREATING -> READY`とする。生成失敗は確保資源をロールバックし`FREE`へ戻す。生成失敗した子をZOMBIEとして残さない。
- 以下のSchedulerによる状態遷移は、独立ContextとSchedulerを導入した実行段階に適用する。初期同期段階では`READY`はspawn済み・未実行を意味し、詳細は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。
- Schedulerは`READY`を選択して`RUNNING`とし、実行権移譲では`RUNNING -> READY`、ブロック時は`RUNNING -> WAITING`、起床時は`WAITING -> READY`とする。
- `exit(status)`開始で`RUNNING -> EXITING`とし、渡された`int32_t`値を終了ステータスとして保存する。最後のスタック・Context等の資源回収をスケジューラ側コンテキストに移してから完了し、`EXITING -> ZOMBIE`へ遷移させる。`exit()`は呼び出し元へ復帰しない。終了値をマスク、符号化、または狭い型へ切り詰めない。
- `EXITING`は実行選択・`wait()`回収・スロット再利用の対象外。親の起床は`ZOMBIE`確定後とする。
- `ZOMBIE`はPID、親PID、終了ステータスを残し、実行資源を保持しない。親の`wait()`で回収して初めて`FREE`となる。
- 状態変更は`ProcessManager`に集約する。排他制御やプリエンプティブ/マルチプロセッサ対応の詳細は未確定。

## 4. プロセステーブル・PID

### 4.1 プロセステーブル

`ProcessManager`が固定長の`std::array<Process, MAX_PROCESSES>`相当のテーブルを所有する。空き枠とPIDの検索は線形探索を基本とする。`MAX_PROCESSES`の値は未確定。汎用管理フレームワークは導入しない。

### 4.2 PIDの原則

- PIDの割当はテーブル添字に依存しない。
- 通常の自動割当は1以上を循環し、既存のPIDと衝突させない。未回収の`ZOMBIE`に付与されたPIDも使用中とする。
- `spawn()`および`wait()`のsyscall戻り値は共通ABIの`int32_t`であるため、有効な正のPIDは`INT32_MAX`以下に制限する。内部`pid_t`の具体型は未確定だが、syscall境界でPIDを`int32_t`へ安全に表現できなければならない。
- PID 0を表現可能にし、特殊な用途のため予約する。当面、自動割当しない。
- `init`にPID 1を与えることを通常の起動構成とする。
- 内部`pid_t`の符号・幅、無効PIDの表現、PID循環割当の具体的な探索・オーバーフロー処理は後日確定する。syscallに公開するPID値は上記の`int32_t`範囲に収める。
- PIDの世代番号付与などは現段階では導入しない。再利用されたPIDを長期間保持する利用者への保証も未定義。

将来の`kill()`において、UNIX互換の`kill(0, sig)`のような特別な意味を導入するかどうかは別途決定する。PID 0が表現可能であることだけをもって、各syscallの引数0の意味まで確定しない。

## 5. 現在実行中のProcessの参照

`ProcessManager`が現在実行中のProcessへの内部ポインタ等を1つ保持し、一般コードに直接公開せず`current()`を経由して取得する。単一CPUを前提とし、ホストOSのスレッドローカルストレージを必須としない。

```cpp
// 概念上のアクセスAPI（確定宣言ではない）
Process& ProcessManager::current();
```

`current()`は**実行中のProcessが必ず存在する文脈で呼ぶ**ことを前提とする。`startup()`中などProcessがまだ存在しない状態では呼ばない。この前提を保証する内部処理、およびProcessが存在しないことを検査するAPIの要否は実装時に決める。

現在のProcessの切替管理はスケジューラ／プロセス管理の責任とし、Context層には引き受けさせない。Process導入後のsyscallのFD処理は`manager.current().fd_table()`を使う。Process導入前の単一`FdTable`方式は旧段階の実装として扱う。

## 6. Context層と実行スタック

### 6.1 確定方針

- プロセスごとに**独立した実行スタック**を用意する。
- ホストLinux環境は`ucontext`（`getcontext()`、`makecontext()`、`swapcontext()`等）により初期化と協調的な実行切替を実現する。
- `setjmp()/longjmp()`を共通のプロセス切り替え機構として使用しない。
- Cortex-M0とMC68000ではそれぞれのCPUに合わせたContext層の実装を設ける。
- `ucontext_t`などのホスト依存型をProcessの一般公開APIに露出させない。
- Context層は実行コンテキストの初期化・保存・復帰を扱い、PIDやFD等のプロセス管理を扱わない。

スタックの確保場所・容量・ガード機構・所有者、およびContext層のAPIや具体的な内部型は未確定。ホスト実装での`ucontext`動作確認も今後の作業とする。将来のプリエンプティブ切替には割り込みやシグナル等の追加設計が必要であり、本書は協調方式からの移行を保証する詳細仕様をまだ規定しない。

## 7. プロセス生成と内蔵ユーザーバイナリ

### 7.1 spawnの方針

`spawn()`は指定プログラムを新規プロセスとして開始する。親アドレス空間の複製を行う純粋な`fork()`は実装しない。`vfork()`の採否は未確定であり、現段階で実装対象に加えない。初期内部APIは内蔵バイナリの関数ポインタを受け取る。初期同期段階における引数、スタックを作らずwait時に子を実行する動作、生成失敗時の処理は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に定める。Scheduler導入後の起動時メモリ割当・実行イメージの詳細は別途検討する。

### 7.2 内蔵バイナリの維持

既存の内蔵ユーザーバイナリ機構は、ファイルシステムが使用できないときにもプログラムを起動できる手段として**維持**する。

```cpp
namespace kernel {
    using UserBinaryType = int (*)(int, const char**);
    constexpr UserBinaryType user_binary(const char* name);
}
```

`user_binary()`は名前から既存の静的リンク済み入口関数を検索し、未登録なら`nullptr`を返す。検索自体はProcess生成やFD操作を行わない。現在の`init`直接呼び出しは暫定実装とし、将来はProcessの独立スタック上で呼び出す構成を検討する。内蔵バイナリが共有アドレス空間で動作することと、完全なプロセス間メモリ隔離は別問題である。

ファイルシステムからロードするユーザーバイナリについては全体仕様の既存技術検証・制約（ホストの位置非依存コード、`kvec`/`dispatch`方式など）を参照し、本書では実行形式を新たに確定しない。

## 8. FD機構との接続

各Processが固定長`FdTable`を所有し、システム全体の`FileDescription`を参照する。`FdTable`はfd番号の対応付けだけを管理し、参照数を操作しない。`FileDescription`の参照数は既存の専用APIで管理する。

```cpp
OptionalFileRef alloc_file_description(FileInterface& target, uint32_t flags);
bool retain_file_description(FileDescription& desc);
void release_file_description(FileDescription& desc);
```

`spawn()`でFDを継承する際は、親子の各fdが**同一FileDescription**を参照する。継承したfdごとに`retain_file_description()`を行い、テーブルの単純コピーで代用しない。共有するFileDescriptionのfile positionも共有される。継承途中の失敗時は子に登録済みのFDを`detach()`し、対応する参照を`release_file_description()`してロールバックする。FDごとにretainと対応する解放を正しく行う。`exit()`時には全FDを切り離して参照を解放する。詳細な例外的失敗・cleanupの扱いはFD仕様に従う。

## 8.1 cwdの所有と寿命

- 各Processはcwd inode番号を保持し、そのinodeへの参照を1つ所有する。
- Filesystemはcwd参照を固定長表で管理する。容量、retain/release操作、orphan時の規則は[`mkdir_rmdir_implimentation_spec.md`](mkdir_rmdir_implimentation_spec.md)に従う。
- initのcwdは`ROOT_INODE`とし、Process公開前にFilesystem側の参照をretainする。
- `spawn()`は親cwd inodeを継承し、子Processを公開する前に参照をretainする。失敗時は部分生成した参照を戻す。
- `chdir()`は[`chdir_implementation_spec.md`](chdir_implementation_spec.md)に従い、新cwdをretainしてからProcessの値を更新し、旧cwdをreleaseする。旧cwd解放後のfinalize failureでは新cwdを維持して`-1`を返す。
- `exit()`はProcessが終了処理中にcwd参照をreleaseする。Filesystem参照解除に失敗した場合はProcess slotを再利用せず、既定の資源解放失敗扱いとする。
- `rmdir()`後もcwd参照を持つProcessは、そのdirectoryを基点としたpathname操作を続けられる。directoryのfinalize条件はFilesystem仕様に従う。

## 9. カーネル起動と初期Process

### 9.1 第1段階：FD・read/write検証用の単一Process起動（決定事項）

本段階ではContext層、`ucontext`、Scheduler、独立スタック、通常の`spawn()`を実装しない。`ProcessManager`の固定長テーブルに`init`用のProcessを1個だけ作り、従来の起動スタック上で内蔵`init`関数を直接呼び出す。ここでのProcessはFD等を所有する論理的な管理単位であり、実行コンテキストの切り替え対象ではない。

- 起動時専用の内部API `bool ProcessManager::initialize_init_process()` を設ける。これは当面の特別な起動経路であり、通常の`spawn()`の代用品ではない。
- 初期化成功時、PIDは1、PPIDは0、状態は`RUNNING`、待機理由は`NONE`とする。Processの`FdTable`は全エントリ未使用とする。
- Filesystemがmount済みの場合、`cwd_inode = ROOT_INODE`を設定し、Filesystemのcwd参照をretainしてからProcessをcurrentとして公開する。
- `current_`をPID 1のProcessへ設定し、通常のFD系syscallが`current().fd_table()`を使用できるようにする。
- Context・スタックの初期化は行わない。初期実装の`Process`型からContextメンバを一時的に省略してよい。
- `RUNNING`は、この段階では`init`が現在実行される論理プロセスであることを示し、Schedulerによる選択を意味しない。
- 初期化に失敗した場合は`init`を実行しない。失敗処理の具体的な戻り値・停止処理は未確定とし、部分的に初期化されたProcessをsyscallから参照可能にしない。
- これは単一Process起動段階に限る特例であり、通常の`spawn()`は既定どおり`CREATING -> READY`を経由する。

起動順序の正本は`kernel_core_spec.md`とする。概念上は、HAL・デバイス・FileDescriptionの準備後にPID 1のProcessを初期化し、その`FdTable`に`initialize_standard_fds()`でfd 0/1/2を設定してから、`user_binary("init")`で取得した入口関数を直接呼ぶ。fd 0/1/2はそれぞれ独立した`FileDescription`を使用し、同じ`ConsoleDevice`実体を参照できる。

`sys_read()` / `sys_write()`は現在Processの`FdTable::get(fd)`から`FileDescription`を取得し、既存の`FileDescription::read()` / `write()`へ委譲する。プロセス管理やFD対象の内部状態をsyscall層から直接変更しない。アクセスモード・不正fdの扱いは`open_file_description_spec.md`に従う。

本段階では、通常のプロセス終了APIではなく、直接呼び出した`init`からの復帰をカーネル側で処理する。戻り値や停止の具体的な処理は従来どおり未確定であり、`exit_current()`が機能するとはみなさない。

### 9.2 後続段階への移行

将来、通常の`spawn()`とSchedulerを導入した段階では、ProcessにContextと独立スタックを追加し、`init`の起動も通常のプロセス生成へ統合することを検討する。内蔵ユーザーバイナリの検索機構は維持する。LinuxホストのContext層には`ucontext`を採用し、カーネル起動／Scheduler側のコンテキストはProcessのコンテキストと区別する。

## 10. ProcessManager内部API（合意事項）

### 10.1 層の責務

syscall層は引数検証・変換、内部APIへの委譲、結果・エラー変換を中心とする薄いラッパとし、プロセス管理の実処理・状態遷移は`ProcessManager`へ集約する。カーネル起動側からもsyscallを経由せずに内部APIを利用する。Schedulerは実行対象を選択し、Context層はコンテキストを切り替える。

### 10.2 公開API

以下は合意済みのAPI構成を示す。`ProcessResult`と`WaitResult`の具体的な型、エラー表現、`spawn()`の引数保持方法は未確定であり、そのままコンパイル可能なヘッダを意味しない。

```cpp
using ProcessRef = std::reference_wrapper<Process>;
using OptionalProcessRef = std::optional<ProcessRef>;
using ConstProcessRef = std::reference_wrapper<const Process>;
using OptionalConstProcessRef = std::optional<ConstProcessRef>;

class ProcessManager {
public:
    ProcessManager() = default;
    ProcessManager(const ProcessManager&) = delete;
    ProcessManager& operator=(const ProcessManager&) = delete;

    OptionalProcessRef find(pid_t pid);
    Process& current();

    ProcessResult spawn(
        UserBinaryType entry,
        int argc,
        const char* const argv[],
        OptionalConstProcessRef parent
    );

    [[noreturn]] void exit_current(int32_t status);
    WaitResult wait_child();
    bool reap_child(pid_t child_pid);

private:
    // Schedulerとの接続：名称と詳細契約は暫定
    void activate(Process& process);
    void suspend(Process& process);
    void wakeup(Process& process);
    void finish_exit(Process& process);

    std::array<Process, MAX_PROCESSES> processes_{};
    pid_t next_pid_ = 1;
    Process* current_ = nullptr;

    friend class Scheduler;
};
```

通常のカーネル利用者に対する公開メソッドは、`find()`、`current()`、`spawn()`、`exit_current()`、`wait_child()`を基本とする。syscall境界で終了値の出力に成功してから子を回収するため、`reap_child(pid_t)`もProcessManagerの内部APIとして設ける。このAPIは指定PIDが呼出し元の子であり、まだ`ZOMBIE`である場合だけ`FREE`へ遷移させる。不一致や状態不整合の場合は失敗し、Process状態を変更しない。`current()`はProcessが存在する文脈でのみ呼ぶ。Scheduler向けの状態更新は一般公開せず、詳細はScheduler仕様と合わせて決定する。内部表現の生ポインタは一般APIへ公開しない。

通常の`spawn()`は`std::cref(manager.current())`等を親として渡す。起動時の`init`生成では`std::nullopt`を指定し、`ppid = 0`とする。`std::nullopt`は「親Processなし」を表し、PID 0のProcessの参照を意味しない。子が起動時に使う`argv`の有効期間は親のスタックに依存させない。

### 10.3 生成・終了・待機の内部処理

- **生成**：独立ContextとSchedulerを導入した段階では、`FREE`スロットを`CREATING`として予約し、PID・親子関係、プログラム・スタック・Context、FD継承を準備する。すべて成功したら`READY`とする。初期同期段階ではスタック・Contextを作らず、詳細仕様に従い`READY`（未実行）にする。途中失敗では確保資源を元に戻して`FREE`へ。PIDの欠番は許容する。
- **終了**：`exit_current(status)`で`EXITING`へ遷移し、`status`を`int32_t`のまま終了値として保存する。FDを解放し、孤児となる子（ZOMBIEを含む）を原則PID 1へ引き継ぐ。Schedulerへ非復帰で切り替えた後、`finish_exit()`相当の処理で終了プロセスが使っていたスタック・Context・実行イメージを回収し、PID・PPID・終了値を残した`ZOMBIE`へ遷移させて親を起床させる。`sys_exit(status)`は`exit_current(status)`へ委譲し、復帰しない。
- **待機**：`wait_child()`は成功時に子PIDと終了値を返すが、その時点では子を`ZOMBIE`のまま保持する。`sys_wait(status_ptr)`は`status_ptr == nullptr`でない場合、syscall境界で出力先を検証し、安全なuser-copy経路で終了値を書き込む。書き込みが成功した後に限り`reap_child(pid)`を呼び、子を`FREE`へ戻す。出力先の検証または書き込みに失敗した場合は`-1`を返し、子を`ZOMBIE`のまま残して次回waitで回収可能にする。`status_ptr == nullptr`なら終了値の出力を省略し、回収を確定する。子がいなければ`ECHILD`相当のエラー、未終了の子だけなら`WAITING(CHILD)`としてスケジューラへ移譲する。初期同期段階では、詳細仕様に従い最古の未実行子を関数呼出しで実行し、結果出力成功後に回収する。特定PID指定の`waitpid()`は現段階で追加しない。
- **init終了**：PID 1のinit終了時は通常の孤児引継ぎ・回収処理と区別し、カーネル側の停止処理へ移行する。停止の詳細は未確定。

### 10.4 今後決定する事項

1. `ProcessResult`、`WaitResult`の構造とエラー表現、syscall ABIへの変換。
2. `pid_t`の具体的なABI、PID上限と循環探索、`MAX_PROCESSES`の数値。
3. スタック・Contextの所有形態、実行イメージ・`argc/argv`の保管、外部バイナリのロード方法。
4. Schedulerとの切替API・`friend`範囲、待機や起床の詳細契約、停止処理。
5. `sleep()`、シグナル、パイプの待機理由と将来のプリエンプティブ・マルチプロセッサ対応。
6. 標準FD初期化時のcleanup失敗の詳細（初期Processへの直接登録方式は9.1節で確定）。

## 11. 最小実装の段階

第1段階のみ実装順序を確定する。後続段階は引き続き候補である。

1. **確定：** PID 1用の最小`Process`、固定長テーブル、`initialize_init_process()`、`current()`、プロセス所有`FdTable`、標準FD設定と`read/write`の検証。ContextとSchedulerは不要。
2. **後続候補：** Context・独立スタックを導入し、内蔵`init`を切り替え可能なProcessとして起動する。
3. **後続候補：** 通常の`spawn()`、FD継承、PID循環割当、ロールバック。
4. **後続候補：** `exit()`・`wait()`、ZOMBIE回収。
5. **後続候補：** 協調スケジューリング、`sleep()`、パイプ。
