# `getdents()` システムコール実装仕様

## 1. 目的と範囲

本書は、directoryを開いたfdからエントリ列を取得するphase 1の`getdents()` syscallを定義する。syscallはfd・ユーザーバッファを検証してFD層へ委譲し、通常ファイル固有処理は`RegularFile`とFilesystem公開APIへ分担する。

対象は固定長32-byte dirent、通常ファイルとdirectoryのFD統合経路、共有file position、partial resultと失敗処理である。`readdir()`系のユーザーライブラリ、名前解決、permission、VFS、可変長dirent、ホストOSのdirent ABI互換は対象外とする。

## 2. 公開ABI

```cpp
int32_t sys_getdents(int32_t fd, void* buf, uint32_t count);
```

- 成功時は`buf`へ格納したバイト数を返す。終端時は`0`を返す。
- 失敗時は`-1`を返す。現在のsyscall ABIが`-1`のみを公開するため、Filesystemの詳細statusをユーザーABIへ追加しない。
- 1レコードは`fs::DIRENT_SIZE`(32 bytes)のdirent wire encodingとする。先頭4 bytesはinode番号LE、続く28 bytesはNUL終端name領域。メモリ上のC++構造体layoutをABIとして使わない。
- 返却レコードはこの順序でオンディスクから読み出した有効entryである。`inode_num == INODE_NUM_NONE`の空slotは返さない。`.`と`..`は通常の有効entryとして返す。
- 名前は不透明なNUL終端byte列。locale変換、UTF検査、pathname加工をしない。

## 3. syscall処理

処理順:

1. `fd`が有効で、現在Processの`FdTable`からACTIVEな`FileDescription`を取得できることを確認する。失敗は`-1`。
2. `count == 0`なら、bufferがnullでも`0`を返す。fdの有効性はこの前に検査する。
3. `count > 0`では`buf != nullptr`、`count <= INT32_MAX`、`buf + count`の範囲にoverflowがないこと、出力範囲が書込み可能であることを状態変更前に検証する。不正範囲は`-1`でoffset不変とする。
4. `count < fs::DIRENT_SIZE`なら`-1`を返し、offset不変とする。
5. `FileDescription::getdents()`へ委譲する。

ユーザーbufferの検証・アクセスはsyscall境界の責務である。検証できないbufferへ直接書き込んではならない。書込み可能性を事前に確定できない環境では、既存の安全なuser-copy経路を使う。存在しないuser-copy機構を作る場合は別途仕様を整合させる。

## 4. FD層・FileInterface連携

`FileDescription`に`getdents(void* buf, uint32_t count)`を追加し、read可否と有効targetを検査した後、I/O対象へ委譲する。`getdents`はoffsetをFileDescriptionに追加せず、通常read/seekと同じI/O対象のpositionを共有する。

許可済みの限定virtual boundaryである`FileInterface`に、専用`getdents()` hookを1つ追加する。通常ファイル以外の対象(`ConsoleDevice`、`Pipe`等)は既定で失敗する。RTTIや`dynamic_cast`による対象判別は禁止する。`RegularFile`のみがhookを実装する。

`RegularFile::getdents()`はFilesystem公開APIの`stat()`を使い、対象inodeがdirectoryであることと現在のsizeを確認する。directoryでなければ`-1`、通常ファイルであっても`-1`とし、FD層の対象誤用を成功扱いしない。inode tableやFATへ直接アクセスしない。sizeが`DIRENT_SIZE`の倍数でなければ破損扱いとする。

## 5. レコード走査とoffset

- `RegularFile`の既存`offset_`を走査位置にする。`FileSystem::open_file_t::offset`を別の正本にしない。
- 呼出し開始時のoffsetが`DIRENT_SIZE`の倍数でなければ`-1`を返し、offset不変とする。これは通常`read()`または`lseek()`で作られた不整列位置の検出である。
- 最大出力容量は`floor(count / DIRENT_SIZE) * DIRENT_SIZE`とする。余った1〜31 bytesは使用しない。
- `offset_ < inode.size`の間、各slotをraw `read_at(inode, offset_, raw, DIRENT_SIZE)`で読み、`dirent_decode()`する。完全な1レコードに満たない成功結果は`ECORRUPT`相当の失敗とする。
- Filesystem read error、statusが`OK`でない結果、または完全な1 record未満の結果では現在slotを消費せず停止する。成功済みrecordが0件なら`-1`、1件以上なら既出record分のbyte数を返す。ユーザーABIに部分成功件数を返した場合、後続errorはその呼出しでは返さない。
- decode成功後、slot位置を越えるようoffsetを32 bytes進める。空slotは出力せず走査を続ける。有効entryはraw serialized bytesをそのままbufferへコピーし、出力byte数を32増やす。
- 出力容量が満杯になった時点で停止する。出力後に余分なslotを先読みしてoffsetを進めない。
- directory sizeまで走査し、出力entryがなければ`0`を返す。EOFに達するまで読み飛ばした空slot分のoffset進行は保持する。
- `offset_`の加算前に32 bytes分の表現範囲とinode size内の位置を検証する。overflowまたはsizeとの矛盾は破損扱いで停止し、未処理位置を越えて進めない。

