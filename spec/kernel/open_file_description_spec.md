# File Description / File Interface 仕様書

## 1. 位置づけ

本書は、Ancient
UNIX風サブセットOSにおけるファイルディスクリプタ（fd）から実際のI/O対象までの共通機構を規定する。

対象は以下とする。

-   プロセスごとのfdテーブル
-   `FileDescription`
-   `FileInterface`
-   `RegularFile`
-   `ConsoleDevice`
-   `Pipe`
-   標準入出力（fd 0/1/2）
-   ファイルシステムおよびHALとの接続境界

ファイルシステム内部（inode、FAT、BlockDevice等）、HAL内部、パイプのバッファリング方式、プロセス管理そのものの詳細は本書の範囲外とする。

`FileDescription`はPOSIXでいう *open file description*
に相当する概念であり、「ファイルそのもの」ではなく、openされたI/O対象に対する共有状態を表す。

------------------------------------------------------------------------

## 2. 全体構成

fdを使用するI/Oは、原則として以下の経路を通る。

``` text
User process
    |
    | read(fd, buf, count)
    | write(fd, buf, count)
    v
syscall dispatcher
    |
    v
+-------------------------------+
| FD / FileDescription層        |
|                               |
| Process::fd_table[]           |
|        |                      |
|        v                      |
| FileDescription               |
|  - FileInterface* target      |
|  - flags                      |
|  - ref_count                  |
+-------------------------------+
              |
              v
+-------------------------------+
| FileInterface                 |
|                               |
| read() / write()              |
+-------------------------------+
       |             |             |
       v             v             v
 RegularFile    ConsoleDevice     Pipe
       |             |
       v             v
 FileSystem          HAL
       |        getchar()/putchar()
       v             |
 inode / FAT /       v
 BlockDevice    Host / Hardware
```

各層の責務は以下の通りとする。

  ---------------------------------------------------------------------------------------------
  層                                  責務
  ----------------------------------- ---------------------------------------------------------
  fdテーブル                          整数fdから`FileDescription`への対応付け

  `FileDescription`                   openされた状態、flags、参照数の保持

  `FileInterface`                     fdから利用可能なファイル的I/O操作の共通インターフェース

  `RegularFile`                       ファイルシステム上の通常ファイルを`FileInterface`へ接続

  `ConsoleDevice`                     コンソールを`FileInterface`へ接続

  `Pipe`                              パイプを`FileInterface`へ接続

  FileSystem                          inode/FAT/BlockDevice等による通常ファイルの実処理

  HAL                                 ホストOS・CPU・ボード依存の最低レベルI/O
  ---------------------------------------------------------------------------------------------

FD層はI/O対象の具体的種類を原則として意識しない。またHALはfdおよび`FileDescription`を意識しない。

------------------------------------------------------------------------

## 3. fdテーブル

各プロセスは固定長のfdテーブルを1個持つ。

fdテーブルは、整数fdからシステム全体の`FileDescription`への対応だけを管理する。`FileDescription`の所有権や参照カウントの増減はfdテーブル自身の責務としない。

内部表現は以下を基本とする。

```cpp
constexpr std::size_t MAX_FD = ...;

class FdTable {
public:
    OptionalFileRef get(int fd);

    int alloc(FileDescription& desc);
    int alloc(int fd, FileDescription& desc);

    OptionalFileRef detach(int fd);

private:
    std::array<FileDescription*, MAX_FD> entries_{};
};
```

共通の別名として以下を用いる。

```cpp
using FileRef = std::reference_wrapper<FileDescription>;
using OptionalFileRef = std::optional<FileRef>;
```

内部の`entries_`では、未使用fdを`nullptr`で表現する。これはテーブル内部の実装詳細であり、公開APIではポインタを返さず、未使用・無効なfdは`std::nullopt`で表現する。

```text
entries_[0] -> FileDescription
entries_[1] -> FileDescription
entries_[2] -> FileDescription
entries_[3] -> nullptr
...
```

同一の`FileDescription`を複数のfdから参照してよい。また、異なるプロセスの`FdTable`が同一の`FileDescription`を参照してよい。

### 3.1 `get()`

```cpp
OptionalFileRef get(int fd);
```

指定fdが有効かつ使用中であれば、その`FileDescription`への参照を返す。

以下の場合は`std::nullopt`を返す。

- `fd < 0`
- `fd >= MAX_FD`
- 対応するエントリが未使用

`get()`はfdテーブルおよび`FileDescription`の参照カウントを変更しない。

### 3.2 `alloc(FileDescription&)`

```cpp
int alloc(FileDescription& desc);
```

未使用の最小fd番号を探し、そのfdを`desc`へ対応付ける。

成功時は割り当てたfd番号を返し、空きfdが存在しない場合は`-1`を返す。

この関数は`FileDescription`の参照カウントを変更しない。呼び出し側は、この登録に対応する有効な参照を既に保有していなければならない。

### 3.3 `alloc(int, FileDescription&)`

```cpp
int alloc(int fd, FileDescription& desc);
```

指定したfd番号へ`desc`を対応付ける。

以下の場合は失敗し、`-1`を返す。

- `fd < 0`
- `fd >= MAX_FD`
- 指定fdが既に使用中

成功時は指定されたfd番号を返す。

既存エントリを暗黙に上書きしてはならない。`dup2()`等で使用中fdを置き換える場合は、呼び出し側が既存fdを`detach()`して必要な参照解放を行った後、本関数で新しい対応を登録する。

本関数も`FileDescription`の参照カウントを変更しない。

### 3.4 `detach()`

```cpp
OptionalFileRef detach(int fd);
```

指定fdが使用中であれば、対応する`FileDescription`への参照を返した上で、そのfdエントリを未使用状態へ戻す。

以下の場合は`std::nullopt`を返し、テーブルを変更しない。

- `fd < 0`
- `fd >= MAX_FD`
- 対応するエントリが未使用

`detach()`は`FileDescription`の参照カウントを変更しない。返された参照に対応する参照数の減算は呼び出し側が行う。

### 3.5 fdテーブルの不変条件

- 未使用エントリの内部表現は`nullptr`とする。
- 非nullエントリは、`G_file_description_table`内の`ACTIVE`な`FileDescription`を指す。
- 公開APIから生ポインタを返さない。
- `alloc()` / `get()` / `detach()`は`FileDescription::ref_count_`を変更しない。
- `FdTable`の単純コピーを、`spawn()`によるfd継承処理の代用としてはならない。継承時には各`FileDescription`の参照数を正しく増加させる。
- fd番号は配列インデックスと一致する。

