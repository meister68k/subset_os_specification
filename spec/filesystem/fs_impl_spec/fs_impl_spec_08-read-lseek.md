# 08 — `read_at()`・`read()`・`lseek()`

> 共通の部分転送とoffset規約は`AGENTS.md`を参照する。

## 1. `read_at()`

```cpp
std::pair<uint32_t, FsStatus> read_at(
    uint32_t inode_num,
    uint32_t offset,
    uint8_t* dst,
    uint32_t len);
```

- inodeを取得しlive状態を確認する。
- `len == 0`は`{0, OK}`とする。
- `offset >= inode.size`は`{0, OK}`とする。
- 読込長は`min(len, inode.size - offset)`とする。
- 計算はunderflow/overflowを避ける。
- open-file offsetは変更しない。
- directoryも同じbyte列primitiveで読み出せる。

## 2. chain read

- offsetから開始cluster indexとcluster内offsetを求める。
- `Fat::walk_to_index()`で開始clusterを得る。
- 必要なblockを順に読む。
- 各blockから要求範囲だけコピーする。
- EOFまでにchainが終われば`ECORRUPT`とする。
- 範囲外clusterやcycleはFAT層で停止する。
- I/O失敗時は既にcopyしたbyte数とstatusを返す。
- 全要求完了または論理EOF到達は`OK`とする。

## 3. `read()` wrapper

- `open_file_t.offset`を開始offsetとして`read_at()`を呼ぶ。
- 実読込byte数だけoffsetを進める。
- errorを伴うpartial readでも実読込分は進める。
- 複数fdが同じopen file descriptionを共有する場合、offsetも共有される。

## 4. directory read

- phase 1ではraw `dirent_t`列を通常readで公開する。
- FS内部はdirent境界を強制しない。
- userlandはserialized dirent size単位で要求する。
- 分断されたdirentを次readへ内部持越しする機構は設けない。

## 5. atime

- phase 1ではreadごとのatime更新を行わない。
- atimeはmtimeと同値の簡略仕様とする。
- read pathからinode writeを発生させない。

## 6. `lseek()`

```cpp
FsStatus lseek(open_file_t* file,
               int32_t offset,
               Whence whence,
               int32_t* out_new_offset);
```

- `SEEK_SET`、`SEEK_CUR`、`SEEK_END`を扱う。
- `SEEK_END`ではinode.sizeを取得する。
- signedな中間値で加算し、overflowを検査する。
- 結果が負なら`EINVAL`とする。
- pipe等のseek不能対象は`ESPIPE`とする。
- 検査成功後だけ`file->offset`を更新する。
- EOFより先へのseekは許可する。

## 7. 完了条件

- size 0/1/511/512/513/1023/1024を読める。
- EOF、EOF+1からのreadが`{0, OK}`となる。
- block境界を跨ぐreadが正しい。
- partial read + I/O errorのcountとoffsetが正しい。
- `SEEK_SET/CUR/END`の正常・overflow・負結果を検証できる。
- directory raw readをwhole dirent単位で検証できる。
- readでinode時刻書込みが発生しない。
