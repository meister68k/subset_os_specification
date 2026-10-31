# 極小UNIX `mv`仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-MV-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの`/bin/mv` |
| 位置づけ | `rename()` syscallでpathnameを変更・移動するユーザーランドコマンド |
| 依拠する仕様 | [`rename_implementation_spec.md`](../kernel/rename_implementation_spec.md)、[`fs_impl_spec_12-rename.md`](../filesystem/fs_impl_spec/fs_impl_spec_12-rename.md)、syscall入口仕様 |

## 1. 目的と範囲

同一Filesystem内でregular fileまたはdirectoryの名前を変更するか、別directoryへ移動する。`mv`は`rename()` syscallの薄い利用例であり、データcopy、`link()`、`unlink()`による代替処理はしない。

初版はsourceとdestinationを各1つ受け付ける。destination置換、複数source、destination directoryへの自動配置、symlink、Filesystemを跨ぐmoveは対象外とする。

## 2. 起動形式

```text
mv source destination
```

- pathnameをちょうど2つ指定する。不足・余分な引数はusage errorとする。
- optionは提供しない。`-f`、`-i`、`-n`、`--`もpathnameとして扱う。
- `-`で始まるpathnameは`./-name`のように指定する。
- 相対pathは呼び出し元Processのcwd、絶対pathはroot基準で解決する。
- `PATH_MAX=128`、`NAME_MAX=27` byteのFilesystem上限を超えるpath/nameを切り詰めない。
- shellはクォート、glob、environment variableを展開しない。受け取った`argv`をそのまま使う。

## 3. 動作

- 引数検査後、`btx::sys_rename(source, destination)`を1回呼ぶ。
- syscallが成功したときのみ終了status 0とする。成功時は標準出力へ何も出さない。
- destinationが別inodeとして既に存在すると失敗する。既存file/directoryの上書きや置換をしない。
- sourceとdestinationが同一inodeを指す場合はrename APIのno-op成功となる。
- regular fileの内容copy、inode作成、link/unlinkによる二段階renameを行わない。
- directory移動、移動先がsource自身または子孫である場合などはFilesystem APIが判定する。
- 移動対象がopen中またはcwd参照中でも、既存FD/cwd inodeをユーザーランドから変更しない。directoryの親関係はrename syscall後のFilesystem状態に従う。

## 4. エラー処理

- 引数不正、syscall失敗時は標準エラーfd=2へ`mv:`で始まる診断を出し、終了status 1とする。
- 診断にはsourceとdestinationを可能な範囲で含める。厳密な文言・errno数値は規定しない。
- 失敗時は`cp`、`ln`、`rm`等へfallbackしない。また逆順renameや`unlink()`を独自に試さない。
- mutation開始後のFilesystem/backend failureは部分変更を残し得る。`mv`は失敗を報告するが、戻り値のみで両pathnameが元の状態だと保証してはならない。
- `write()`の部分書込みは継続し、0/負値では診断を停止する。診断失敗で再帰しない。

代表的な失敗はsource不在、path途中のnon-directory、destination存在、root/`.`/`..`、directory self/descendant move、pathname長超過、Filesystem未mount、I/O/破損である。具体的statusはrename syscall仕様に従う。

## 5. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `btx::sys_rename(oldpath, newpath)` | pathname変更・移動 |
| OSラッパ | `btx::sys_write()`相当 | 標準エラー診断 |
| ilibc | `std::strlen()`または既存format API | pathname診断 |
| ランタイム | `main()`の戻り値 | Process終了status |

- `stat()`による事前checkや`open()`による一時FD作成を行わない。source/destination間のraceをユーザーランドで解決しない。
- 動的メモリ確保、`FILE*`、`printf()`、Filesystem内部API、host OS固有`rename()`を使わない。
- Process cwd、FD table、既存open file positionを変更しない。

## 6. 受入確認項目

1. regular fileの同一directory内renameと、異なるdirectoryへのmoveで内容・inode番号が維持される。
2. directory移動後に`.`/`..`、親link count、相対pathname、`getcwd()`が整合する。
3. open中fileとcwd参照中directoryの移動後も既存参照が有効である。
4. destination既存、source不在、root/`.`/`..`、self/descendant move、path上限を失敗として診断する。
5. sourceとdestinationが同一inodeの場合に成功し、状態を変更しない。
6. `mv`が`sys_rename()`を1回だけ呼び、copy/link/unlink fallbackをしない。
7. rename backend failure時は失敗statusを返し、コマンドが無断でcleanupを行わない。

## 7. 非対応機能

- 既存destinationの置換、force/interactive/no-clobber option。
- 複数source、destination directoryへの一括move。
- symlink、cross-filesystem move、copy-and-delete fallback。
- atomicity、crash consistency、rollback保証。