------------------------------------------------------------------------
## 4. FileDescription

### 4.1 役割と型

`FileDescription`は、一回のopenによって生成されたI/O対象のオープン状態を表す。POSIXでいうopen file descriptionに相当する概念であり、「ファイルそのもの」ではない。

外部からの直接的な状態破壊を避けるため、単純な公開`struct`ではなく、状態を隠蔽するクラスとして実装する。

概念上、以下の構成とする。

```cpp
class FileDescription {
public:
    FileDescription() = default;
    FileDescription(const FileDescription&) = delete;
    FileDescription& operator=(const FileDescription&) = delete;

    int32_t read(void* buf, uint32_t count);
    int32_t write(const void* buf, uint32_t count);
    int32_t getdents(void* buf, uint32_t count);
    int32_t seek(int32_t offset, uint32_t whence);
    int32_t ioctl(uint32_t request, void* argp);

    uint32_t flags() const;

private:
    std::optional<std::reference_wrapper<FileInterface>> target_;
    uint32_t flags_ = 0;
    int16_t ref_count_ = -1;

    // FileDescription管理APIからのみ寿命状態を変更できるようにする。
};
```

共通の別名として以下を用いる。

```cpp
using FileRef = std::reference_wrapper<FileDescription>;
using OptionalFileRef = std::optional<FileRef>;
```

`FileDescription`はコピー禁止とする。`dup()`や`spawn()`は`FileDescription`自体を複製せず、同一オブジェクトへの参照を共有する。

### 4.2 `target_`

`target_`は実際のI/O対象を表す`FileInterface`へのoptionalな参照である。

```text
FileDescription
      |
      +-- target_ --> RegularFile
      |
      +-- target_ --> ConsoleDevice
      |
      +-- target_ --> Pipe
```

状態との対応は以下を基本とする。

- `FREE`: `target_ == std::nullopt`
- `ACTIVE`: `target_`は有効な`FileInterface`を参照する
- `RELEASING`: cleanup完了まで必要に応じて`target_`を保持する

これにより、`FileDescription`の公開・管理APIから`nullptr`による対象不存在の表現を排除する。

`FileDescription`自身は、対象が通常ファイル、コンソール、パイプのいずれであるかを原則として判定しない。

### 4.3 file position

`FileDescription`自身はfile positionを保持しない。

通常ファイルのfile positionは、そのopenに対応する`RegularFile`が`offset_`として保持する。`dup()`や`spawn()`で同一`FileDescription`を共有する場合は同一`RegularFile`も共有されるため、file positionも共有される。別々の`open()`では別々の`RegularFile`が生成されるため、file positionは独立する。

コンソールやパイプなどfile positionを持たない対象では、位置状態を持たない。

### 4.4 `flags_`

`flags_`はopen時のアクセスモードおよび必要なopen flagsを保持する。

少なくとも`read()` / `write()`時のアクセス可否判定に利用する。

具体的なフラグ値および`O_RDONLY` / `O_WRONLY` / `O_RDWR` / `O_CREAT` / `O_TRUNC`等の扱いはシステムコール仕様およびファイルシステム仕様に従う。

### 4.5 `ref_count_`

`ref_count_`は、その`FileDescription`の使用状態と参照数を表す符号付き整数とする。

```text
ref_count_ == -1 : FREE      （未使用）
ref_count_ ==  0 : RELEASING （最終参照が外れ、解放処理中）
ref_count_ >   0 : ACTIVE    （使用中。値は参照fd数）
```

正常な未使用値は`-1`のみとする。

`ref_count_`はFileDescription管理APIからのみ変更する。通常のfd操作やI/O処理が直接増減してはならない。

新規割り当ては`FREE(-1)`からのみ行う。`RELEASING(0)`から`ACTIVE`へ戻してオブジェクトを復活させてはならない。

### 4.6 `FileDescription`テーブル

すべての`FileDescription`は、カーネル全体で共有する固定長テーブルに格納する。

```cpp
constexpr std::size_t MAX_OPEN_FILES = ...;

std::array<FileDescription, MAX_OPEN_FILES>
    G_file_description_table;
```

グローバル変数には`G_`接頭辞を付ける。

`G_file_description_table`はシステム全体のopen file descriptionを保持し、プロセスごとの`FdTable`とは別のものである。

空きエントリは`ref_count_ == -1`で判定する。`ref_count_ == 0`のエントリは解放処理中であり、空きとして扱ってはならない。

初期化時には全エントリを`FREE`状態とし、`target_ = std::nullopt`とする。

空きエントリの探索は単純な線形探索でよい。

### 4.7 `FileDescription`管理API

`G_file_description_table`の確保・参照増加・解放は、以下の管理APIを通して行う。

```cpp
OptionalFileRef alloc_file_description(FileInterface& target, uint32_t flags);
bool retain_file_description(FileDescription& desc);
int32_t release_file_description(FileDescription& desc);
```

これらはカーネル内部APIであり、ユーザー空間へ公開しない。

#### 4.7.1 `alloc_file_description(FileInterface&, uint32_t)`

```cpp
OptionalFileRef alloc_file_description(FileInterface& target, uint32_t flags);
```

`G_file_description_table`から未使用の`FileDescription`を1個確保し、I/O対象とopen flagsを同時に設定する。`target`は呼び出し側が寿命を保証する。`FileDescription`はこの参照先を所有しない。

処理は以下とする。

1. `flags`の有効性を検査する（許容フラグはsyscall仕様に従う）。
2. `ref_count_ == -1`の`FREE`エントリを先頭から線形探索する。
3. `target_ = target`、`flags_ = flags`とする。
4. 最後に`ref_count_ = 1`として`ACTIVE`状態へ遷移させる。
5. その`FileDescription`への参照を返す。

成功時は`OptionalFileRef`、無効なflagsまたは空きエントリが存在しない場合は`std::nullopt`を返す。

失敗時は`G_file_description_table`を変更しない。

初期参照数1は、fd登録前は呼び出し側が保有し、登録成功後にfdテーブルへ引き渡す参照を表す。登録失敗時は呼び出し側がreleaseする。

デフォルトコンストラクタは`public`な`FileDescription() = default;`とし、メンバ初期値によって`FREE(-1)`、`target_ == std::nullopt`の状態を作る。追加の初期化用setterは設けない。

#### 4.7.2 `retain_file_description()`

```cpp
bool retain_file_description(FileDescription& desc);
```

既存の`ACTIVE`な`FileDescription`への参照を1つ追加する。

成功条件は以下とする。

- `desc`が`G_file_description_table`内のエントリである。
- `ref_count_ > 0`である。
- `ref_count_`が`int16_t`の上限に達していない。

