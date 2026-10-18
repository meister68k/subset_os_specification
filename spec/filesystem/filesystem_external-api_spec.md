# 外部仕様 — `Filesystem<Backend>` 利用者 API

> 本書は `src/filesystem.h` を中心とした、ファイルシステム**を利用する側**(syscall 層・ライブラリ呼び出し側)の外部仕様である。レイヤー内部の実装契約は `spec/00`〜`spec/11`、全エラー定義は `spec/errors.md`、全レイヤー共通の制約は `AGENTS.md` を上位仕様とする。本書とコードの乖離があれば `filesystem.h` の現状を正とし、仕様矛盾として報告する。

## 1. 範囲と対象外

- 対象: `Filesystem<Backend>` が public に提供する操作と、それに関連する公開型・定数。
- 利用者は `Filesystem<Backend>` だけを経由し、マウント済み FS の内部層を直接操作しない。
- phase 1 の実装対象である。phase 2 のクラスタ再利用・fsck・クリッシュ共有等は対象外。
- シンボリックリンク、VFS、FS を跨ぐ pathname 移動は対象外。

### 1.1 非公開(利用禁止)な内部層

| ヘッダ / 型 | 扱い |
|---|---|
| `Fat<Backend>` (`fat.h`) | 内部層。公開経路なし |
| `InodeTable<Backend>` (`inode_table.h`) | 内部層。公開経路なし |
| `Directory<Backend>` (`directory.h`) | 内部層。`creat`/`mkdir`/`unlink`/`resolve` が一時利用 |
| `BlockDevice<Backend>` (`block_device.h`) | 内部層。`mount`/`unmount` が open/close |
| `FilesystemTestProbe` | テスト専用 friend。本番呼び出し禁止 |

- マウント済み FS に対して上記へ直接アクセスする API は提供しない。
- private メンバ `fat_` / `inode_table_` / `device_` / `open_files_` へは触れない。

## 2. 必要なヘッダと公開型

```cpp
#include "filesystem.h"   // Filesystem, OpenFileTable, open_file_t, Whence, ResolveMode
#include "fs_status.h"    // FsStatus
#include "fs_format.h"    // 定数、superblock_t, disk_inode_t, dirent_t
#include "mount_state.h"  // mount_state_t (mount_state() の参照先)
#include "serialization.h" // directory 列挙時の dirent_decode() に必要
```

- これらのヘッダが提供する型・定数・関数はすべて `namespace fs` 内にある(`fs/src/*.h`)。
- 本書のシグネチャ例(§4〜§12)は `namespace fs` 内の宣言をそのまま示す。呼び出し側から書く利用例(§10, §14)は `fs::` で修飾する。

### 2.1 結果型 `FsStatus`

- 全操作の結果は `FsStatus`(成功は常に `FsStatus::OK`)。
- byte 転送を伴う操作は `std::pair<uint32_t, FsStatus>`(先頭 = 実転送バイト数、次 = 停止理由)。
- 内部状態としてグローバル `errno` を使わない。OS `errno` への変換は **syscall 層でのみ**行う。

| 値 | 利用者から見た意味 |
|---|---|
| `OK` | 成功(EOF・長さ 0 の read も含む) |
| `EIO` | ブロック転送失敗 |
| `ERANGE` | offset / size / 変換結果が表現・FS 容量外 |
| `ENODEV` | 未 mount、デバイス利用不能 |
| `ENOSPC` | クラスタ・inode・open-file 表の枯渇 |
| `ENOENT` | 名前・パス成分が存在しない |
| `EEXIST` | 存在が禁止される対象が存在する(`mkdir()`、`link()` 等) |
| `EISDIR` | 通常ファイルが必要な操作へ directory を渡した、または Parent 解決で最終名が無い |
| `ENOTEMPTY` | `rmdir()`対象directoryに`.`/`..`以外の有効entryがある |
| `EINVAL` | 他に専用値が無い不正引数(空パス、負 seek、null 等) |
| `ESPIPE` | seek 不能対象(pipe)へ `lseek()` |
| `ENOTDIR` | パス途中の成分が directory でない |
| `ENAMETOOLONG` | 名前 27 バイト超、またはパス 128 バイト以上 |
| `EMLINK` | `nlink == UINT16_MAX` から増加 |
| `ECORRUPT` | オンディスク構造・chain が形式/不変条件違反 |
| `EBUSY` | 既 mount への再 `mount()`、Backend 固有 busy |
| `EWP` | 書込保護 Backend 用(予約) |

詳細な選択基準は `spec/errors.md` を参照。

### 2.2 定数(`fs_format.h`)

