# `stat` システムコール実装仕様

## 1. 目的

本書はAncient UNIX風サブセットOSの`stat()`システムコールと、そのために追加するFilesystem属性取得APIの実装契約を定める。AIエージェントは本書を直接の実装指示として扱い、既存のpathname解決・inode/FAT・プロセス管理の境界を維持すること。

対象は通常ファイルとディレクトリの**pathnameによる属性取得**である。既存の`creat`/`open`/`close`/`read`/`write`は実装済みとする。`fstat`、シンボリックリンク、デバイス特殊ファイルの属性取得は今回の対象外とする。

本段階のゴールは疑似ユーザープログラムから以下の経路を成立させること。

```text
btx::sys_stat(path, buf)
  -> kernel::sys_stat(path, buf)         // 薄いラッパ
  -> kernel::kernel_stat(path, buf)      // cwd・pathname解決・属性変換
       -> Filesystem::resolve_path(cwd_inode, path, &inode)
       -> Filesystem::stat(inode, &info)
            -> inode読出し・検証
            -> FATチェーン計数
       -> fs::FileStat から struct stat への変換
```

## 2. 前提と設計原則

- システムコールは薄いラッパとし、`sys_stat()`は`kernel_stat()`へ委譲する。
- pathname解決は`kernel_stat()`内から既存の`Filesystem::resolve_path()`を呼ぶ。`Filesystem::stat()`には**inode番号だけ**を渡し、pathnameやプロセスを渡さない（合意済みB案）。
- Filesystem利用者は公開`Filesystem<Backend>` APIのみを利用する。カーネルから`InodeTable`、`Fat`、`disk_inode_t`にアクセスしない。
- OS用`struct stat`とFS用`fs::FileStat`を分離し、オンディスクinode形式をOSのABIとして公開しない。
- `stat()`はfdやopen file descriptionを確保しない。`FdTable`、`RegularFile`、`FileDescription`、Filesystemの`open_file_t`に触らない。
- `stat()`は参照専用。inodeの時刻、size、offset、FATを変更しない。`atime`の更新も行わない。
- 動的メモリ確保、RTTI、例外、新規の仮想関数は追加しない。既存FSの故障検出・`ECORRUPT`安全停止方針に従う。
- FSの同一インスタンスへの並行操作は既存仕様どおり外部で直列化する。
- 公開APIの命名や型名に既存コードとの差異がある場合、**意味を維持した最小変更**とし、同等の並行APIを重複作成しない。

## 3. ユーザー向け `struct stat` と定数

### 3.1 `struct stat`

ユーザー空間とカーネルで共通に参照する`sys/stat.h`へ、UNIX系で一般的な主要13メンバを定義する。現段階は**メンバ名と意味の互換**を目的とし、既存OSとのバイナリABI互換や現行POSIXのナノ秒時刻APIを要求しない。

```cpp
#include <stdint.h>

struct stat {
    uint32_t st_dev;
    uint32_t st_ino;
    uint32_t st_mode;
    uint32_t st_nlink;
    uint32_t st_uid;
    uint32_t st_gid;
    uint32_t st_rdev;
    int64_t  st_size;
    int64_t  st_atime;
    int64_t  st_mtime;
    int64_t  st_ctime;
    int32_t  st_blksize;
    int64_t  st_blocks;
};
```

- `st_size`は符号付き64bitだが、現FSの論理サイズ上限は`uint32_t`の範囲とする。
- `st_atime`等は**UNIX epoch起点の秒数**とし、現FSの時刻値（下位32bit運用）を64bitへ拡張する。`atime = mtime`とする既存FS phase 1契約を維持する。
- `st_blksize`は推奨I/O単位として512、`st_blocks`は**実際に割当済みの512バイトブロック数**とする。両者を混同しない。
- 当面`st_dev = 0`（単一ルートFSの固定識別値）、`st_uid = 0`、`st_gid = 0`、`st_rdev = 0`とする。これらをinodeへ永続化しない。
- 同一ターゲットのカーネルとユーザー空間は同じヘッダとABI設定でビルドする。Cortex-M0/MC68000間の構造体メモリ配置の一致は要求しない。`packed`を付けない。
- 本OS独自の`struct stat`がホストlibcの同名構造体と競合しないよう、既存ユーザー向けヘッダ配置・ビルド規則に従う。ホストの`<sys/stat.h>`を不用意に混在させない。

### 3.2 ファイル種別・権限ビット

既存FSのmode形式と整合するUNIX型の定数を共有する。