成功時は`ref_count_`を1増加させて`true`を返す。

失敗時は`ref_count_`を変更せず`false`を返す。

`FREE(-1)`および`RELEASING(0)`状態から参照を増加させてはならない。

#### 4.7.3 `release_file_description()`

```cpp
int32_t release_file_description(FileDescription& desc);
```

`FileDescription`の参照を1つ解放する。

この関数は、対応するfdが`FdTable`から切り離された後に呼び出すことを前提とする。

処理は以下とする。

1. `ref_count_`を1減らす。
2. 減算後も`ref_count_ > 0`なら、そのまま終了する。
3. `ref_count_ == 0`になった場合は`RELEASING`状態へ遷移する。
4. `target_`に関連する必要なcleanupを行う。
5. `target_ = std::nullopt`、`flags_ = 0`とする。
6. cleanup完了後、最後に`ref_count_ = -1`として`FREE`状態へ戻す。

cleanupに失敗し、安全に再利用できない場合は`RELEASING(0)`を維持し、そのスロットを再利用してはならない。

### 4.8 `FileDescription`のI/O

`read()` / `write()` / `seek()`は`FileDescription`自身の操作として提供してよい。

これらのメソッドは少なくとも以下を担当する。

- `flags_`によるread/writeアクセス可否検査
- `target_`が有効であることの確認
- `FileInterface::read()` / `write()` / `getdents()` / `seek()`への委譲

file positionの保持・更新は`FileDescription`ではなく、seek可能なI/O対象自身の責務とする。通常ファイルでは`RegularFile::offset_`が唯一の正本となる。

これにより、コンソールやパイプへ通常ファイル固有のoffset引数を強制しない。

------------------------------------------------------------------------
## 5. FileInterface

### 5.1 役割

`FileInterface`は、fdからアクセス可能な対象に共通の「ファイル的I/O操作」を与える抽象インターフェースである。

通常ファイルだけでなく、コンソールやパイプも同じ`read()` / `write()`システムコールから扱えるようにすることを目的とする。通常ファイル固有のfile positionは共通I/Oシグネチャへ持ち込まない。

### 5.2 インターフェース

概念上、以下の純粋抽象クラスとする。

```cpp
class FileInterface {
public:
    virtual ~FileInterface() = default;

    virtual void init() {}
    virtual int32_t read(void* buf, uint32_t count) = 0;
    virtual int32_t write(const void* buf, uint32_t count) = 0;
    virtual int32_t getdents(void* buf, uint32_t count) { return -1; }
    virtual int32_t seek(int32_t offset, uint32_t whence) { return -1; }
    virtual int32_t release() { return 0; }
};
```

I/Oの戻り値は成功時に転送バイト数、失敗時に負値とする。部分転送が発生した場合は、1バイト以上転送できていれば正の転送数を優先して返してよい。

`getdents()`はdirectory列挙用の限定hookであり、既定実装は失敗する。`RegularFile`だけがoverrideし、対象inodeがdirectoryであることをFilesystem公開APIで確認する。処理とoffset規則は`getdents_implementation_spec.md`に従う。RTTIによる対象判別は行わない。

`seek()`はseek可能性を明示する共通hookであり、この段階でvirtualスタブを設ける。seek不能な対象は既定実装の`-1`を使用してよい。通常ファイルは`RegularFile::seek()`でoverrideする。`whence`の具体値は`lseek()`仕様に従う。

`release()`は当該`FileDescription`の最終参照が外れた時に1回だけ呼ばれる終端cleanup hookである。常駐デバイスは既定の空実装を利用できる。

`virtual`の使用は、このような異なるI/O対象を同一のfd機構へ接続する実装差し替え境界に限定して許可する。RTTIは使用しない。`dynamic_cast`および`typeid`には依存しない。

------------------------------------------------------------------------

## 6. RegularFile

### 6.1 役割

`RegularFile`は、ファイルシステム上の通常ファイルを`FileInterface`へ接続するアダプタである。

``` text
FileDescription
      |
      v
FileInterface
      ^
      |
 RegularFile
      |
      v
 FileSystem
      |
      v
 inode / FAT / BlockDevice
```

`RegularFile`はファイルシステム内部のinode、FAT、BlockDevice等をFD層へ直接公開しない。

### 6.2 I/O

`RegularFile::read()`および`RegularFile::write()`は、既存のFileSystem APIの`read_at()` / `write_at()`を利用して通常ファイルへのI/Oを行う。

file positionは`RegularFile::offset_`が保持する。read/writeが正の転送byte数を返した場合のみ、そのbyte数だけ`offset_`を進める。Filesystem側`open_file_t::offset`はFD統合経路では使用しない。

`RegularFile::seek()`はこの`offset_`を変更する通常ファイル固有実装とする。具体的な`SEEK_SET` / `SEEK_CUR` / `SEEK_END`処理は`lseek()`実装時に確定するが、共通virtualインターフェースはこの段階で固定する。

`RegularFile`は1回の`open()`に対応するFilesystem側`open_file_t*`を1個だけ保持する。`dup()`や`spawn()`でfdが共有されてもFilesystemの`dup_file()`は呼ばず、同じ`FileDescription`および同じ`RegularFile`を共有する。

`RegularFile::release()`は保持している`open_file_t*`に対してFilesystemの`close()`を1回だけ呼び、その後自身を未使用状態へ戻す。Filesystemの`close()`がエラーを返しても、同APIの契約上open entry自体は解放済みであるため、同じhandleへ再度`close()`してはならない。

`RegularFile`実体は動的確保せず固定長テーブルで管理する。空き枯渇時は`open()`を失敗させる。

### 6.3 BlockDeviceとの関係

既存ファイルシステムの`BlockDevice`抽象は維持する。

HALの導入によって`BlockDevice`を置き換えない。

したがって通常ファイルの経路は、

``` text
fd
 -> FileDescription
 -> RegularFile
 -> FileSystem
 -> inode / FAT / BlockDevice
```

で完結し、通常のファイルI/OがHALのキャラクタI/O
APIを経由することはない。

------------------------------------------------------------------------

## 7. ConsoleDevice

### 7.1 役割

`ConsoleDevice`はコンソールを`FileInterface`へ接続する。

``` cpp
class ConsoleDevice final : public FileInterface {
public:
    void init() override;

    int32_t read(void* buf, uint32_t count) override;
    int32_t write(const void* buf, uint32_t count) override;
    int32_t ioctl(uint32_t request, void* argp) override;
};
```

