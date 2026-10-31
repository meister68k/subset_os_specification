# 12 — Filesystem `rename()` API

## 1. 範囲

同一Filesystem内で、既存のregular fileまたはdirectoryの名前を変更する、または別のdirectoryへ移動するAPIを定める。初版はdestination置換を行わない。symlink、cross-filesystem rename、crash consistency、permission判定は対象外とする。

同一Filesystemへの並行mutationは禁止し、Filesystem利用側で直列化する。phase 1のディスク更新はtransactionalではないため、電源断・Backend I/O failureをまたぐatomicityは保証しない。

## 2. 公開API

```cpp
FsStatus rename(uint32_t cwd_inode,
                const char* oldpath,
                const char* newpath);
```

- `oldpath`と`newpath`は同じFilesystem名前空間で解決する。
- 絶対pathはroot inodeから、相対pathは`cwd_inode`から解決する。
- syscall層はユーザーpathを安全に検証・copyしてから呼ぶ。Filesystem APIはコピー後のNUL終端pathを受け取る。
- 成功時はsource inodeの内容・inode番号を保ったままdirectory entryを移す。file dataはcopyしない。
- destinationが別inodeの既存entryなら`EEXIST`で失敗し、置換しない。
- symlinkがないため、pathname componentをsymlink経由でたどらない。

## 3. pathnameと事前検証

状態変更前に次をすべて検証する。

1. 両pathが非null、非空で、`PATH_MAX`未満である。
2. `resolve_path(oldpath)`でsource inodeを解決し、rootなら`EBUSY`で拒否する。
3. `resolve_parent()`でold/new pathそれぞれの親inodeと最終nameを解決する。`/`、末尾slash、最終nameのないpathは既定pathname errorで拒否する。
4. source entryとinodeが存在し、liveで整合している。sourceはregular fileまたはdirectoryに限る。
5. source最終nameが`.`または`..`なら`EINVAL`として拒否する。
6. destination nameが存在しない。既存destinationがsourceと同じinodeの場合は例外としてno-op成功とする。
7. destination parentがlive directoryであることを確認する。
8. sourceがdirectoryなら、destination parentから`..`をrootまでたどりsource自身へ到達しないことを確認する。source自身またはその子孫への移動は`EINVAL`。走査はinode table容量を上限としてboundedにし、循環・不正な`..`は`ECORRUPT`とする。
9. cross-parent moveではstaged source link countの増加と、directory sourceの場合のnew parent link countの増加が上限内であることを確認する。sourceまたはnew parentのnlinkが`UINT16_MAX`なら`EMLINK`。old parentのnlinkが減算不能な不正値なら`ECORRUPT`とする。

sourceとdestinationが同一inodeならno-op成功とし、name、link count、時刻を変更しない。これは同一pathnameと、hard link aliasをdestinationに指定した場合を含む。

## 4. 成功時の更新規則

- sourceとdestinationの親が同じ場合はentry名だけを変更する。移動元・移動先の中間状態を外部へ公開しない。
- 親が異なる場合はentryを新しい親へ移し、移動元entryを取り除く。regular fileの`nlink`は操作前後で不変とする。
- directoryを親間移動する場合、child directoryの`..`を新parentへ更新する。childの`nlink`は不変とし、旧parentの`nlink`を1減らし、新parentの`nlink`を1増やす。同一parent内の名前変更では親link countを変えない。
- directoryの`.`は自身を指すまま維持する。directory data、file data、inode番号、mode、既存FD、cwd inode参照は変更しない。
- 更新対象directoryの`mtime`/`ctime`と移動対象inodeの`ctime`はFilesystem時刻規約に従い更新する。`atime == mtime`の共通規約を維持する。
- 移動中directoryをcwdとして保持するProcessがあってもinode参照は変更しない。`..`更新により、その後の相対pathname解決と`getcwd()`再構成は新しい親階層に従う。

## 5. Directory層との連携・更新順序

同一parent内の名前変更では、Directory層に次の内部操作を設ける。

```cpp
FsStatus dir_rename_entry(uint32_t parent_inode,
                          const char* old_name,
                          const char* new_name,
                          uint32_t expected_inode);
```

既存slotのnameを更新し、source entryが`expected_inode`を指すこと、destination nameが未使用であることを検証する。旧entryを先に削除してから新entryを追加してはならない。成功時にinode link countを変更しない。

異なるparent間ではFilesystem層がDirectoryの`dir_add_entry()`、`dir_remove_entry()`と、次の`.`/`..`更新APIを組み合わせる。