| 定数 | 値 | 利用者向け意味 |
|---|---:|---|
| `FS_BLOCK_SIZE` | 512 | ブロック/クラスタ・バイト数 |
| `FS_MAGIC` | `0x58545442` (LE 数値) | superblock magic(ディスク上は `42 54 54 58`) |
| `FAT_FREE` / `FAT_EOF` | `0x0000` / `0xFFFF` | FAT エントリ(通常は内部) |
| `INODE_NUM_NONE` | 0 | 未使用 dirent の inode 番号・番兵 |
| `ROOT_INODE` | 1 | ルート directory の inode |
| `CLUSTER_NONE` | `0xFFFFFFFF` | 「cluster なし」番兵 |
| `NAME_MAX` | 27 | 名前の最大バイト数(NUL 除く) |
| `NAME_FIELD_SIZE` | 28 | dirent の name 領域(NUL 込み) |
| `PATH_MAX` | 128 | パス全体の最大バイト数(有効長は 127+NUL) |
| `INODE_SIZE` | 64 | inode レコード長(phase 1 固定) |
| `DIRENT_SIZE` | 32 | dirent レコード長・directory read の単位 |
| `SUPERBLOCK_SIZE` | 512 | superblock サイズ |
| `S_IFREG` / `S_IFDIR` | `0100000` / `0040000` | mode 種別ビット |
| `S_IFREE` / `S_IUNUSED` | `0000000` / `0170000` | 未使用 / 削除済み inode |
| `ROOT_MODE` / `ROOT_NLINK` | `S_IFDIR\|0777` / 2 | ルート初期値 |

### 2.3 メモリ側レコード

```cpp
struct superblock_t;   // mount 後 superblock() が返す検証す検証済み値(読取専用)
struct mount_state_t;  // mount 後 mount_state() が返す runtime 会計(読取専用)
struct disk_inode_t;   // inode のメモリ表現
struct dirent_t;       // dirent のメモリ表現 { uint32_t inode_num; char name[28]; }
struct FileStat;       // stat() の結果 {
                       //   uint32_t inode; uint16_t mode; uint16_t nlink;
                       //   uint32_t size; uint32_t blocks;  // 512-byte units
                       //   uint64_t mtime; uint64_t ctime; }
```

- これらをディスクへ直接書き込む API は利用者に提供しない。
- `dirent_t` の解釈は `dirent_decode()` を使う(§10)。
- `FileStat` は `stat()` が返すメモリ用結果型(§13)。ディスク表現とは独立に構築され、
  FS が管理する事実だけを持つ。uid / gid / dev 等の OS 固定値は含めない。
  `disk_inode_t` をそのまま返す API ではない。

### 2.4 open file description

```cpp
struct open_file_t {
    uint32_t inode_num;  // 対象 inode
    uint32_t offset;     // 共有される現在位置
    uint32_t ref_count;  // 参照数(dup/fork で増加)
    bool is_pipe;        // pipe 等(現状の open_file() は常に false)
};
```

- `open_file_t*` は `open_file()` / `dup_file()` が返し、`read` / `write` / `lseek` / `close` / `dup_file` に渡す。
- フィールドの直接書き換えはしないこと(特に `offset` / `ref_count`)。
- 同一 `open_file_t*` を共有する記述子は **offset を共有**する。

### 2.5 列挙型

```cpp
enum class Whence {          // lseek の基準位置(POSIX SEEK_* と同値 0/1/2)
    kSEEK_SET = 0,
    kSEEK_CUR = 1,
    kSEEK_END = 2,
};

enum class ResolveMode {     // pathname 解決の目的
    Target,                  // 最終 component まで解決
    Parent,                  // 最終名の直前で停止(親 inode + 名前)
};
```

- syscall 境界では `<unistd.h>` の `SEEK_SET` / `SEEK_CUR` / `SEEK_END` をこの列挙へ変換する(値は一致)。

## 3. Backend 契約

`Filesystem<Backend>` はコンパイル時に Backend を選ぶ。Backend は物理媒体固有の同期 512 バイト block I/O のみを提供する。

```cpp
struct Backend {
    using OpenArg = /* backend 固有 */;
    FsStatus open(OpenArg arg);
    FsStatus read(uint32_t blkno, uint8_t* buf);
    FsStatus write(uint32_t blkno, const uint8_t* buf);
    FsStatus close();
};
```

- `open` 失敗時は Backend 自身を閉状態へ戻す。
- `close` は終端操作。エラーを返しても呼び出し側(`unmount`)は再試行しない。
- Backend 内で例外を送出しない。OS エラーは `FsStatus` へ変換する。
- FS レイアウト・FAT・inode の知識を持ち込まない。
- 詳細は `spec/02-block-device.md` §3。

### 3.1 イメージ準備

- ホスト開発では `ruby mkfs.rb IMAGE CAPACITY` で空イメージを作成する(`mkfs-tools.md`)。
- `mkfs.rb` / `mkfs_chk.rb` は `fs_start_block = 1` 固定。
- 本番ターゲットでは同等の mkfs または事前書き込み済みイメージを用意する。

## 4. ライフサイクル

```cpp
template <class Backend>
class Filesystem {
public:
    using NowFn = uint32_t (*)();

    Filesystem();

    FsStatus mount(typename Backend::OpenArg arg, uint32_t total_blocks,
                   uint32_t fs_start_block, NowFn now = nullptr);
    FsStatus unmount();
    bool is_mounted() const;
    const superblock_t& superblock() const;
    const mount_state_t& mount_state() const;
    // … §5 以降の操作 …
};
```

