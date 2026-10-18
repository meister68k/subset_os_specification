# 01 — シリアライズ

> 共通のLE固定・直接キャスト禁止規約は`AGENTS.md`を上位仕様とする。

## 1. 責務

- メモリ上の値とオンディスクのバイト列を相互変換する。
- ホストのエンディアン、アラインメント、構造体パディングをディスク形式から隔離する。
- superblock、inode、dirent、FATエントリを同一方針で扱う。

## 2. レイヤー

```text
byteswap<T>() / LE整数プリミティブ
├── buf_writer / buf_reader        本体・mkfs用
└── binary_out / binary_in         ホストテスト専用
```

- `byteswap<T>()`はiostream非依存の純粋関数とする。
- `buf_writer`/`buf_reader`は`uint8_t[]`を対象とする。
- 本体用処理は仮想関数、streambuf、localeを使用しない。
- `binary_out`/`binary_in`はダンプ・比較用であり、本体へリンクしない。

## 3. 必須API

```cpp
void superblock_encode(const superblock_t& in, uint8_t* out);
FsStatus superblock_decode(const uint8_t* in, superblock_t* out);

void inode_encode(const disk_inode_t& in, uint8_t* out);
FsStatus inode_decode(const uint8_t* in, disk_inode_t* out);

FsStatus dirent_encode(const dirent_t& in, uint8_t* out);
FsStatus dirent_decode(const uint8_t* in, dirent_t* out);
```

- 実際の宣言は既存コードのconst指定と引数順に合わせてよい。
- decodeは構文上不正な表現を検出した場合`ECORRUPT`を返す。
- `dirent_encode()`は名前領域内にNUL終端がない場合`ENAMETOOLONG`を返す。
- 上位層にしか判断できない意味検証は上位層へ残す。
- 失敗時に半端な出力値を利用可能な状態にしない。

## 4. 整数処理

- 16/32/64bit値はバイト単位でLE変換する。
- `memcpy`を使う場合も、ネイティブ値をそのまま永続化してはならない。
- 64bit時刻領域は下位32bit値をLEで書き、上位32bitを0にする。
- FAT値は2バイトLEとして読み書きする。
- バッファ残量不足を事前に検出する。

## 5. 構造別契約

### superblock

- `00-overview.md`記載順で各フィールドを符号なし32bit LEとして配置する。
- 未使用領域はゼロで埋める。
- encodeは動的な`mount_state_t`を一切含めない。

### inode

- `mode`、`nlink`、`uid`、`gid`は16bit LEとする。
- `size`と`first_cluster`は32bit LEとする。
- 各時刻はディスク上8バイトを消費する。
- メモリ上の自然アラインメントとディスク上のオフセットを混同しない。

### dirent

- `inode_num`を先頭4バイトへLEで配置する。
- 続けて`name[28]`を配置する。
- 名前領域は、最初のNULより前にある0〜27バイトの不透明なバイト列として扱う。
- シリアライズ層は文字コード、空名、`/`の有無を意味検証しない。
- encode前に名前領域内の28バイト以内にNUL終端があることを確認する。
- decode後の名前領域は必ず安全に終端されていることを確認する。
- 名前領域内にNULがないdecode入力は`ECORRUPT`とする。
- 空名はシリアライズ表現として許容する。使用中direntの空名を拒否する責務はdirectory層に置く。

## 6. 完了条件

- 各構造について既知バイト列とのencode比較が通る。
- 各構造について既知バイト列からのdecodeが通る。
- encode→decodeのround-tripが通る。
- 既知LEバイト列のテストに加え、ネイティブ値の`memcpy`等へ依存していないことをコードレビューで確認する。
- 非アライン位置のバイト列でも未定義動作やfaultを起こさない。
- 予約領域が常に0である。