`ConsoleDevice::ioctl()`は [`ioctl_implementation_spec.md`](ioctl_implementation_spec.md) のTTY判定と共有echo ON/OFF/GET要求を受理する。これらの要求は `argp == nullptr` を使う。

`init()`は`FileInterface::init()`のオーバーライドで、echoを既定ON、保留EOFをなしに初期化する。`HAL::init_hal()`の後、`kernel::initialize_devices()`が非公開デバイス登録テーブルを登録順に走査して呼び出す。

当面、コンソールはcanonical modeや行単位バッファリングを持たないbyte streamとする。
入力byteのechoだけは`ConsoleDevice`が共有状態で管理し、既定ONとする。echoのON/OFFは
`ioctl_implementation_spec.md` のecho要求で切り替える。Ctrl-D (EOT, `0x04`) は入力EOFで、
EOT自体はデータとして返さずechoもしない。read中にEOTより前のbyteを取得済みなら、その
部分成功を返してpending EOFを次のreadで一度報告する。

以下は本段階では`ConsoleDevice`の責務としない。

-   canonical mode / 行編集
-   端末属性
-   高度なバッファリング

必要になった段階で別途仕様化する。

### 7.2 HALとの関係

`ConsoleDevice`はHALのキャラクタI/O APIを利用する。

概念上、出力は以下の経路となる。

``` text
write(fd, ...)
   |
FileDescription
   |
ConsoleDevice::write()
   |
HAL::putchar()
   |
Host / Hardware
```

入力は以下の経路となる。

``` text
read(fd, ...)
   |
FileDescription
   |
ConsoleDevice::read()
   |
HAL::getchar()
   |
Host / Hardware
```

HALはコンソールドライバそのものではなく、`ConsoleDevice`から利用される機種依存境界である。

既存HAL仕様に従い、

``` cpp
namespace HAL {

void init_hal();
void putchar(char ch);
int32_t getchar();
uint32_t get_tick_count();
bool wait_for_tick_change(uint32_t observed_tick);

}
```

を基本APIとする。

`HAL::getchar()`はblocking
APIとし、正常時は0〜255のbyte値、負値は入力終了またはエラーを表す。

`HAL::wait_for_tick_change()`は、協調Schedulerが実行可能Processのない状態でtickを待つためのblocking APIである。host実装の時計源・待機・失敗契約は[`host_hal_tick_wait_spec.md`](host_hal_tick_wait_spec.md)に従う。

文字コード変換、CR/LF変換、echoはHALでは行わない。echoとEOTの処理はConsoleDeviceの責務とする。

### 7.3 Linux/POSIX端末との既知の非互換（2026-10-25 調査記録）

`cat`による標準入力の対話転送をLinuxと比較して差を調査した。方針は現状維持（実装変更なし）であり、以下は既知の非互換情報として記録する。ホストHALは実端末でcanonical/ECHOを無効化して起動するため、端末処理はすべて本OS側（`ConsoleDevice`）の責務となり、以下はそのままユーザーに見える挙動差になる。

差の原因は独立した3つに分解できる。

1.  **行バッファがない（canonical mode欠如）** — §7.1で延期と明記した項目。Linuxは改行入力で行単位で`read()`へ渡すため、コンソールから直接読むコマンドは行ごとに処理できる。本OSはbyte streamのため、改行を契機とした渡り方にならない。
2.  **`read()`が`count`充足まで返さない（短縮readなし）** — `ConsoleDevice::read()`は`count`バイト充足かHAL入力終了/EOTまでブロックする。`cat`は512バイトを要求するため、Ctrl-D入力または512バイト打ち込みまで1バイトも返らず、転送出力もその時点まで遅れる。Linuxは到着済みbyteがあれば短い`read()`を返すため、canonical modeがなくてもchar単位で出力される。観察された「全入力後にまとめて出力される」の直接原因はこの要因であり、canonical mode欠如とは別の要因である。
3.  **制御byteのline discipline処理がない** — LinuxはBS/DEL等を編集処理してデータに混入させず、`^H`等のキャリーエコーで表示する。本OSのechoは生byteのため、BS (`0x08`) は出力時にホスト端末でカーソルを左へ動かすだけで編集はされず、データにも`0x08`が残る。入力中にBSを打ち込むと`cat > file`等の出力内容がLinuxと異なる。

対象範囲（現時点の実装）:

-   sh0（`btx::readline()`）は該当しない。readlineは1バイト読みと自前編集を行うため、LinuxでICANONを無効にして動く行編集プログラムと同じ構造で動作する。
-   `fgets`/`gets`は1バイトずつ読むため要因2の実害は小さいが、要因3の制御byte混入は受ける。現状コンソール入力での利用はない（`init`はファイル読み取り）。
-   `cat`と`fread`等の`count > 1`の直接読み取りが要因2の全該当。`cat`の対話利用は非現実的と判断し現状維持とした。
-   Ctrl-DのEOFは§7.1のpending EOFで対応済みで、canonical modeの部分行flushに相当する。
-   端末由来の信号（Ctrl-C等）は本項目とは別の未実装項目である（`subset_os_overview_spec.md`のLv.2計画）。

将来、(1) readlineを使わない対話入力プログラムを増やす、(2) POSIX端末ツールを移植する、のいずれかを狙う場合は、要因1〜3の解消とmode切替ioctl（readlineが自前編集を行うための非canonical切替を含む）が必要になり、その時点で§7.1の「別途仕様化」に着手する。

------------------------------------------------------------------------

## 8. 標準入出力

### 8.1 fd 0/1/2

stdin / stdout / stderrも通常のfd機構に載せる。

最初のプロセスまたは疑似ユーザープログラムを開始する前に、カーネルが概念的に以下を設定する。

``` text
fd 0 --> FileDescription --> ConsoleDevice
fd 1 --> FileDescription --> ConsoleDevice
fd 2 --> FileDescription --> ConsoleDevice
```

これにより、

``` cpp
read(0, buf, n);
write(1, buf, n);
write(2, buf, n);
```

は通常の`read()` / `write()`システムコールとして処理される。

### 8.2 ファイルシステムとの関係

本OSでは`mknod`を採用しないため、コンソールをファイルシステム上のデバイスファイルとして`open()`する経路は設けない。

fd 0/1/2はカーネルが直接fdテーブルへ登録する。

ただし初期化後は、FD層から見て標準入出力を特別扱いしない。通常の`FileDescription`と`FileInterface`を経由する。

### 8.3 stdin / stdout / stderrの関係

fd 0/1/2にはそれぞれ独立した`FileDescription`を1個ずつ用意し、初期参照数を1とする。

