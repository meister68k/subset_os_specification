# 11 — `mkdir()`

> 名前、directory表現、`nlink`上限、metadata更新順序、phase 1非再利用の共通規則は`AGENTS.md`を上位仕様とする。pathname解決は`10-pathname-link.md`、directory内容の操作は`06-directory.md`に従う。

## 1. 範囲

- phase 1でsubdirectoryを新規作成する。
- `mkdir()`は親directory inodeと単一の名前componentを受け取る低レベルAPIとする。
- フルパスからの作成は呼び出し側が`resolve_parent()`後に`mkdir()`を呼ぶ。
- `rmdir()`はphase 1対象であり、詳細は`spec/kernel/mkdir_rmdir_implimentation_spec.md`に従う。
- `rename()`、空間回収、fsckは本仕様の対象外とする。`unlink()`はdirectoryを引き続き`EISDIR`で拒否する。

## 2. API

```cpp
FsStatus mkdir(uint32_t dir_inode_num,
               const char* name,
               uint16_t mode,
               uint32_t* out_inode_num);
```

- `dir_inode_num`は新規directoryを追加する親inode番号とする。
- `name`はpathname解決後の1〜27バイトの単一componentとする。
- `mode`から許可9ビットだけを保持し、umaskは適用しない。
- `out_inode_num`は成功時に限り新規inode番号で確定する。
- 戻り値は`FsStatus`とし、成功は`FsStatus::OK`とする。

## 3. 事前検査とstatus

状態を変更する前に次を検査する。

- 未mountは`ENODEV`とする。
- `name == nullptr`または`out_inode_num == nullptr`は`EINVAL`とする。
- 空名または`/`を含む名前は`EINVAL`とする。
- 27バイトを超える名前は`ENAMETOOLONG`とする。
- 親inodeがlive directoryでなければ、通常ファイルは`ENOTDIR`、未使用・削除済みinodeは`ECORRUPT`とする。
- `dir_lookup()`で同名が見つかれば、対象種別にかかわらず`EEXIST`とする。
- `.`と`..`は通常direntとして存在するため、正常な親では同じく`EEXIST`とする。
- dangling direntやdirectory構造の不整合は`ECORRUPT`とする。
- inode、cluster、directory領域の枯渇は`ENOSPC`とする。
- 親の`nlink == UINT16_MAX`で増加できない場合は`EMLINK`とする。
- block I/O失敗は`EIO`またはBackend由来のstatusとする。
- `dir_lookup()`の`ENOENT`だけを新規作成の開始条件とし、その他の失敗はそのまま返す。

## 4. 成功時の永続状態

新規directory inodeは次の値を持つ。

| フィールド | 値 |
|---|---|
| `mode` | `S_IFDIR | (mode & 0777)` |
| `nlink` | 2(親からの名前と自身の`.`) |
| `uid` / `gid` | 0 |
| `size` | `FS_BLOCK_SIZE` |
| `first_cluster` | 新規割当済みdirectory data cluster |
| `atime` / `ctime` / `mtime` | 作成時の`now()` |

- directory data clusterは全バイトを0で初期化してから露出する。
- 先頭direntの`.`は新規inode自身を指す。
- 2番目direntの`..`は`dir_inode_num`を指す。
- 残りのdirent slotは全ゼロとする。
- 親inodeの`nlink`は子の`..`参照を反映して1増加する。
- 新規directoryの内容を書くためにユーザ向け`write()`を使わない。directory層のhelperまたは同等の内部経路で`dirent_encode()`する。

## 5. 更新順序

親directoryに名前を公開する前に、新規directoryを完全に初期化する。次の順序を変更してはならない。

1. 引数、名前、親inodeを検証する。
2. `dir_lookup()`で同名が存在しないことを確認する。
3. 親の`nlink`を読み、増加可能であることを確認する。
4. `alloc_inode()`で新規inode番号を単調割当する。
5. 新規inodeの全フィールドを初期化し、`nlink = 2`、`size = 0`、`first_cluster = CLUSTER_NONE`の未公開directoryとして永続化する。
6. directory data clusterを1個割り当て、全ゼロで初期化する。
7. `.`と`..`を`dirent_encode()`で書き、`size = FS_BLOCK_SIZE`と`first_cluster`を含む完成状態を永続化する。
8. `inc_nlink(dir_inode_num)`で親のlink countを先に増やす。
9. `dir_add_entry(dir_inode_num, name, new_inode_num)`で作成済みdirectoryを公開する。
10. すべて成功した場合だけ`*out_inode_num`を確定する。