### 4.1 `mount()`

```cpp
FsStatus mount(typename Backend::OpenArg arg,
               uint32_t total_blocks,
               uint32_t fs_start_block,
               NowFn now = nullptr);
```

| 引数 | 意味 |
|---|---|
| `arg` | Backend 固有の open 引数(パス、制御ブロック等) |
| `total_blocks` | デバイス総ブロック数(容量上界。superblock の `total_blocks` 以下でなければならない) |
| `fs_start_block` | superblock が在るブロック。格納値と一致必須 |
| `now` | `ctime` / `mtime` 生成用時刻プロバイダ。`nullptr` なら常に 0 |

手順(内部): Backend open → `fs_start_block` から superblock 読み/decode → 全項目検証 → FAT/inode table 読み → `mount_state_t` 再構築 → mounted。

| 結果 | 条件 |
|---|---|
| `OK` | mounted へ遷移 |
| `EBUSY` | 既に mount 済み |
| `ECORRUPT` | superblock 検証・再構築失敗(追加 I/O なし) |
| `EIO` / Backend 由来 | open / superblock 読込失敗 |
| `ENODEV` 等 | Backend open 失敗 |

- 失敗時はデバイスを閉じ、runtime 状態を無効化し、通常 API を使えない未 mount 状態に戻る。
- 成功時のみ `is_mounted() == true`。以降の FS 操作は未 mount なら `ENODEV`。

### 4.2 時刻プロバイダ

```cpp
static uint32_t my_now();                 // 下位 32bit UNIX 時刻を返す
fs.mount(arg, total, start, &my_now);     // 決定的な注入例
fs.mount(arg, total, start);              // 省略時: 常に 0
```

- コアはホスト clock を直接呼ばない(`AGENTS.md` §14)。
- phase 1: 時刻フィールドは 64bit 領域の下位 32bit のみ使用、上位は 0。
- `mtime` / `ctime` は内容・状態変更で更新。`atime` は phase 1 で `mtime` と同値(read では更新しない)。

### 4.3 `unmount()`

| 結果 | 条件 |
|---|---|
| `OK` | FAT 同期 + device close 成功。runtime 無効化済み |
| `ENODEV` | 未 mount |
| `EIO` 等(sync) | FAT 同期失敗。**mounted / open を維持**し再試行可能 |
| Backend close 由来 | 同期成功後の close 失敗。mounted 解除・runtime 無効化済み。**close 再試行不可** |

- open-file 表は mount 単位で初期化される。unmount 前に全 `close()` を完了させることが利用者責務(未 close の entry は RAM 上で消える)。

### 4.4 読取専用アクセサ

- `is_mounted()` — mount 済みか。
- `superblock()` — 検証済みジオメトリ(未 mount は初期値)。
- `mount_state()` — 再構築済み runtime 会計(未 mount は初期値)。
- いずれも const 参照。書き換え API は無い。

## 5. 操作の分類

| 種別 | 戻り値 | API |
|---|---|---|
| 成否 | `FsStatus` | `mount` `unmount` `creat` `mkdir` `lseek` `open_file` `dup_file` `unlink` `close` `resolve` `resolve_path` `resolve_parent` `stat` |
| byte 転送 | `pair<uint32_t, FsStatus>` | `write_at` `read_at` `read` `write` |

### 5.1 offset の二層構造

| API | offset 引数 | open-file `file->offset` |
|---|---|---|
| `read_at` / `write_at` | 有り | **変更しない**(byte 範囲プリミティブ) |
| `read` / `write` | なし(開始位置 = `file->offset`) | **実転送分だけ**進める |
| `lseek` | 提案位置(相対/絶対) | 検査成功後にのみ更新 |

- partial 転送 + エラーでも、成功したバイト数だけ offset を進める。
- 成功フル転送は `{要求長, OK}`。EOF read・長さ 0 は `{0, OK}`。
- `stat` は open を行わない。open-file table・`open_file_t::offset`・inode の時刻・
  FAT を一切変更しない(§13)。

## 6. pathname 解決

```cpp
FsStatus resolve(uint32_t cwd_inode, const char* path, ResolveMode mode,
                 uint32_t* out_inode, char out_name[NAME_MAX + 1]);

FsStatus resolve_path(uint32_t cwd_inode, const char* path,
                      uint32_t* out_inode);

FsStatus resolve_parent(uint32_t cwd_inode, const char* path,
                        uint32_t* out_dir_inode,
                        char out_name[NAME_MAX + 1]);
```

- `cwd_inode` — 相対パスの開始 directory。プロセス cwd の保管は process 層の責務。絶対パスでは未使用(先頭 `/` で root 開始)。
- `resolve_path` = `resolve(..., Target, ...)`。最終 inode を返す。
- `resolve_parent` = `resolve(..., Parent, ...)`。**最終名の存在確認は行わない**(呼び出し側の `creat` / `mkdir` / `unlink` 等が後続で確認)。
- 読み取り専用。書込みは行わない。