- fd 0: `O_RDONLY`
- fd 1: `O_WRONLY`
- fd 2: `O_WRONLY`

I/O対象となる`FileInterface`実体は共有してよい。初期構成は単一の`G_console_device`を3つのfdで共有する。

### 8.4 初期化APIと起動順序

```cpp
bool initialize_standard_fds(
    FdTable& fd_table,
    FileInterface& dev_stdin,
    FileInterface& dev_stdout,
    FileInterface& dev_stderr
);
```

引数はfd番号順（0, 1, 2）。`initialize_standard_fds()`はカーネル内部の起動用関数であり、fd 0/1/2が空いていることを前提とする。使用中なら上書きせず失敗する。3つのFileDescriptionを`alloc_file_description(target, flags)`で独立に確保し、`FdTable::alloc(fd, desc)`で登録する。`FdTable`は参照数を変更しない。

初期化の順序は以下を基本とする。

1. カーネルランタイムと静的オブジェクトを初期化し、FileDescriptionテーブルを全FREEにする。
2. `HAL::init_hal()`を呼ぶ。
3. `kernel::initialize_devices()`を呼ぶ。非公開のデバイス登録テーブルを登録順に走査し、各エントリの`FileInterface::init()`を呼ぶ。ConsoleDevice等の個別`init()`を起動側から重複して呼ばない。
4. 最初のプロセスの管理領域と空の`FdTable`を準備する。
5. `initialize_standard_fds(init_fd_table, G_console_device, G_console_device, G_console_device)`を呼ぶ。
6. 成功した場合だけinit（当面は疑似ユーザープログラム）を起動する。

途中で失敗した場合、関数が今回登録したfdを`detach()`し、今回確保した全FileDescriptionの参照を`release_file_description()`で解放して`false`を返す。cleanupエラーがあっても各FileDescriptionの終端解放規約に従って後始末する。失敗後はinitを起動しない。成功時は`true`を返す。標準fdの作成に追加の`retain`は必要ない。

ゲストOSのfd 2とホストOSのstderrを対応付ける必要はない。初期構成ではゲストfd 1/2とも`ConsoleDevice`を介して`HAL::putchar()`へ出力し、ホストのstderrはカーネル自身の診断・デバッグ出力に使用できるよう区別する。

### 8.5 spawnによる継承

`spawn()`は指定されたfdを子プロセスへ継承する。`inherit_fds == nullptr && inherit_fd_count == 0`の場合は呼び出し元プロセスの全open fdを継承し、それ以外では指定リストのfdだけを継承する。各子fdは同じ番号で同じ`FileDescription`を参照し、参照数を増加させる。詳細は[`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)に従う。

したがってルートプロセスにfd
0/1/2を設定すれば、通常のfd継承機構によって子孫プロセスへ標準入出力を伝播できる。

標準入出力専用の継承機構は設けない。

------------------------------------------------------------------------

## 9. Pipe

`Pipe`は`pipe()`実装時に`FileInterface`の実装として追加する。

概念上、

``` text
FileInterface
      ^
      |
     Pipe
```

とすることで、パイプも通常ファイルやコンソールと同じ`read()` /
`write()`システムコールから利用する。

`pipe()`は読み出し側fdと書き込み側fdの一対を生成する。

``` text
fd[0] --> FileDescription --> Pipe
fd[1] --> FileDescription --> Pipe
```

read/write endpointの生成とFD登録は[`pipe_implementation_spec.md`](pipe_implementation_spec.md)に従う。buffer、blocking、EOF等のI/O詳細は[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)に従う。

------------------------------------------------------------------------

## 10. システムコールとの対応

### 10.1 read / write

`read()` / `write()` syscallは`FdTable::get()`で`FileDescription`へのoptionalな参照を取得し、`FileDescription`のI/Oメソッドへ処理を委譲する。

```text
sys_read(fd, buf, count)
    |
    +-- FdTable::get(fd)
    |
    +-- FileDescription::read(...)
    |
    +-- FileInterface::read(...)
```

```text
sys_write(fd, buf, count)
    |
    +-- FdTable::get(fd)
    |
    +-- FileDescription::write(...)
    |
    +-- FileInterface::write(...)
```

syscall層は`FileDescription::target_`や`ref_count_`を直接操作しない。

### 10.2 open / creat

通常ファイルの`open()`では、FileSystemによってpathnameを解決または必要に応じて作成し、Filesystemの`open_file()`でhandleを確保し、そのhandleを保持する`RegularFile`を固定長テーブルから確保する。続いて`alloc_file_description(regular_file, flags)`で新しい`FileDescription`を確保・初期化し、`FdTable::alloc()`へ登録する。

確保順序は概念上、`Filesystem handle -> RegularFile -> FileDescription -> fd`とし、途中失敗時は未所有の資源を逆順に解放する。ただし`FileDescription`確保後は`RegularFile`の所有を`FileDescription`へ引き渡すため、fd登録失敗時は`release_file_description()`だけを呼び、`RegularFile`やFilesystem handleを重ねて解放してはならない。

別々の`open()`は、同じpathnameを指定した場合でも別々のFilesystem handle、`RegularFile`、`FileDescription`を生成する。`dup()`はこれらを新規生成せず同一`FileDescription`を共有する。

`creat()`は`open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)`相当の互換ラッパーとして扱う。詳細なフラグ処理とrollbackは`open_close_implementation_spec.md`に従う。

### 10.3 dup / dup2

`dup(fd)`は元fdと同じ`FileDescription`を参照する新しいfdを生成する。新しい`FileDescription`は生成しない。

概念上の手順は以下とする。

1. `FdTable::get(oldfd)`で参照を取得する。
2. `retain_file_description()`で参照数を増加させる。
3. `FdTable::alloc(desc)`で最小の空きfdへ登録する。
4. fd登録に失敗した場合は、追加した参照を`release_file_description()`で戻す。

`dup2(oldfd, newfd)`を実装する場合は`FdTable::alloc(newfd, desc)`を使用できる。

- `oldfd`が無効なら失敗する。
- `oldfd == newfd`なら有効性確認後、そのfdをそのまま返す。
- `newfd`が使用中の場合は`detach(newfd)`し、その参照を解放する。
- 元の`FileDescription`への参照を適切にretainした上で`newfd`へ登録する。

置換途中の失敗時に参照数やfdテーブルを不整合にしないよう、具体的な操作順序は`dup2()`の詳細仕様で定める。

### 10.4 close

`close(fd)`は`FdTable::detach(fd)`で対応付けを外し、得られた`FileDescription`参照を`release_file_description()`へ渡す。

`detach()`自体は参照カウントを変更しない。`release_file_description()`が最終参照を解放する場合のみ`FileInterface::release()`が呼ばれる。したがって`dup()`等で共有された`FileDescription`は、一方のfdをcloseしてもFilesystem handleを閉じない。

最終cleanupでエラーが発生した場合は`close()`は`-1`を返すが、fdは既にdetach済みであり再利用不能とする。二度目の`close(fd)`は失敗する。

### 10.5 lseek

`lseek()`は`FdTable::get()`で対象`FileDescription`を取得し、`FileDescription::seek()`から`FileInterface::seek()`へ委譲する。境界計算、position更新、失敗時の不変条件は`lseek_implementation_spec.md`に従う。

seek不能な対象は`FileInterface::seek()`の既定実装により`-1`を返す。通常ファイルは`RegularFile::seek()`をoverrideし、自身の`offset_`を変更する。

これによりseek可能性の判定にRTTIを使用しない。

### 10.6 pipe

`pipe()`はパイプ用の`FileInterface`実装を生成し、読み出し用・書き込み用の`FileDescription`を確保してfdテーブルへ登録する。

fd登録に失敗した場合は、途中まで確保した`FileDescription`参照を正しく解放する。

endpoint pairの生成、FileDescription作成、FdTableへの原子的な登録は[`pipe_implementation_spec.md`](pipe_implementation_spec.md)に従う。fd登録失敗時は途中まで確保したFileDescription参照とPipe資源をrollbackする。read/writeのblocking、EOF等は[`pipe_io_implementation_spec.md`](pipe_io_implementation_spec.md)に従う。

------------------------------------------------------------------------
## 11. 所有権と寿命

以下の原則を採用する。

1. `G_file_description_table`が`FileDescription`実体の格納領域を提供する。
2. `FdTable`は`FileDescription`を所有せず、対応付けのみを保持する。
3. 複数fdおよび複数プロセスのfdが同じ`FileDescription`を共有できる。
4. `FileDescription::ref_count_`の変更はFileDescription管理APIだけが行う。
5. `FileDescription`は`FileInterface`へのoptionalな参照を保持する。
6. `FileInterface`実装の具体的な所有方式は対象ごとに定める。
7. カーネル内部APIでは、生ポインタが不可欠でない場合は参照を優先する。
8. 「値が存在しない可能性」を返すAPIでは`std::optional<std::reference_wrapper<T>>`を基本候補とする。
9. 生ポインタは固定長テーブル等の内部実装で必要な場合に限定し、可能な限り公開APIへ露出させない。

`ConsoleDevice`のようにシステム全体で単一実体を共有できる対象と、`RegularFile`や`Pipe`のように個別の寿命管理が必要な対象を、同一の所有方式へ無理に統一しない。

`FileDescription`は固定長`std::array`で保持し、動的メモリ確保には依存しない。

`std::optional<std::reference_wrapper<T>>`は参照先の寿命を管理しない。参照先が有効であることは、本仕様の`ref_count_`および各`FileInterface`実装の寿命規約によって保証する。

------------------------------------------------------------------------
## 12. エラー処理

ユーザー空間から見たシステムコールの失敗は、全体仕様のシステムコール規約に従い`-1`として返す。

カーネル内部で`FileInterface`がどのようにエラー種別を返すかは、既存のカーネル内部エラー表現との整合を取って決定する。

少なくとも以下はエラーとして扱う。

-   fdが範囲外
-   fdが未使用
-   読み出し不可のfdに対する`read()`
-   書き込み不可のfdに対する`write()`
-   seek不能な対象に対する`lseek()`
-   `FileInterface`側で発生したI/Oエラー

エラー検査の詳細な順序は各システムコールの詳細仕様で定める。

------------------------------------------------------------------------

## 13. `putstrk`との関係

`putstrk`はカーネルデバッグ用であり、通常の標準入出力機構を意図的にバイパスする。

したがって、

``` text
normal write:

