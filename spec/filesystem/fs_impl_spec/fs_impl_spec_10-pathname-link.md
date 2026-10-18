# 10 — pathname解決・`link()`

> PATH/NAME上限、名前のバイト列、`nlink`、orderingの共通規則は`AGENTS.md`を参照する。

## 1. resolve API

```cpp
enum class ResolveMode { Target, Parent };

FsStatus resolve(uint32_t cwd_inode,
                 const char* path,
                 ResolveMode mode,
                 uint32_t* out_inode,
                 char out_name[NAME_MAX + 1]);

FsStatus resolve_path(uint32_t cwd_inode,
                      const char* path,
                      uint32_t* out_inode);

FsStatus resolve_parent(uint32_t cwd_inode,
                        const char* path,
                        uint32_t* out_dir_inode,
                        char out_name[NAME_MAX + 1]);
```

- `resolve_path()`は最終targetまで解決する。
- `resolve_parent()`は最終名の直前で止まる。
- wrapperは共通の`resolve()`を使用する。

## 2. 開始位置

- path先頭が`/`ならroot inode 1から開始する。
- それ以外は呼び出し側指定`cwd_inode`から開始する。
- cwdの保管はprocess層の責務である。
- 空文字列は`EINVAL`とする。
- 128バイト以上のpathは`ENAMETOOLONG`とする。

## 3. component走査

- 連続する`/`は空componentとして無視する。
- componentは`/`または終端NULまでの1〜27バイトとする。
- componentの文字コードは検査せず、NUL以外の各バイトを不透明な値として扱う。
- componentを固定長bufferへcopyする前に長さを検査する。
- 現在inodeがdirectoryでなければ`ENOTDIR`とする。
- `dir_lookup()`で次inodeへ進む。
- `.`と`..`も通常lookupする。
- rootの`..` direntはrootを指す。
- symlink展開は行わない。

## 4. Parent mode

- 最終componentを`out_name`へ返す。
- 親directory inodeを返す。
- `/`だけのpathはfile nameを持たないため`EISDIR`とする。
- `foo/`のような末尾slashもfile nameを持たないため`EISDIR`とする。
- 最終名について存在確認は呼び出し側操作が行う。

## 5. syscall側の使用

- `creat`、`mkdir`、`unlink`は`resolve_parent()`後に低レベル操作を呼ぶ。
- `open`、`stat`、`access`、`chdir`は`resolve_path()`を使う。
- `chdir`は解決後inodeがdirectoryであることを確認する。

## 6. `link()`

```cpp
FsStatus link(uint32_t cwd_inode,
              const char* oldpath,
              const char* newpath);
```

順序を変更してはならない。

1. `resolve_path(oldpath)`でsource inodeを得る。
2. source inodeを取得し、directoryなら`EISDIR`とする。
3. `resolve_parent(newpath)`で新しい親と名前を得る。
4. `dir_lookup()`し、既存なら`EEXIST`とする。
5. `inc_nlink(source)`を先に行う。
6. `dir_add_entry()`で新名を追加する。
7. add失敗時は`09-unlink-close.md`の`drop_link(source)`でrollbackする。

- direntを先に追加してはならない。
- `inc_nlink()`は`UINT16_MAX`で`EMLINK`を返す。
- rollbackの`dec_nlink()`後も通常は`nlink > 0`なので、`finalize_if_orphaned()`はinode内容を変更しない。
- `drop_link()`失敗時は新しいdirentを作らず、増加したlink countの残留を許容する。
- link成功・rollbackで`ctime`規約を守る。
- filesystemを跨ぐlinkは対象外とする。

## 7. 完了条件

- absolute/relative pathを解決できる。
- repeated slash、`.`、`..`、root `..`を処理できる。
- 空path、長いcomponent、長いpath、途中non-directoryを拒否できる。
- parent modeで末尾slashを拒否できる。
- hard link後に双方の名前が同じinodeを指す。
- destination存在、directory source、nlink上限を拒否できる。
- dir add失敗時は`drop_link()`でrollbackし、成功時にnlinkが元へ戻る。
- rollback失敗時も新しいdirentが残らない。