### 6.1 パス規則

| 入力 | 結果 |
|---|---|
| 空文字列 `""` | `EINVAL` |
| パス長 ≥ 128 | `ENAMETOOLONG` |
| component > 27 バイト | `ENAMETOOLONG` |
| 途中成分が非 directory | `ENOTDIR` |
| 成分が存在しない | `ENOENT` |
| 先頭 `/` | root (inode 1) から開始 |
| それ以外 | `cwd_inode` から開始 |
| 連続する `/` | 空 component として無視 |
| `.` / `..` | 通常の dirent として lookup(root の `..` は root) |
| Parent で末尾 `/`(含む `"/"`) | `EISDIR`(最終名が無い) |
| Target で `"/"`・`"//"` 等 | 開始 inode(root)を返す |
| symlink | 展開しない(未サポート) |

- 名前は NUL 終端不透明バイト列。文字コードは解釈・正規化しない。
- `/` は pathname 区切り。component 内に含めない(NUL も component 内にない)。
- `resolve` の `out_name` は Parent で必須(`nullptr` は `EINVAL`)。Target 時の `out_name` 内容は規定しない(`resolve_path` は内部 buffer を渡すため利用者は気に不要)。

## 7. ファイル作成 — `creat()`

```cpp
FsStatus creat(uint32_t dir_inode_num, const char* name, uint16_t mode,
               uint32_t* out_inode_num);
```

- **親 directory の inode + 単一名**(フルパスではない)。フルパスから使う場合は `resolve_parent()` の後を呼ぶ。
- `name`: 1〜27 バイト、`/` 不可、空不可。不正は `EINVAL` / `ENAMETOOLONG`。
- `mode`: 許可 9 バイトのみ保持(`mode & 0777`)。種別は常に `S_IFREG`。phase 1 で umask は適用しない。
- `out_inode_num`: 成功時のみ確定。`nullptr` は `EINVAL`。

### 7.1 既存名の挙動(V7 `creat(2)` 互換)

| 既存対象 | 結果 |
|---|---|
| 通常ファイル | **truncate**(論理 `size = 0`)して同一 inode を返す。`EEXIST` にはならない |
| directory | `EISDIR` |
| dangling dirent(未使用/削除済み inode 指す) | `ECORRUPT` |
| 不存在 | 新規作成(§7.2) |

- truncate は旧 cluster を phase 1 では再利用しない。`mtime` / `ctime` を更新する。

### 7.2 新規作成の順序と失敗時

1. inode 割当・初期化・永続化(`nlink=1`, `size=0`, `first_cluster=CLUSTER_NONE`)
2. directory へ dirent 追加

- 手順 1 失敗: dirent を作らない(変化なし)。
- 手順 2 失敗: `drop_link()` で rollback。inode は `S_IUNUSED` へ(`S_IFREE` には戻さない)。rollback 自体の失敗は leak を許容し、手順 2 の status を返す。
- 名前だけが未初期化 inode を指す瞬間は作らない。
- inode 番号は単調割当。削除済みスロットは phase 1 で再利用しない。

### 7.3 directory 作成 — `mkdir()`

```cpp
FsStatus mkdir(uint32_t dir_inode_num, const char* name, uint16_t mode,
               uint32_t* out_inode_num);
```

- **親 directory inode + 単一名**を受け取る。フルパスは`resolve_parent()`後に呼ぶ。
- `name`は1〜27バイト、`/`不可、空不可。`mode`は許可9ビットだけを保持し、種別は`S_IFDIR`とする。
- `out_inode_num`は成功時のみ新規directory inode番号で確定する。`nullptr`は`EINVAL`とする。
- 同名の通常ファイルまたはdirectoryが存在する場合は`EEXIST`とする。`.`と`..`も正常な親では同名として`EEXIST`になる。
- 新規directoryは1clusterを持ち、`size = 512`、`nlink = 2`とする。`.`は自身、`..`は親を指す。
- 子の`..`参照のため親の`nlink`を1増やす。親の`nlink == UINT16_MAX`は`EMLINK`とする。
- inode、cluster、`.`、`..`を完全に初期化し、親の`nlink`を増やしてから親direntへ公開する。
- 失敗時は公開済みの未初期化directoryを残さず、rollback失敗では破損よりinode、cluster、link countのリークを優先する。
- 詳細な更新順序、rollback、故障注入条件は`spec/11-mkdir.md`に従う。
- phase 1の`rmdir()`は空directoryを削除する。詳細は`spec/kernel/mkdir_rmdir_implimentation_spec.md`に従う。

### 7.4 directory削除 — `rmdir()`

```cpp
FsStatus rmdir(uint32_t dir_inode_num, const char* name);
```

- 親directory inodeと単一名を受け取る。pathname解決は呼び出し側で行う。
- 対象は空directoryに限る。`.`/`..`以外の有効entryがあれば`ENOTEMPTY`。
- link count、open/cwd参照、失敗時の更新順序は`spec/kernel/mkdir_rmdir_implimentation_spec.md`に従う。

