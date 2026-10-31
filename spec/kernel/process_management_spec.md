# プロセス管理仕様書 (`process_management_spec.md`)

## 1. 目的と位置づけ

本書はAncient UNIX風サブセットOSにおけるプロセス管理の基本構造、および今後の設計に対する制約を定める。現時点までに合意した項目を仕様とし、ProcessManager内部の具体的な結果型・エラー表現等の未確定事項は区別する。Schedulerの動作契約は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)を正とする。

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
- 各Processは生成時に指定された実行名を固定長で保持する。`spawn(name, argv, inherit_fds, inherit_fd_count)`では`name`をProcess名とし、`argv[0]`とは独立して扱う。初期Process `init`の名前は`init`とする。名前の最大長とコピー規則は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。
- 実行中のProcessを指す内部参照（概念的にはポインタ）を管理し、通常のsyscallはこれを経由してプロセス固有のFD等へアクセスする。
- プロセスの管理責務（生成・状態・回収）と、実行対象を選ぶSchedulerの責務を分離する。協調切替・選択順・待機連携は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)に従う。
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
    +-- 協調Scheduler（[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)）
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
    char name_[128]; // spawn nameのコピー。上限と初期化規則はspawn/wait仕様に従う
    ProcessState state_ = ProcessState::FREE;
    WaitReason wait_reason_ = WaitReason::NONE;
    FdTable fd_table_;
    uint32_t cwd_inode_; // Filesystemのcwd参照をProcessが1つ所有する
    // ContextはHOST_UCONTEXTで有効。実装型はContext層に隠す。
    int32_t exit_status_ = 0;

    friend class ProcessManager;
};
```

この例は実際のコンストラクタや型の確定を意味しない。Contextの共通内部表現とビルド時の有効化条件は[`process_context_implementation_spec.md`](process_context_implementation_spec.md)に従う。`FdTable&`を公開する場合も、そのクラス自身の操作契約を迂回した操作を許す趣旨ではない。

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

`WaitReason`は`ProcessState`と独立して保持する。協調Schedulerは`NONE`、`CHILD`、`SLEEP`に加え、[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)に定めるpipe read/writeの`IO`待機を実装する。device I/Oの`IO`待機は対象外とする。通常の非待機状態では`NONE`を用いる。

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
- Scheduler有効時の状態遷移、dispatchとwait連携は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)に従う。Scheduler無効の`NO_CONTEXT`互換段階では`READY`はspawn済み・未実行を意味し、wait駆動方法は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。
- Schedulerは協調Scheduler仕様に従い`READY`を選択して`RUNNING`とし、yieldでは`RUNNING -> READY`、blockでは`RUNNING -> WAITING`、起床時は`WAITING -> READY`とする。
- `exit(status)`開始で`RUNNING -> EXITING`とし、渡された`int32_t`値を終了ステータスとして保存する。Scheduler有効時はScheduler Contextへ戻った後に実行資源を回収して`ZOMBIE`へ遷移させる。`exit()`は呼び出し元へ復帰しない。終了値をマスク、符号化、または狭い型へ切り詰めない。
- `EXITING`は実行選択・`wait()`回収・スロット再利用の対象外。親の起床は`ZOMBIE`確定後とする。
- `ZOMBIE`はPID、親PID、終了ステータスを残し、実行スタック・Context等の実行資源を保持しない。親の`wait()`で回収して初めてProcess枠が`FREE`となる。
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

`ProcessManager`が現在実行中のProcessへの内部ポインタ等を1つ保持し、一般コードに直接公開せず`current()`を経由して取得する。Scheduler Context上ではProcessが実行主体ではないため、現在Process参照を空にする。単一CPUを前提とし、ホストOSのスレッドローカルストレージを必須としない。

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

ホストLinuxの協調Context、独立スタック、ガードページ、Context無効の互換モードは[`process_context_implementation_spec.md`](process_context_implementation_spec.md)で定める。協調Schedulerの選択・dispatch規則は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)で定める。将来のプリエンプティブ切替には割り込み等の別設計が必要である。マイコン向けContext実装はCPU固有バックエンドとして後続仕様で定める。

## 7. プロセス生成と内蔵ユーザーバイナリ

### 7.1 spawnの方針

`spawn()`は指定プログラムを新規プロセスとして開始する。親アドレス空間の複製を行う純粋な`fork()`は実装しない。`vfork()`の採否は未確定であり、現段階で実装対象に加えない。初期内部APIは内蔵バイナリの関数ポインタを受け取る。引数、生成失敗時の処理、Scheduler無効のwait駆動互換動作は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)、Scheduler有効時の動作は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)に従う。

### 7.2 内蔵バイナリの維持

既存の内蔵ユーザーバイナリ機構は、ファイルシステムが使用できないときにもプログラムを起動できる手段として**維持**する。

```cpp
namespace kernel {
    using UserBinaryType = int (*)(int, const char**);
    constexpr UserBinaryType user_binary(const char* name);
}
```

`user_binary()`は名前から既存の静的リンク済み入口関数を検索し、未登録なら`nullptr`を返す。検索自体はProcess生成やFD操作を行わない。初期同期段階では`init`を起動Contextから直接呼ぶ。協調Scheduler段階では`init`も独立Context上で実行する。内蔵バイナリが共有アドレス空間で動作することと、完全なプロセス間メモリ隔離は別問題である。

ファイルシステムからロードするユーザーバイナリについては全体仕様の既存技術検証・制約（ホストの位置非依存コード、`kvec`/`dispatch`方式など）を参照し、本書では実行形式を新たに確定しない。

## 8. FD機構との接続

各Processが固定長`FdTable`を所有し、システム全体の`FileDescription`を参照する。`FdTable`はfd番号の対応付けだけを管理し、参照数を操作しない。`FileDescription`の参照数は既存の専用APIで管理する。

```cpp
OptionalFileRef alloc_file_description(FileInterface& target, uint32_t flags);
bool retain_file_description(FileDescription& desc);
void release_file_description(FileDescription& desc);
```

`spawn()`が継承するfd番号は`inherit_fds`と`inherit_fd_count`で指定する。`nullptr, 0`は親の全open fd、非NULL pointerとcount 0は継承なし、それ以外は指定されたfdだけを同じ番号で継承する。親子の各継承fdは**同一FileDescription**を参照する。継承したfdごとに`retain_file_description()`を行い、テーブルの単純コピーで代用しない。共有するFileDescriptionのfile positionも共有される。継承途中の失敗時は子に登録済みのFDを`detach()`し、対応する参照を`release_file_description()`してロールバックする。`exit()`時には子の全FDを切り離して参照を解放する。詳細な検証と失敗条件は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。

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

Contextを生成しない`NO_CONTEXT`互換モードでは従来の同一スタック・wait駆動実行を維持する。`HOST_UCONTEXT`で協調Schedulerを有効にする段階では、`init`を含む各Processに独立Contextとスタックを用意し、`kernel::startup()`からScheduler run loopへ入る。`init`入口をカーネル起動Contextから直接呼び出さない。Context仕様とScheduler仕様に従う。

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
        const int32_t* inherit_fds,
        uint32_t inherit_fd_count,
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

通常のカーネル利用者に対する公開メソッドは、`find()`、`current()`、`spawn()`、`exit_current()`、`wait_child()`を基本とする。syscall境界で終了値の出力に成功してから子を回収するため、`reap_child(pid_t)`もProcessManagerの内部APIとして設ける。このAPIは指定PIDが呼出し元の子であり、まだ`ZOMBIE`である場合だけ`FREE`へ遷移させる。不一致や状態不整合の場合は失敗し、Process状態を変更しない。`current()`はProcessが存在する文脈でのみ呼ぶ。Scheduler向け状態変更とdispatchの境界は[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)に従う。内部表現の生ポインタは一般APIへ公開しない。

通常の`spawn()`は`std::cref(manager.current())`等を親として渡す。起動時の`init`生成では`std::nullopt`を指定し、`ppid = 0`とする。`std::nullopt`は「親Processなし」を表し、PID 0のProcessの参照を意味しない。子が起動時に使う`argv`の有効期間は親のスタックに依存させない。

### 10.3 生成・終了・待機の内部処理

- **生成**：`FREE`スロットを`CREATING`として予約し、PID・親子関係、プログラム・引数・FD継承を準備する。Scheduler有効時は専用スタックとContextも初期化し、全準備成功後に`READY`とする。`NO_CONTEXT`ではスタック・Contextを作らず、wait時に親スタック上で実行する。途中失敗では確保資源を元に戻して`FREE`へ。PIDの欠番は許容する。
- **終了**：`exit_current(status)`で`EXITING`へ遷移し、`status`を`int32_t`のまま終了値として保存する。FDを解放し、孤児となる子（ZOMBIEを含む）を原則PID 1へ引き継ぐ。Scheduler有効時はScheduler Contextへ戻った後に終了したProcessのContext/stackを解放し、`ZOMBIE`として確定して待機中の親を起床させる。`reap_child()`は終了情報とProcess枠を回収する。`NO_CONTEXT`では子入口のreturnを終了として扱う。`sys_exit(status)`はContext有効時に`exit_current(status)`へ委譲し、復帰しない。
- **待機**：`wait_child()`は成功時に子PIDと終了値を返すが、その時点では子を`ZOMBIE`のまま保持する。`sys_wait(status_ptr)`は`status_ptr == nullptr`でない場合、syscall境界で出力先を検証し、安全なuser-copy経路で終了値を書き込む。書き込みが成功した後に限り`reap_child(pid)`を呼び、子を`FREE`へ戻す。出力先の検証または書き込みに失敗した場合は`-1`を返し、子を`ZOMBIE`のまま残して次回waitで回収可能にする。`status_ptr == nullptr`なら終了値の出力を省略し、回収を確定する。子がいなければ`ECHILD`相当のエラー、未終了の子だけなら親を`WAITING(CHILD)`にしてSchedulerへ制御を渡す。wait駆動方式は`NO_CONTEXT`互換モードに限り最古の未実行子へ切り替える。特定PID指定の`waitpid()`は現段階で追加しない。
- **init終了**：PID 1のinit終了時は通常の孤児引継ぎ・回収処理と区別し、カーネル側の停止処理へ移行する。停止の詳細は未確定。

### 10.4 今後決定する事項

1. `ProcessResult`、`WaitResult`の構造とエラー表現、syscall ABIへの変換。
2. `pid_t`の具体的なABI、PID上限と循環探索、`MAX_PROCESSES`の数値。
3. マイコンContextの内部表現、実行イメージ・`argc/argv`の保管、外部バイナリのロード方法。
4. MCU向けHAL timer/idle機構での正数`sleep()`有効化時期、およびScheduler run loop終了後の停止処理。ホストHALは[`host_hal_tick_wait_spec.md`](host_hal_tick_wait_spec.md)で定める。
5. シグナル、パイプの待機条件と将来のプリエンプティブ・マルチプロセッサ対応。
6. 標準FD初期化時のcleanup失敗の詳細（初期Processへの直接登録方式は9.1節で確定）。

## 11. 最小実装の段階

第1段階のみ実装順序を確定する。後続段階は引き続き候補である。

1. **確定：** PID 1用の最小`Process`、固定長テーブル、`initialize_init_process()`、`current()`、プロセス所有`FdTable`、標準FD設定と`read/write`の検証。ContextとSchedulerは不要。
2. **次段階：** `HOST_UCONTEXT`と`NO_CONTEXT`のビルド選択、および通常の`spawn()`子に対するContext・独立スタックを導入する。起動時`init`の専用Context化は別途扱う。
3. **後続候補：** FD継承、PID循環割当、Context生成を含むrollbackを確認する。
4. **後続候補：** `exit()`・`wait()`・ZOMBIE回収をwait駆動Context方式へ接続する。
5. **後続候補：** 協調Scheduler（[`cooperative_scheduler_spec.md`](cooperative_scheduler_spec.md)）、正数`sleep()`のMCU向けHAL機構、パイプ、マイコン固有Context backend。
