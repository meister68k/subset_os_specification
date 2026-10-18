# `unlink()` システムコール実装仕様

## 1. 目的と範囲

本書はpathnameを受け取る`unlink()` syscallの実装契約を定める。カーネルはProcessのcwdを基点にpathnameを親directory inodeと最終nameへ分解し、Filesystem公開APIへ委譲する。link count、open-but-unlinked lifetime、disk更新順序の正本はFilesystem層仕様とする。

phase 1ではregular file等の対応済み非directory inodeの名前を1つ削除する。directory削除は`rmdir()`の責務であり、`unlink()`では拒否する。symlink、VFS、cross-filesystem操作、クラスタ/inode再利用、permission判定は対象外とする。

## 2. syscall ABI

```cpp
int32_t sys_unlink(const char* path);
```

- 成功時は`0`、失敗時は`-1`を返す。
- 現行ABIは失敗時の`-1`のみを規定する。グローバルerrnoや本syscall専用のerrno機構は追加しない。
- Filesystem内では`FsStatus`を維持し、syscall境界で既定の失敗値へ変換する。
- syscall番号は`syscall_entry_spec.md`等の番号決定を待ち、本書では割り当てない。

## 3. 処理経路と責務

```text
btx::sys_unlink(path)
  -> syscall dispatcher
  -> kernel::sys_unlink(path)               // ABI引数・結果の薄い変換
  -> kernel::kernel_unlink(Process&, path)  // cwd・pathname解決
  -> Filesystem::resolve_parent(cwd, path)
  -> Filesystem::unlink(parent_inode, name)
       -> dir_lookup / inode確認
       -> dir_remove_entry
       -> drop_link
       -> 必要ならfinalize_if_orphaned
```

- syscall handlerはFD table、inode、dirent、FATを直接操作しない。
- kernel内部APIはsyscallを通さず呼べる形とし、Processとpathを受け取り`FsStatus`を返す。
- Filesystemには親inodeと単一componentを渡す。Filesystem API内でpathname/cwd解決を重ねない。
- 同一Filesystemへの並行操作は禁止し、既定の直列化規則を守る。

概念上の内部API:

```cpp
FsStatus kernel_unlink(Process& process, const char* path);
```

名前空間・具体的なクラス名は既存実装に合わせてよいが、syscallとFilesystemの責務境界を変えない。

## 4. syscall側の入力検証とpathname解決

1. null pathを拒否する。
2. ユーザーpathを`PATH_MAX`未満のNUL終端文字列として安全に取り込む。上限までNULがなければ`ENAMETOOLONG`相当で失敗する。ユーザー領域を未検証のままFilesystemへ渡さない。
3. 現在Processの`cwd_inode()`を取得し、`resolve_parent(cwd, path, &parent_inode, name)`を呼ぶ。
4. resolve失敗時はその失敗をsyscall失敗へ変換し、Filesystem mutationを行わない。
5. resolve成功後、`Filesystem::unlink(parent_inode, name)`を1回呼ぶ。

Absolute pathはrootから、relative pathはProcess cwdから解決する。`.`と`..`は通常のdirent解決に従う。最終対象がdirectoryならFilesystem側で`EISDIR`となる。`/`のみ、または末尾slash付きpathは既定`resolve_parent()`契約により`EISDIR`となり、mutationしない。途中componentがdirectoryでない場合は`ENOTDIR`相当で停止する。

この段階で`open()`や一時FDを作らない。`stat()`を事前に呼んで対象を重複確認する必要もない。対象inode種別とdirentの整合はFilesystem::unlink()が検証する。

## 5. Filesystem unlink契約

```cpp
FsStatus unlink(uint32_t parent_inode_num, const char* name);
```

順序は変更しない。

1. 親directoryとnameを検証し、`dir_lookup()`で対象inode番号を得る。
2. 対象inodeを取得・検証する。無効dirent、未使用/削除済みinode等の破損は`ECORRUPT`とする。
3. 対象がdirectoryなら`EISDIR`を返す。rootもdirectoryとして拒否する。
4. `dir_remove_entry(parent_inode_num, name, &removed_inode)`で親direntを削除する。
5. removeが成功した場合だけ`drop_link(removed_inode)`を呼ぶ。