## 8. open / dup / close

```cpp
std::pair<open_file_t*, FsStatus> open_file(uint32_t inode_num);
FsStatus dup_file(open_file_t* file);
FsStatus close(open_file_t* file);
```

### 8.1 `open_file(inode_num)`

- pathname 解決後の**検証済み inode 番号**から open file description を割り当てる。path 解決は呼び出し側(`resolve_path` 等)の責務。
- 新 entry: `offset = 0`, `ref_count = 1`, `is_pipe = false`。
- 未 mount: `ENODEV`。未使用/削除済み inode: `ECORRUPT`。inode 範囲外等: 下層の `EINVAL` / `ECORRUPT`。
- **固定表枯渇(`kCapacity == 32`): `ENOSPC`**。同時に開ける description は FS インスタンスあたり 32。

### 8.2 `dup_file(file)`

- 同一 entry の `ref_count` を 1 増やす(`dup` / `fork` 子側)。**offset は共有**される。
- 表外・解放済み entry: `EINVAL`。`ref_count == UINT32_MAX`: `ERANGE`(増やさない)。未 mount: `ENODEV`。

### 8.3 `close(file)`

1. 有効 entry 検査 → 表外/二度目: `EINVAL`
2. `ref_count--`。まだ正なら `OK`(entry と inode は維持)
3. 最後の参照: inode 番号を保持して entry を解放 → `nlink == 0` かつどの entry からも参照されない場合のみ orphan finalize(`S_IUNUSED`)

- `close` は **`nlink` を変更しない**。
- finalize 失敗時も entry は解放済み。`close` はエラーを返し、二度目の close は `EINVAL`。fd は使えない。
- 未 mount: `ENODEV`。

### 8.4 open-but-unlinked

- open 中に `unlink` された inode は、**最後の close まで** read/write 可能。
- 同一 inode を指す別の entry が残る間は finalize しない。
- 最後の close でのみ `S_IUNUSED` へ遷移。
- phase 1: クラッシュ時の open table 丧失により orphan が残留し得る。mount 中の推測回収はしない。

## 9. 読み書き

### 9.1 `write_at()` — byte 範囲プリミティブ

```cpp
std::pair<uint32_t, FsStatus> write_at(uint32_t inode_num, uint32_t offset,
                                       const uint8_t* src, uint32_t len);
```

| 入力 | 結果 |
|---|---|
| 未 mount | `{0, ENODEV}` |
| `len == 0` | `{0, OK}`(null `src` 許容) |
| `src == nullptr` (`len > 0`) | `{0, EINVAL}` |
| `offset + len` が `uint32_t` で overflow | `{0, ERANGE}`(状態変更なし) |
| `offset + len > data_blocks * 512` | `{0, ERANGE}`(最大ファイル/FS 容量) |
| chain 不足・破損 | `{0, ECORRUPT}` または割当時の `ENOSPC` 等 |
| 途中 I/O 失敗 | `{成功済みバイト数, 停止 status}` |

- open-file offset は変更しない。
- 新 cluster は zero-fill。EOF 後の gap はゼロとして読める。
- 部分 block は read-modify-write。完全上書き block は read 省略可。
- 1 バイトでも成功したら inode の `size = max(old, offset+written)`、`first_cluster`、時刻を更新。inode 更新に失敗した byte は成功数に含めない。
- **directory の inode へ呼ばないこと。** directory 判定は本関数では行わない。ユーザ書き込みは必ず §9.3 の `write()` を使う。

### 9.2 `read_at()` — byte 範囲プリミティブ

```cpp
std::pair<uint32_t, FsStatus> read_at(uint32_t inode_num, uint32_t offset,
                                      uint8_t* dst, uint32_t len);
```

| 入力 | 結果 |
|---|---|
| 未 mount | `{0, ENODEV}` |
| `len == 0` | `{0, OK}` |
| `offset >= size`(EOF) | `{0, OK}` |
| `dst == nullptr` (`len > 0`) | `{0, EINVAL}` |
| live でない inode | `{0, ECORRUPT}` |
| 有効分の読取 | `{n, OK}`、`n = min(len, size - offset)` |
| 途中 I/O / chain 破損 | `{コピー済み, 停止 status}` |

- open-file offset・inode 時刻(атime 等)を一切変更しない。
- directory も同一 byte 列 primitive で読める(§10)。

### 9.3 `read()` / `write()` — open-file wrapper

```cpp
std::pair<uint32_t, FsStatus> read(open_file_t* file, uint8_t* dst, uint32_t len);
std::pair<uint32_t, FsStatus> write(open_file_t* file, const uint8_t* src, uint32_t len);
```

- `file == nullptr` → `{0, EINVAL}`。
- 開始位置は `file->offset`。実転送分だけ offset を進める。
- `write()` は対象が directory なら **`{0, EISDIR}`**(open-file 層で拒否)。`read()` は directory を読める。
- 共有 description の offset は複数 fd 間で共有される。

