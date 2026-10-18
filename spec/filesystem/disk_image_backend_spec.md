# ディスクイメージ Backend 仕様書 (`disk_image_backend_spec.md`)

## 1. 目的と位置づけ

本書は、Ancient UNIX風サブセットOSの `Filesystem<Backend>` から利用する
ホスト用ディスクイメージ Backend `DiskImageBackend` の外部仕様を定める。

`DiskImageBackend` は、ホスト上の既存ファイルを固定長ブロックデバイスとして扱い、
`Filesystem<Backend>` / `BlockDevice<Backend>` に同期1ブロックI/Oを提供する。

本Backendは以下を行わない。

- ディスクイメージファイルの新規作成
- ディスクイメージのtruncate
- `mkfs`
- `fsck`
- superblock / FAT / inode / directory の解釈
- Filesystemとしての妥当性検査
- 故障注入
- I/Oログ・呼び出し回数等のテスト観測
- バッファキャッシュ

ディスクイメージが有効なFilesystemを保持していること、および必要な初期化・検査を
済ませていることは利用者側の責任とする。

本Backendはテスト専用 `FakeBlockBackend_DiskImage` の
同期1ブロックI/Oと状態遷移を参考にするが、
`FakeControl`、故障注入、観測機能は持たない。

---

## 2. Backend契約

既存の `Filesystem<Backend>` / `BlockDevice<Backend>` 契約に従い、
継承・virtualを用いない単純な型として実装する。

```cpp
struct DiskImageBackend {
    using OpenArg = const char*;

    FsStatus open(OpenArg path);
    FsStatus read(uint32_t blkno, uint8_t* buf);
    FsStatus write(uint32_t blkno, const uint8_t* buf);
    FsStatus close();

private:
    std::FILE* file_ = nullptr;
    uint32_t capacity_blocks_ = 0;
    bool is_open_ = false;
};
```

`FS_BLOCK_SIZE` は既存Filesystem仕様に従い 512 byte とする。

---

## 3. 所有権

`DiskImageBackend` は `open()` に成功した時点で、
内部の `std::FILE*` を所有する。

所有権は `close()` で終了する。

`OpenArg` として渡されたパス文字列自体は所有しない。
`open()` 完了後にパス文字列を保持する必要はない。

Backendは動的メモリ確保を必要としない。

---

## 4. `OpenArg`

```cpp
using OpenArg = const char*;
```

`OpenArg` は既存ディスクイメージファイルへのパスを表す。

以下を前提とする。

- NUL終端文字列である。
- 呼び出し時に有効である。
- 書込み可能な既存ファイルを指す。

Backendはパス文字列を保存しない。

---

## 5. 状態

Backendは次の2状態を持つ。

```text
CLOSED
  |
  | open()
  v
OPEN
  |
  | close()
  v
CLOSED
```

初期状態は `CLOSED` とする。

`CLOSED` 状態では、

```cpp
file_ = nullptr;
capacity_blocks_ = 0;
is_open_ = false;
```

を満たす。

`OPEN` 状態では、

```cpp
file_ != nullptr;
capacity_blocks_ > 0;
is_open_ = true;
```

を満たす。

---

## 6. `open()`

### 6.1 シグネチャ

```cpp
FsStatus open(OpenArg path);
```

### 6.2 動作

既存ファイルをread/write可能かつtruncateしないモードで開く。

概念上は次を用いる。

```cpp
std::fopen(path, "r+b");
```

ファイルを新規作成してはならない。
既存内容をtruncateしてはならない。

open後にファイルサイズを取得し、
そのサイズが `FS_BLOCK_SIZE` の正の整数倍であることを確認する。

成功時は、

```text
capacity_blocks_ = file_size / FS_BLOCK_SIZE
is_open_ = true
```

とする。

### 6.3 ファイルサイズ取得

実装は `std::fseek()` / `std::ftell()` を用いてよい。

概念上は次の手順とする。

```text
fseek(file, 0, SEEK_END)
ftell(file)
fseek(file, 0, SEEK_SET)
```

ファイルサイズ取得失敗は `FsStatus::EIO` とする。

### 6.4 エラー

| 条件 | 戻り値 |
|---|---|
| 既にopen済み | `FsStatus::EBUSY` |
| `path == nullptr` | `FsStatus::EINVAL` |
| `fopen(..., "r+b")` 失敗 | `FsStatus::ENODEV` |
| ファイルサイズ取得失敗 | `FsStatus::EIO` |
| ファイルサイズが0以下 | `FsStatus::EIO` |
| ファイルサイズが `FS_BLOCK_SIZE` の倍数でない | `FsStatus::EIO` |
| block数が `uint32_t` で表現できない | `FsStatus::EIO` |
| 成功 | `FsStatus::OK` |

