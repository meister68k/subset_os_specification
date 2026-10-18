# カーネル共通仕様書 (`kernel_core_spec.md`)

## 1. 目的と位置づけ

本書は、Ancient UNIX風サブセットOSで、複数のサブシステムを横断する比較的小規模な**カーネルの動作仕様**を定める。

- `subset_os_overview_spec.md` は目標・全体構成・開発段階を定める。
- `AGENTS.md` は実装全体の共通規約を定める。
- 本書は、個別サブシステムに帰属しない協調動作と、その順序・前提条件・失敗時の扱いを定める。
- 個々のAPIの詳細は各サブシステム仕様書に置き、本書では必要な参照と呼出し契約だけを記載する。

現時点で本書が定義する機能は `kernel::startup()` の起動シーケンスおよび内蔵ユーザーバイナリ機構とする。今後、横断的で独立仕様書を作るほど大きくない機能が必要になれば章を追加する。未要求の共通機構は設計しない。

## 2. カーネル起動処理

### 2.1 役割

`kernel::startup()` はカーネルの初期化を順序付け、最初のユーザープログラム（`init`、当面は疑似ユーザープログラム）の実行開始までを統括する。

各サブシステムの内部初期化アルゴリズムを `startup()` に重複実装しない。起動後の通常のI/O処理は既存のFD機構を使用する。

### 2.2 前提条件

- `startup()` の実行開始前に、必要なC++ランタイム初期化と静的オブジェクトの構築が完了していること。特に登録テーブルが参照する静的デバイス実体が有効であること。
- `FileDescription`の固定長テーブルは、初回利用前にすべて `FREE`（`ref_count == -1`）であること。既存のデフォルト初期化で成立する場合、起動時に二重にクリアする必要はない。
- `startup()` は通常のカーネル起動経路から一度呼ばれる。再起動・再実行時の振る舞いは本書で定義しない。

### 2.3 起動順序

```text
（startup() 呼出し前）C++ランタイム・静的オブジェクト構築
    |
kernel::startup()
    |
    +-- HAL::init_hal()
    |
    +-- kernel::initialize_devices()
    |      +-- 非公開のG_char_devicesを登録順に走査
    |      +-- 各エントリのFileInterface::init()を呼出し
    |
    +-- kernel::initialize_root_filesystem()
    |      +-- G_root_filesystem を起動時固定構成で mount
    |      +-- mount失敗時はinitを起動しない
    |
    +-- FileDescriptionテーブルの使用準備を確認
    |
    +-- ProcessManager::initialize_init_process()
    |      +-- PID 1 / PPID 0 / RUNNING / WaitReason::NONE
    |      +-- Process所有の空FdTableを準備しcurrentへ設定
    |
    +-- initialize_standard_fds(current_process.fd_table(),
    |                           G_console_device,
    |                           G_console_device,
    |                           G_console_device)
    |      +-- fd 0: O_RDONLY
    |      +-- fd 1: O_WRONLY
    |      +-- fd 2: O_WRONLY
    |
    +-- kernel::user_binary("init") で内蔵プログラムを検索
    |      +-- 見つからなければ起動失敗
    |
    +-- argc=1, argv={"init", nullptr} でinitを直接呼び出す
    |
    +-- initから復帰した場合、root filesystemをunmount
           +-- G_root_filesystem.unmount()
           +-- unmount後にstartup()を終了
```

ここでの依存関係は次の通り。

1. HALを利用するデバイスの初期化より先に `HAL::init_hal()` を完了させる。
2. `kernel::initialize_devices()` をroot filesystemのmountおよび標準FDの初期化より先に完了させる。起動処理から個別デバイスの `init()` を重ねて呼ばない。
3. `kernel::initialize_root_filesystem()` は、root filesystemが利用するBackendに必要なデバイス初期化後に呼ぶ。成功するまではpathnameを伴うFilesystem APIをsyscall側から利用しない。
4. `initialize_standard_fds()` より先に、使用可能なFileDescriptionテーブル、PID 1のProcessが所有する空のFdTable、初期化済みのデバイスを用意する。
5. root filesystemのmountと標準FD初期化の両方が成功するまでは `init` のユーザーコードを実行しない。
6. 本実装では内蔵`init`が関数から復帰した場合をカーネル終了経路とみなし、`startup()`はroot filesystemをunmountしてから終了する。通常稼働中にroot filesystemをunmountしてはならない。

