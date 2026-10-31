# 極小UNIX `ln`仕様書（ハードリンク）

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-LN-001 |
| 版 | 0.1（実装前ドラフト） |
| 対象 | サブセットOSの`/bin/ln` |
| 位置づけ | `link()` syscallでhard linkを作成するユーザーランドコマンド |
| 依拠する仕様 | [`link_implementation_spec.md`](../kernel/link_implementation_spec.md)、[`filesystem_design_spec.md`](../filesystem/filesystem_design_spec.md)、syscall入口仕様 |

## 1. 目的と範囲

既存の通常ファイルに新しいpathnameを追加する。`ln`は`link()` syscallの薄いユーザーランド利用例とし、inodeやdirectory entryを直接操作しない。

初版はhard linkのみ対応し、sourceとdestinationを各1つ受け付ける。symlink、directory、複数source、destination directoryへの自動配置、force/interactive optionは扱わない。

## 2. 起動形式

```text
ln source destination
```

- pathnameをちょうど2つ指定する。不足・余分な引数はusage errorとする。
- optionは提供しない。`-s`、`-f`、`-i`、`--`もpathnameとして扱う。
- `-`で始まるpathnameは`./-name`のように指定する。
- 絶対pathと相対pathを受け付ける。相対pathは呼び出し元Processのcwdを基準とする。
- pathname上限は`PATH_MAX=128`、component上限は`NAME_MAX=27` byteとし、切り詰めない。
- shellはクォートやglobを展開しない。受け取った`argv`をそのまま使う。

## 3. 作成動作

- 引数検査後、`link(source, destination)` syscallを1回呼び出す。
- 成功時はdestination名がsourceと同じinodeを指す。ファイル内容の複製や新inode作成は行わない。
- destinationがすでに存在する場合は失敗し、既存entryを置換しない。
- sourceがdirectoryの場合はOS側が拒否する。directory hard linkを作成しない。
- 同一Filesystem内のみ対応する。現在は単一Filesystemを前提とし、異なるFilesystem間移動・linkは対象外。
- open中のsourceでもlink可能であり、source FDやoffsetは変化しない。
- sourceまたはdestination pathに対する事前`stat()`、`open()`、一時FD作成は行わない。

## 4. 出力・エラー・終了ステータス

- 成功時は標準出力へ何も出さず、終了status 0とする。
- 引数不正、syscall失敗時は標準エラーfd=2へ`ln:`で始まる診断を出し、終了status 1とする。
- 診断にはsourceとdestinationを可能な範囲で含める。厳密な文言やerrno数値の表示形式は規定しない。
- syscall ABIが失敗時に`-1`のみ返すため、`ln`はFilesystemの`FsStatus`を直接参照しない。
- `write()`が部分書込みした場合は残りを再送する。0または負値で診断出力を停止し、再帰的な診断を行わない。

## 5. 使用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `link(oldpath, newpath)` / `btx::sys_link()` | hard link作成 |
| OSラッパ | `write()` | 標準エラーへの診断 |
| ilibc | `std::strlen()`または既存format API | pathname表示・診断 |
| ランタイム | `main()`の戻り値 | Process終了status |

- 動的メモリ確保、`FILE*`、`printf()`、host OS固有API、Filesystem内部APIを使わない。
- syscall失敗後に`unlink()`で独自rollbackしない。link syscallのFilesystem更新・rollback契約に従う。
- fd 0/1/2やProcess cwdを変更しない。

## 6. 受入確認項目

1. 引数不足・超過でusageを出し、syscallを呼ばずstatus 1となる。
2. 通常fileへのlink成功後、両pathが同一inodeと内容を示し、nlinkが1増える。
3. 一方の名前を`rm`で削除しても、もう一方から内容へ到達できる。
4. 既存destination、missing source、directory source、invalid pathで診断しstatus 1となる。
5. open中sourceへのlink後も既存FDとoffsetが維持される。
6. syscall失敗時にコマンドがFilesystem内部APIや独自cleanupへ迂回しない。

## 7. 非対応機能

- symlink作成（`ln -s`）。
- directory hard link。
- 既存destinationの置換（`-f`）、確認（`-i`）、option終端（`--`）。
- 複数source、destination directoryへの一括配置。
- Filesystemを跨ぐlink。rename/moveは`mv`コマンドの機能であり、本コマンドでは提供しない。
