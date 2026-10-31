# 極小UNIX `ls` 仕様書

| 項目 | 内容 |
|---|---|
| 文書ID | SUBSET-LS-001 |
| 版 | 0.3（`-l`詳細表示・`-p` directory suffix） |
| 対象 | サブセットOSの `/bin/ls` |
| 位置づけ | UNIX V7に着想を得た、単純なディレクトリ列挙コマンド |
| 依拠する仕様 | `getdents_implementation_spec.md`（dirent wire record）、`filesystem_external-api_spec.md`、`filesystem_design_spec.md`、`stat_implementation_spec.md`、ilibc APIリファレンス |

## 1. 目的と範囲

指定したディレクトリ内の名前、または指定した通常ファイル自身の名前を標準出力へ列挙する。出力は常に1エントリ1行とし、端末幅に応じた多段組やソートは行わない。`-l`で属性を含む詳細表示、`-p`でdirectory名への`/`付加を行う。ディレクトリ列挙と`stat()`を検証できることを重視する。

本書はユーザランドコマンドの仕様であり、OS・ファイルシステム側の公開APIの新設は要求しない。ただしディレクトリに対する`open/read`と`stat`のユーザ向けシステムコール経路が使用可能であることが前提となる。

## 2. 起動形式とオプション

```text
ls [-alp] [path ...]
```

- 引数なしならカレントディレクトリ `.` を表示する。
- `path` はファイルまたはディレクトリを指定できる。絶対パス、相対パスを許容する。
- `-a` は名前が `.` で始まるエントリも表示する。指定がなければ隠しエントリを省略する。
- `-l` は各entryのmode、link数、UID、GID、size、更新時刻を含む詳細形式で表示する。
- `-p` はdirectory entry名の末尾に `/` を付ける。directoryでないentryには付けない。
- `-a`、`-l`、`-p`は併用できる。`-alp`のような短いoptionの結合、および`-a -l -p`のような分割指定を許容する。optionの重複は無視する。
- 未知のoption文字を含むoption tokenはエラーとする。引数全体を検査してから出力を開始する。
- `--`、`--help`、`--version` は非対応。`-`で始まるpathを指定するには`./-name`のように記述する。単独の`-`もoption tokenとして扱い、未対応option errorとする。
- `-R`（再帰）、`-d`、`-i`、`-t`、`-r`等は非対応。
- オプションはパス引数の前にのみ許容する。最初のパス引数以後は、後続をすべてパスとして解釈する。
- ワイルドカード、クォート、環境変数は展開しない。シェルが生成した `argv` をそのまま利用する。

## 3. 出力形式・表示順序

- 通常形式ではエントリ名をASCII/バイト列として1行に出力する。詳細形式ではfield列の後にエントリ名を出力する。各行の末尾にはLF (`\n`) を1つ付加する。
- 表示順序は**ディレクトリ内部の記録順（物理dirent順）**とし、辞書順ソートを行わない。
- 通常形式は名前だけを表示する。`-l`時は詳細形式を使い、`-p`時はdirectory名に`/`を付ける。
- `inode_num == 0` の空きスロットは表示しない。
- `-a` なしの場合、`name[0] == '.'` のエントリは表示しない。`-a` ありなら `.` および `..` もディレクトリに実在する場合に限り表示する。仮想的には生成しない。
- ファイルを直接指定した場合、通常形式はそのパスを1行で表示する（例：`ls /tmp/a` → `/tmp/a\n`）。詳細形式ではそのpathを対象に取得した`stat`情報を前置し、pathを表示名とする。直接指定には隠し名除外規則を適用しない。`-p`は直接指定した通常fileの名前にsuffixを付けない。
- 単一ディレクトリの内容では、ディレクトリ自身の名前や見出しを表示しない。
- 複数の`path`を指定したときは引数順に処理し、**各ディレクトリの先頭に `path:` の見出し**を付ける。対象間は空行1行で区切る。通常ファイルは自身の指定パス1行のみを表示する。順序のグルーピングや並べ替えは行わない。
- 文字コード変換・特殊文字エスケープは行わない。ファイル名はOSが保証するNUL終端の名前を使う。

### 3.1 動作例（記録順は例示）

```text
$ ls
bin
etc
readme
$ ls -a
.
..
bin
etc
.hidden
readme
$ ls /etc/inittab
/etc/inittab
$ ls -p
bin/
etc/
readme
$ ls -l
drwxrwxrwx 2 0 0 512 0 bin
drwxrwxrwx 2 0 0 512 0 etc
-rw-r--r-- 1 0 0 23 0 readme
```