FD経路テストの新しい第1段階では、PID 1の最小Processを生成し、そのProcessの`FdTable`を使用する。Context層、`ucontext`、Scheduler、独立スタックおよび通常の`spawn()`はまだ要求しない。`init`は既存の呼び出しスタック上で直接実行する。詳細は`process_management_spec.md`の9.1節。

### 2.4 デバイス初期化と登録テーブル

- `kernel::initialize_devices()` は `kernel::find_char_device()` と同じデバイス管理実装ファイルに定義する。
- 非公開の `G_char_devices` を登録順に走査し、`FileInterface::init()` を仮想呼び出しする。`startup()` は登録テーブルの内部表現を直接操作しない。
- 同じ実体が複数のmajor番号に登録されている場合も、各エントリで `init()` を呼ぶ。多重初期化への対応はデバイスドライバの責務とする。
- デバイス登録は静的に定義され、起動時の動的登録は行わない。
- 現行の `initialize_devices()` は `void` を返す。共通の初期化失敗通知やロールバックは定義しない。

詳細は `device_driver_spec.md` を参照。

### 2.5 標準入出力の準備

初期構成では、単一の `G_console_device` をfd 0/1/2のI/O対象として共有する。ただし、**FileDescriptionは3個独立**して確保する。

| fd | アクセスモード | 初期I/O対象 |
|---:|---|---|
| 0 | `O_RDONLY` | `G_console_device` |
| 1 | `O_WRONLY` | `G_console_device` |
| 2 | `O_WRONLY` | `G_console_device` |

標準FDの設定は、以下の既存APIに委譲する。

```cpp
bool initialize_standard_fds(
    FdTable& fd_table,
    FileInterface& dev_stdin,
    FileInterface& dev_stdout,
    FileInterface& dev_stderr
);
```

引数はfd 0/1/2の順序とする。将来、標準入出力を別々のデバイスへ割り当てる場合でも、このAPIは変更しない。

標準FDを設定する際、major/minor番号の検索は必須ではない。`startup()` は既知のデバイス実体を直接渡してよい。`FdTable`での対応付け、`FileDescription`の確保・解放、途中失敗時のロールバックは `open_file_description_spec.md`（改訂版の該当仕様）に従う。

ホストOSのstderrはゲストOSのfd 2と同一視しない。ホストstderrはカーネル自身の診断出力などに使用できる。

### 2.6 失敗時の扱い

- `kernel::initialize_root_filesystem()` が `FsStatus::OK` 以外を返した場合、`init` を起動しない。
- `initialize_standard_fds()` が `false` を返した場合、`init` を起動しない。
- `user_binary("init")` が `nullptr` を返した場合も、呼び出しは行わない。失敗後の停止・後始末の詳細は未確定とする。
- `initialize_standard_fds()` の内部で今回確保・登録したFD／FileDescriptionを後始末する契約はFD仕様書に従う。cleanup不能のスロットは `RELEASING` に維持する。
- FD機構の完成前や障害時に必要な診断出力には、FDを経由しないカーネル用の出力経路（既存の `putstrk` 等）を用いる。
- `startup()` 自体の戻り値、致命的エラー後の停止方式、プロセス管理初期化失敗時の詳細な復旧処理は、関係する仕様が確定するまで未定とする。仮のエラーコード体系や再試行機構を導入しない。
- `init`復帰後のroot filesystem unmount失敗は起動失敗ではなく終了処理失敗として扱う。Filesystemがmount状態を維持する失敗では、正常に後始末できたものとして扱わない。

### 2.7 init実行開始直前の条件

- HALの初期化が完了している。
- 登録されたキャラクタデバイスについて `initialize_devices()` の走査が完了している。
- root filesystemがmount済みであり、`kernel::root_filesystem()`から参照できる。
- FileDescription管理テーブルを使用できる。
- PID 1のProcessが所有するFdTableが存在し、fd 0/1/2がそれぞれACTIVEな別個のFileDescriptionを参照する。
- 3つのFileDescriptionの初期参照数は各1であり、対象とアクセスモードが正しく設定されている。
- `user_binary("init")` が有効なエントリポイントを返す。
- 起動前の処理に失敗していない。

