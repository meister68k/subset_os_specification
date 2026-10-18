# 04 — mount・superblock・`mount_state_t`

> 共通のchecked arithmeticと`ECORRUPT`安全停止は`AGENTS.md`を参照する。

## 1. mount順序

1. Backendをopenする。
2. 呼び出し側指定の`fs_start_block`からsuperblockを1ブロック読む。
3. `superblock_decode()`する。
4. superblock全項目を検証する。
5. 検証成功後だけFATとinode tableを読む。
6. `mount_state_t`を再構築する。
7. 全処理成功後だけmounted状態へ遷移する。

- 失敗時は通常FS APIを利用可能にしない。
- 適切にBackendをcloseし、半端なcache/stateを破棄する。

## 2. superblock必須検証

- `magic`が本FSの定数と一致する。
- `block_size == FS_BLOCK_SIZE == 512`である。
- 格納された`fs_start_block`がmount引数と一致する。
- `total_blocks`がデバイス容量以下である。
- `fat_start_block`、`fat_blocks`、`data_start_block`、`data_blocks`の全計算がoverflowしない。
- superblock、FAT、data領域が互いに重複しない。
- 全領域がFS範囲内に収まる。
- FATが`data_blocks`個の2バイトentryを収容できる。
- `data_blocks <= 65535`である。
- `inode_size != 0`である。
- `512 % inode_size == 0`である。
- `inode_size == INODE_SIZE(64)`である(phase 1固定。decode契約が64バイト読込みのため、より小さい値は再構築時のバッファ外読込みを招く)。
- `inode_table_start_cluster < data_blocks`である。
- phase 1では`inode_table_start_cluster == 0`である。
- `root_inode == 1`である。
- 失敗は`ECORRUPT`とし、未検証値をアドレス計算へ使用しない。

## 3. RAM専用状態

```cpp
struct mount_state_t {
    uint32_t next_free_cluster_hint;
    uint32_t next_free_inode;
    uint32_t inode_table_last_cluster;
    uint32_t inode_table_capacity;
};
```

- この構造はディスクへencodeしない。
- superblockへ同等フィールドを追加しない。
- FATとinode tableが唯一の真実の情報源である。

## 4. cluster hint再構築

phase 1では使用済みclusterが0から連続するため、FAT上のFREE境界を求める。
境界は単一の前方走査で確定し、走査がprefix不変条件の検証を兼ねる。

```text
first_free = data_blocks   # FREE未検出時のデフォルト
for i in [0, data_blocks):
    v = FAT[i]
    if first_free == data_blocks:
        if v == FAT_FREE: first_free = i   # 境界確定
    else:
        if v != FAT_FREE: return ECORRUPT  # 境界後のused混入
next_free_cluster_hint = first_free
```

- `[0, first_free)` がFREEを含まないことは「最初のFREE」の定義から自明で、個別の検証は不要。
- 走査はFAT全ブロックを一度ずつ読む。1ブロックFATキャッシュにより各物理ブロックは最大1回のreadで収まるため、境界推定のためだけの二分探索は行わない。
- FAT read失敗を伝播する。
- 境界後(index >= first_free)にusedが現れる等、prefix不変条件に反する状態は`ECORRUPT`とする。
- 全使用済みならhintは`data_blocks`となり、次のallocが`ENOSPC`となる。

## 5. inode table再構築

- `inode_table_start_cluster`から末尾まで1回だけ走査する。
- 走査cluster数を数える。
- 最終clusterを`inode_table_last_cluster`とする。
- `inode_table_capacity = cluster_count * (512 / inode_size)`とする。
- 全レコードをdecodeし、`mode != S_IFREE`の最大indexを探す。
- `next_free_inode = highest_ever_used_index + 1`とする。
- `S_IUNUSED`も「一度使われた」として最大index候補に含める。
- live判定は`mode != S_IFREE && mode != S_IUNUSED`とする。
- chain範囲外、cycle、FREE混入は`ECORRUPT`とする。
- 容量乗算とindex加算をchecked arithmeticで行う。

## 6. unmount

- mounted状態を確認する。
- FAT cache等のdirty状態をsyncする。
- sync失敗時はBackendをcloseせず、mounted状態とdeviceのopen状態を維持してstatusを返す。この場合はunmountを再試行できる。
- sync成功後に`BlockDevice::close()`を1回呼ぶ。
- `BlockDevice::close()`の成否にかかわらず、呼び出し後はmounted状態を解除し、mount時に構築したruntime stateを無効化する。
- `BlockDevice::close()`が失敗した場合はそのstatusを返すが、unmountまたはBackend closeを再試行しない。
- superblockは書き戻さない。

## 7. 完了条件

- 各必須検証を1項目ずつ壊したimageでmount拒否できる。
- 未検証値を使ったout-of-bounds I/Oが起きない。
- empty余地、全cluster使用、削除済みinodeを含む状態を再構築できる。
- FAT cycleとinode-table chain破損を停止できる。
- mount失敗後に通常APIが動作しない。
- mount/unmountを繰り返して状態が残留しない。