```cpp
#define S_IFMT   0170000
#define S_IFREG  0100000
#define S_IFDIR  0040000
#define S_ISREG(m) (((m) & S_IFMT) == S_IFREG)
#define S_ISDIR(m) (((m) & S_IFMT) == S_IFDIR)
```

権限ビットは下位9bit（`0777`）で表す。既存FSの`fs::S_IFREG`/`fs::S_IFDIR`と**数値が一致**することをテストする。OSヘッダをFSのオンディスク定義に依存させない。必要な`S_IRUSR`等の標準的権限定数は同じヘッダに追加してよいが、未対応ファイル種別を正常に扱えると偽装してはならない。

## 4. FS属性構造体 `fs::FileStat`

Filesystemの公開型として、ディスク表現と独立した次の結果型を追加する。

```cpp
namespace fs {
struct FileStat {
    uint32_t inode;
    uint16_t mode;
    uint16_t nlink;
    uint32_t size;
    uint32_t blocks; // 512-byte units
    uint64_t mtime;
    uint64_t ctime;
};
}
```

- `FileStat`はFSが管理する事実のみを表し、uid/gid/dev等のOS固定値を含めない。
- 現phase 1では1クラスタ = 512バイトなので、`blocks`は**現在inodeから到達可能な割当クラスタ数**と等しい。
- FSに存在しない独立した`atime`は持たない。カーネル側で`st_atime = info.mtime`とする。
- `FileStat`はメモリ内API用。encode/decodeされるオンディスクレコードとして扱わない。

## 5. `Filesystem::stat()` の追加契約

```cpp
FsStatus stat(uint32_t inode_num, FileStat* out);
```

### 5.1 処理

1. mount状態と引数を検査する。
2. 指定inode番号が有効範囲にあることを検査する。
3. 既存のinode管理内部APIでinodeレコードを読み、型、live状態、必要な不変条件を検証する。
4. inodeの`mode`、`nlink`、`size`、`mtime`、`ctime`を取得する。
5. inodeの`first_cluster`から**FATチェーンを実走査**し、現在到達可能なクラスタ数を数える（§5.2）。
6. ローカル`FileStat result{}`へ全項目を構築し、**すべて成功した最後に**`*out = result`を実行する。
7. `FsStatus::OK`を返す。

`Filesystem::stat()`内で`resolve_path()`を呼ばない。`open_file()`や`close()`も呼ばない。inode読出しに必要な内部I/Oは認めるが、書込み・資源割当・時刻変更は禁止する。

### 5.2 FATチェーンの計数

- `first_cluster == CLUSTER_NONE`ならクラスタ数は0。`size != 0`なら構造不整合として`ECORRUPT`。
- それ以外では先頭クラスタから終端`FAT_EOF`まで、既存FAT内部APIを利用してチェーンを順にたどる。
- 各クラスタ番号の有効範囲、予約値・空き値の不正混入、循環、チェーン長の上限を検査する。いずれも破損なら`ECORRUPT`。
- **`size`から`ceil(size / 512)`を計算して`blocks`の代わりに返すことは禁止**する。実際のチェーン長を数える。
- サイズとチェーン長の対応について、既存FSが保証する不変条件を確認・適用する。ゼロ長だが割当済みのチェーンなど、正当か不正か既存内部契約で判断できない状態を推測で改変しない。その場合は実装時に仕様矛盾を報告する。
- directoryも`st_blocks`の計数対象とする（`.`/`..`やdirentの格納に利用するデータクラスタを含む）。
- truncate/unlink等で切り離された、inodeから到達不能な旧チェーンや回収されないクラスタは含めない。
- クラスタ数計数の上限はFS幾何情報の範囲で管理し、整数オーバーフローや無限ループを起こさない。破損時は処理を安全に中止する。
- 計数の最適化・キャッシュ・inodeへのクラスタ数追加は今回は行わない。計算量`O(チェーン長)`を許容する。

### 5.3 エラーと出力不変条件

| 条件 | 戻り値 |
|---|---|
| 正常な通常ファイル・directory | `FsStatus::OK` |
| 未mount | `FsStatus::ENODEV` |
| `out == nullptr` | `FsStatus::EINVAL` |
| inode番号の範囲外・0等の不正指定 | `FsStatus::EINVAL`（既存下層契約に従う） |
| 未使用/削除済みinode、inode/FATの破損 | `FsStatus::ECORRUPT` |
| inode等の読み取りI/O失敗 | `FsStatus::EIO`またはBackend由来のエラー |

- **失敗時は`*out`を一切変更しない**。`out == nullptr`の場合は書き込まない。
- 既存Filesystemの`FsStatus`選択と不整合がある場合、下層の契約を無断変更せず整合箇所を報告する。
- `stat`実行だけでFS内open-file表の使用数は増減しない。