## 10. directory 列挙(raw dirent)

- phase 1 の `read()` / `read_at()` は **直列化済み `dirent_t` の生バイト列**を返す。
- FS 側は dirent 境界を強制しない。userland は **必ず `DIRENT_SIZE`(32) 単位**で要求する。分割持ち越しバッファは無い。
- 各レコードは `dirent_decode(buf, &de)` で decode する。
- 上記はFilesystemのraw directory read契約である。syscall `getdents()`は空slotを除外して有効な32-byte recordを返す。詳細は`spec/kernel/getdents_implementation_spec.md`を参照する。

```cpp
uint32_t dir_ino = 0;
if (fs.resolve_path(cwd, "/some/dir", &dir_ino) != fs::FsStatus::OK) { /* … */ }

auto of = fs.open_file(dir_ino);
if (of.second != fs::FsStatus::OK) { /* … */ }

uint8_t raw[fs::DIRENT_SIZE];
for (;;) {
    const auto r = fs.read(of.first, raw, fs::DIRENT_SIZE);
    if (r.second != fs::FsStatus::OK || r.first < fs::DIRENT_SIZE) break;
    fs::dirent_t de;
    if (fs::dirent_decode(raw, &de) != fs::FsStatus::OK) break;
    if (de.inode_num == fs::INODE_NUM_NONE) continue;   // 空き slot
    // de.name は NUL 終端の不透明バイト列(最大 27)
}

fs.close(of.first);
```

- `inode_num == INODE_NUM_NONE`(0) の slot は空き。名前だけで空判定しない。
- directory の一般 `write()` は `EISDIR`。内容変更は `creat` / `mkdir` / `unlink` の内部経路のみ。

## 11. `lseek()`

```cpp
FsStatus lseek(open_file_t* file, int32_t offset, Whence whence,
               int32_t* out_new_offset);
```

| 条件 | 結果 |
|---|---|
| `file` または `out_new_offset` が null | `EINVAL` |
| `file->is_pipe` | `ESPIPE` |
| 未知の `whence` | `EINVAL` |
| 提案位置 < 0 | `EINVAL`(`offset` / `out_new_offset` 不変) |
| 提案位置 > `INT32_MAX` | `ERANGE`(同上) |
| 成功 | `file->offset` と `*out_new_offset` を更新して `OK` |

- 基準: `kSEEK_SET` = 0、`kSEEK_CUR` = 現在 offset、`kSEEK_END` = inode `size`。
- **EOF より先への seek は許可**。その後の write は §9.1 の契約に従う。
- 現状 `open_file()` は `is_pipe = false` のため `ESPIPE` は通常経路では発生しない(将来の pipe 用)。

## 12. `unlink()`

```cpp
FsStatus unlink(uint32_t dir_inode_num, const char* name);
```

- 親 directory inode + 単一名(`creat` と同じ形)。フルパスは `resolve_parent()` の後。
- **順序(内部):** lookup → inode 取得 → directory 拒否 → **dirent 削除** → `nlink--` → 必要なら finalize。逆転禁止。
- directory(ルート含む)の unlink は常に `EISDIR`。
- 名前が不存在 → `ENOENT`。dirent が無効 inode を指す → `ECORRUPT`。
- remove 失敗時は `nlink` を減らさない。
- open 中のファイル: 名前と `nlink` が消えても内容は最後の `close` まで保持(§8.4)。
- phase 1: cluster・inode スロットは再利用しない(`free_chain` は非回収)。
- pathname syscallのcwd解決、user path検証、FDへの影響は`spec/kernel/unlink_implementation_spec.md`に従う。

## 13. 属性取得 — `stat()`

```cpp
FsStatus stat(uint32_t inode_num, FileStat* out);
```

- **inode 番号だけ**を受け取る。pathname は受け取らない(仕様確定済みの B 案)。
  pathname から属性を取る経路は、呼び出し側で `resolve_path()` → `stat()` と
  2段に分ける(`kernel::kernel_stat()` の経路、stat_implementation_spec.md §6)。
- 読み取り専用。`open_file()` を呼ばず、open-file table・offset・inode の時刻・
  FAT を変更しない。`atime` の更新も行わない。
- `out` は全項目の構築に成功した最後にだけ書き込み、失敗時は `out` を変更しない。
- `blocks` は `ceil(size / 512)` で計算しない。inode から到達できる**実 FAT チェーン長**
  (512バイト単位)。truncate 後に size 0 でも chain が残る場合は `blocks >= 1` になる。
- 未知のファイル種別を正常として扱わない。種別ビットが通常ファイル / directory の
  どちらでもない inode は `ECORRUPT` で拒否する。

| 結果 | status |
|---|---|
| 成功。`*out` に属性が入る | `OK` |
| 未 mount | `ENODEV` |
| `out == nullptr` | `EINVAL` |
| 範囲外の inode 番号 | `EINVAL` |
| 未使用 / 削除済み / 種別不明の inode | `ECORRUPT` |
| `size` があるのに `first_cluster` が無い | `ECORRUPT` |
| FAT チェーンの FREE 混入・範囲外参照・循環 | `ECORRUPT` |
| inode 読出しの I/O 失敗 | `EIO`(または Backend 由来の status) |

