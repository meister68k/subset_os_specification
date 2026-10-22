# `getcwd()` システムコール実装仕様

## 1. 目的と範囲

本書は、呼出し元Processのcwdをrootからの絶対pathnameとして取得する`getcwd()` syscallを定める。Processは引き続きcwd inodeへの参照を所有する。pathname文字列をProcessテーブルに常時保持せず、呼び出しごとにFilesystemのdirectory関係から再構成する。

phase 1はroot filesystem 1個、通常directory、`.` / `..`を含む既定pathname規則を対象とする。symlink、mount namespace、chroot、複数filesystem間のcwd、directory renameは対象外とする。

## 2. syscall ABI

```cpp
int32_t sys_getcwd(char* buf, uint32_t capacity);
```

- 成功時はNUL終端した絶対pathnameを`buf`へ格納し、**終端NULを含むbyte数**を返す。
- 失敗時は`-1`を返す。errnoやglobal error状態は追加しない。
- root cwdの出力は`/`とNULであり、戻り値は`2`。
- `capacity`は出力先bufferのbyte容量。pathnameと終端NULの全体がcapacityに収まらなければ失敗する。
- `buf == nullptr`、`capacity == 0`、読み取り不能なcwd状態、cwdがroot配下の名前付きdirectoryとして到達不能な場合は失敗する。
- 失敗時は`buf`へpathnameの部分結果を返さない。syscall番号は共通syscall ABI仕様で割り当てる。

`capacity`のABI幅は`uint32_t`とする。kernelでは`capacity`を固定上限`PATH_MAX`以下に制限して一時領域を確保し、加算・narrowingを検証する。Filesystemのpathname規則上、pathname本文は`PATH_MAX`未満であり、終端NULを含めて`PATH_MAX` byte以内に収める。

## 3. pathname再構成の方針

Processテーブルにcwd inodeを保持する現行設計を維持する。`getcwd()`はcwd inodeから親方向へ辿り、各親directory内で子inodeを指す名前を検索し、componentを逆順に固定長一時bufferへ積む。rootまで到達後、`/`から始まるpathnameを組み立てる。

Processに「親inodeと名前」を保存してpathnameを表現する方式は採用しない。cwd inodeは相対pathname解決とcwd参照寿命の正本であり、親子directory関係と名前はFilesystemのdirectory構造の責務とする。

## 4. Filesystem公開API要件

既存Filesystem APIは名前からinodeを引く`dir_lookup(parent, name, ...)`を持つが、親directory内からinodeに対応する名前を検索する公開APIが不足している。以下と同等のFilesystem公開APIを追加する。

```cpp
FsStatus find_name_by_inode(uint32_t parent_inode_num,
                            uint32_t child_inode_num,
                            char out_name[NAME_MAX + 1]);
```

- `parent_inode_num`はlive directory、`child_inode_num`はlive directoryでなければならない。
- 親directoryの有効direntを検査し、`child_inode_num`を指すentry名をbyte単位で返す。
- `.`と`..`は子component名として返さない。対象childが親自身である検索は失敗とする。
- 一致する名前がちょうど1つなら成功する。見つからない場合は`ENOENT`相当、複数一致・壊れたdirent・不正inodeは`ECORRUPT`相当とする。
- `out_name`は成功時にのみ更新し、NUL終端する。名前の最大長は`NAME_MAX`。
- directoryのsize、dirent形式、inode状態をFilesystem自身で検査する。上位層からinode table、FAT、オンディスク構造へ直接アクセスしない。

この逆引きAPIは`Filesystem<Backend>`の公開境界に置き、pathnameの再構成をカーネルからFilesystem私有状態へ侵入して実装しない。

## 5. 処理経路

```text
btx::sys_getcwd(buf, capacity)
  -> syscall dispatcher
  -> kernel::sys_getcwd(buf, capacity)       // ABI・出力buffer検証
  -> kernel::kernel_getcwd(Process&, temp)   // cwd inodeから構成
       -> Filesystem::stat(cwd_inode, ...)
       -> Filesystem::dir_lookup(child, "..", &parent_inode)
       -> Filesystem::find_name_by_inode(parent_inode, child_inode, name)
       -> 固定長buffer内でpathnameを構成
  -> 構成完了後にbufへcopy-out
```

syscall handlerはProcessの`cwd_inode_`を直接操作せず、Process管理APIからcwd inodeを読み取る。Filesystem処理は公開API経由とし、出力pathnameの組立とABI戻り値変換はkernel syscall層が担う。

## 6. 再構成アルゴリズム

