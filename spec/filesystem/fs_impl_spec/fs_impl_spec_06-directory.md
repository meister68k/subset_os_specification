# 06 — ディレクトリ

> dirent形式、名前のバイト列、pathname共通上限は`AGENTS.md`と`00-overview.md`を参照する。

## 1. モデル

- ディレクトリは固定長dirent配列を内容とする通常ファイルである。
- `inode.size`は有効dirent数ではなく、確保済みディレクトリ領域の容量を表す。
- directory chainを1クラスタ確保するたびに`inode.size`を512増やす。
- directory操作は汎用chain read/write helperを利用する。
- ユーザの一般`write()`からdirectory内容を変更させない。`creat()`、`mkdir()`、`unlink()`等のdirectory層経由の操作だけが変更する。

## 2. API

```cpp
FsStatus dir_lookup(uint32_t dir_inode_num, const char* name,
                    uint32_t* out_inode_num);
FsStatus dir_add_entry(uint32_t dir_inode_num, const char* name,
                       uint32_t inode_num);
FsStatus dir_remove_entry(uint32_t dir_inode_num, const char* name,
                          uint32_t* out_inode_num);
FsStatus dir_is_empty(uint32_t dir_inode_num, uint32_t expected_parent_inode_num,
                      bool* out_is_empty);
```

- 全APIは対象inodeがlive directoryであることを確認する。
- pathname由来の名前は、pathname層で検証済みの1〜27バイトのcomponentとして受け取る。
- 名前の文字コードは解釈せず、NUL以外のバイト列として比較する。
- `.`と`..`を特別な文字列アルゴリズムで処理せず、通常direntとして扱う。

## 3. lookup

- 先頭から`inode.size`内のdirentを順にdecodeする。
- 空きentryは読み飛ばす。
- byte列として完全一致する名前を探す。
- 発見時はinode番号を返す。
- 未発見は`ENOENT`とする。
- 無効なinode番号、壊れたNUL終端、使用中direntの空名、使用中dirent名に含まれる`/`、size不整合は`ECORRUPT`とする。

## 4. add_entry

- directory層自身は重複名検査を行わない。
- 呼び出し側が操作固有の重複規則を適用する。
- 既存の空きslotがあればそこへ書く。
- 空きslotがなければdir chainを1cluster延長する。
- 新clusterは全ゼロにしてからentryを書く。
- entryは`dirent_encode()`で書く。
- 必要な場合だけdirectory sizeを増やす。
- 内容変更時にdirectoryの`mtime`と`ctime`を更新する。
- inode更新を含む失敗時の最悪結果をリーク側へ寄せる。

## 5. remove_entry

- 先に対象をlookupする。
- 対象slotをゼロ化して空きに戻す。
- directoryのchainやsizeを縮小しない。
- 内容変更時にdirectoryの`mtime`と`ctime`を更新する。
- 対象inodeの`nlink`はこの関数で変更しない。
- `unlink()`がremove成功後に`dec_nlink()`する。

## 6. 空directory判定

- 対象はlive directoryでなければならない。
- `.`が自身を指すentryとしてちょうど1つ、`..`が`expected_parent_inode_num`を指すentryとしてちょうど1つ存在することを検証する。
- これら以外の有効entryがあれば`out_is_empty = false`、なければ`true`とする。
- 必須entryの欠落・重複・誤ったinode参照、またはdirent/chain破損は`ECORRUPT`とする。
- 出力値は走査が成功した場合にだけ確定する。

## 7. root初期状態

- rootはinode 1である。
- root directoryには`.`と`..`の2entryを作る。
- 両entryともinode 1を指す。
- rootは1クラスタ確保済みなので、sizeは512とする。
- `.`と`..`より後の未使用slotはゼロであり、sizeの範囲内で空きslotとして走査する。

## 8. 完了条件

- lookupのhit/missを検証できる。
- 空きslot再利用とcluster追加の両方を検証できる。
- direntがblock境界に接するケースを検証できる。
- removeがentryだけを消し、対象inodeを変更しない。
- `.`と`..`を通常lookupできる。
- 空directory判定が必須entryを検証し、他entryを検出できる。
- malformed direntを`ECORRUPT`で停止できる。