- 単体テストは `fs/test/stat_test.cpp`。上位からの完全経路は
  `kernel/test/kernel_stat_test.sh` で検証する。

## 14. 典型的な利用手順

### 14.1 マウント → ファイル作成 → 書き込み → アンマウント

```cpp
#include "filesystem.h"
#include "fs_status.h"
#include "fs_format.h"

using Fs = fs::Filesystem<MyBackend>;

// 1. イメージ準備(ホスト開発例): ruby mkfs.rb fs.img 1M

Fs fs;   // 関数内の局所オブジェクト。ファイルスコープに置くと namespace fs と競合する
MyBackend::OpenArg arg;          // 例: { "fs.img", … }
const uint32_t total_blocks = 2048;   // 1M / 512
const uint32_t fs_start = 1;          // mkfs.rb 固定

// 2. mount(時刻プロバイダ省略 → すべて 0)
if (fs.mount(arg, total_blocks, fs_start) != fs::FsStatus::OK) {
    /* 適切に処理 */
}

// 3. resolve_parent + creat(フルパスで作る場合)
uint32_t dir_ino = 0;
char name[fs::NAME_MAX + 1];
if (fs.resolve_parent(fs::ROOT_INODE, "note.txt", &dir_ino, name) != fs::FsStatus::OK) { /* … */ }

uint32_t ino = 0;
if (fs.creat(dir_ino, name, static_cast<uint16_t>(0644), &ino) != fs::FsStatus::OK) { /* … */ }

// 4. open → write → close
auto of = fs.open_file(ino);
if (of.second != fs::FsStatus::OK) { /* … */ }

const uint8_t data[] = { 'h', 'i' };
const auto w = fs.write(of.first, data, sizeof(data));
if (w.second != fs::FsStatus::OK || w.first != sizeof(data)) { /* partial / 失敗 */ }

fs.close(of.first);

// 5. unmount
fs.unmount();
```

### 14.2 読み込み

```cpp
uint32_t ino = 0;
if (fs.resolve_path(fs::ROOT_INODE, "note.txt", &ino) != fs::FsStatus::OK) { /* ENOENT 等 */ }

auto of = fs.open_file(ino);
uint8_t buf[64];
const auto r = fs.read(of.first, buf, sizeof(buf));
// r.first = 実読込数。r.second == OK かつ r.first < sizeof(buf) なら EOF 付近
fs.close(of.first);
```

### 14.3 読み替え(seek)

```cpp
int32_t pos = 0;
fs.lseek(of.first, 0, fs::Whence::kSEEK_END, &pos);     // size へ
fs.lseek(of.first, -10, fs::Whence::kSEEK_CUR, &pos);   // 相対。負なら EINVAL
fs.lseek(of.first, 512, fs::Whence::kSEEK_SET, &pos);   // 絶対。EOF 超可
```

### 14.4 削除

```cpp
uint32_t dir_ino = 0;
char name[fs::NAME_MAX + 1];
if (fs.resolve_parent(fs::ROOT_INODE, "note.txt", &dir_ino, name) == fs::FsStatus::OK) {
    fs.unlink(dir_ino, name);   // directory なら EISDIR
}
```

### 14.5 dup 共有 offset

```cpp
auto a = fs.open_file(ino);
fs.dup_file(a.first);                 // ref_count = 2、offset 共有
fs.write(a.first, data, n);           // a 経由で書いた位置は dup 先でも同じ
fs.close(a.first);                    // ref_count = 1、finalize しない
fs.close(a.first);                    // 最後の参照。orphan なら finalize
```

### 14.6 directory作成

```cpp
uint32_t parent_ino = 0;
char name[fs::NAME_MAX + 1];
if (fs.resolve_parent(fs::ROOT_INODE, "docs", &parent_ino, name) == fs::FsStatus::OK) {
    uint32_t dir_ino = 0;
    fs.mkdir(parent_ino, name, static_cast<uint16_t>(0755), &dir_ino);
}
```

- `mkdir()`は作成対象の親inodeと単一名を受け取る。
- phase 1の削除は`rmdir(parent_ino, name)`で行う。空directoryのみ削除できる。

## 15. phase 1 の制約(利用者へ)