write()
 -> fd
 -> FileDescription
 -> FileInterface
 -> ConsoleDevice
 -> HAL::putchar()

debug output:

putstrk()
 -> HAL::putchar()
```

という2経路を許容する。

`putstrk`を`FileDescription`や`ConsoleDevice`へ依存させないことで、fd機構が未初期化または故障している段階でも最低限のデバッグ出力を可能とする。

------------------------------------------------------------------------

## 14. 実装上の制約

-   `FileInterface`の実装差し替え境界では限定的にC++の`virtual`を使用してよい。
-   RTTIは使用しない。
-   `dynamic_cast`および`typeid`は使用しない。
-   `FileInterface`の共通操作は起動時初期化用の`init()`、I/O用の`read()` / `write()`、seek hookの`seek()`、要求委譲用の`ioctl()`、最終参照解放時の`release()`とする。`init()`・`seek()`・`ioctl()`・`release()`はデフォルト実装を持つ。seek不能・未対応要求の対象は各hookの`-1`を利用できる。ioctl要求のABIと対応対象は個別仕様に従う。
-   HALへfd、`FileDescription`、ファイルシステム等の上位概念を持ち込まない。
-   ファイルシステムの`BlockDevice`抽象は維持し、HALへ統合しない。
-   コンソールは当面byte streamとし、端末制御機能を先行して追加しない。

------------------------------------------------------------------------

## 15. 実装段階との対応

syscallディスパッチ、疑似ユーザープログラム機構、およびホスト環境用HALは実装済みである。
したがって本仕様の次の実装対象は、これらの間を接続するfd共通基盤とする。

導入順序は以下とする。

-   fdテーブル、`FileDescription`、`FileInterface`、`ConsoleDevice`およびfd 0/1/2の初期化を導入する。
-   既存の`read` / `write` syscallをfd共通基盤へ接続し、標準入出力を通して動作確認する。
-   `close`を接続し、`FileDescription`の参照カウントと寿命管理を完成させる。
-   その後`RegularFile`を接続し、通常ファイルの`open` / `read` / `write` / `close`へ拡張する。
-   `dup`実装時に`FileDescription`の共有を利用する。
-   `spawn`実装時にfdテーブル継承と`FileDescription`共有を接続する。
-   `pipe`実装時に`FileInterface`の実装として`Pipe`を追加する。

これにより、先にFD共通基盤とコンソール経路を独立して検証し、その後に通常ファイル、`dup`、`spawn`、`pipe`を段階的に追加する。

------------------------------------------------------------------------

## 16. 実装計画

### 16.1 方針

現時点でsyscallディスパッチ、疑似ユーザープログラム機構、ホスト環境用HALは完成している。
次の実装では、通常ファイルを直ちに接続するのではなく、まず

```text
疑似ユーザープログラム
    |
    v