失敗時は開いた `FILE*` があれば閉じ、
`CLOSED` 状態へ戻す。

---

## 7. 容量

`capacity_blocks_` はディスクイメージファイルの実サイズから計算した
Backend物理容量を表す。

```text
capacity_blocks_ = file_size / FS_BLOCK_SIZE
```

上位の `BlockDevice<DiskImageBackend>` が保持する `total_blocks_` は
論理的な利用上限であり、通常は次を満たすことを利用者側の責任とする。

```text
BlockDevice total_blocks <= DiskImageBackend capacity_blocks
```

Backend自身は `read()` / `write()` で必ず `capacity_blocks_` を検査するため、
上位から誤ったblock番号が渡されてもファイル末尾を越えてI/Oしてはならない。

---

## 8. `read()`

### 8.1 シグネチャ

```cpp
FsStatus read(uint32_t blkno, uint8_t* buf);
```

### 8.2 動作

指定blockを1blockだけ読み出す。

概念上の処理は次の通り。

```cpp
const long offset =
    static_cast<long>(blkno) * static_cast<long>(FS_BLOCK_SIZE);

if (std::fseek(file_, offset, SEEK_SET) != 0) {
    return FsStatus::EIO;
}

if (std::fread(buf, FS_BLOCK_SIZE, 1, file_) != 1) {
    return FsStatus::EIO;
}
```

部分転送は成功として扱わない。
`FS_BLOCK_SIZE` byteを完全に読み出せなければ `EIO` とする。

### 8.3 検査順序

次の順序で検査する。

1. open状態
2. `buf != nullptr`
3. `blkno < capacity_blocks_`
4. seek
5. 1block read

### 8.4 戻り値

| 条件 | 戻り値 |
|---|---|
| 未open | `FsStatus::ENODEV` |
| `buf == nullptr` | `FsStatus::EINVAL` |
| `blkno >= capacity_blocks_` | `FsStatus::ERANGE` |
| seek失敗 | `FsStatus::EIO` |
| 1block完全にreadできない | `FsStatus::EIO` |
| 成功 | `FsStatus::OK` |

Backend検査で失敗した場合は `buf` の内容を変更しない。

---

## 9. `write()`

### 9.1 シグネチャ

```cpp
FsStatus write(uint32_t blkno, const uint8_t* buf);
```

### 9.2 動作

指定blockへ1blockだけ書き込む。

概念上の処理は次の通り。

```cpp
const long offset =
    static_cast<long>(blkno) * static_cast<long>(FS_BLOCK_SIZE);

if (std::fseek(file_, offset, SEEK_SET) != 0) {
    return FsStatus::EIO;
}

if (std::fwrite(buf, FS_BLOCK_SIZE, 1, file_) != 1) {
    return FsStatus::EIO;
}
```

部分転送は成功として扱わない。
`FS_BLOCK_SIZE` byteを完全に書けなければ `EIO` とする。

### 9.3 検査順序

次の順序で検査する。

1. open状態
2. `buf != nullptr`
3. `blkno < capacity_blocks_`
4. seek
5. 1block write

### 9.4 戻り値

| 条件 | 戻り値 |
|---|---|
| 未open | `FsStatus::ENODEV` |
| `buf == nullptr` | `FsStatus::EINVAL` |
| `blkno >= capacity_blocks_` | `FsStatus::ERANGE` |
| seek失敗 | `FsStatus::EIO` |
| 1block完全にwriteできない | `FsStatus::EIO` |
| 成功 | `FsStatus::OK` |

Backend検査で失敗した場合はディスクイメージ内容を変更しない。

---

## 10. write時のflush

`write()` ごとに `std::fflush()` を呼ぶことは必須としない。

初期実装では、

```text
write() -> fwrite()
close() -> fclose()
```

でよい。

`fclose()` によりstdio bufferはflushされる。

将来、`Filesystem::sync()` とホストファイルの永続反映を厳密に対応させる必要が
生じた場合は、Backend共通契約への `sync()` / `flush()` 追加を別途検討する。

現段階で `DiskImageBackend` 独自のflush APIを追加してはならない。

---

## 11. `close()`

### 11.1 シグネチャ

```cpp
FsStatus close();
```

### 11.2 動作

open中の `FILE*` を `std::fclose()` する。

`fclose()` の成否にかかわらず、Backendは終端的に `CLOSED` 状態へ遷移する。

概念上は次の通り。