## 6. カーネル内部API `kernel::kernel_stat()`

```cpp
namespace kernel {
int32_t kernel_stat(const char* path, struct stat* buf);
}
```

### 6.1 処理順序

1. `path == nullptr`または`buf == nullptr`なら失敗する。
2. current processのcwd inode番号を取得する。`open()`で利用する既存のcwd取得規則・暫定root扱いを**再利用**し、stat固有の別実装を作らない。
3. `Filesystem::resolve_path(cwd_inode, path, &inode_num)`で最終inodeを取得する。相対パス、絶対パス、`/`、`.`、`..`等の意味は既存のパス解決契約どおり。
4. `Filesystem::stat(inode_num, &info)`を呼び出す。
5. ローカル`struct stat result{}`を構築し、以下の表のとおり各値を設定する。
6. **最後に**`*buf = result`を実行し、`0`を返す。

| ユーザー向けメンバ | 値 |
|---|---|
| `st_dev` | `0` |
| `st_ino` | `info.inode` |
| `st_mode` | `info.mode` |
| `st_nlink` | `info.nlink` |
| `st_uid` / `st_gid` | `0` |
| `st_rdev` | `0` |
| `st_size` | `info.size` |
| `st_atime` | `info.mtime` |
| `st_mtime` | `info.mtime` |
| `st_ctime` | `info.ctime` |
| `st_blksize` | `fs::FS_BLOCK_SIZE`（512） |
| `st_blocks` | `info.blocks` |

### 6.2 戻り値・エラー

- 成功時`0`、失敗時`-1`。
- `resolve_path()`や`Filesystem::stat()`の失敗は成功として扱わず、既存syscallで採用しているエラー報告規約に変換する。FS自体はグローバル`errno`を持たない。
- 現行OSのsyscallが`-1`のみを返し`errno`を提供していない場合、今回だけ独立した`errno`機構を新設しない。エラー変換の拡張は既存OSの共通ルールが確定してから行う。
- pathnameの未存在は`ENOENT`、途中の非directoryは`ENOTDIR`、不正パスは`EINVAL`/`ENAMETOOLONG`等の既存`resolve_path()`の結果を維持する。
- 失敗時の`buf`は不変。fd/open-file/offsetを変更しない。

### 6.3 概念コード

```cpp
int32_t kernel::kernel_stat(const char* path, struct stat* buf)
{
    if (path == nullptr || buf == nullptr) return -1;

    uint32_t inode = 0;
    const auto resolved = G_filesystem.resolve_path(
        current_process().cwd_inode, path, &inode);
    if (resolved != fs::FsStatus::OK) return -1;

    fs::FileStat info{};
    const auto status = G_filesystem.stat(inode, &info);
    if (status != fs::FsStatus::OK) return -1;

    struct stat result{};
    result.st_dev = 0;
    result.st_ino = info.inode;
    result.st_mode = info.mode;
    result.st_nlink = info.nlink;
    result.st_uid = result.st_gid = result.st_rdev = 0;
    result.st_size = info.size;
    result.st_atime = info.mtime;
    result.st_mtime = info.mtime;
    result.st_ctime = info.ctime;
    result.st_blksize = fs::FS_BLOCK_SIZE;
    result.st_blocks = info.blocks;
    *buf = result;
    return 0;
}
```

`G_filesystem`/`current_process()`は**仮名**。既存のFS参照取得・プロセス管理APIに置き換えること。引数ポインタのユーザー/カーネル間安全性確認は本OSの共通syscall境界規約に従い、`stat`だけに仮想メモリ検査機構を新設しない。

## 7. `kernel::sys_stat()`

```cpp
int32_t kernel::sys_stat(const char* path, struct stat* buf)
{
    return kernel_stat(path, buf);
}
```

- syscallディスパッチは既存の自動生成規約に従い接続する。
- `sys_stat()`でpathname解決、FSアクセス、`FdTable`操作、構造体のメンバ変換を行わない。
- ユーザー向け宣言は既存の`btx::sys_*`呼び出し規約に整合させ、`struct stat`の共通ヘッダを利用する。

## 8. 実装順序

1. `sys/stat.h`に`struct stat`とファイル種別定数を定義。既存FS mode値との一致を検証。
2. ファイルシステム公開型`fs::FileStat`を追加。
3. `Filesystem::stat(inode, out)`を実装。inode属性読出し、FATチェーン計数、エラー時出力不変を単体テスト。
4. `kernel::kernel_stat()`を実装。既存のcurrent process/cwd参照、`resolve_path()`、FS属性変換を接続。
5. `kernel::sys_stat()`とユーザー向けラッパを既存syscall生成方式で接続。
6. 疑似ユーザープログラムでend-to-endテストおよび既存I/O回帰テストを実施。