### 2.8 カーネルとFilesystemの接続

通常ファイル用`creat()` / `open()`等のpathnameベースsyscallを実装する前提として、カーネル共通部から起動時にmount済みのroot filesystemへ到達できる経路を設ける。

本段階では`mount(2)` syscall、VFS、mount table、統合mount名前空間を導入しない。root filesystemは**起動時固定構成**とし、カーネル内部だけがmount操作を行う。

#### 2.8.1 root filesystem実体

root filesystemの実体は静的寿命を持つカーネル資源とする。グローバル実体を置く場合は共通命名規則に従い`G_`接頭辞を用いる。

概念上は次の形とする。

```cpp
using RootFilesystem = Filesystem<PlatformFilesystemBackend>;

extern RootFilesystem G_root_filesystem;

FsStatus initialize_root_filesystem();
RootFilesystem& root_filesystem();
```

`PlatformFilesystemBackend`は共通カーネル仕様で特定のBackend型に固定しない。ホスト版では`DiskImageBackend`、実機ではSDカードや内蔵flash等、ビルド対象に応じた既存Backendを用いる。Backend差異をsyscall実装へ持ち込んではならない。

`root_filesystem()`はmount処理を行わず、既に構築済みの`G_root_filesystem`への参照を返すだけの薄い内部アクセサとする。syscallや通常ファイルアダプタから`G_root_filesystem`を直接参照する記述を散在させず、このアクセサを接続点とする。

#### 2.8.2 起動時mount

`initialize_root_filesystem()`は、対象Backendに必要な起動パラメータを組み立て、既存の`Filesystem::mount()`を1回呼ぶ。

概念上の処理は以下とする。

```text
initialize_root_filesystem()
    |
    +-- Backend固有のOpenArgを取得
    +-- total_blocksを取得
    +-- fs_start_blockを取得
    |
    +-- G_root_filesystem.mount(
            arg,
            total_blocks,
            fs_start_block,
            now_provider)
```

`arg`、`total_blocks`、`fs_start_block`の具体値はホスト/実機の構成に依存するため、本共通仕様では固定しない。これらを`creat()` / `open()` syscall側で組み立ててはならない。

Filesystemの時刻providerについて、適切なUNIX時刻源がまだ存在しない段階では`nullptr`を渡してよい。`HAL::get_tick_count()`は起動後経過時間でありUNIX時刻ではないため、暗黙にFilesystemの時刻providerへ流用しない。

成功時は`FsStatus::OK`を返し、以後`root_filesystem().is_mounted() == true`でなければならない。失敗statusは握りつぶさず呼び出し元の`kernel::startup()`へ返す。

#### 2.8.3 syscall層との境界

pathnameを扱う通常ファイルsyscallは、root filesystemがmount済みであることを起動時前提とし、Filesystemへの接続には`kernel::root_filesystem()`を使用する。

例えば`creat()`の内部処理は概念上、

```text
current process / cwd
    |
    +-- kernel::root_filesystem()
    |
    +-- resolve_parent(...)
    +-- Filesystem::creat(...)
    +-- Filesystem::open_file(...)
    |
    +-- RegularFile
    +-- FileDescription
    +-- FdTable
```

となる。

Filesystemがテンプレート型であることを理由に、syscallごとにBackend型やBlockDeviceを直接操作してはならない。FAT、inode table、BlockDevice等の下位層へsyscallから直接アクセスすることも禁止する。

当面current processのcwd機構が未完成でroot inodeを暫定利用する場合、その暫定規則はpathname syscall共通の1か所に局所化する。各`sys_creat()` / `sys_open()`へ`ROOT_INODE`を個別に直書きしない。

#### 2.8.4 mount syscallとの関係

本節の起動時mountは、将来の`mount(2)` syscallとは別物である。

- 現段階ではroot filesystemは1個でよい。
- ユーザー空間からmount/unmountする機能は提供しない。
- VFSやmount tableを先行実装しない。
- 将来`mount(2)`や複数filesystem名前空間を導入するときは、`root_filesystem()`の利用箇所をVFS側の解決APIへ置き換えられるよう、直接グローバル参照を避ける。

Filesystem側が複数インスタンスを扱える設計であることと、現段階のカーネルがroot filesystem 1個だけを利用することは矛盾しない。

