# 07 — `creat()`・`write_at()`

> 共通のstatus、overflow、部分転送、時刻規約は`AGENTS.md`を上位仕様とする。本書末尾の`write_at()`境界値テストは必須である。

## 1. `creat()`契約

```cpp
FsStatus creat(uint32_t dir_inode_num,
               const char* name,
               uint16_t mode,
               uint32_t* out_inode_num);
```

- pathname解決後の「親directory inode + 単一名」を受け取る低レベルAPIとする。
- 名前と親directoryを変更前に検証する。
- modeから許可9ビットを保持する。
- phase 1ではumaskを適用しない。
- 作成時刻として`ctime`、`mtime`、`atime`をproviderから設定する。

## 2. 既存名の処理

- `dir_lookup()`が既存inodeを返した場合、V7の`creat(2)`挙動を採用する。
- 既存対象が通常ファイルなら同じinodeをtruncateして再利用する。
- `EEXIST`を返さない。
- 既存対象がdirectoryなら`EISDIR`を返す。
- truncateでは論理sizeを0にする。
- phase 1では旧clusterを再利用可能領域へ戻さない。
- `first_cluster`の扱いはphase 1の非再利用規約と整合させる。
- truncateは`mtime`と`ctime`を更新する。
- 成功時に既存inode番号を返す。

## 3. 新規作成の順序

1. `dir_lookup()`で非存在を確認する。
2. `alloc_inode()`でinode番号を確保する。
3. inode全フィールドを初期化する。
4. `mode = S_IFREG | permission_bits`とする。
5. `nlink = 1`、`uid = gid = 0`とする。
6. `size = 0`、`first_cluster = CLUSTER_NONE`とする。
7. 時刻を初期化する。
8. `put_inode()`する。
9. `dir_add_entry()`で名前を公開する。
10. 成功時だけ出力inode番号を確定する。

- `put_inode()`失敗時はdirentを作らず、そのstatusを返す。まだlive inodeの永続化に成功していないため、`dec_nlink()`は呼ばない。
- `dir_add_entry()`失敗時は、`09-unlink-close.md`の`drop_link()`でrollbackする。
- rollbackでは`dec_nlink()`後に`finalize_if_orphaned()`を行う。この時点ではopenされていないため、正常時は`finalize_inode()`によって`S_IUNUSED`へ遷移する。
- rollback後のinodeを`S_IFREE`へ戻さない。
- `drop_link()`失敗時は名前を公開せず、inodeまたはlink countの残留を許容する。dangling direntを作るより安全な状態を優先する。
- 名前だけが未初期化inodeを指す瞬間を作らない。

## 4. `write_at()`契約

```cpp
std::pair<uint32_t, FsStatus> write_at(
    uint32_t inode_num,
    uint32_t offset,
    const uint8_t* src,
    uint32_t len);
```

- open-file offsetは変更しない。
- `len == 0`は`{0, OK}`とする。
- null bufferは長さ0以外で拒否する。
- `required_end = offset + len`を事前検査する。
- 最大ファイルサイズまたはFS容量を超える要求は`ERANGE`とする。
- 返すbyte countは実際に永続化できたバイト数とする。
- 上位の`write()`だけがopen-file offsetを処理済みbyte数だけ進める。
- directoryへのユーザwrite拒否は上位層で行う。

## 5. cluster確保

- size 0かつ未割当ファイルの最初のwriteでは先頭clusterをallocする。
- 既存chainが必要cluster数に足りなければ末尾から逐次extendする。
- 必要な分だけ確保する。
- 倍々確保やchain全体コピーは行わない。
- FAT枯渇時は可能な範囲の部分writeと`ENOSPC`を返せる。
- 未初期化データが読めないよう新clusterを適切にzero-fillする。
- EOFより後へwriteする場合、論理gapはゼロとして読める状態にする。

## 6. block書込み

- 先頭・末尾の部分blockはread-modify-writeする。
- 1blockを完全上書きする場合は不要なreadを省略してよい。
- chain内の必要clusterがpremature EOFなら`ECORRUPT`とする。
- 成功したblock/byteだけを`written`へ加える。
- I/O失敗後に次blockへ進まない。
- 部分成功では`{written, stopping_status}`を返す。

## 7. inode更新

- 新sizeは`max(old_size, offset + written)`とする。
- size計算もoverflow検査する。
- 内容またはsizeが変わった場合、`mtime`と`ctime`を更新する。
- `atime`はphase 1で`mtime`と同値に保つ。
- inodeのsize/cluster情報を更新できなかったbyteを成功数へ含めない。
- データとinode更新の失敗境界をfault injectionで明示する。

## 8. 必須境界値テスト

DESIGN 11.1.15のテスト条件をこのレイヤーの完了条件として組み込む。

### ファイルサイズ境界

- size 0。
- size 1。
- size 511。
- size 512。
- size 513。
- size 1023。
- size 1024。

### `write_at()`の必須ケース

- `offset=0, len=1`。
- `offset=511, len=2`。
- `offset=512, len=1`。
- `offset=512, len=512`。
- `offset + len`が`uint32_t`境界直前。
- `offset + len`がoverflowする値。
- EOF位置へのwrite。
- EOF+1以上へseekした後のwriteとgapの読み戻し。
- 先頭cluster未割当からのwrite。
- 既存chain内だけで完了するwrite。
- chainを1cluster延長するwrite。
- 複数cluster延長するwrite。
- FAT枯渇前に一部だけ書けるwrite。
- 最初のblock read失敗。
- 中間block write失敗。
- inode更新失敗。

### 関連境界

- inode tableのcluster境界にあるinodeへのwrite。
- 最大有効cluster付近のchain延長。
- 最大inode番号付近の位置計算。
- `creat()`の同名通常ファイルtruncate。
- `creat()`の同名directory拒否。
- `dir_add_entry()`失敗時に`drop_link()`で行うorphan rollback。
- `put_inode()`失敗時にdirentが作られないこと。

## 9. 完了条件

- 上記必須テストが全て通る。
- 部分writeのbyte countとstatusが両方正しい。
- rollback後に名前付き不正inodeが残らない。
- `dir_add_entry()`失敗後、rollback成功時にinodeが`S_IUNUSED`になる。
- overflowする要求で割当・I/O・状態変更が起きない。
