# 09 — `unlink()`・open file・`close()`

> 共通の`nlink`上限、ordering、phase 1非再利用は`AGENTS.md`を参照する。

## 1. `open_file_t`

```cpp
struct open_file_t {
    uint32_t inode_num;
    uint32_t offset;
    uint32_t ref_count;
    bool is_pipe;
    /* 必要なRAM専用cache */
};
```

- 実フィールドは既存設計に合わせる。
- `OpenFileTable`が固定配列として所有する。
- `Filesystem<Backend>`が`OpenFileTable`を所有する。
- process fd tableはopen-file entryへの参照だけを持つ。
- `dup`/`fork`で同じentryを共有し、`ref_count`を増やす。
- 共有entryはoffsetも共有する。

## 2. `unlink()`

```cpp
FsStatus unlink(uint32_t dir_inode_num, const char* name);
```

順序を変更してはならない。

1. `dir_lookup()`でtargetを得る。
2. target inodeを取得する。
3. directoryなら`EISDIR`を返す。
4. `dir_remove_entry()`で名前を先に消す。
5. remove成功後だけ`drop_link(target)`する。

- 逆順は禁止する。
- `drop_link()`は`dec_nlink()`後に`finalize_if_orphaned()`を行う。
- name削除後の`drop_link()`失敗はinodeまたはlogical storageのleakを生み得るが、dangling nameより安全である。
- directory削除は常に拒否する。

## 3. `finalize_if_orphaned()`

`Filesystem<Backend>`がopen-file表とinodeテーブルを調停するための共通helperとする。

```cpp
FsStatus finalize_if_orphaned(uint32_t inode_num);
```

1. inodeを取得する。
2. `nlink != 0`なら内容を変更せず`OK`を返す。
3. `open_files_.inode_is_open(inode_num)`なら内容を変更せず`OK`を返す。
4. `inode_table_.finalize_inode(inode_num)`を呼び、そのstatusを返す。

- `inode_is_open()`は特定entryの`ref_count`だけでなく、open-file表全体を走査する。
- `InodeTable`はopen-file表を参照しない。logical storageとinodeをfinal状態へ移す処理は、`05-inode-table.md`の`finalize_inode()`が担う。
- `nlink == 0`かつopen中のinodeでは、`mode`、`size`、`first_cluster`を維持する。

link countを1減らし、必要ならfinalizeする操作も`Filesystem<Backend>`の共通helperとする。

```cpp
FsStatus drop_link(uint32_t inode_num);
```

1. `inode_table_.dec_nlink(inode_num)`を呼ぶ。
2. 失敗した場合はそのstatusを返す。
3. `finalize_if_orphaned(inode_num)`を呼び、そのstatusを返す。

- `unlink()`、`creat()`のrollback、`link()`のrollbackは原則として`drop_link()`を使う。
- `dec_nlink()`成功後にfinalizeが失敗した場合、減算済みのlink countは維持され、orphanの残留を許容する。

## 4. `close()`

1. 有効entryであることを確認する。
2. `ref_count`を1減らす。
3. まだ0でなければ`OK`を返す。
4. 最後の参照ならinode番号を保持してentryを解放する。
5. `finalize_if_orphaned()`を呼び、そのstatusを返す。

- `close()`は`nlink`を変更しない。
- open中にunlinkされたinodeは最後のcloseまでread/write可能である。
- 同じinodeを指す別のopen-file entryが残っていればfinalizeしない。
- 最後のentryを解放した後にfinalizeが失敗した場合も、fdとentryは解放済みとする。`close()`はエラーを返し、二度目のcloseを拒否する。
- finalize失敗時は`nlink == 0`のlive inodeが残留し得るが、entryを復活させない。

## 5. phase 1クラッシュ制約

- open-but-unlinked状態でクラッシュするとRAMのopen tableは失われる。
- 該当inodeが再起動後に残留する可能性を許容する。
- mount中に推測で回収しない。
- この制約はfsckまたはphase 2の課題である。

## 6. 完了条件

- 通常unlinkでdirentが消え、`nlink`が減る。
- directory unlinkを拒否する。
- remove失敗時に`nlink`が変わらない。
- open中unlink後も既存fdからread/writeできる。
- dupされた最後でないcloseではfinalizeしない。
- 同一inodeの別open-file entryが残るcloseではfinalizeしない。
- 最後のcloseでのみorphanが`S_IUNUSED`になる。
- 最後のcloseでfinalizeが失敗してもfdとentryが解放済みである。
- `nlink` underflowと`ref_count` underflowを防ぐ。