## 9. 必須テスト

少なくとも以下を自動テストする。

1. 新規作成したサイズ0の通常ファイルは`S_ISREG(st_mode)`が真、`st_size=0`、`st_blocks=0`。
2. `write()`で1バイト書いたファイルは`st_size=1`、実クラスタ数に応じて`st_blocks=1`。
3. 512バイトを超えて書いたファイルの`st_blocks`が実FATチェーン長に一致する。
4. ディレクトリに対して成功し、`S_ISDIR(st_mode)`が真。`/`の`st_ino`は`ROOT_INODE`。
5. ディレクトリについても`st_blocks`を実チェーンから取得する。
6. 絶対パスと相対パスで同一inodeに解決されるケースでは属性が一致する。
7. 存在しない最終パスは失敗し、出力バッファを変更しない。
8. 途中のパス成分が通常ファイルなら失敗する。
9. 空パス、長すぎるパス、`nullptr`の`path`/`buf`は失敗する。
10. `Filesystem::stat()`の範囲外inode・未使用inode・未mountをテストする。
11. FATチェーンの範囲外参照、循環、FREE/予約エントリ混入が`ECORRUPT`になり、出力構造体が不変。
12. サイズ非0で`first_cluster == CLUSTER_NONE`なら`ECORRUPT`。
13. inode読出しI/O故障時にエラーを返し、出力が不変。
14. `st_blksize == 512`、`st_blocks`は論理サイズの単純な切上げ値ではなくチェーン長から得ること。
15. `st_uid/st_gid/st_dev/st_rdev`は契約した固定値、`st_atime == st_mtime`。
16. `stat`前後でFD数、`FileDescription`数、Filesystem open-file数および既存fdのoffsetが変化しない。
17. `open -> write -> stat -> read -> close`の既存機能を壊さない（読出しは適切なオフセット／再open後に行う）。
18. 同一ファイルを別々にopenした状態でも`stat`の結果はinode属性を反映し、open file descriptionに依存しない。
19. unlink済みでpathnameが存在しない場合の`stat(path)`は失敗する。open-but-unlinkedの既存read/write動作は回帰しない。
20. FSのmode定数とユーザー向け`S_IFREG`/`S_IFDIR`が一致する。

## 10. 実装完了条件

疑似ユーザープログラムから次の検証が通ること（宣言・フラグの名前は既存コードと一致させる）。

```cpp
int32_t fd = btx::sys_creat("/test.txt", 0644);
// fd >= 0
btx::sys_write(fd, "Hello, World\n", 13);
btx::sys_close(fd);

struct stat info{};
int32_t ret = btx::sys_stat("/test.txt", &info);
// ret == 0
// S_ISREG(info.st_mode)
// info.st_size == 13
// info.st_blocks == 1
// info.st_blksize == 512

ret = btx::sys_stat("/missing.txt", &info);
// ret == -1
```

`creat -> write -> close -> stat`が成立し、ファイルを再度openせずに属性が取得できること。また、`sys_stat`が`kernel_stat`への委譲のみとなり、filesystemからinode属性を取得する処理が`Filesystem::stat()`に閉じていること。

## 11. 既存仕様との整合・実装時の注意

- FS外部仕様（現行）では`stat`は未提供（§14）。今回の`Filesystem::stat()`は**新規公開API**であり、実装完了後はFS外部仕様を更新する。
- FATチェーン走査は既存の内部FAT契約に合わせて実装する。**公開FS APIしか見えない段階で存在しない内部メソッドを創作しない**。チェーン走査で不足する内部操作は既存FS実装を確認して必要最小限のみ追加する。
- `st_blocks`の正確さには、既存inode/FATチェーンの不変条件の理解が必要。内部実装を確認せず「sizeで計算すれば正しい」と置き換えない。
- FS phase 1は単一FS・uid/gid固定・時刻の下位32bit運用。OSの`struct stat`は将来拡張余地を残すが、今回新しい永続フィールドやディスク形式変更を導入しない。
- ファイルシステムの破損時安全停止と、システムコールの通常エラー（`-1`）は層を分ける。既存のエラー規約を壊して独自`errno`を追加しない。
- 本書は設計・実装指示書であり、`Filesystem::stat()`など新APIが既に実装されていることを示すものではない。