1. `buf`、`capacity`、出力範囲の書込み可能性を検証する。出力範囲検証の前にユーザーバッファへ書き込まない。
2. 現在Processのcwd inodeを取得し、`Filesystem::stat()`でlive directoryであることを確認する。
3. cwd inodeが`ROOT_INODE`なら`/\0`を一時bufferに構成する。
4. root以外では、現在inodeの`..` entryから親inodeを得る。rootの`..`はrootを指すFilesystem契約に従う。
5. 親inodeのdirectory内で現在inodeを指すentry名を`find_name_by_inode()`で取得する。
6. `/component`分の容量を含めて検査し、componentを逆順用の固定長bufferへ追加する。`PATH_MAX`を越える前に失敗させ、整数wraparoundを許さない。
7. 親inodeがrootでなければ現在inodeを親へ移し、手順4へ戻る。走査回数は最大`PATH_MAX`までとし、rootへ到達しない循環や破損関係は失敗とする。
8. component列を逆順にして、先頭`/`と終端NULを含む絶対pathnameを一時bufferに完成させる。
9. `result_length`をpathnameと終端NULを含むbyte数として計算する。`result_length <= capacity`を再確認する。
10. 完成済み一時bufferをユーザー`buf`へcopy-outし、`result_length`を返す。

途中で失敗した場合、呼出し元Processのcwd、FD、Filesystem metadataを変更しない。ユーザーbufferには部分pathnameを出力しない。ユーザー領域の安全なcopy-in/copy-outは既存のsyscall境界契約に従い、MMUの存在を仮定しない。ターゲットで書込み可能性を検査できない場合の安全なuser-memory契約はsyscall入口仕様で別途確定する。

## 7. 到達不能cwdとエラー

| 条件 | 内部扱い | syscall結果 |
|---|---|---:|
| 正常なcwd | `OK` | NULを含むpathname byte数 |
| null buffer / capacity 0 / 不正範囲 | `EINVAL`相当 | `-1` |
| capacity不足 | `ERANGE`または`ENAMETOOLONG`相当 | `-1` |
| cwd inodeがroot配下の親direntから見つからない | `ENOENT`相当 | `-1` |
| cwdがdirectoryでない、親関係が不正 | `ENOTDIR` / `ECORRUPT`相当 | `-1` |
| Filesystem未mount、I/O失敗 | `ENODEV` / `EIO`相当 | `-1` |
| 循環またはpathname上限超過 | `ECORRUPT` / `ENAMETOOLONG`相当 | `-1` |

syscall ABIはエラー原因を個別の戻り値として公開せず、全て`-1`に変換する。Filesystem内部statusは内部層で保持し、OS global `errno`を追加しない。

`rmdir()`済みdirectoryをcwdとして保持しているProcessは、既存契約どおりそのinodeを基点に相対pathname操作を続けられる。ただし親directory内にそのinodeを指す名前がないため、`getcwd()`は失敗する。削除済みdirectory名を推測・キャッシュして返してはならない。

## 8. 同期と副作用

- Filesystemのdirectory走査は、既存Filesystemの直列化契約内で行う。走査中に同一filesystemのdirectory mutationが割り込まないよう、Filesystem全体で定めた同期規則を使う。
- `getcwd()`はpathname解決、`open()`、FD割当、cwd参照のretain/releaseを行わない。
- directory内容、inode、FAT、時刻、Process cwd、FileDescription、file positionを変更しない。
- 並行`rename()`は現段階の対象外。将来追加する場合は、pathname再構成と親・名前の一貫性を保証する同期・再試行契約をFilesystem仕様へ追加する。

## 9. 受入確認項目

- root cwdで`/\0`を返し、戻り値が2である。
- root直下および複数階層のcwdで正しい絶対pathnameとNULを返す。
- `.`、`..`、連続slashを経た`chdir()`後も、実際のdirectory entry名に基づく絶対pathnameを返す。
- pathnameとNULがcapacityにちょうど収まる場合は成功し、1 byte不足では失敗してbufferへ部分pathnameを残さない。
- null buffer、capacity 0、未mount、I/O故障、破損dirent、親循環を失敗として扱う。
- `rmdir()`済みcwdで相対pathname操作は既存契約どおり可能だが、`getcwd()`は失敗する。
- 成功・失敗のいずれでもcwd参照数、FD、Filesystem metadata、file positionが不変。
- 逆引きAPIが0件・1件・複数件の一致を区別し、不正なdirentを検出する。

## 10. 関連仕様

- syscall入口、user-memory検証、番号: [`syscall_entry_spec.md`](syscall_entry_spec.md)
- cwd inodeの所有・継承・寿命: [`process_management_spec.md`](process_management_spec.md)
- cwd参照とrmdir後の寿命: [`mkdir_rmdir_implimentation_spec.md`](mkdir_rmdir_implimentation_spec.md)
- pathname解決とdirectory API: [`filesystem_external-api_spec.md`](../filesystem/filesystem_external-api_spec.md)、[`fs_impl_spec_06-directory.md`](../filesystem/fs_impl_spec/fs_impl_spec_06-directory.md)、[`fs_impl_spec_10-pathname-link.md`](../filesystem/fs_impl_spec/fs_impl_spec_10-pathname-link.md)
- cwd変更: [`chdir_implementation_spec.md`](chdir_implementation_spec.md)
