# `chdir()` システムコール実装仕様

## 1. 目的と範囲

本書はProcessのcurrent working directory (cwd)を変更する`chdir()` syscallを定める。syscall境界でpathを安全に取り込み、Filesystemの公開pathname・属性APIでtargetを特定し、Process管理経由でcwd参照を交換する。

phase 1ではroot filesystem 1個、通常directory、`.`/`..`を含む既定pathname解決を扱う。permission、symlink、VFS/mount namespace、cross-filesystem cwdは対象外とする。

## 2. syscall ABI

```cpp
int32_t sys_chdir(const char* path);
```

- 成功時は`0`、失敗時は`-1`を返す。
- 現行ABIは失敗時の`-1`のみを定義する。errno機構を新設せず、内部のFilesystem statusをsyscall境界で既定の失敗値へ変換する。
- syscall番号は本書で割り当てない。

## 3. 処理経路と責務

```text
btx::sys_chdir(path)
  -> syscall dispatcher
  -> kernel::sys_chdir(path)                // user path検証・結果変換
  -> kernel::kernel_chdir(Process&, path)   // target検査・cwd参照交換
       -> Filesystem::resolve_path(old_cwd, path, &target_inode)
       -> Filesystem::stat(target_inode, &info)
       -> Filesystem cwd-ref retain(target_inode)
       -> Process::cwd_inode_ = target_inode
       -> Filesystem cwd-ref release(old_cwd)
```

syscall handlerはthin adapterとし、cwdフィールドを直接書き換えない。Processのcwd更新はProcess管理APIに集約する。Filesystem利用者は`Filesystem<Backend>`公開APIだけを使用し、`InodeTable`、`disk_inode_t`、FATへ直接アクセスしない。

概念上の内部API:

```cpp
FsStatus kernel_chdir(Process& process, const char* validated_path);
```

実際の命名とProcessManagerへの委譲方法は既存実装規約に合わせる。引数pathはカーネル側の検証済みNUL終端コピーとする。

## 4. Pathとtarget検証

1. `path == nullptr`を拒否する。
2. user pathを固定長`PATH_MAX` bufferへ安全に取り込む。NULを含む長さがPATH_MAX未満であることを確認し、終端NULがなければ`ENAMETOOLONG`相当とする。検査前にpathname APIを呼ばない。
3. 現cwdを読み、`resolve_path(cwd, path, &target_inode)`を呼ぶ。absolute pathはrootから、relative pathはcwdから解決する。
4. `Filesystem::stat(target_inode, &info)`で属性を得る。
5. `(info.mode & S_IFMT) == S_IFDIR`を確認する。通常file等は`ENOTDIR`相当で拒否する。

空pathは`EINVAL`、途中componentがdirectoryでない場合は`ENOTDIR`、不存在componentは`ENOENT`、長すぎるpath/componentはpathname仕様のstatusを返す。連続slash、`.`、`..`、root `..`の扱いは既存pathname仕様に従う。末尾slashは`resolve_path()`の既定規則に従って解決し、targetがdirectoryなら許可する。

Filesystem未mountや破損・I/O failureは元の内部statusを保つ。syscallはそれらを`-1`へ変換し、cwdを変更しない。

## 5. cwd参照の交換順序

各Processはcwd inodeを保持し、そのinodeへの参照を1つ所有する。Filesystemの固定長cwd-reference table、容量、孤立directoryのfinalize規則は[`mkdir_rmdir_implimentation_spec.md`](mkdir_rmdir_implimentation_spec.md)に従う。

処理順序は次の通り。

1. target解決とdirectory種別検査を完了する。
2. Filesystemでtarget cwd参照をretainする。
3. retain成功後にProcessのcwd inodeをtargetへ更新する。
4. 旧cwdのFilesystem参照をreleaseする。

