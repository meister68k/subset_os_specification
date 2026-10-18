# `mkfs`仕様

> 永続形式、LE encode、checked arithmetic、phase 1定数は`AGENTS.md`と`00-overview.md`を参照する。

## 1. 入力

- `total_blocks`: デバイス全体のブロック数。
- `fs_start_block`: 予約領域直後のFS開始位置。
- Backendまたは書込み先image。
- フォーマット識別magic等の固定値。

- `fs_start_block`より前は変更しない。
- MBR等の作成は本仕様の責務外とする。

## 2. 容量計算

```text
available_blocks = total_blocks - fs_start_block - 1
fat_blocks       = ceil(available_blocks / 257)
data_start_block = fs_start_block + 1 + fat_blocks
data_blocks      = available_blocks - fat_blocks
```

512バイトのFAT 1ブロックは256個のFAT entryを持つ。FAT自身も利用可能領域から
1ブロックを消費するため、最小の十分なFAT長は上式で直接求める。

```text
data_blocks <= fat_blocks * 256
```

- 全減算の前にunderflowを検査する。
- `data_blocks * 2`の前にoverflowを検査する。
- ceil計算で`+511`を無検査で行わない。
- 旧来の不動点反復は、容量によって2値間を振動するため使用しない。
- `data_blocks >= 2`を要求する。
- `data_blocks <= 65535`を要求する。
- 最小構成はsuperblock 1 + FAT 1 + data 2である。
- よって`total_blocks >= fs_start_block + 4`を要求する。

## 3. bootstrap順序

1. 入力と容量計算結果を全て検証する。
2. FAT領域を全ゼロ化する。
3. cluster 0をinode table用に確保し、`FAT[0]=FAT_EOF`とする。
4. superblockの不変フィールドを構築する。
5. `inode_table_start_cluster=0`、`root_inode=1`とする。
6. inode table clusterを全ゼロ化する。
7. inode 1をroot directoryとして初期化する。
8. root directory用clusterを1個確保する。
9. root data clusterを全ゼロ化する。
10. `.`と`..`を作り、両方inode 1を指すようencodeする。
11. root inodeの`first_cluster`、`size`、時刻を設定する。
12. inode 1をencodeしてinode tableへ書く。
13. 必要な同期を行う。
14. superblockは手順3以降の任意の時点で`fs_start_block`へ書いてよい。

- デバッグ用Rubyツールは作成途中の異常終了に対する整合性を保証しない。
- 途中生成されたimageは破棄して再作成する。
- allocator hint等の動的帳簿をsuperblockへ書かない。
- 全persistent fieldは専用encode関数を使う。

## 4. root inode

- inode番号は1。
- modeは`S_IFDIR`と適切なpermission bits。
- `nlink`は設計で定めたroot初期値を用いる。
- `uid=gid=0`。
- sizeは確保済みdirectory容量である512。
- `first_cluster`はroot data cluster。
- `.`と`..`は両方inode 1。

## 5. 完了条件

- `fs_start_block=0`の最小imageを作成しmountできる。
- 予約領域ありのimageを作成し、予約領域が不変である。
- `fs_start_block + 4`未満を拒否する。
- FAT block数が変わる境界容量を検証する。
- 65535 cluster上限付近を検証する。
- 全算術overflow/underflow入力をI/O前に拒否する。
- 作成imageの既知byte dumpがLE仕様と一致する。
- mount後にroot、`.`、`..`をlookupできる。

## 6. デバッグ用Rubyツールの制約

- `mkfs.rb`と`mkfs_chk.rb`は`fs_start_block=1`固定とする。
- `mkfs.rb`は既存image全体を破棄して再作成し、予約block 0も保存しない。
- 生成直後の予約block 0はゼロとする。
- `mkfs_chk.rb`は`mkfs.rb`直後の未使用imageだけを検証する。
- 一般的なfsckではなく、使用開始後のimageは検査対象外とする。
- 大きく破損したimageに対する安全な解析や整形されたエラー終了は保証しない。