`dir_remove_entry()`が失敗した場合、targetのlink countやorphan状態を変更しない。`drop_link()`が失敗した場合はそのstatusを返し、削除済みの名前を再作成しない。link count減算後にfinalizeが失敗した場合も減算を戻さず、`nlink == 0`のlive orphanを許容する。最初に確定したfailure statusを後段cleanup errorで上書きしない。

一つのinodeを指す別名が残っている間は`nlink > 0`を維持し、finalizeしない。最後の名前を外した後でもopen-file handleが参照する間はinodeの`mode`、`size`、`first_cluster`と内容を維持する。最後のopen handleのrelease時にorphan finalizationを試みる。

## 6. `open()` / `close()` / `dup()`との関係

- `unlink()`は既存fdをcloseせず、fd table、FileDescription参照数、file positionを変更しない。
- open中にunlinkされたファイルは、その既存fdのaccess modeで許可されたread/write/seekを継続できる。
- `dup()`や`spawn()`で同一FileDescriptionを共有するfdがある場合も、最終参照が解放されるまでFilesystem handleはopenである。
- 最後の名前を削除し、open handleもなくなった時点でFilesystemがorphanをfinalizeする。phase 1の`free_chain()`は空間を再利用しないため、cluster/inode slotは再利用可能に戻らない。
- `unlink()`の成功はファイル内容の即時消去を意味しない。既存open参照とFilesystemのorphan lifetimeに従う。

## 7. 結果とエラー

| 条件 | Filesystem結果 / syscall結果 |
|---|---|
| 正常に名前を削除 | `FsStatus::OK` / `0` |
| null・空path | `EINVAL`相当 / `-1` |
| path/name上限超過 | `ENAMETOOLONG` / `-1` |
| 最終対象が存在しない | `ENOENT` / `-1` |
| 途中componentがdirectoryでない | `ENOTDIR` / `-1` |
| 最終対象がdirectory(root、`.`/`..`経由を含む) | `EISDIR` / `-1` |
| Filesystem未mount | `ENODEV` / `-1` |
| directory構造、dirent、inodeが不整合 | `ECORRUPT` / `-1` |
| remove前のblock I/O failure | `EIO`またはBackend status / `-1` |
| dirent削除後のdrop/finalize failure | failure status / `-1`。名前は削除済みのまま |

通常のsyscall失敗は`-1`へ変換する。Filesystemでglobal errnoを使わず、現在のABIにerrno設定がないことを理由に新しいglobal stateを追加しない。

## 8. 必須検証

- 相対/絶対path、cwd基点、root基点、`.`/`..`を含むpathで対象を正しく解決する。
- 空path、null、PATH_MAX境界、NAME_MAX境界、末尾slash、`/`、途中non-directoryを検証する。
- regular fileのunlink後に名前が消え、link countが1減少する。
- hard linkが複数ある場合、一名の削除では他名とinode内容が維持される。
- directoryおよびrootを`EISDIR`で拒否し、状態不変である。
- missing name、dangling dirent、未使用/削除済みinodeを検証する。
- dirent削除失敗時にlink count不変、drop_link/finalize失敗時に名前非復元・最初のstatus保持を確認する。
- open中unlink後も既存FDのaccess modeで許可された操作を継続でき、dupされた参照の最後のcloseでのみfinalize判定する。
- unlink自体がfd table、FileDescription、file offsetを変更しない。
- 成功値を返す失敗経路がなく、失敗時にFilesystemの既定API以外から下位層へ侵入しない。

## 9. 関連仕様

- Filesystem unlink/drop_link/open orphan契約: `spec/filesystem/fs_impl_spec/fs_impl_spec_09-unlink-close.md`
- 公開Filesystem API: `spec/filesystem/filesystem_external-api_spec.md` §12
- pathname解決: `spec/filesystem/fs_impl_spec/fs_impl_spec_10-pathname-link.md`
- link count・inode lifetime: `spec/filesystem/filesystem_design_spec.md` §18
- syscall ABI: `spec/kernel/syscall_entry_spec.md`
- open file description / close: `spec/kernel/open_close_implementation_spec.md`、`spec/kernel/open_file_description_spec.md`