syscall dispatcher
    |
    v
fd table
    |
    v
FileDescription
    |
    v
FileInterface
    |
    v
ConsoleDevice
    |
    v
HAL
```

の経路を完成させる。

この経路を先に確立することで、ファイルシステム固有処理と切り離してfd管理、参照カウント、アクセスモード、標準入出力を検証できる。
その後`RegularFile`を追加し、既存FileSystemを同じFD APIへ接続する。

### 16.2 実装順序

#### 第1段階: `FileInterface`

以下の純粋抽象クラスを定義する。

```cpp
class FileInterface {
public:
    virtual ~FileInterface() = default;

    virtual void init() {}
    virtual int32_t read(void* buf, uint32_t count) = 0;
    virtual int32_t write(const void* buf, uint32_t count) = 0;
    virtual int32_t seek(int32_t offset, uint32_t whence) { return -1; }
    virtual int32_t release() { return 0; }
};
```

`init()`はデバイス登録テーブル経由の起動時初期化に使用し、通常ファイルやPipeは必要がなければ継承した空実装を使う。`seek()`はこの段階でvirtualスタブとして定義し、seek不能な対象は既定の`-1`を利用する。

**完了条件**

- `FileInterface`を継承したテスト用クラスから`read()` / `write()`を呼び出せる。
- `seek()`をoverrideしない実装では`-1`が返る。
- RTTIを必要としない。
- `dynamic_cast` / `typeid`を使用しない。

#### 第2段階: `FileDescription`

`FileDescription`をコピー禁止のクラスとして実装する。

```cpp
class FileDescription {
public:
    FileDescription() = default;
    FileDescription(const FileDescription&) = delete;
    FileDescription& operator=(const FileDescription&) = delete;

    int32_t read(void* buf, uint32_t count);
    int32_t write(const void* buf, uint32_t count);
    int32_t seek(int32_t offset, uint32_t whence);

private:
    std::optional<std::reference_wrapper<FileInterface>> target_;
    uint32_t flags_ = 0;
    int16_t ref_count_ = -1;
};
```

**完了条件**

- `FREE / RELEASING / ACTIVE`の状態を表現できる。
- `target_`の不存在を`std::nullopt`で表現できる。
- 一般の呼び出し側から`ref_count_`を直接変更できない。
- `FileDescription`のコピーが禁止される。
- publicなデフォルトコンストラクタでFREE状態の固定長配列を構築できる。
#### 第3段階: `FileDescription`テーブルと管理API

`FileDescription`はカーネル全体で共有する固定長`std::array`に格納する。

```cpp
constexpr std::size_t MAX_OPEN_FILES = ...;

std::array<FileDescription, MAX_OPEN_FILES>
    G_file_description_table;
```

管理APIは参照中心とする。

```cpp
using FileRef = std::reference_wrapper<FileDescription>;
using OptionalFileRef = std::optional<FileRef>;

OptionalFileRef alloc_file_description(FileInterface& target, uint32_t flags);
bool retain_file_description(FileDescription& desc);
int32_t release_file_description(FileDescription& desc);
```

**完了条件**

- 初期状態で全エントリが`FREE(-1)`かつ`target_ == std::nullopt`となる。
- `alloc_file_description(target, flags)`がtarget・flagsを設定後に`FREE`エントリを`ACTIVE(1)`へ遷移させて参照を返す。
- 満杯時は`std::nullopt`を返し、テーブルを変更しない。
- `retain_file_description()`がACTIVE状態だけを増加できる。
- 参照数のoverflow / underflowを防止できる。
- 最後のreleaseで`RELEASING(0)`を経由し、cleanup完了後にのみ`FREE(-1)`へ戻る。
- `RELEASING`を空きとして誤認しない。
#### 第4段階: fdテーブル

各プロセス用の`FdTable`を実装する。

内部表現は固定長ポインタ配列とするが、公開APIから生ポインタを返さない。

```cpp
class FdTable {
public:
    OptionalFileRef get(int fd);

    int alloc(FileDescription& desc);
    int alloc(int fd, FileDescription& desc);

    OptionalFileRef detach(int fd);

private:
    std::array<FileDescription*, MAX_FD> entries_{};
};
```

`alloc(desc)`は未使用の最小fdへ登録する。

`alloc(fd, desc)`は指定fdが空いている場合だけ登録し、既存エントリを暗黙に上書きしない。

`detach()`はfdテーブルから対応付けを外すだけであり、参照カウントを変更しない。

**完了条件**

- 未使用の最小fdを割り当てられる。
- 指定fdへ割り当てられる。
- 使用中fdへの指定割り当てを拒否できる。
- 負のfd、上限以上のfd、未使用fdを適切に扱える。
- `get()` / `detach()`が不在時に`std::nullopt`を返す。
- 公開APIから生ポインタが露出しない。
- 複数fdから同じ`FileDescription`を参照できる。
#### 第5段階: `ConsoleDevice`

`ConsoleDevice`を`FileInterface`の実装として追加する。

```cpp
class ConsoleDevice final : public FileInterface {
public:
    void init() override;

    int32_t read(void* buf, uint32_t count) override;
    int32_t write(const void* buf, uint32_t count) override;
    int32_t ioctl(uint32_t request, void* argp) override;
};
```

`write()`は各byteを`HAL::putchar()`へ渡す。

`read()`は`HAL::getchar()`を利用する。

初期実装では対話用途を優先し、`count > 0`の場合でも1 byte取得した時点で返してよい。
canonical modeや行単位バッファリングは導入しない。echoは既定ONで各通常入力byteを出力し、
ioctlでOFFにもできる。EOT (`0x04`) を受信したとき、そのread内に先行byteがあればその
部分成功を返して次readで一度だけ0を返す。先行byteがなければ直ちに0を返す。

`count == 0`ではHALを呼ばず0を返す。

`HAL::getchar()`が負値を返した場合のEOFとエラーの区別は、HAL側で現状区別されていないため、既存どおり取得済みbyteがあれば部分成功、なければ0を返す。

**完了条件**

- `HAL::init_hal()`後の`kernel::initialize_devices()`から、登録された`ConsoleDevice::init()`が仮想呼び出しされる。
- `ConsoleDevice::write()`からHALのコンソール出力へbyte列を出力できる。
- `ConsoleDevice::read()`はecho ON/OFFに従い、EOTをデータbyteではなくEOFとして扱う。
- `ConsoleDevice::read()`からホスト標準入力を読み出せる。
- `count == 0`を正しく処理できる。

#### 第6段階: fd 0/1/2の初期化

疑似ユーザープログラム開始前に、8.4節の`initialize_standard_fds()`を用いて標準入出力をfdテーブルへ設定する。

```text
fd 0 --> FileDescription(stdin)  --> ConsoleDevice
fd 1 --> FileDescription(stdout) --> ConsoleDevice
fd 2 --> FileDescription(stderr) --> ConsoleDevice
```

3つのfdは別々の`FileDescription`を使用する。

初期flagsは概念上以下とする。

```text
fd 0 : O_RDONLY
fd 1 : O_WRONLY
fd 2 : O_WRONLY
```

`ConsoleDevice`実体は初期構成では1個共有する。`initialize_standard_fds()`の引数で個別のI/O対象を指定できる。

**完了条件**

- fd 0/1/2が使用済みとしてfdテーブルに登録される。
- 途中失敗時に今回の登録・参照を後始末し、initを起動しない。
- 次の一般fd割り当てが3から開始される。
- stdin/stdout/stderrのアクセスモードが独立している。

#### 第7段階: `read` / `write` syscallとの接続

既存syscallディスパッチの`read` / `write`実装をfd共通基盤へ接続する。

`read()`は概念上以下の順序で処理する。

```text
fd検査
  |
  v