#### 2.8.5 unmount

通常のカーネル稼働中にroot filesystemをunmountする経路は設けない。
ユーザー空間からのunmountも本段階では提供しない。

ただし、本実装では内蔵`init`を通常のC++関数として直接呼び出しており、`init`が関数から復帰した場合をカーネル終了経路として扱う。そのため`kernel::startup()`は、`init`復帰後にroot filesystemがmount済みであれば `G_root_filesystem.unmount()` を1回呼び、その完了後に`startup()`を終了する。

概念上の終端処理は次の通りとする。

```text
init(...)
  |
  +-- 復帰
        |
        +-- root filesystemがmount済みか確認
        |
        +-- G_root_filesystem.unmount()
        |
        +-- startup()終了
```

`Filesystem::unmount()`はFilesystem側の既存契約に従う。特に、open handleを残したままunmountしないことは呼び出し側の責任である。このため、init終了時点で通常ファイル用fdやFilesystem open handleが残存しないことをカーネル終了処理の前提とする。

unmount結果の扱いは以下とする。

- `FsStatus::OK`: 正常にunmountされたものとして`startup()`終了へ進む。
- sync失敗等によりFilesystemがmount状態を維持するエラー: エラーを診断可能な形で扱い、未完了のまま正常終了扱いにしない。
- Backend close由来のエラーでFilesystemが既にunmount状態へ遷移した場合: 再度`unmount()`やBackend `close()`を試みない。

本段階では一般化された`shutdown_filesystem()`やカーネル停止APIは新設しない。終了時unmountは`startup()`内の固定root filesystem後始末として扱い、将来カーネル停止処理を独立化する時点で再配置を検討する。

## 3. 内蔵ユーザーバイナリ機構

### 3.1 目的と範囲

ファイルシステムや実行ファイルローダーに依存せず、カーネルと一体にリンクしたユーザープログラムを起動する仕組みを定める。初期段階の `init` の起動、および疑似ユーザープログラムの実行に利用する。実メモリ空間の分離、プロセス生成、スケジューリング、ファイルシステム経由での実行形式ロードは本節の対象外とする。

実装の現状は `internal_binary.h` および `startup.cpp` に基づく。ここでいう「内蔵バイナリ」は独立した実行ファイルをロードすることではなく、**静的リンク済みの関数を呼び出すこと**を指す。

### 3.2 プログラムの形式と検索API

```cpp
namespace kernel {
    using UserBinaryType = int (*)(int, const char**);
    constexpr UserBinaryType user_binary(const char* name);
}
```

- 内蔵プログラムの入口は `extern "C" int init(int, const char**);` のような関数である。
- `user_binary(name)` は名前に一致する入口関数を返し、未登録の場合は `nullptr` を返す。
- 現行テーブルでは文字列 `"init"` が `init` 関数に対応する。検索キーは現時点ではフルパス `"/sbin/init"` ではない。
- テーブルは `internal_binary.h` に静的に定義され、現行実装では `constexpr` な名前・関数ポインタの対と終端用の門番を利用する。動的な登録・削除機能は持たない。
- テーブル内で使用する関数ポインタは有効な静的リンク済みプログラムを指す。引数 `name` は有効なNUL終端文字列であることを呼び出し側の前提とする（現行実装はnull引数を検査しない）。
- `user_binary()` はプログラムの検索のみ担当し、FD設定・プロセス生成・プログラム実行をしない。

### 3.3 initの起動

`kernel::startup()` はHAL、デバイス、FileDescription、および標準FDの準備を終えた後で、`user_binary("init")` を呼び出す。戻り値が非nullであれば、次の引数で関数を直接呼び出す。

```cpp
const char* argv[] = {"init", nullptr};
UserBinaryType entry = user_binary("init");
if (entry != nullptr) {
    entry(1, argv);
}
```

これは呼び出し規約を示す概念コードであり、上記の `nullptr` 分岐に続く起動失敗処理は別途定める。現行 `startup.cpp` では検索を `constexpr` に評価して直接呼び出している。いずれの場合も、内蔵 `init` 起動にファイルシステムのmountや `/sbin/init` の読み込みを必要としない。

`init` 関数からの復帰値をどう扱うか、終了後のカーネル状態や再起動・停止方針はまだ確定しない。