```cpp
FsStatus dir_set_dotdot(uint32_t child_inode,
                        uint32_t expected_old_parent,
                        uint32_t new_parent);
```

`dir_set_dotdot()`はchildがlive directoryで、`.`と`..`が各1個あり、`..`が`expected_old_parent`を指すことを検証して更新する。Directory層はinode/parent link countを変更しない。

異なるparent間のmoveは、全検証後、次の順で処理する。

1. staged destination entryを安全に扱うため、source inodeのlink countを1増やす。
2. sourceがdirectoryなら、新parentのlink countを1増やす。上限・不整合はmutation前に検査する。
3. destination parentへsource inodeの新entryを追加する。
4. sourceがdirectoryなら、childの`..`をold parentからnew parentへ変更する。
5. old parentからsource entryを削除する。この削除成功をnamespace移動のcommit pointとする。
6. commit後、staged source linkを1減らす。sourceがdirectoryならold parentのlink countも1減らす。

step 1〜4の失敗、またはstep 5の失敗では、変更済みmetadataとdestination entryを逆順でrollbackする。rollbackが失敗した場合は最初のfailureを返す。dangling direntや、finalize済みinodeを指すentryを作らない。step 5成功後のlink count/parent count cleanup failureはfailure statusを返すが、namespace変更は取り消さず新pathnameを有効に保つ。したがって、mutation開始後のfailureはsource名が消えている場合がある。

同一parent内renameはsingle existing slotのname updateとし、parent/source link countを変更しない。すべての経路でFilesystem APIは動的メモリを使わず、既存固定上限内で処理する。disk write中の電源断整合性は保証しない。

## 6. 更新失敗・crash制約

- pathname解決・種別・cycle・destination検査の失敗は、Filesystem状態を変更しない。
- mutation開始後のcommit前Backend failureではrollbackを試み、最初のfailure statusを返す。rollback failureではdangling direntやfinalize済みinodeへの参照を作らず、修復可能なlink count/metadataの残留を許容する。
- namespace commit後のmetadata cleanup failureではfailure statusを返すが、pathname moveは完了済みである。呼出し側は失敗を報告し、古いpathnameの存在を仮定しない。
- syscallの失敗値だけから「一切変更されなかった」と推測してはならない。mutation開始後の失敗では、呼出し側は`mv`コマンド仕様どおり失敗を報告し、独自に逆renameや`unlink()`を行わない。
- 電源断・kernel crash時のoperation atomicityは保証しない。journal、shadow copy、fsckは導入しない。
- 初版では同一Filesystem内のrenameのみなので`EXDEV`は通常発生しない。将来複数Filesystemを導入した場合は、Filesystemをまたぐrenameを`EXDEV`で拒否する。

## 7. 結果status

| 条件 | status |
|---|---|
| rename成功または同一inode no-op | `OK` |
| null/empty path | `EINVAL` |
| path/name超過 | `ENAMETOOLONG` |
| source不存在 | `ENOENT` |
| path途中componentがdirectoryでない | `ENOTDIR` |
| source/destination親/entry構造の不整合 | `ECORRUPT` |
| sourceまたはdestination parentが対応外種別 | `EISDIR` / `ENOTDIR` |
| destinationが別inodeで既存 | `EEXIST` |
| sourceが`.`、`..`、root | `EINVAL` / `EBUSY` |
| directoryを自身の子孫へ移動 | `EINVAL` |
| directory cycle/link count更新が不正、staged nlink上限 | `ECORRUPT` / `EMLINK` |
| directory entry追加領域不足 | `ENOSPC` |
| disk/backend failure | `EIO`またはBackend status |

## 8. 必須確認項目

1. 同一parent内でregular fileとdirectoryの名前を変更でき、inode番号・内容が維持される。
2. 異なるparent間でregular fileを移動でき、nlinkが不変である。
3. 異なるparent間でdirectoryを移動でき、childの`.`/`..`、両parentのnlink、後続relative pathと`getcwd()`が整合する。
4. open中regular file、cwd参照中directoryを移動しても既存参照が有効である。
5. source/destination同一inode aliasはno-op成功となる。
6. destination exists、source missing、root/`.`/`..`、self/descendant move、path上限、途中non-directoryで事前状態変更がない。
7. 境界ごとのBackend failureで最初のstatusを保持し、dangling direntや無効な`.`/`..`を残さない。rollback failureも注入する。
8. inode/cluster/directory table上限でoverflowせず、範囲外を書き換えない。
