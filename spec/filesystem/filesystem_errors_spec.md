# エラー仕様 — `FsStatus`

> `AGENTS.md`の共通結果規約を上位仕様とする。本書は全コード一覧と選択基準を定義する。

## 1. 列挙

```cpp
enum class FsStatus {
    OK,
    EIO,
    ERANGE,
    ENODEV,
    ENOSPC,
    ENOENT,
    EEXIST,
    EISDIR,
    ENOTEMPTY,
    EINVAL,
    ESPIPE,
    ENOTDIR,
    ENAMETOOLONG,
    EMLINK,
    ECORRUPT,
    EBUSY,
    EWP,
};
```

- この列挙は`fs/src/fs_status.h`の`namespace fs`内に定義される。`fs`の外側からは`fs::FsStatus`で参照する。

## 2. 使い分け

| 値 | 使用条件 |
|---|---|
| `OK` | 操作が正常終了した。EOFや長さ0のreadも含む |
| `EIO` | BackendまたはブロックI/Oが失敗した |
| `ERANGE` | offset、size、変換結果が表現・FS容量範囲外 |
| `ENODEV` | デバイスが未open、存在しない、利用不能 |
| `ENOSPC` | クラスタ、inode、dirent、固定テーブルが枯渇した |
| `ENOENT` | 指定名またはパス成分が存在しない |
| `EEXIST` | `mkdir()`や`link()`の新しい名前等、存在が禁止される対象が存在する |
| `EISDIR` | 通常ファイルが必要な操作へディレクトリを渡した |
| `ENOTEMPTY` | `rmdir()`対象directoryに`.`/`..`以外の有効entryがある |
| `EINVAL` | 空パス、負になるseek等、他に専用値がない不正引数 |
| `ESPIPE` | pipe等のseek不能対象に`lseek()`した |
| `ENOTDIR` | パス途中の成分がディレクトリでない |
| `ENAMETOOLONG` | 名前が27バイト超、またはパスが上限以上 |
| `EMLINK` | `nlink == UINT16_MAX`から増加しようとした |
| `ECORRUPT` | オンディスク構造またはチェーンが仕様違反 |
| `EBUSY` | 既にopen中の`BlockDevice`に対する再open、またはBackend固有のbusy状態 |
| `EWP` | 将来の書込保護Backend用予約 |

## 3. API形

- 成否だけを返す操作は`FsStatus`を返す。
- byte countを伴う操作は`std::pair<uint32_t, FsStatus>`を返す。
- partial I/Oでは処理済みバイト数と停止理由を同時に返す。
- syscall層だけが`FsStatus`をOS errnoへ変換する。

## 4. 優先規則

- 引数検査は状態変更より前に行う。
- オンディスク矛盾を検出した場合は`ECORRUPT`を優先する。
- Backend I/O失敗は`EIO`またはBackend由来の適切な値を返す。
- 一度確定した具体的な失敗を、後段の汎用エラーで上書きしない。
- EOFはエラーではない。
- `creat()`で同名の通常ファイルが存在することはエラーではなくtruncate条件である。
- `mkdir()`で同名の通常ファイルまたはdirectoryが存在することは`EEXIST`である。
- `rmdir()`で`.`/`..`以外の有効entryが残るdirectoryを削除しようとした場合は`ENOTEMPTY`である。
- `link()`で新名が存在することは`EEXIST`である。
- ディレクトリに対する`creat`、`write`、`unlink`、`link`元は`EISDIR`とする。
