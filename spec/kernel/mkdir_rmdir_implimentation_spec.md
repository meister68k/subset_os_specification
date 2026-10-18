# `mkdir()` / `rmdir()` 実装仕様（phase 1）

## 1. 目的と範囲

phase 1で`mkdir(2)`と`rmdir(2)`を実装する。syscall層は引数を検証・変換し、Processのcwdを取得してpathnameを解決し、Filesystemの公開APIへ委譲する。ディスク形式、directory操作、link count、orphan寿命の詳細はFilesystem層が担う。

同一Filesystemへの並行呼び出しは禁止し、操作を直列化する。crash consistency、inode/cluster再利用、再帰削除、rename、権限検査、umask、VFSは扱わない。失敗rollbackでは最初の失敗statusを維持し、安全なリークまたは過剰link countを許容する。

## 2. syscall動作

### 2.1 `mkdir(path, mode)`

`mkdir(path, mode)`はユーザーpathを`PATH_MAX`内で安全に取り込み、`resolve_parent(current.cwd_inode(), path, ...)`で親inodeと最終名を得て、`Filesystem::mkdir(parent, name, mode, ...)`を呼ぶ。`mode`は許可9ビットを使い、umaskは適用しない。相対pathはcwd、絶対pathはrootから解決する。ポインタとpathの検証は状態変更前に完了する。

### 2.2 `rmdir(path)`

`rmdir(path)`はまず`resolve_parent()`を呼ぶ。最終名が`.`または`..`なら`EINVAL`を返す。`resolve_parent()`が`/`等の最終名なしで`EISDIR`を返した場合は`resolve_path()`で対象を確認し、root inodeなら`EBUSY`とする。それ以外は`Filesystem::rmdir(parent, name)`を呼ぶ。次のエラーを返す。

| 条件 | errno / `FsStatus` |
|---|---|
| root inodeを削除対象とする | `EBUSY` |
| 最終componentが`.`または`..` | `EINVAL` |
| 対象名が存在しない | `ENOENT` |
| 対象がdirectoryでない | `ENOTDIR` |
| directoryに`.`/`..`以外の有効entryがある | `ENOTEMPTY` |
| dirent、inode、chain等が破損 | `ECORRUPT` |

空path、末尾slash、path/name上限、途中componentがdirectoryでない場合はpathname層の既定statusを返す。失敗時に成功値を返さない。

## 3. Filesystem API

```cpp
FsStatus mkdir(uint32_t parent_inode_num, const char* name, uint16_t mode,
               uint32_t* out_inode_num);
FsStatus rmdir(uint32_t parent_inode_num, const char* name);
```

`mkdir()`の入力、inode初期化、公開順序、rollbackは`fs_impl_spec_11-mkdir.md`に従う。`rmdir()`は親inodeと単一componentを受け取る低レベルAPIであり、pathname解決は行わない。名前と親の検証規則はmkdirと共通とする。

## 4. `rmdir()`の検査・更新順序

状態変更前にmount状態、引数、名前、親directory、対象dirent、対象inodeを検証する。rootは`EBUSY`、`.`/`..`は`EINVAL`、非directoryは`ENOTDIR`とする。directory層の空判定は`.`と`..`が各々1個存在し、正しいinodeを指すことを確認し、他の有効direntがあれば`ENOTEMPTY`を返す。不正構造は`ECORRUPT`とする。

検証後の順序:

1. 親から対象direntを削除する。削除が失敗したらlink countを変更しない。
2. 対象directoryの`.`と親からの名前に対応する2 linkを`drop_link(child)`で落とす。
3. 子の2回の減算が成功した後、親の子directory参照1つを`drop_link(parent)`で落とす。

子の2回目の減算または親の減算が失敗した場合、名前は復元しない。最初の失敗を返し、残ったinode/link countのリークを許容する。順序を逆転させて親を先に減らしてはならない。

## 5. orphanとdirectory参照寿命

Filesystemは既存のopen-file参照に加え、cwd参照を固定長テーブルで追跡する。テーブルの1 entryは1 Processのcwd所有参照を表し、容量は`MAX_PROCESSES + 1`とする。Process数上限に対して`chdir()`のretain-new-before-release-oldを可能にする。テーブルの追加・削除はFilesystemの公開内部APIを通し、上位層は表現へ直接アクセスしない。

- Processはcwd inodeを所有する。initはrootをretainし、`spawn()`は親cwdをretainして継承する。
- `chdir()`は新cwdをretainしてからProcessのcwdを交換し、旧cwdをreleaseする。新cwd retainが失敗した場合は旧cwdを維持する。
- `exit()`は他の終了資源処理に従ってcwd参照をreleaseする。
- `rmdir()`されたdirectoryにopen-fileまたはcwd参照があれば、unlink後も参照元から利用できる。cwdである間、そのcwdを基点とする相対解決とdirectoryへの作成・削除を許可する。
- `finalize_if_orphaned()`はopen-file参照とcwd参照の両方がなく、directoryの場合は空である場合に限りfinalizeする。参照がなくても内容が残るdirectoryはinodeを維持し、孤立treeとして保持する。
- 参照が消えた孤立directoryに子directoryが残っている場合、再帰削除せずdirectory inodeをfinalizeしない。phase 1ではその孤立treeを到達不能なまま保持し、inode/clusterを再利用しない。
- root cwd参照は初期Process終了まで保持する。Filesystem unmount前には全process cwd/open handleが解放済みであること。

## 6. `mkdir()`の既定契約

新directoryは1 cluster、`size = FS_BLOCK_SIZE`、`nlink = 2`、`.`は自身、`..`は親を指す。親`nlink`は1増加する。inode、cluster、direntを初期化後に親direntを公開する。詳細とrollbackは`spec/filesystem/fs_impl_spec/fs_impl_spec_11-mkdir.md`を正本とする。

## 7. `FsStatus`とerrno変換

`FsStatus::ENOTEMPTY`を追加する。Filesystem内部ではglobal errnoを使わない。syscall境界で`ENOTEMPTY`、`ENOTDIR`、`EINVAL`、`EBUSY`等を対応するOS errnoへ変換する。`FsStatus`の全一覧とmappingはfilesystem error/external API仕様を更新する。

## 8. 必須検証

- root直下、相対path、絶対path、`.`, `..`を含むmkdir/rmdir。
- 空directoryの削除、`.`/`..`を含む非空directory、通常file、root、missing name、`.`/`..`の各errno。
- mkdirのmode、`.`/`..`、親子link count、重複名、資源枯渇。
- remove entry、childの各drop_link、parent drop_link境界へのI/O故障注入。失敗時は最初のstatusを保持し、公開名が復活しないこと。
- open中およびcwd中のdirectory削除後の参照利用、spawn cwd継承、chdir保持交換、exit release、最後の参照後のfinalize判定。
- cwd参照固定表の容量上限、容量超過、retain/releaseの二重操作と不正inodeを検査する。
- 孤立directoryに子directoryが残るケースで再帰削除せず、inode/clusterを再利用しない。

## 9. 関連仕様

本書はsyscall統合とcwd参照寿命の正本とする。Filesystem層の詳細は`filesystem_external-api_spec.md`、`filesystem_errors_spec.md`、`fs_impl_spec_06-directory.md`、`fs_impl_spec_09-unlink-close.md`、`fs_impl_spec_10-pathname-link.md`、`fs_impl_spec_11-mkdir.md`を参照する。Process cwdの所有・spawn/chdir/exit連携は`process_management_spec.md`にも反映する。