```cpp
const int rc = std::fclose(file_);

file_ = nullptr;
capacity_blocks_ = 0;
is_open_ = false;

return rc == 0 ? FsStatus::OK : FsStatus::EIO;
```

### 11.3 戻り値

| 条件 | 戻り値 |
|---|---|
| 未open | `FsStatus::ENODEV` |
| `fclose()` 成功 | `FsStatus::OK` |
| `fclose()` 失敗 | `FsStatus::EIO` |

`fclose()` 失敗後も再closeしてはならない。

Backend closeは終端操作であり、上位 `BlockDevice` / `Filesystem::unmount()` から
再試行されない既存契約に従う。

---

## 12. 転送仕様

| 項目 | 仕様 |
|---|---|
| block size | `FS_BLOCK_SIZE` = 512 byte |
| 転送単位 | 常に1block |
| 転送方式 | `fseek` + `fread` / `fwrite` |
| 同期性 | API呼出し自体は同期 |
| read buffer | 呼び出し側所有 |
| write buffer | 呼び出し側所有 |
| image file | Backendがopen中の `FILE*` を所有 |
| 部分転送 | なし |
| バッファキャッシュ | Backendでは持たない |
| writeごとのflush | 必須ではない |

---

## 13. `BlockDevice<Backend>` との責務分担

`DiskImageBackend` はBackend単体でも安全に動作できるよう、
open状態、null buffer、block範囲を検査する。

上位の `BlockDevice<DiskImageBackend>` が同様の検査を行う場合でも、
Backend側の検査は削除しない。

理由は、Backendがホストファイルへ直接アクセスする最下層であり、
ファイル末尾を越えるI/OをBackend自身でも防止するためである。

---

## 14. Filesystemとの境界

`DiskImageBackend` はFilesystemレイアウトを解釈しない。

以下はすべて上位層または利用者側の責務とする。

```text
image作成
superblock
FAT
inode
directory
pathname
mkfs
fsck
mount時のFilesystem検証
```

ディスクイメージ内容が未初期化、破損、またはFilesystemとして不正であっても、
Backendの `open()` 自体は、ファイルがblock deviceとして扱えるサイズなら成功してよい。

その後の `Filesystem::mount()` が内容を読み取り、
Filesystem仕様に従って `ECORRUPT` 等を返すことはBackendの責務ではない。

---

## 15. テスト専用Fakeとの相違

`FakeBlockBackend_DiskImage` から引き継ぐのは、
既存ファイルを `"r+b"` で扱うこと、1block同期I/O、
基本status、状態遷移だけとする。

本番 `DiskImageBackend` には以下を持ち込まない。

- `FakeControl`
- `next_*_status`
- block指定故障注入
- payload指定故障注入
- I/Oログ
- read/writeカウンタ
- write履歴
- last read/write記録

---

## 16. ホストstdioのサイズ制約

初期実装では可搬性と単純さを優先し、
`std::fseek()` / `std::ftell()` を使用してよい。

この場合、扱える最大ディスクイメージサイズは
ホスト処理系の `long` およびstdio実装の制約を受ける。

本OSの当面の用途ではこれを許容する。

より大きなimageが必要になった場合は、
`fseeko()` / `ftello()` またはプラットフォーム固有64bit seek APIへの移行を
別途検討する。

---

## 17. 最小検証項目

実装時には少なくとも以下を確認する。

1. 既存の正しいサイズのimageを `open()` できる。
2. 二重 `open()` が `EBUSY` になる。
3. 存在しないimageの `open()` が `ENODEV` になる。
4. 0 byte imageを拒否する。
5. `FS_BLOCK_SIZE` の倍数でないimageを拒否する。
6. 未openの `read()` / `write()` / `close()` が `ENODEV` になる。
7. null bufferが `EINVAL` になる。
8. `blkno == capacity_blocks_` が `ERANGE` になる。
9. block 0を書いてreadすると同じ512 byteを取得できる。
10. 最終blockを書いてreadすると同じ512 byteを取得できる。
11. `close()` 後に同じimageを再openするとwrite内容が保持されている。
12. `open()` により既存image内容がtruncateされない。
13. Backendは動的メモリ確保を行わない。

---

## 18. 実装上の禁止事項

- 例外を使用しない。
- RTTIを使用しない。
- virtualを導入しない。
- ディスクイメージファイルを暗黙に新規作成しない。
- `open()` 時にtruncateしない。
- Filesystem内部構造を解釈しない。
- `mkfs` / `fsck` / image生成機能を追加しない。
- テスト専用故障注入機能を本番Backendへ追加しない。
- Backend独自のmount/unmount概念を追加しない。