FileDescription取得
  |
  v
read可能なflagsか確認
  |
  v
target->read(...)
```

`write()`も同様に、

```text
fd検査
  |
  v
FileDescription取得
  |
  v
write可能なflagsか確認
  |
  v
target->write(...)
```

とする。

この段階では接続対象が`ConsoleDevice`のみであり、file positionは存在しない。

**完了条件**

疑似ユーザープログラムから、

```cpp
write(1, "Hello\n", 6);
```

を呼び出した際、

```text
guest
 -> syscall dispatcher
 -> fd table
 -> FileDescription
 -> FileInterface
 -> ConsoleDevice
 -> HAL::putchar()
```

の経路で画面へ出力される。

また、

- `write(0, ...)`を拒否する。
- `read(1, ...)`および`read(2, ...)`を拒否する。
- 不正fdを拒否する。

#### 第8段階: `close`

`close(fd)`をfd共通基盤へ接続する。

処理は以下とする。

```text
fd検査
  |
  v
FdTable::detach(fd)
  |
  v
OptionalFileRefを取得
  |
  v
release_file_description(desc)
  |
  +-- 他参照あり --> ACTIVEのまま終了
  |
  +-- 最終参照 --> RELEASING(0)
                    |
                    +--> cleanup
                    |
                    +--> FREE(-1)
```

fd 0/1/2も通常のfdとしてclose可能とする。
標準入出力専用の保護は設けない。

**完了条件**

- `close(1)`後の`write(1, ...)`が失敗する。
- closeされたfdを後で再利用できる。
- 最後の参照が閉じられたとき`RELEASING`へ遷移し、後処理完了後にのみ`FREE`へ戻る。

#### 第9段階: `FileDescription`共有機構の検証

`dup()`や`spawn()`本体に先立ち、複数fdが同じ`FileDescription`を共有する動作を内部テストする。

```text
fd 3 --+
       +--> FileDescription A
fd 4 --+
```

この状態で、

```text
A.ref_count == 2
```

となることを確認する。

一方のfdをcloseした後もAが生存し、最後のfdをcloseした時点で解放されることを確認する。

**完了条件**

- 1つの`FileDescription`を複数fdで安全に共有できる。
- 一方のcloseが他方を無効化しない。
- 最後のcloseでのみ解放される。

ここまでを**FD共通基盤完成**のマイルストーンとする。

### 16.3 次段階: `RegularFile`

FD共通基盤完成後に`RegularFile`を追加する。

```cpp
class RegularFile final : public FileInterface {
    ...
};
```

通常ファイルの経路は、

```text
fd
 -> FileDescription
 -> RegularFile
 -> FileSystem
 -> inode / FAT / BlockDevice
```

とする。

`RegularFile`を導入した段階で、通常ファイル固有のfile positionを`RegularFile::offset_`として保持する。

概念上、

```text
read:
  RegularFile::offset_をFilesystem::read_at(inode, offset, ...)へ渡す
  -> 成功したbyte数だけRegularFile::offset_を進める

write:
  RegularFile::offset_をFilesystem::write_at(inode, offset, ...)へ渡す
  -> 成功したbyte数だけRegularFile::offset_を進める

seek:
  FileDescription::seek(...)
  -> RegularFile::seek(...)
  -> RegularFile::offset_を更新
```

とする。Filesystem側`open_file_t::offset`と`ref_count`はFD統合経路では共有状態の正本として使用しない。`open_file_t`は既存Filesystem APIとの互換を保つため残し、1個の`RegularFile`が1個の`open_file_t*`を保持する。

### 16.4 後続機能との接続

FD共通基盤完成後は、以下の順で既存機構へ接続する。

1. `RegularFile` + `open()` / 通常ファイル`read()` / `write()` / `close()`
2. `creat()`
3. `lseek()`
4. `dup()`
5. `spawn()`によるfdテーブル継承
6. `Pipe`

`dup()`は新しい`FileDescription`を生成せず、元fdと同一の`FileDescription`を参照する。

`spawn()`によるfd継承も同様に、継承対象fdごとに`FileDescription`の参照数を増加させる。

`Pipe`は`FileInterface`の新しい実装として追加し、syscallディスパッチの`read()` / `write()`へ対象種別による分岐を追加しない。

### 16.5 初期テスト項目

FD共通基盤の実装時には、少なくとも以下を検証する。

1. `FileDescription`の参照数増減
2. `G_file_description_table`とoptional参照ベース管理APIの初期化・確保・retain・release・再利用と状態遷移
3. fdテーブルの最小空き番号割り当て
4. fdの範囲外・未使用検査
5. `ConsoleDevice::write()`
6. `ConsoleDevice::read()`
7. fd 0/1/2の初期化
8. `write(1, ...)`からHALまでの到達
9. `write(2, ...)`からHALまでの到達
10. `read(0, ...)`からHALまでの到達
11. `write(0, ...)`の拒否
12. `read(1, ...)` / `read(2, ...)`の拒否
13. `close(1)`後の`write(1, ...)`の拒否
14. 同一`FileDescription`を2つのfdから共有
15. 一方のfdをcloseしても他方が有効
16. 最後のfdをcloseした時点で`FileDescription`を解放

これらが完了した時点で、`RegularFile`、`dup()`、`spawn()`、`Pipe`を載せるためのFD共通基盤は完成したものとする。