- 新規directoryの初期化途中に親の名前から到達可能にしてはならない。
- 親の`nlink`増加はdirent追加より先に行う。追加後の`nlink`更新失敗で過少countの公開directoryを残さないためである。
- phase 1はcrash consistencyを保証しないが、各API失敗は以下のrollbackを実施する。

## 6. 失敗時のrollback

- inodeの最初の永続化より前の失敗ではdirentを作らず、`drop_link()`を呼ばない。
- `nlink = 2`のinode永続化後に初期化が失敗した場合は、`drop_link(new_inode_num)`を2回使い、最後に`S_IUNUSED`へfinalizeする。
- 1回目の`drop_link()`が失敗したら、2回目は行わない。
- `inc_nlink(parent)`が失敗した場合も、新規directoryを2回の`drop_link()`でfinalizeする。
- `inc_nlink(parent)`後の`dir_add_entry()`失敗では、先に新規directoryを2回の`drop_link()`でfinalizeし、その成功後だけ`dec_nlink(parent)`で親の増分を戻す。
- 新規directoryのrollbackが失敗した場合、新規inodeと親のlink count増分のリークを許容する。到達可能な未初期化directoryや過少link countより安全な側に停止するためである。
- `dec_nlink(parent)`のrollback失敗では過剰countをリークし、他のmetadata変更は行わない。
- rollback中に別のstatusが発生しても、`mkdir()`は操作を停止させた最初のstatusを返す。
- rollbackでinodeを`S_IFREE`へ戻してはならない。clusterとinode slotはphase 1で再利用しない。
- この順序を共有helperに集約し、呼び出し側ごとに部分rollbackを複製しない。

## 7. pathnameと公開APIの使い方

```cpp
uint32_t parent_inode = 0;
char name[fs::NAME_MAX + 1];
fs::FsStatus status = fs.resolve_parent(cwd_inode, path, &parent_inode, name);
if (status == fs::FsStatus::OK) {
    uint32_t new_inode = 0;
    status = fs.mkdir(parent_inode, name, mode, &new_inode);
}
```

- absolute pathとrelative pathの開始位置は`resolve_parent()`が処理する。
- 空path、末尾slash、PATH/NAME上限はpathname層のstatusをそのまま返す。
- Processが所有するcwd inodeの参照数はFilesystemの固定長cwd参照表で追跡する。詳細は`spec/kernel/mkdir_rmdir_implimentation_spec.md`を参照。

## 8. 必須テスト

### 8.1 基本動作

- root直下にdirectoryを作成し、`resolve_path()`で到達できるか。
- modeの種別と許可ビット、`nlink == 2`、`size == 512`、時刻を検証できるか。
- `.`が自身、`..`が親を指し、親の`nlink`が1増えるか。
- 未使用dirent slotが全ゼロか。
- 作成したdirectory下で`creat()`とさらなる`mkdir()`が成功するか。
- absolute/relative path、`.`、`..`を含む解決が正しいか。

### 8.2 入力と上限

- null引数、空名、`/`を含む名前、27/28バイト名を検証する。
- 親が通常ファイル、未使用inode、削除済みinode、範囲外inodeの場合を検証する。
- 同名の通常ファイル/directoryと`.`/`..`が`EEXIST`になるか。
- 親の`nlink == UINT16_MAX`で`EMLINK`となり、状態変更しないか。
- inode tableのcluster境界と拡張、FAT/inode枯渇、親の空slot再利用とchain拡張を検証する。

### 8.3 破損と故障注入

- dangling dirent、壊れた親chain、壊れた新規cluster chainを`ECORRUPT`で停止できるか。
- inode永続化、cluster割当、zero-fill、`.`/`..`書込み、親`nlink`増加、親dirent追加の各境界でread/write失敗を注入する。
- 新規directoryと親`nlink`のrollbackそれぞれに失敗を注入する。
- 失敗時に親から到達可能な未初期inodeが残らず、`out_inode_num`が変更されないか。
- rollback成功時に新規inodeが`S_IUNUSED`となり、`S_IFREE`へ戻らないか。
- rollback失敗時に最初の失敗statusを保持し、dangling nameではなくリーク側で停止するか。

## 9. 完了条件

- 新規directoryは公開時点でinode、cluster、`.`、`..`が全て初期化済みである。
- 親と子の`nlink`が成功時の規約を満たす。
- 上記の境界値・破損・故障注入テストが全て通る。
- directory内容の書込みはdirectory層を通し、上位層がFAT、inode-table offset、物理blockを直接操作しない。
- phase 1の非再利用、非並行、crash consistencyなしの制約を維持する。
