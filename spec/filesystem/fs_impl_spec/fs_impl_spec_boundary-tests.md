# 境界条件・共通完了テスト

> `AGENTS.md`のレビュー・テスト規約を上位仕様とする。本書は各層の受入テスト索引である。

## 1. 算術

- `offset + len`の正常上限、ちょうどoverflow、overflow超過。
- `inode_num * inode_size`の正常上限とoverflow。
- ceil divisionの加算overflow。
- cluster→block変換のoverflow。
- region start + lengthのoverflow。
- 負になるseek結果。
- signed seek加算の正負overflow。
- 拒否時にstateとI/O回数が変化しないこと。

## 2. block device

- block 0。
- 最終有効block。
- `blkno == total_blocks`。
- `blkno > total_blocks`。
- 未open read/write。
- null buffer。
- 未openかつnull buffer、null bufferかつ範囲外blockなど複数違反時の検査優先順位。
- 二重openは`EBUSY`となり、Backend open回数と保持状態が変化しない。
- Backend open失敗後は閉状態かつ容量0で、Backend自身も閉状態に戻る。
- 未open closeは`ENODEV`となり、Backend close回数が変化しない。
- close成功時と失敗時のどちらもBackend closeは1回だけで、閉状態かつ容量0へ遷移する。
- close失敗後の再closeは`ENODEV`となり、Backend closeを再試行しない。
- open/read/write/close fault injection。

## 3. serialization

- 既知LE byte vector。
- 各recordのround-trip。
- 非アラインbuffer位置。
- reserved領域0。
- direntの0/1/27 byte名（0 byteはシリアライズ上の構造テスト）。
- 28 byte名の拒否。
- NULなしdirentの拒否。

## 4. mount/superblock

- magic不一致。
- block size不一致。
- mount引数とstored fs_start不一致。
- deviceを越えるtotal blocks。
- 領域重複。
- FAT容量不足。
- `data_blocks`上限超過。
- inode size 0。
- inode sizeが512を割り切らない。
- inode-table cluster不正。
- root inode不正。
- superblock検証失敗後に追加I/Oしないこと。
- used-prefix FATの境界が先頭・中間・末尾。
- inode table 1clusterと複数cluster。
- `S_IUNUSED`を含むnext inode再構築。
- unmountのsync失敗時はmounted/open状態を維持し、closeを呼ばず再試行できる。
- unmountのclose失敗時はmounted状態を解除し、closeを再試行しない。

## 5. FAT

- FAT entryのblock境界前後。
- cluster 0。
- 最大有効cluster。
- 範囲外cluster。
- 1cluster chain。
- 複数cluster chain。
- chain内`FAT_FREE`。
- premature `FAT_EOF`。
- self cycle。
- multi-cluster cycle。
- hop数が`data_blocks`超過。
- 全cluster割当済みの`ENOSPC`。
- cache dirty writeback failure。

## 6. inode

- inode 0の番兵扱い。
- root inode 1。
- inode recordのcluster境界前後。
- inode-table extension。
- zero-fill failure。
- 最大inode付近の位置計算。
- `nlink=0`のdecrement拒否。
- `nlink=UINT16_MAX`のincrement拒否と`EMLINK`。
- `S_IFREE`、live、`S_IUNUSED`の判定。

## 7. directory/path

- 空directory内lookup miss。
- 先頭・中間・末尾slot hit。
- 空きslot再利用。
- directory cluster追加。
- dirent境界。
- malformed inode参照。
- 使用中direntの空名と`/`を含む名前を`ECORRUPT`として拒否。
- 空path。
- `/`。
- relative path。
- repeated slash。
- `.`、`..`、root `..`。
- trailing slash in parent resolution。
- 27 byte component。
- 28 byte component。
- 非ASCII byteを含むcomponentを文字コード検査なしで処理。
- path limit直前と上限到達。
- intermediate non-directory。

## 8. read/write/seek

- file size 0/1/511/512/513/1023/1024。
- `write_at(0, 1)`。
- `write_at(511, 2)`。
- `write_at(512, 1)`。
- `write_at(512, 512)`。
- EOF read。
- EOF seek。
- EOF+1 seek。
- EOF beyond writeとzero gap。
- negative seek result。
- partial read/write + I/O failure。
- FAT枯渇時のpartial write。
- required chainのpremature EOF。
- directory raw read。
- directory user write拒否。
- error時のshared offset更新量。

## 9. creat/mkdir/link/unlink/close

- 新規creat。
- 既存regularのtruncate。
- 既存directoryの`EISDIR`。
- inode put失敗時rollback。
- dir add失敗時rollback。
- mkdir成功時の`.`/`..`、子`nlink == 2`、親`nlink + 1`、全ゼロの未使用dirent slot。
- mkdirの同名file/directoryの`EEXIST`、親`nlink == UINT16_MAX`の`EMLINK`。
- mkdirのinode、cluster、zero-fill、`.`/`..`、親`nlink`、親direntの各更新境界への故障注入。
- mkdir rollback成功時の`S_IUNUSED`と、rollback失敗時のdangling name防止・最初のstatus保持。
- rmdir: 空directory成功、missing name、非directory、root、`.`/`..`、非空directoryの各status。
- rmdir: 親entry削除と子2 link/親1 linkのdrop境界への故障注入。失敗時の名前非復元と最初のstatus保持。
- cwd/open参照中のrmdir後も利用できること、最後の参照解放後のorphan finalize判定、孤立treeの非再帰保持。
- hard link生成。
- link destinationの`EEXIST`。
- directory hard link拒否。
- link add失敗時nlink rollback。
- unlink lookup miss。
- directory unlink拒否。
- dir remove失敗時nlink不変。
- unlink後のnlink減少。
- open中unlink。
- open中unlink後のread/write。
- dup/fork共有offset。
- 非最終close。
- 最終closeでorphan finalize。
- `S_IUNUSED`へ遷移し`S_IFREE`へ戻らないこと。

## 10. `mkfs`

- 最小4-block FS（`fs_start_block`基準）。
- 最小未満。
- 予約block 0個と複数個。
- FAT block数の切替境界。
- 旧不動点式が振動していた境界（`fs_start_block=1`, `total_blocks=260`）。
- `fat_blocks=ceil((total_blocks-fs_start_block-1)/257)`の前後。
- data cluster 65535と超過。
- root inode/direntのknown-byte確認。
- 作成直後のmountとroot lookup。
- root directory inodeのsizeが512であること。

## 11. fault injection原則

- fake Backendで各I/O境界を1回ずつ失敗させる。
- statusだけでなく、書込み回数、対象block、残存stateを検証する。
- partial transferはbyte countとstatusを同時にassertする。
- ordering規則は呼出しlogで検証する。
- 破損入力では有限回のI/Oで停止することを検証する。