詳細形式のfield順は`mode link-count uid gid size mtime name`とする。`mtime`は`stat.st_mtime`のUNIX epoch秒を10進数で表示する。時刻文字列化やtimezone変換は行わない。modeはfile type (`d` directory、`-` regular file)とowner/group/otherの`rwx` bitを10文字で表示する。UID/GIDは名前解決せず10進数とする。

`-p`のslashは表示上のsuffixであり、pathnameやFilesystem entryを変更しない。`-l`と併用した場合はlong rowのname field末尾へ付加する。

## 4. ディレクトリ取得方法：`getdents()`

ディレクトリ列挙には`getdents()`（syscall番号66、`getdents_implementation_spec.md`）を使う。0.1のraw dirent直接読み出し方式は変更した。

| 項目 | 値・契約 |
|---|---|
| 取得API | `getdents(fd, rec, 32)`（1レコード単位） |
| `DIRENT_SIZE` | 32バイト |
| 1レコードのwire形式 | 先頭4バイトがinode番号（リトルエンディアン `uint32_t`）、続く28バイトがNUL終端name |
| 空きエントリ | カーネル側で除外済み（`ls`では判別・除外の必要なし） |
| 名前の終端NUL | カーネル側で保証済み（不正direntは`getdents()`が拒否する） |
| 終端 | `getdents()`の戻り値`0` |
| 実行環境 | Cortex-M0 / MC68000 共通でオンディスク表現はLE固定（デコードはカーネル側） |

- `ls`はレコードのname領域（オフセット4から28バイト）をバイト列として直接使う。inode番号は基本表示に使わないため`ls`側でデコードしない。
- 生データを`struct dirent*`へ直接キャストしない。カーネルがwire形式の正当性（LE inode、name NUL、空slot除外）を保証する。
- ユーザーランドから`fs::dirent_decode()`をリンク・参照しない。on-disk形式のカーネル内部ヘッダもユーザランドへ公開しない。
- `getdents()`が返すレコードは有効entryのみであるため、空slotスキップや不完全direntの復元は`ls`側で行わない。
- `getdents()<0`はエラーとして扱う。バッファ境界を越える読み込みや同期化のためのseekは行わない。
- **前提条件:** OS側がdirectory fdに対する`getdents()`を許容すること（本OSでは実装済み、`getdents_implementation_spec.md` §8）。

## 5. 対象パスの判定

1. 処理対象の`path`へ `stat(path, &st)` を行う。
2. `S_ISDIR(st.st_mode)` ならディレクトリとして `open(path, O_RDONLY, ...)`、続いて`getdents()`でdirentを1レコードずつ列挙する。
3. ディレクトリ以外のファイルは、指定された`path`文字列を1行だけ出力する（このため開く必要はない）。
4. `stat()` が失敗した場合、診断を出してその対象をスキップする。
5. 通常形式で`-p`がない場合、列挙中のentryごとに`stat()`は行わず名前だけを出力する。
6. `-l`または`-p`が指定されている場合、各directory entryの完全なpathnameを組み立てて`stat()`し、種別・long metadataを取得する。`-l`と`-p`を併用する場合は同じ`stat()`結果を使う。
7. entry pathの構築は固定長`PATH_MAX` buffer内で行う。長さ超過またはentry `stat()`失敗は診断してエラーを記録する。`-l`ではmetadataのないrowを出さずそのentryをskipする。`-p`のみではslashの有無を判断できないため、suffixなしの名前を出力する。

`S_ISDIR`および`struct stat`はOS側の`<sys/stat.h>`（UNIX互換の公開ABI）を用いる。`open()`はOS仕様の実際の3引数形式 `open(path, flags, mode)` に合わせる。読取専用では第3引数は0とする。

`-l`のmodeは`st_mode`からfile typeと`0777` permission bitを抽出する。表示形式は`d`または`-`に続き、owner/group/otherの各`rwx`三文字を連結した10文字とする。link count、UID、GID、size、mtimeはそれぞれ`st_nlink`、`st_uid`、`st_gid`、`st_size`、`st_mtime`を10進表記する。所有者名・group名、ACL、symlink情報は扱わない。

## 6. エラー処理・終了ステータス

| 状況 | 動作 |
|---|---|
| 不明なオプション | fd=2へ診断し、列挙開始前に終了ステータス1 |
| `stat()` 失敗 | エラー表示、そのパスをスキップし次へ |
| `-l`時のentry `stat()`失敗 | エラー表示、そのentry rowをskipし列挙を継続 |
| `-p`時のentry `stat()`失敗 | エラー表示、suffixなしのentry名を表示し列挙を継続 |
| entry pathnameが`PATH_MAX`を超える | entry `stat()`失敗と同様に扱い、エラーを記録 |
| `open()` 失敗 | エラー表示、そのディレクトリをスキップし次へ |
| `getdents()` 失敗 | エラー表示、当該ディレクトリの列挙を中止し次のパスへ |
| `close()` 失敗 | 診断表示し失敗を記録、次へ |
| stdoutの `write()` が0または負値 | 診断し、コマンド全体を中断 |
| stdoutへの正の部分書き込み | 未送信分を繰り返し書く |
| エラーが一件もない | 終了ステータス0 |
| 何らかのエラーがあった | 終了ステータス1 |

