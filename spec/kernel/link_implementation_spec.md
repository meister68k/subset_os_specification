# `link()`システムコール実装仕様

## 1. 目的と範囲

本書は、既存の通常ファイルに新しいpathnameを追加するハードリンク作成syscall `link()`の契約を定める。kernel syscall handlerは引数検証と結果変換を行い、Processのcwdとpathnameを使う内部APIへ委譲する。dirent、inode link count、rollbackの処理順序はFilesystem仕様に従う。

phase 1では同一Filesystem内の非directory inodeへのhard linkだけを扱う。directory hard link、symlink、異なるFilesystem間のlink、rename、permission判定は対象外とする。

## 2. syscall ABI

```cpp
int32_t sys_link(const char* oldpath, const char* newpath);
```

- syscall番号は9とする（V7 `link`と同じ番号）。
- 成功時は0、失敗時は-1を返す。errnoや新たなエラー取得APIを追加しない。
- `oldpath`は既存sourceを指定し、`newpath`は新しい名前を指定する。既存destinationの置換はしない。
- Filesystem内では`FsStatus`を保持し、syscall境界で成功/失敗のABIへ変換する。

## 3. 処理経路と責務

```text
btx::sys_link(oldpath, newpath)
  -> syscall dispatcher (9)
  -> kernel::sys_link(oldpath, newpath)        // ABIの薄い変換
  -> kernel::kernel_link(current, oldpath, newpath) // Process cwd・安全なpath copy
  -> Filesystem::link(cwd_inode, oldpath, newpath)  // pathname解決・link操作
```

概念上のkernel内部API:

```cpp
fs::FsStatus kernel_link(Process& process,
                         const char* oldpath,
                         const char* newpath);
```

- syscall handlerはFilesystemのinode、dirent、nlinkやdisk表現を直接操作しない。
- kernel内部APIはsyscall dispatcherを通さず利用できる。
- 両pathはFilesystemへ渡す前にそれぞれ`PATH_MAX`内のNUL終端文字列として安全に検証・copyする。null、未終端、上限超過を検出した場合はFilesystem状態を変更しない。
- 相対pathは呼び出し元Processのcwd、絶対pathはrootを基点とする。
- Filesystem単体への並行mutationは既定の直列化条件に従う。

## 4. Filesystem操作

Filesystem公開APIは次の形式とする。

```cpp
FsStatus link(uint32_t cwd_inode,
              const char* oldpath,
              const char* newpath);
```

このAPIはpathname解決も行い、kernel層から親inodeやdirentを直接渡さない。実装順序は[`fs_impl_spec_10-pathname-link.md`](../filesystem/fs_impl_spec/fs_impl_spec_10-pathname-link.md) §6に従う。

1. `resolve_path(cwd_inode, oldpath)`でsource inodeを解決する。
2. source inodeを検証し、directoryなら`EISDIR`で拒否する。
3. `resolve_parent(cwd_inode, newpath)`でdestinationの親と最終nameを解決する。最終名のない`/`や末尾slashは拒否する。
4. destination名をlookupし、既存なら`EEXIST`とする。`ENOENT`の場合だけ続行し、その他のlookup失敗は伝播する。
5. sourceのlink countを増加する。上限なら`EMLINK`としてdirentを変更しない。
6. destination direntを追加する。
7. dirent追加に失敗した場合は共通`drop_link(source)`でrollbackする。rollback失敗時は最初の失敗statusを返し、新direntを残さずlink countの残留を許容する。

direntをlink countより先に公開してはならない。成功時はsourceとdestinationが同一inodeを指し、ctimeと親directory更新はFilesystemのdirent/link仕様に従う。

## 5. 結果・エラー

| 条件 | Filesystem結果 | syscall結果 |
|---|---|---:|
| 新しいhard link作成成功 | `OK` | 0 |
| null/空path | `EINVAL`相当 | -1 |
| path/name上限超過 | `ENAMETOOLONG` | -1 |
| sourceが存在しない | `ENOENT` | -1 |
| path途中componentがdirectoryでない | `ENOTDIR` | -1 |
| sourceがdirectory | `EISDIR` | -1 |
| destination名が既に存在 | `EEXIST` | -1 |
| sourceのlink count上限 | `EMLINK` | -1 |
| inode/dirent/FAT資源不足 | `ENOSPC`または下位status | -1 |
| Filesystem未mount、I/O、破損 | `ENODEV`、`EIO`、`ECORRUPT`等 | -1 |
| dirent追加後のrollback failure | 最初のfailure status | -1。名前は公開せずlink count残留を許容 |

Filesystem状態更新後に下位処理が失敗した場合は、その時点までに仕様上確定した変更を巻き戻さない。ユーザー側には詳細statusを公開せず、診断の詳細規則は既存ABIに従う。

## 6. FD・cwdとの関係

- `link()`は既存FD、FileDescription、file offset、Process cwdを変更しない。
- sourceがopen中でも新しい名前を追加できる。どちらか一方の名前を後で`unlink()`しても、他方またはopen参照が残る限りinode内容は保持される。
- cwdを指すdirectory inodeへのhard linkは作らない。directory sourceは拒否する。

## 7. 必須確認項目

1. syscall番号9が`sys_link(oldpath, newpath)`へdispatchされ、成功0/失敗-1を返す。
2. 相対/絶対pathがcwd/root基準で解決される。
3. 成功後、sourceとdestinationが同一inodeを指し、nlinkが1増える。
4. destinationが既存、sourceがdirectory、source不在、各path不正の場合にFilesystem mutationが起きない。
5. nlink上限、dirent追加失敗、rollback失敗で仕様どおりの順序・statusとなり、dangling nameを公開しない。
6. open中sourceへのlinkと、その後の片方のunlinkがopen参照寿命を壊さない。
7. `link()`がFD table、FileDescription、offset、cwdを変更しない。