- 1または2の失敗ではProcess cwdと旧参照を変更しない。
- targetと旧cwdが同じinodeの場合も同じ順序を使う。固定表の一時追加容量により安全に交換できる。
- 新cwd参照を確立する前に旧cwd参照をreleaseしてはならない。
- 4で旧cwd参照のrelease/finalizeが失敗した場合、新cwdはretain済みでProcessにも設定済みなので、新cwdを維持しsyscallは`-1`を返す。旧参照が残る場合は安全側のリークとして保持し、後段cleanup失敗で先行failureを上書きしない。このpartial state changeを実装・テストで明示する。
- cwd交換はfd table、open file description、file positionを変更しない。

## 6. 孤立cwdと他のprocess操作

- `rmdir()`済みdirectoryをcwdとして保持しているProcessは、そのcwdを基点としたrelative path解決を継続できる。`chdir()`から`"."`へ移る場合も有効なdirectoryとして扱う。
- 別Processがそのdirectoryをcwdとして保持していれば、新Processのcwd retain/releaseは各々独立参照として数える。
- `spawn()`は親cwdをretainして子へ継承する。`exit()`はProcessのcwd参照をreleaseする。詳細は`process_management_spec.md`とmkdir/rmdir実装仕様に従う。
- Filesystemがcwd/open参照を持つ孤立directoryをいつfinalizeできるかはFilesystem仕様が正本であり、本syscallがinode内容を直接finalizeしない。

## 7. エラーと副作用

| 条件 | 内部status相当 | syscall結果 |
|---|---|---:|
| 正常なdirectoryへ変更 | `OK` | `0` |
| null/空path | `EINVAL` | `-1` |
| path/component上限超過 | `ENAMETOOLONG` | `-1` |
| path component不存在 | `ENOENT` | `-1` |
| path途中またはtargetが非directory | `ENOTDIR` | `-1` |
| Filesystem未mount | `ENODEV` | `-1` |
| inode/dirent/chain破損 | `ECORRUPT` | `-1` |
| block I/O失敗 | `EIO`またはBackend status | `-1` |
| cwd参照表枯渇 | `ENOSPC` | `-1`。cwd不変 |
| target retain失敗 | Filesystem status | `-1`。cwd不変 |
| 旧cwd release後のfinalize failure | Filesystem status | `-1`。新cwdへ遷移済み |

通常の検証失敗ではcwdを変更しない。旧cwd release後のfinalize failureだけは§5で定めた状態遷移後の失敗となる。成功・失敗いずれも`Process::cwd_inode_`の値以外に永続inode metadataを変更しない。

## 8. 必須検証

- absolute/relative path、root、`.`、`..`、repeated slash、末尾slash。
- 通常fileへのchdir、missing path、途中non-directory、空/null/長すぎるpathを拒否しcwd不変。
- Filesystem `stat()`でdirectory modeを確認し、inode tableへ直接アクセスしない。
- 成功時にtarget参照retain後にProcess cwdが変わり、旧参照をreleaseすること。
- target retain failureやcwd表容量上限でProcess cwdと旧参照を維持すること。
- old/newが同一inodeの交換を含め、retain-before-release順を検証すること。
- old cwdがrmdir済みのorphanである場合の移動、open/cwd参照が残る間の寿命を確認すること。
- 旧cwd release後のfinalize failureで新cwdが設定済みのまま`-1`を返し、危険なrollbackやdangling cwdを作らないこと。
- `chdir()`がfd、FileDescription、共有file offsetに影響しないこと。

## 9. 関連仕様

- Process cwd所有・継承・終了: `spec/kernel/process_management_spec.md` §8.1、§9.1
- cwd参照表とrmdir後の寿命: `spec/kernel/mkdir_rmdir_implimentation_spec.md`
- pathname解決: `spec/filesystem/fs_impl_spec/fs_impl_spec_10-pathname-link.md`
- Filesystem属性API: `spec/kernel/stat_implementation_spec.md`、`spec/filesystem/filesystem_external-api_spec.md`
- syscall ABI: `spec/kernel/syscall_entry_spec.md`、`spec/kernel/syscall_generator_spec.md`