例: slot列が`[., .., a, empty, b]`で、出力容量が2 recordなら最初の呼出しは`.`と`..`を返して3番目slotの直後で止まる。次の呼出しは`a`を返し、empty slotをoffset上で消費してから`b`を返す。

## 6. offset共有と通常`read()`との関係

- 同じ`FileDescription`を共有するfd (`dup()`や`spawn()`継承を含む)は同じ`RegularFile::offset_`を使う。
- `getdents()`は有効entryを返すが、offsetはraw directory slot上の位置を示す。空slotのスキップもoffsetへ反映する。
- directoryに対する通常`read()`は既定どおりraw byte列を返し、`getdents()`とoffsetを共有する。そのためpartial `read()`後の不整列offsetでは`getdents()`が失敗する。`lseek()`で32-byte境界へ戻せる。
- directoryへの`write()`は既定どおり拒否する。
- 並行read/getdents/seekはFilesystemおよびFD既定の直列化規則に従う。操作の途中で同じoffsetを並行更新しない。

## 7. エラーと境界

現行syscall ABIでは以下はいずれも`-1`となる。errnoの新設やグローバルerrnoの追加はしない。

| 条件 | 内部の扱い |
|---|---|
| 無効fd | FD層エラー |
| read不可のopen flags | `FileDescription`のread拒否 |
| null/範囲外/書込み不能buffer | syscall引数エラー。offset不変 |
| countが1〜31 bytes | `EINVAL`相当。offset不変 |
| 有効fdがdirectoryでない、または非filesystem target | `ENOTDIR`相当の失敗 |
| FS未mount / I/O失敗 | `ENODEV` / `EIO`相当の失敗 |
| 不正dirent、chain破損、sizeとdataの矛盾 | `ECORRUPT`相当の失敗 |
| error以前に1件以上を返却済み | 返却済みbyte数。次の呼出しで未処理位置から再開 |
| 正常な終端 | `0` |

FS statusはFilesystem内部で維持し、syscall handlerは既定のsyscall失敗規約へ変換する。部分成功時に失敗statusを成功済みbyte数で覆い隠す規則はこのsyscallの戻り値契約であり、FS statusそのものを書き換える意味ではない。

## 8. 実装完了条件

- regular file、console等への`getdents`を拒否し、directory fdからのみentryを返す。
- `.`/`..`を含む有効entryを返し、空slotを出力しない。name bytesとwire byte orderを保つ。
- `count`が0、1〜31、32、複数record+余剰byteの場合を検証する。
- EOF、空directory以外の空slot群、複数回read、offset不整列、seek後の再開を検証する。
- `read()` / `getdents()` / `dup()`共有のoffset挙動を確認する。
- invalid fd、read access拒否、user buffer不正、I/O故障、壊れたdirent、部分成功後の失敗で戻り値とoffsetが規則どおりである。
- `FileDescription`経由のread flag検査と`FileInterface`専用hookを使い、syscallから`RegularFile`の私有状態やFilesystem内部層へ直接侵入しない。

## 9. 関連仕様

- 全体syscall一覧とユーザー向け引数・戻り値: `spec/subset_os_overview_spec.md`
- directory raw readとdirent wire format: `spec/filesystem/filesystem_external-api_spec.md` §10、`spec/filesystem/fs_impl_spec/fs_impl_spec_06-directory.md`
- `FileDescription` / `FileInterface` / `RegularFile`: `spec/kernel/open_file_description_spec.md`、`spec/kernel/open_close_implementation_spec.md`
- syscall入口とABI: `spec/kernel/syscall_entry_spec.md`、`spec/kernel/syscall_generator_spec.md`