| 項目 | 制約 |
|---|---|
| 空間回収 | cluster・inode スロットを**再利用しない**(単調 bump)。削除後の総使用量は減らない |
| crash consistency | なし。途中までの書込みが残り得る |
| fsck | なし。検出した破損は `ECORRUPT` で fail closed |
| thread | **同一 `Filesystem` インスタンスへの並行呼び出し禁止**。外部で直列化する |
| `link()` | `spec/10` §6 に定義があるが、**`filesystem.h` の公開 API には未導入** |
| `mkdir()` | 親inode + 単一名で新規directoryを作成。詳細は`spec/11-mkdir.md` |
| `rmdir()` | phase 1で提供。詳細は`spec/kernel/mkdir_rmdir_implimentation_spec.md` |
| `rename` / `chmod` | 公開 API なし(`stat` は §13 で提供) |
| symlink / 跨 FS | 未サポート |
| uid / gid | 常に 0 |
| umask | `creat` / `mkdir` に適用しない |
| open-file 容量 | FS あたり 32(`ENOSPC`) |
| 最大ファイルサイズ | `data_blocks * 512`(FS データ容量全体) |
| `data_blocks` 上限 | 65535 |
| 時刻 | 下位 32bit UNIX、`atime == mtime`、read で `atime` 更新なし |
| 例外 / RTTI | 本体は `-fno-exceptions -fno-rtti`。例外を送出しない |
| iostream | 本体非依存。`dirent` 等の変換は `serialization.h` |

## 16. syscall 境界での errno 変換

- FS 内部は `FsStatus` のまま扱う。
- システムコール境界でのみ、OS `errno` へ変換する。例(方針のみ):

| `FsStatus` | 想定 errno |
|---|---|
| `OK` | (成功) |
| `EIO` | `EIO` |
| `ERANGE` | `ERANGE` / `EOVERFLOW` |
| `ENODEV` | `ENODEV` / `ENXIO` |
| `ENOSPC` | `ENOSPC` |
| `ENOENT` | `ENOENT` |
| `EEXIST` | `EEXIST` |
| `EISDIR` | `EISDIR` |
| `ENOTEMPTY` | `ENOTEMPTY` |
| `EINVAL` | `EINVAL` |
| `ESPIPE` | `ESPIPE` |
| `ENOTDIR` | `ENOTDIR` |
| `ENAMETOOLONG` | `ENAMETOOLONG` |
| `EMLINK` | `EMLINK` |
| `ECORRUPT` | `EIO`(破損は報告し続ける。握りつぶさない) |
| `EBUSY` | `EBUSY` |

- 変換表の厳密な対応は syscall 層の仕様で確定する(本書は FS 側の契約を示す)。
- `Whence` ↔ `SEEK_*` の変換も syscall 層(数値は一致)。
- `stat()` が返す `ENODEV` / `EINVAL` / `ECORRUPT` / `EIO` と、`resolve_path()` が返す
  `ENOENT` / `ENOTDIR` / `ENAMETOOLONG` は本表の対応どおり。`stat` 専用の新規 status は
  追加しない。

## 17. 前提チェックリスト

利用開始前に次を確認する。

1. Backend が §3 の 4 操作を例外なしで提供している。
2. イメージが `mkfs` 済みで、`mount` の `total_blocks` / `fs_start_block` が superblock と一致する。
3. 同一 `Filesystem` インスタンスを直列アクセスしている。
4. フルパス操作では `resolve_path` / `resolve_parent` → inode + 単一 name の形へ変換している。
5. directory read を 32 バイト単位で行い、`dirent_decode` している。
6. open 系エラー後、成功分の `open_file_t*` だけ `close` している。
7. `unmount` 前に不要な open entry をすべて `close` している。
8. partial 転送は byte 数と status の両方を処理している。
9. `ECORRUPT` を握りつぶさず、再マウントまたは fsck/再作成へ切り替えている。
10. `write_at` を directory inode に呼ばない(§9.1)。
11. `stat()` は open せずに呼び、open-file を消費しないことを確認する(§13)。

## 18. FD共通基盤から利用する場合の統合規約

本節は上位のFD共通基盤(`FileDescription` / `RegularFile`)から本APIを利用する場合の追加規約である。Filesystem単体APIの互換性を保つため`open_file_t`、`read(open_file_t*)`、`write(open_file_t*)`、`dup_file()`は残すが、FD統合経路ではopen file descriptionの共有状態を上位`FileDescription`へ一本化する。

- 1回の上位`open()`につき`open_file()`を1回だけ呼ぶ。
- 得られた`open_file_t*`は1個の`RegularFile`が保持する。
- 上位`FileDescription`がfile positionとfd参照数を保持するため、FD統合経路では`open_file_t::offset`と`open_file_t::ref_count`を共有状態の正本として使用しない。
- `RegularFile::read_at()` / `write_at()`は`open_file_t::inode_num`と上位から渡されたoffsetを使い、本APIの`read_at()` / `write_at()`を呼ぶ。
- `dup()`や`spawn()`によるfd共有では`dup_file()`を呼ばない。同じ`FileDescription` / `RegularFile` / `open_file_t*`を共有する。
- 上位`FileDescription`の最終参照解放時だけ、`RegularFile`から本APIの`close(open_file_t*)`を1回呼ぶ。
- `close()`がエラーを返した場合でも本仕様8.3節どおりentryは解放済みであるため、同じ`open_file_t*`へ再度`close()`してはならない。

この統合規約は既存Filesystem APIの削除・改名を要求しない。将来Filesystem単体の`open_file_t`を簡素化する場合は別変更として扱う。
