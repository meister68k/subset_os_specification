# 03 — FAT

> 共通のLE変換、算術、破損時停止、phase 1非再利用規則は`AGENTS.md`を参照する。

## 1. 責務

- FAT16相当テーブルのentry取得・更新を行う。
- クラスタを単調割当する。
- FATチェーンを安全に走査・延長する。
- 1ブロックFATキャッシュをFSインスタンス単位で管理する。

## 2. クラス形

```cpp
template<class Backend>
class Fat {
public:
    FsStatus get(uint32_t cluster, uint16_t* out);
    FsStatus set(uint32_t cluster, uint16_t value);
    std::pair<uint32_t, FsStatus> alloc();
    FsStatus free_chain(uint32_t head_cluster);
    std::pair<uint32_t, FsStatus> walk_to_index(uint32_t head, uint32_t index);
    std::pair<uint32_t, FsStatus> find_tail(uint32_t head);
    std::pair<uint32_t, FsStatus> extend(uint32_t tail);
    FsStatus sync();
};
```

- シグネチャの細部は既存宣言に合わせてよい。
- クラスタ番号とstatusを同時に返す場合、曖昧な番兵だけで失敗を表現しない。

## 3. FAT entry位置計算

- `byte_offset = cluster * sizeof(uint16_t)`をchecked multiplicationで求める。
- `fat_block_index = byte_offset / 512`とする。
- `offset_in_block = byte_offset % 512`とする。
- 物理ブロックは`fat_start_block + fat_block_index`とする。
- clusterが`data_blocks`以上ならI/Oを行わない。
- FAT領域外になる計算は`ECORRUPT`とする。
- entryは必ずLE encode/decodeする。

## 4. FATキャッシュ

- 1ブロックだけキャッシュしてよい。
- キャッシュ状態はインスタンスメンバとする。
- block番号、valid、dirtyを明示的に管理する。
- 別ブロックへ切り替える前にdirty blockを書き戻す。
- 書き戻しに失敗した場合、失敗を返し新blockへ切り替えない。
- read失敗時にcache validを立てない。
- `sync()`はdirty blockを書き戻す。
- unmount前に`sync()`する。

## 5. alloc

- phase 1では`mount_state_.next_free_cluster_hint`から1個割り当てる。
- hintが`data_blocks`以上なら`ENOSPC`とする。
- 対象entryが`FAT_FREE`であることを確認する。
- 想定外に非FREEなら、mount再構築不変条件違反として安全停止する。
- 確保entryを`FAT_EOF`へ設定する。
- 設定成功後だけhintを進める。
- 失敗時にhintを進めない。

## 6. walk_to_index

- index 0はhead自身を返す。
- 各hopで現在clusterの範囲を確認する。
- 次entryが`FAT_FREE`なら`ECORRUPT`とする。
- 必要位置より前で`FAT_EOF`なら`ECORRUPT`とする。
- hop数が`data_blocks`を超える前に停止する。
- 循環チェーンで無限ループしてはならない。

## 7. find_tail

- headから`FAT_EOF`まで前方走査する。
- 範囲外、FREE、過剰hopを`ECORRUPT`とする。
- `FAT_EOF`を持つclusterを返す。

## 8. extend

1. tailが有効で、現在`FAT_EOF`であることを確認する。
2. `alloc()`でnew clusterを確保する。
3. tail entryをnew clusterへ変更する。
4. new clusterは`alloc()`によって`FAT_EOF`になっている。
5. 途中失敗では既存chainを壊さないことを優先する。

## 9. free_chain

- phase 1では再利用を行わない。
- 呼び出し形は将来差替え可能なまま維持する。
- phase 1でFAT entryを`FAT_FREE`へ戻さない。
- 上位層は将来の実装と同じ呼び出し経路を使用してよい。
- `head_cluster`が`data_blocks`以上なら`ECORRUPT`を返す。
- 有効な`head_cluster`では`FsStatus::OK`を返す。
- どちらの場合もI/Oを発行せず、FATを変更しない。
- phase 1ではチェーン走査を行わない。

## 10. 完了条件

- entryのblock境界前後をget/setできる。
- dirty切替とsyncの失敗をfault injectionで検証できる。
- allocation枯渇を`ENOSPC`で返せる。
- 有効chainの任意indexとtailを取得できる。
- 範囲外link、FREE混入、premature EOF、cycleを`ECORRUPT`で停止できる。
- 複数FSインスタンスのcacheとhintが混線しない。