### 3.4 当面のFdTableと将来のプロセス管理

FD経路の初期テストでは、PID 1の最小Processを用意し、その**Processが所有する`FdTable`**を使用する。`initialize_standard_fds()` はこのテーブルにfd 0/1/2を設定してから `init` を実行する。`FdTable`はPID 1のProcessが所有する。`sys_read` / `sys_write`は`ProcessManager::current().fd_table()`からFDを取得する。

将来 `spawn()` 等を実装するときにも、内蔵プログラムの名前から入口関数を検索する仕組みと、プロセスやFDの管理責務は分離する。共通ローダー等の新しい抽象化は現段階で追加しない。

## 4. 関連仕様と変更時の扱い

- `device_driver_spec.md`：`kernel::initialize_devices()`、`kernel::find_char_device()`、デバイス登録テーブルとドライバ側 `init()` の契約。
- `filesystem_design_spec.md` / `external-api.md`：`Filesystem<Backend>`、`mount()`、pathname解決、`creat()`、`open_file()`等のFilesystem側契約。
- `open_file_description_spec.md`（現行改訂版）：`FileInterface`、`FileDescription`、`FdTable`、`initialize_standard_fds()`の動作。
- `internal_binary.h`：内蔵ユーザープログラムの登録表と検索関数の現行宣言・実装。
- `startup.cpp`：現在の `kernel::startup()` の実装（本書のFD・デバイス初期化は今後反映する設計）。
- `subset_os_overview_spec.md`：プロセス起動や機能段階を含む全体方針。
- `AGENTS.md`：C++使用制限、所有権、エラー処理、静的初期化、テスト等の横断規約。

**起動順序の正本は本書**とする。各サブシステム仕様書には、その初期化関数の詳細・必要な前提条件だけを定義し、起動全体の詳細な順序は本書を参照する。既存文書中の起動順序の記述を整理する場合は、個別APIの仕様まで削除しない。

## 5. 最小限の検証項目

- HAL初期化後に `kernel::initialize_devices()` が呼ばれ、その完了後にroot filesystemをmountすること。
- `initialize_root_filesystem()`成功後に`root_filesystem().is_mounted()`がtrueであり、失敗時は`init`を実行しないこと。
- root filesystemのBackend依存引数がsyscall層へ漏れず、`creat()` / `open()`等が`kernel::root_filesystem()`経由でFilesystemへ到達すること。
- `initialize_devices()` が登録エントリを走査し、同一実体が複数登録されていても規定どおり呼び出すこと（詳細はデバイス仕様書）。
- `init`実行直前にfd 0/1/2が適切なアクセスモードで利用可能であること。
- `initialize_standard_fds()`が失敗した場合に`init`が実行されないこと。登録済みFDの後始末はFD仕様書のテストで検証する。
- 起動時に`find_char_device()`を呼ばなくても標準FDを初期化できること。
- `user_binary("init")` は `init` の入口を返し、未登録名には `nullptr` を返すこと。
- 標準FDの準備が成功してから内蔵 `init` が呼び出されること。失敗時は呼び出さないこと。
- テスト時のPID 1のProcessが所有するFdTableにfd 0/1/2を設定でき、プロセス管理機構を前提としないこと。

## 6. 今後追加する横断的仕様

必要になった時点で、本書に独立した章を追加する。現段階ではカーネル停止、共通診断、その他の横断処理のAPIや動作を先行して確定しない。

## 7. FD検証用の最小Process起動段階（追加決定）

本段階では`ProcessManager::initialize_init_process()`でPID 1（PPID 0）のProcessを`RUNNING`として登録し、Context・Scheduler・独立スタック・`spawn()`を実装せずに内蔵`init`を直接呼び出す。標準FDは当該Processの`FdTable`へ直接登録する。syscall層は`current().fd_table()`を通じて`FileDescription`にアクセスする。

初期Process生成、標準FD初期化、内蔵バイナリ検索のいずれかに失敗した場合は`init`を実行しない。作成済み資源の適切な後始末は各サブシステムの既存契約に従い、カーネル停止・復旧方法は未確定とする。`init`関数からの復帰時の扱いも引き続き未確定とし、この段階で一般の`exit()`を実装したものとはみなさない。