- 診断はfd=2へ出力し、行頭には`ls:`を付ける。正確なエラー文言・errno数字の表示は実装依存。
- 診断自体が失敗しても再帰的にエラー処理しない。
- `ls`が開いたディレクトリFDは、エラー時を含め必ず閉じる。fd=0/1/2は閉じない。
- 複数パスの中でエラーが起きても、書き込み失敗以外は後続のパスを処理する。

## 7. 利用API・実装制約

| 区分 | API | 用途 |
|---|---|---|
| OSラッパ | `stat()` | パスの種類を判定 |
| OSラッパ | `open()`、`getdents()`、`close()` | ディレクトリ読み込み |
| OSラッパ | `write()` | stdoutと診断出力 |
| ilibc | `std::strlen()` | 名前・パスの文字列長 |
| ilibc | `std::strcmp()`等 | `-a`、`-l`、`-p`のoption判定 |
| ローカル処理 | mode/整数変換 | permission文字列と10進fieldの固定buffer生成 |
| ローカル処理 | name領域の切り出し | レコード先頭4バイトのinode分を除いたnameバイト列を使う |

- `FILE*`、`malloc/free`、`printf()`、浮動小数点、RTTI、例外を使用しない。
- direntは32-byte bufferで1件ずつ処理する。entry pathnameと表示rowには`PATH_MAX`/固定長row bufferを使い、ディレクトリ全件をメモリに蓄積しない。
- `-l`のrow bufferはNULとLFを含め208 byte以上とし、書き込み前に構築長を検査する。10進変換とmode生成は固定長で行い、`printf()`や動的メモリ確保を使わない。
- `stat` とdirentレコードの公開ヘッダ定義は、OS本体のABIとの一致を優先する。ホストPOSIXで同じバイナリをそのまま動作させることは想定しない。
- `getdents()`を経由するため、`ls`はFSオンディスク形式を直接参照しない。列挙APIの抽象化はカーネルの`getdents()`境界が担う。

## 8. 受入テスト

1. 引数なしでカレントディレクトリの非隠しエントリを1行ずつ表示する。
2. `-a`指定で隠しエントリを表示する。存在する `.` / `..` を表示し、存在しない場合は生成しない。
3. 列挙順が実ディレクトリの記録順であることを確認する。
4. 空きslot（inode=0）を表示しない。
5. 単独の通常ファイル指定では指定パスそのものを表示する。
6. 複数の対象を引数順に処理し、ディレクトリには見出しを付ける。
7. `stat/open/getdents` 失敗時に診断し、後続の対象を処理する。
8. 複数レコードを複数回の`getdents()`呼出しでも欠落なく列挙する。
9. 空slot非表示・不正dirent拒否・name NUL終端はカーネル`getdents()`の契約であり、`ls`は返却レコードのnameバイト列をそのまま表示する。MC68000でも構造体キャストをしないことを確認する。
10. stdoutの部分writeを再送し、write失敗時に停止する。
11. 開いたFDを必ず閉じ、既存の標準FDを閉じない。
12. 存在しないパスを混在させた場合、列挙可能な後続パスは表示し、戻り値1とする。
13. `-l`で各entryのmode、nlink、uid、gid、size、mtime、nameを指定field順で表示する。directoryとregular fileのtype/permission文字列を確認する。
14. `-p`でdirectoryだけに`/`を付け、通常fileには付けない。`-ap`、`-lp`、`-alp`および分割optionを同じ意味で処理する。
15. `-l`または`-p`でentry `stat()`が失敗したとき、定義したrow skip/name-only処理と最終status 1を確認する。
16. entry pathの`PATH_MAX`境界・超過、最大長name、最大値fieldのrow生成でbuffer越境しない。

## 9. 補足

1. **option:** `-a`、`-l`、`-p`を採用した。ソートや再帰表示は引き続き対象外。
2. 複数パスの見出し・空行区切りは本書の規則に従う。
3. **ディレクトリ形式の公開:** `getdents()`方式 (0.2) により、on-disk形式をユーザランドへ公開する必要はなくなった。wire recordの境界は`getdents_implementation_spec.md`が正本である。
4. `-l`と`-p`でentryごとの`stat()`を使う。`getdents()`のwire record、`stat()`のABIがそれぞれの取得仕様の正本である。
5. **表示順:** ソートを省くため、出力順序はFSの内部配置に依存する。これは初版の意図的な簡略化である。
