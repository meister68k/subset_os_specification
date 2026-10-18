# 05 — inodeテーブル

> inodeの共通永続形式、phase 1 lifetime、checked arithmeticは`AGENTS.md`と`00-overview.md`を参照する。

## 1. 責務

- inode番号をinode-table chain内のレコード位置へ変換する。
- inodeをdecode/encodeして取得・更新する。
- inodeを単調割当し、必要時にinode-table chainを延長する。
- `nlink`更新と、上位層の判定後にorphanをfinalizeする経路を提供する。
- open-file表は参照せず、永続inodeの操作に限定する。

## 2. 位置計算

```text
byte_offset = inode_num * inode_size
chain_index = byte_offset / 512
offset_in_cluster = byte_offset % 512
```

- inode番号0は通常の割当対象にしない。
- 乗算を事前検査する。
- `chain_index`のclusterは`Fat::walk_to_index()`で取得する。
- `offset_in_cluster + inode_size <= 512`を保証する。
- `inode_size`が512を割り切ることはmountで検証済みである。

## 3. get

```cpp
FsStatus get(uint32_t inode_num, disk_inode_t* out);
```

1. 引数とinode番号範囲を確認する。
2. inode-table内位置をchecked計算する。
3. 対象clusterをFAT経由で得る。
4. 物理blockを読む。
5. 対象レコードを`inode_decode()`する。
6. decode/意味検証に成功後、出力を確定する。

## 4. put

```cpp
FsStatus put(uint32_t inode_num, const disk_inode_t& in);
```

1. inode値と位置を検証する。
2. 対象blockを読む。
3. 一時buffer内のレコードだけを`inode_encode()`で更新する。
4. block全体を書き戻す。

- inode更新はread-modify-writeとする。
- 他レコードを変更してはならない。
- phase 1ではinode専用cacheを持たない。

## 5. alloc_inode

- `next_free_inode < inode_table_capacity`ならその番号を返してhintを進める。
- 容量不足ならinode-table末尾へclusterを1個extendする。
- 新clusterを全ゼロで書く。
- zero-fill成功後だけ末尾と容量を更新する。
- 新容量計算をoverflow検査する。
- allocation失敗は`ENOSPC`とする。
- 返却後、呼び出し側が完全初期化したinodeを`put()`する。
- 途中失敗で割当済みと誤認するhint更新を行わない。

## 6. link count

### inc_nlink

```cpp
FsStatus inc_nlink(uint32_t inode_num);
```

- inodeをgetする。
- live inodeであることを確認する。
- `UINT16_MAX`なら`EMLINK`を返す。
- 1増加し、`ctime=now()`としてputする。

### dec_nlink

```cpp
FsStatus dec_nlink(uint32_t inode_num);
```

- inodeをgetする。
- live inodeであることを確認する。
- `nlink == 0`なら減算せず`ECORRUPT`を返す。
- 1減少し、`ctime=now()`とする。
- その結果をputする。
- `nlink`が0に到達しても、`mode`、`size`、`first_cluster`を変更しない。
- open-file表を参照せず、finalizeも行わない。

### finalize_inode

```cpp
FsStatus finalize_inode(uint32_t inode_num);
```

- inodeをgetする。
- live inodeかつ`nlink == 0`であることを確認する。条件不成立なら`ECORRUPT`を返し、inodeを変更しない。
- `first_cluster != CLUSTER_NONE`なら`Fat::free_chain()`を呼ぶ。
- `size = 0`、`first_cluster = CLUSTER_NONE`、`mode = S_IUNUSED`、`ctime = now()`としてputする。
- `S_IFREE`へ戻してはならない。
- 対象inodeがopenされていないことは呼び出し側の事前条件とする。`InodeTable`自身はopen状態を検査しない。

`nlink == 0`のlive inodeは、open中にunlinkされた場合およびphase 1でクラッシュ後に残留した場合に現れ得る正常な永続状態である。`get()`やinodeの意味検証は、この状態だけを理由に`ECORRUPT`としてはならない。open-file表を所有する上位層が、`09-unlink-close.md`の`finalize_if_orphaned()`で`nlink == 0`かつ非openを確認した後に`finalize_inode()`を呼ぶ。

## 7. 完了条件

- inode番号がcluster境界の前後にあるケースをget/putできる。
- 隣接inodeを壊さずread-modify-writeできる。
- inode-tableを複数clusterへ延長できる。
- 新clusterが全ゼロである。
- allocation途中のI/O失敗でstateが先行しない。
- `nlink`上限・下限を安全に扱える。
- `dec_nlink()`で0に到達してもinode内容を維持できる。
- `finalize_inode()`が`nlink == 0`のlive inodeだけを`S_IUNUSED`へ遷移させる。
