# 独自ファイルシステム実装プロジェクト 設計仕様書

> **各層仕様同期版**: `00-overview.md`〜`12-rename.md`のphase 1契約を反映する。共通規約はルートの`AGENTS.md`、各レイヤーの詳細な契約・完了条件は各層仕様書を参照する。本書のAPI例・手順も同じ契約に従い、実装や検証の完了を示すものではない。

UNIX V7風の自作OS(サンデープロジェクト)における、ファイルシステム単体実装の設計まとめ。
**実装言語は C++ とする**。ただし、Cortex-M0/68000への移植性とコードサイズを優先し、C++機能には4.4節の制限を設ける。

---

## 1. プロジェクトの背景・最終目標

- UNIX V7(1979年)の主要システムコールのうち、B-shell が「パイプとリダイレクトが効くレベルでそれっぽく動く」(Lv2)のに必要な約20個の syscall サブセットを、シングルユーザー限定で実装するのが最終目標。
  - 例: fork, execve, exit, wait, open, close, read, write, creat, pipe, dup, lseek, unlink, chdir, getpid, signal, kill, times, stat/access など。`umask`はphase 1では実装しない。
- カーネル全体は将来的にホスト上のプロセスとして模擬実装し(トラップの代わりに関数呼び出し、プロセスはスレッド/ucontextで模擬、メモリ管理は当面malloc任せでスワップ不要)、後で実機/エミュレータへの移植を検討する。
- **実機ターゲットは Cortex-M0 または 68000。** そのため C/C++ 実装は「アラインメント制約」「エンディアン差異」「コードサイズ制約」を最初から意識する。
- 今回はカーネル全体ではなく、**「ファイルシステム部分」だけを先に単体実装**する。理由: 設計の自由度が高く面白いこと、ディスクイメージという1ファイルに閉じてテストしやすいこと。

---

## 2. ファイルシステムの基本設計方針

### 2.1 ブロック管理: FAT方式

- レイアウト: `[予約セクタ群] [スーパーブロック] [FATテーブル] [データブロック領域]`。ファイルシステムと無関係な予約セクタを先頭に複数置いてよい(詳細は3.1節)。
- `FAT[n] = 次のクラスタ番号`(0=空き、`FAT_EOF`=チェーン終端)。空き領域管理もFATテーブルが兼ねる。
- FAT12のビットパッキングは避け、**FAT16相当(uint16配列)** にする(実装の単純さ優先)。
- クラスタサイズ = ブロックサイズ = **512バイト**(MS-DOSに倣う)、1:1対応。
- V7方式のbmap(直接/間接/二重間接ブロック)より大幅に単純になることを理由に採用。

### 2.2 メタデータ: Unix流(inodeとディレクトリエントリの分離)

- ディレクトリエントリは `{名前, inode番号}` のみ。サイズや属性は一切持たない。
- 理由: 名前とメタデータの分離により、**ハードリンク(link(2))が自然に実装できる**(複数のディレクトリエントリが同じinode番号を指せる)。unlinkは「エントリ削除→`drop_link()`でリンク数減算→`nlink == 0`かつ非openならfinalize」という処理になる。phase 1ではクラスタとinode slotを再利用しない。

### 2.3 inodeテーブルの持ち方(採用方針: 可変長・チェーン方式)

- **inodeテーブル自体を「クラスタ0から始まるFATチェーンを持つ特殊ファイル」として実装する。**
- 固定長配列(当初のAI提案)は不採用。理由: mkfs時に決めた上限を超えると詰む(イメージ再作成しかない)という制約が、運用上受け入れがたいというユーザーの意向を採用。
- テーブル拡張(新規inode割り当てで容量不足時)は、通常ファイルのクラスタ拡張と**全く同じ経路**(`chain_extend`)を使う。コード重複を避ける。
- 循環参照の回避: inode番号→データ位置の変換にはinodeテーブルの中身を見る必要があるが、inodeテーブル自身の位置をinode経由で引くことはできない。これは**スーパーブロックがinodeテーブルの先頭クラスタを直接持つ**特殊ケースとして割り切る。
- inode番号0は未使用の番兵として予約。**ルートディレクトリはinode番号1固定**。

### 2.4 既知のトレードオフ

- lseekのランダムアクセスがFATチェーンの先頭からの走査になり、理論上O(n)(V7のbmapはO(1)に近い)。Lv2で扱うファイルサイズは小さいため、当面はキャッシュなしの素朴な実装で実害なしと判断。将来「open中ファイルごとに最後に訪れたクラスタ位置をキャッシュする」最適化を追加できる(MS-DOS系ドライバと同様の手法)。
- メタデータ(inode)とブロック管理(FAT)という2つの独立した構造の整合性維持がバグの温床になりやすい。→ 3.5節「空き領域回収の段階的方針」でこの発生機会自体を最小化する設計にしている。

### 2.5 類似の既存実装(参考・反面教師)

- Linuxの **UMSDOS**(1992年〜、Jacques Gelinas作)がほぼ同じコンセプト(FATでブロック管理+隠しファイルでUnix風メタデータを保持)。ただしDOS FATフォーマット完全互換を律儀に守る必要があり負担になっていた。メンテナンス不足で2.6.11カーネルで削除。
- 今回の独自FSはディスクフォーマットの外部互換性を一切気にしなくてよいので、UMSDOSが背負っていた負担(DOS側ツールとの相互運用性)は不要。FATとUnix流メタデータという「アイデア」だけを借りる。

---

## 3. ディスク上のレイアウト仕様

### 3.1 全体レイアウト

```
[ブロック0..fs_start_block-1 : 予約領域(ファイルシステム仕様外)]
[ブロックfs_start_block    : スーパーブロック]
[ブロックfat_start_block..fat_start_block+fat_blocks-1 : FATテーブル]
[ブロックdata_start..                     : データ領域(クラスタ0..data_blocks-1)]
```

データ領域のクラスタ0は、mkfs時点で必ずinodeテーブルの先頭クラスタとして割り当てられる(`inode_table_start_cluster == 0` 固定)。

#### 3.1.1 先頭予約セクタ(MBR等)

ファイルシステム領域の先頭に、**ファイルシステムと無関係な予約セクタを0個以上置いてよい**。典型例としてMS-DOS形式のMBRを1セクタ置くことができる。予約セクタの内容・形式・用途はファイルシステム仕様の範囲外とする。ファイルシステムは、呼び出し側から与えられた`fs_start_block`を起点として自身の領域を解釈する。

予約セクタの具体的なフォーマットはファイルシステム仕様では規定しない。MBRを置く場合の内容は、使用するブートローダ/パーティション管理方式の仕様に従う。

**ファイルシステム開始位置**は`mount()`の引数として外部から与える。`fs_start_block`をスーパーブロック自身から取得する方式にはせず、スーパーブロックを読むための位置情報をファイルシステム外部に置く。


### 3.2 基本パラメータ

| 項目 | 値 |
|---|---|
| ブロック/クラスタサイズ | 512バイト固定 |
| FATエントリ幅 | uint16_t(FAT16相当) |
| FAT特殊値 | `FAT_FREE=0x0000`, `FAT_EOF=0xFFFF` |
| inode番号0 | 未使用番兵 |
| inode番号1 | ルートディレクトリ固定 |
| ディレクトリエントリ名前長 | 27バイト + NUL = 28バイト |
| inodeレコード長 | `INODE_SIZE=64`バイト(phase 1固定) |
| direntレコード長 | `DIRENT_SIZE=32`バイト |
| pathname上限 | `PATH_MAX=128`、`NAME_MAX=27`バイト |

フォーマット識別子`FS_MAGIC`は`00-overview.md`の定義を参照し、ホストのエンディアンに依存せずLEのディスク表現と比較する。

### 3.3 オンディスク構造体(フィールド定義)

> **注意(重要な方針転換、4節参照)**: 以下はディスク上のフィールド構成の「仕様」であり、Cの構造体を直接ディスクバッファへキャストするのではなく、**明示的な encode/decode(シリアライズ)関数経由でアクセスする**。理由はCortex-M0のアラインメント制約と68000のエンディアン差異のため(4節で詳述)。

**スーパーブロック(`fs_start_block`で指定されるブロック)**

> **方針転換(13節参照)**: スーパーブロックは**mkfs時に一度書いたら運用中は二度と書き換えない、不変な仕様情報のみ**を持つ。以前ここに含めていた「動的な帳簿」(次に割り当てるinode番号、bumpアロケータのヒント、inodeテーブルの末尾クラスタ/容量)は**ディスクには一切持たず**、mount時にFAT・inodeテーブルの実データから再構築するRAM専用の`mount_state`(13節)に分離した。これによりスーパーブロックのキャッシュ・dirty管理という問題自体が消滅する。

| フィールド | 型 | 意味 |
|---|---|---|
| magic | uint32 | フォーマット識別子 |
| block_size | uint32 | 512固定 |
| total_blocks | uint32 | イメージ全体のブロック数 |
| fs_start_block | uint32 | ファイルシステム開始ブロック(予約領域直後。mount時にも同じ値を指定する) |
| fat_start_block | uint32 | FAT領域開始ブロック(通常`fs_start_block + 1`) |
| fat_blocks | uint32 | FAT領域のブロック数 |
| data_start_block | uint32 | データ領域開始ブロック |
| data_blocks | uint32 | データ領域のクラスタ数(=FATエントリ数) |
| inode_size | uint32 | 1inodeレコードのバイト数(phase 1は64固定) |
| inode_table_start_cluster | uint32 | inodeテーブルの先頭クラスタ(mkfs時0固定) |
| root_inode | uint32 | ルートディレクトリのinode番号(1固定) |

(旧: `inode_table_last_cluster`, `next_free_inode`, `inode_table_capacity`, `next_free_cluster_hint` は`mount_state`へ移動。13節参照)

**inode(inodeテーブルというファイルの中の1レコード)**

| フィールド | 型 | 意味 |
|---|---|---|
| mode | uint16 | ファイルモード |
| nlink | uint16 | ハードリンク数 |
| uid | uint16 | ユーザーID(予約，現在は0固定) |
| gid | uint16 | グループID(予約，現在は0固定) |
| size | uint32 | 通常ファイルは論理長、directoryは確保済み容量(バイト) |
| atime | uint64領域 | 最終アクセス時刻。現在使用する値は下位32bitのunsigned UNIX time、上位32bitは予約(0)。現在はmtimeと同一値 |
| ctime | uint64領域 | inodeステータス変更時刻。現在使用する値は下位32bitのunsigned UNIX time、上位32bitは予約(0) |
| mtime | uint64領域 | 内容更新時刻。現在使用する値は下位32bitのunsigned UNIX time、上位32bitは予約(0) |
| first_cluster | uint32 | データ先頭クラスタ。`CLUSTER_NONE(0xFFFFFFFF)`で未割当 |

ファイルモードはMINIXに準じ，上位ビットから`[ファイル種別 4ビット][予約(0固定) 3ビット][アクセス許可 9ビット]`。
ファイル種別は 通常ファイル S_IFREG=0100000 ， ディレクトリ S_IFDIR=0040000。
使用中inodeのアクセス許可9ビットは`creat(mode)`で指定された値を保持する。phase 1では`umask`を適用しない。

時刻フィールドはディスク上では各8バイトを確保するが、フェーズ1では`uint32_t`のunsigned UNIX timeとして扱う。上位32bitは予約領域とし0にする。したがって2038年問題(符号付き32bit UNIX timeのオーバーフロー)は発生しない。将来64bit UNIX timeへ移行する場合は予約領域を利用できる。

`ctime`はUNIX/POSIXのinodeステータス変更時刻として扱う。`mtime`はファイル内容の更新時刻、`atime`は現在はmtimeと同一値とする簡略仕様とする。具体的な更新タイミングは16〜17節で定義する。
独自のファイル種別として 未使用 S_IFREE=0000000 ，削除済み S_IUNUSED=0170000。削除済みは一度使われたinodeの最終処理が完了した状態を示す。`nlink == 0`でもopen中ならliveのまま維持する。(13.4節・18.2節参照)

**ディレクトリエントリ**

> **フィールド順序の方針**: `inode_num`(固定長)を先頭、`name`を末尾に置く。理由: `name`は将来27バイトを超える可変長化の可能性があり、可変長部分を末尾に置くことで、固定長ヘッダ部分だけを見て次エントリへスキップする、といった処理と相性を良くしておく。

| フィールド | 型 | 意味 |
|---|---|---|
| inode_num | uint32 | `INODE_NUM_NONE` なら空きエントリ |
| name | char[28] | NUL終端の不透明なバイト列。空きエントリは `name[0]=='\0'` |

### 3.4 mkfsのブートストラップ手順

0. 予約セクタを使用する場合は、ファイルシステム外部の仕様に従って初期化する。MBR等を置かない場合、`fs_start_block`は0でもよい。
1. FAT領域をゼロ初期化(全クラスタ空き)。
2. クラスタ0を確保し `FAT[0] = FAT_EOF` にする(inodeテーブルの先頭かつ唯一のクラスタ)。
3. スーパーブロック(`fs_start_block`)に `inode_table_start_cluster = 0` を設定(不変フィールドのみ。旧`inode_table_last_cluster`/`inode_table_capacity`はmount時に再構築するため、mkfsでは書かない)。
4. inode番号0は番兵として飛ばし、inode番号1(ルートディレクトリ)を確保する(旧`next_free_inode`のディスク書き込みは廃止。同様にmount時再構築)。
5. inodeテーブルとルートdirectory data clusterを全ゼロで初期化する。ルートのinodeは`mode = S_IFDIR | 0777`、`nlink = 2`、`uid = gid = 0`、`size = 512`とし、`first_cluster`とprovider由来の時刻を設定する。先頭に`.`と`..`(ともにinode 1)をencodeし、残りのdirent slotは全ゼロとする。

phase 1のinodeレコード長は`INODE_SIZE = 64`、dirent長は`DIRENT_SIZE = 32`とする。メモリ上の`sizeof`をディスク上の長さとして使わない。

この手順により、以降の「ファイル作成」「inode取得」は全て同じ汎用コードパス(4節・5節)に乗る。

### 3.5 空き領域回収の段階的方針(フェーズ1→フェーズ2)

- **フェーズ1(当面の実装)**: bumpアロケータ的にFAT/inodeを進めるだけ。unlinkしても回収しない。壊れたら/溢れたらmkfsし直す運用。`fat_free_chain`・inodeの再利用ロジックは実質スタブ。
- **フェーズ2(将来のリファクタリング)**: ビットマップ的に本当に回収する方式へ移行。コスト差は数十〜100行程度と見積もる。
- クラスタとinodeの回収方針を**同じタイミング・同じ思想**で揃えることで、片方だけ凝ることによる非対称な複雑さを避ける。
- フェーズ1では「解放」という操作自体がほぼ存在しないため、inode/FATという2構造間の整合性を崩す機会そのものが最小化される。整合性検証(fsck相当)が本当に必要になるのはフェーズ2以降でよい。
- FAT12/16に加え、小規模マイコン用にFAT8も面白い、というアイデアあり(未着手・検討課題)。

---

## 4. シリアライズ方針(エンディアン・アラインメント対応)

### 4.1 方針転換の経緯

当初、`#pragma pack(1)` を付けたC構造体をディスクバッファへ直接キャストする案で開始したが、実機ターゲット(Cortex-M0 / 68000)確定を受けて不採用とした。理由:

| 懸念 | 内容 |
|---|---|
| アラインメント制約 | Cortex-M0(ARMv6-M)は非アラインアクセス機構を持たずHardFaultする。68000もword/long(2B以上)アクセスは偶数アドレス必須でAddress Errorになる |
| 標準準拠性 | `#pragma pack`は処理系拡張。C++の型パニング(reinterpret_castでの構造体↔バイト列変換)はstrict aliasing上未定義動作の領域 |
| エンディアン非対応 | Cortex-M0(通常LE)と68000(BE)の間でネイティブキャスト方式はディスクイメージ互換性を持てない |

### 4.2 採用方針: 明示的 encode/decode

- 構造体を直接キャストせず、**「構造体 ⇔ バイト列」を明示的な関数で変換**する(FatFs等の実績ある組み込みFAT実装と同じ手法)。
- メモリ上の構造体(`inode_t`等)はディスクレイアウトの制約から解放され、自然アラインメントの普通のC構造体としてよい(速い・単純)。ディスクレイアウトは「仕様書」(3.3節)として独立させる。
- **オンディスクのエンディアンはリトルエンディアンに固定**(Cortex-M0のネイティブに合わせる。開発ホスト(x86, LE)・M0ターゲットではエンディアン変換が恒等関数になりデバッグしやすい。68000側のみ変換コストを払う)。

### 4.3 レイヤー構成(プリミティブ層 / ターゲット層 / テスト層)

```
[プリミティブ層]  byteswap<T>()  ― 純粋関数、iostream非依存。ホスト・ターゲット共通
        │
        ├─→ [ターゲット/mkfs本体用]  buf_writer / buf_reader
        │      uint8_t buf[N] への直接書き込み。streambuf・仮想関数・ロケール一切なし
        │
        └─→ [テストハーネス専用]      binary_out / binary_in (std::ostream/istream連携)
               開発機上でのダンプ・比較・デバッグ表示にのみ使用。本体には組み込まない
```

- ユーザー提供のiostreamベースライブラリは**テストコード専用**と確認済み。本体(FS本体・mkfs等)には組み込まない。
- 移植時に修正が必要な点: `__int64`→`uint64_t`、`unsigned long`→`uint32_t`(環境によって幅が異なるため)、`__BIG_ENDIAN__`→GCC/Clang組み込みの`__BYTE_ORDER__`/`__ORDER_LITTLE_ENDIAN__`/`__ORDER_BIG_ENDIAN__`(newlibベアメタル環境では`<endian.h>`起源のマクロが無い場合がある)。
- 本体用`buf_writer`/`buf_reader`は「型名でエンディアンの意図を表現する」という見た目(`.little<uint32_t>(v)`等)を踏襲しつつ、中身は`memcpy`と整数演算のみで完結させ、iostream起因のコードサイズ増加を避ける。

シリアライズAPIは`01-serialization.md`に従う。

```cpp
// ディスク上の固定長表現とメモリ上の値を相互変換する。
void superblock_encode(const superblock_t& in, uint8_t* out);
FsStatus superblock_decode(const uint8_t* in, superblock_t* out);
void inode_encode(const disk_inode_t& in, uint8_t* out);
FsStatus inode_decode(const uint8_t* in, disk_inode_t* out);
FsStatus dirent_encode(const dirent_t& in, uint8_t* out);
FsStatus dirent_decode(const uint8_t* in, dirent_t* out);
```

decodeは不正な表現を`ECORRUPT`で拒否し、失敗した出力を利用可能な状態にしない。dirent encodeは28バイトの名前領域にNUL終端がなければ`ENAMETOOLONG`、decodeは`ECORRUPT`を返す。空名・`/`・文字コードの意味検証はシリアライズ層で行わず、使用中direntの空名や`/`はdirectory層が拒否する。予約領域はゼロでencodeし、時刻は下位32bitをLEで書き上位32bitを0とする。

### 4.4 C++機能制限(確定事項)

| 機能 | 方針 |
|---|---|
| 例外(try/catch/throw) | **禁止** |
| RTTI | **禁止** |
| テンプレート | **許可**(`if constexpr`等も可) |
| new/malloc(ユーザプロセスのヒープ、B種) | 未決定でよい(Lv2のsyscall一覧に`brk`/`sbrk`が含まれないため、現スコープ外) |
| malloc(ホスト上のmkfs/FSライブラリの一時バッファ) | 素直にlibc任せでよい(ホスト専用ツールのため実機制約なし) |

---

## 5. メモリ管理方針(A種/B種の切り分け)

「メモリ管理」を一括りにせず、性質の異なる2種類として扱う。

| | 対象 | 性質 | 方針 |
|---|---|---|---|
| **A. OS内部の資源管理** | バッファキャッシュのブロック、inodeキャッシュエントリ、プロセス管理構造体、オープンファイルテーブル等 | 種類・最大個数を設計時に見積もれる、可変長ではない | **固定サイズのプールアロケータを自作**(V7の`proc[]`/`inode[]`等の静的配列思想を踏襲)。O(1)・断片化なし・決定的 |
| **B. ユーザプロセスのヒープ** | 将来ユーザプログラムが`malloc(3)`を呼ぶ領域(`brk`/`sbrk`) | 汎用・可変長・予測不能 | **現スコープ外、未決定でよい**(Lv2のsyscall一覧に含まれない) |

A種を自前プールにする理由: 汎用mallocは断片化し、確保時間が非決定的になる。カーネル内部(正しさの検証がただでさえ大変な部分)にこの不確実性を持ち込むのは得策でない。固定サイズプールは空きリストの着脱のみで実装コストも低い(数十行程度)。

この方針は3層の「疑似デバイスドライバ」設計(6節)における`struct buf`の実体や、7節のFATブロックキャッシュにもそのまま適用される想定。

---

## 6. 疑似ブロックデバイスドライバ設計

### 6.1 モデルの選定: V7の`bdevsw`/`strategy`方式を模倣

MS-DOSのINT 13h(コード4=INPUT、コード8=OUTPUT)や、モダンLinuxの`submit_bio`と対比した上で、**V7の`d_strategy`+バッファキャッシュ(`bread`/`bwrite`)モデル**を模倣する方針とした。理由:

1. 将来「実機/エミュレータへの移植」時、V7の`d_strategy`インターフェースに寄せておくと対応関係が分かりやすい。
2. 「向きをフラグ、単一エントリポイント」の方が、将来の非同期化(スレッド/ucontext模擬との相性)の余地を残せる。
3. バッファキャッシュ層は今回省略し、同期・非バッファリングの`read`/`write`に単純化する(Lv2の必須要件ではなく後回しにできる最適化のため)。

### 6.2 実装方式: テンプレートクラス `BlockDevice<Backend>`(複数デバイス同時マウント対応)

**経緯**: 当初は「namespaceによる疑似シングルトン」(実質1インスタンス限定、リンク時にバックエンドを選ぶ)を検討したが、**SDカードとマイコン内蔵flash ROMを同時に(=別々の独立したファイルシステムとして)使う**想定が確定したため不採用。仮想関数(vtable)によるランタイムポリモーフィズムも選択肢だが、Cortex-M0のコードサイズ予算を優先し、**コンパイル時にバックエンド型を確定するテンプレートクラス**を採用する。

**バックエンド(デバイス固有の実処理)**: 継承・virtualを使わない、ただの構造体。

```cpp
struct DiskImageBackend {       // ホスト: ディスクイメージファイル
    using OpenArg = const char*;
    std::FILE* fp = nullptr;
    fs::FsStatus open(const char* path);
    fs::FsStatus read(uint32_t blkno, uint8_t* buf);
    fs::FsStatus write(uint32_t blkno, const uint8_t* buf);
    fs::FsStatus close();
};

struct SdCardBackend {          // 将来・実機: SDカード(SPI/SDIO経由)
    using OpenArg = SpiHandle*;
    // ...
};

struct InternalFlashBackend {   // 将来・実機: マイコン内蔵flash ROM
    using OpenArg = std::nullptr_t;
    // ...
};
```

**共通ラッパー(Layer 1、バックエンドに依らない共通ロジックのみ集約)**:

```cpp
template<typename Backend>
class BlockDevice {
public:
    FsStatus open(typename Backend::OpenArg arg, uint32_t total_blocks);
    FsStatus read(uint32_t blkno, uint8_t* buf);   // open状態→null→範囲の順で検査する
    FsStatus write(uint32_t blkno, const uint8_t* buf);
    FsStatus close();
private:
    Backend  backend_;
    bool     is_open_ = false;
    uint32_t total_blocks_ = 0;
};
```

**同時利用例**(狙い通りの用途):

```cpp
fs::BlockDevice<SdCardBackend>        sd_dev;
fs::BlockDevice<InternalFlashBackend> flash_dev;
sd_dev.open(&spi_handle, sd_total_blocks);
flash_dev.open(nullptr, flash_total_blocks);
```

各インスタンスが独立した`backend_`状態(fd/SPIハンドル等)を持つため状態の混線がない。vtableもテンプレートの動的部分もなく、`read`/`write`はほぼ確実にインライン化される。副次的に、同じBackend型を2インスタンス作れば「同種デバイス2枚差し」にも追加コストなしで対応できる。

### 6.3 Layer 1の契約

| 項目 | 内容 |
|---|---|
| 転送単位 | 常に1ブロック(512B)固定、部分転送なし |
| 同期性 | 同期・即時完了 |
| バッファ所有権 | 呼び出し側が確保。この層はコピーのみ(読込は`uint8_t*`、書込は`const uint8_t*`) |
| 境界チェック | `BlockDevice<Backend>`の共通ラッパー側で一元的に実施(`blkno >= total_blocks_`) |
| オープン/クローズ | マウント時に1回、アンマウント時に1回。呼び出しごとの開閉はしない(インスタンスメンバでfd/ハンドル相当を保持) |
| `blkno`の型 | `uint32_t`(512B×2^32 ≒ 2TB相当まで表現可) |

Layer 2(V7の`strategy`相当、向き分岐のみの薄い関数)は「将来の差し替えポイント」として概念上は残すが、実装上は`BlockDevice<Backend>::read`/`write`という対称な2メンバ関数で十分に代替されており、独立した関数として今すぐ用意する必要はない。将来バッファキャッシュ(`bread`/`bwrite`相当)を追加する場合は、`BlockDevice<Backend>`と`Filesystem<Backend>`内部の`Fat<Backend>`の間に挟むのが自然。

`02-block-device.md`に従い、状態遷移は次のように定義する。

- 二重openは`EBUSY`で拒否し、Backendを呼ばない。Backend open失敗時は閉状態・容量0を維持する。Backendは失敗時に自身の途中状態を後始末する。
- read/writeはopen状態、nullバッファ、block範囲の順に検査し、それぞれ`ENODEV`、`EINVAL`、`ERANGE`を返す。検査成功後だけBackendを呼び、そのstatusを返す。
- 未openのcloseは`ENODEV`で拒否する。open中のcloseはBackendを1回だけ呼び、成否にかかわらず閉状態・容量0へ移す。Backend closeは終端操作であり、失敗しても再試行しない。

### 6.4 エラー体系

FS内部の操作結果は**`FsStatus`に統一**する。`Status`と`FsError`は使用しない。`FsStatus`はエラーそのものではなくFS操作の結果ステータスを表すため、成功値として`FsStatus::OK`を含む。

```cpp
enum class FsStatus {
    OK,
    EIO,
    ERANGE,
    ENODEV,
    ENOSPC,
    ENOENT,
    EEXIST,
    EISDIR,
    EINVAL,
    ESPIPE,
    ENOTDIR,
    ENAMETOOLONG,
    EMLINK,
    ECORRUPT,
    EBUSY,
    EWP,
};
```

命名は`OK`およびUNIX errnoに対応する大文字形式とする。`ECORRUPT`はディスク上のFS構造が仕様に適合しない場合に用いる独自エラーである。`EBUSY`は二重openやBackend固有のbusy状態に使う。`EWP`はwrite-protected Backend向けに予約する。

単純な操作は`FsStatus`を返し、データ転送と結果ステータスを同時に返す操作は`std::pair<uint32_t, FsStatus>`を返す。syscall層で必要に応じてUNIX errnoへ変換する。

### 6.5 ターゲットデバイスの想定

- 開発時: ホストのディスクイメージファイル(疑似RAMディスク)。「遅延やDMA割り込みといった非同期性を考えなくてよい」という単純化として扱う。
- 実機: SDカードまたはフラッシュメモリ(HDDは荷が重い、FDDは論外)。
- **SDカードを最終ターゲットに選んだこと自体が設計上のメリット**: SDカードは内部コントローラがFTL相当(消去・ウェアレベリング)を吸収し、ホスト側にはHDDと同じ「セクタ番号指定の単純なブロックデバイス」として見える。生のNANDフラッシュを直接叩く場合と異なり、Layer 1をシンプルな対称read/writeのままにできる決定打になっている。

### 6.6 C++によるファイルシステム内部構成

ファイルシステム本体は、デバイスドライバ層だけでなく、**状態を保持する内部機能をクラスとして構成する**。ただし、すべてをクラス化するのではなく、状態を持つ資源管理単位をクラスとし、ディスク構造体・シリアライズ処理・小さな変換処理は`struct`/自由関数のままにする。

#### 6.6.1 基本方針

- `BlockDevice<Backend>`は6.2節の方針どおりテンプレートクラスとする。
- ファイルシステム全体を`Filesystem<Backend>`クラスとして表現する。
- `Filesystem<Backend>`は、以下の状態管理クラスを所有する。
  - `Fat<Backend>`: FATテーブル操作、FATキャッシュ、クラスタ割り当て
  - `InodeTable<Backend>`: inodeテーブルへのアクセス、inode割り当て
  - `OpenFileTable`: Filesystem側のopen handle管理。open中inodeの寿命(open-but-unlinked)を追跡する。OS全体の共有open file descriptionの正本は上位FD層の`FileDescription`とする
- ディレクトリ操作、pathname解決、通常ファイルのread/write等は、原則として`Filesystem<Backend>`の公開メンバ関数として提供する。これらは現段階では独立した常駐状態を持たないため、専用クラスを必須としない。
- `disk_inode_t`、`dirent_t`、`superblock_t`等のオンディスク構造は単純な`struct`として保持する。ディスク表現をC++オブジェクトの内部表現と混同しない。
- `inode_encode()`/`inode_decode()`等のシリアライズ処理、`byteswap()`等のプリミティブ処理は自由関数とする。
- 実装はすべて`namespace fs`内に置く(`fs/src/*.h`)。本書のクラス定義・メンバ宣言・擬似コード例は`namespace fs`を省略して示し、FSの外側から書き上げる例(6.2節のBackend定義・利用例、外部API仕様の利用例)のみ`fs::`で修飾する。

#### 6.6.2 所有関係

```text
Filesystem<Backend>
│
├─ BlockDevice<Backend> device_
│
├─ Fat<Backend> fat_
│    └─ device_への参照
│
├─ InodeTable<Backend> inode_table_
│    └─ fat_ / device_ / superblock情報への参照
│
├─ OpenFileTable open_files_
│
├─ superblock_t superblock_
└─ mount_state_t mount_state_
```

`Filesystem<Backend>`は各サブシステムの所有者となる。サブシステム間の依存関係はコンストラクタで参照を渡して明示する。グローバル変数やnamespace内のstatic状態に依存しない。

この構成により、次のようにファイルシステムごとの状態を完全に分離できる。

```cpp
template<typename Backend>
class Filesystem {
public:
    FsStatus mount(...);  // Backend引数・デバイス容量・fs_start_blockを外部から渡す
    FsStatus unmount();
    FsStatus sync();

    // ファイルシステム公開API
    FsStatus creat(...);
    FsStatus mkdir(...);
    FsStatus unlink(...);
    FsStatus link(...);
    FsStatus resolve_path(...);
    FsStatus resolve_parent(...);

    std::pair<uint32_t, FsStatus> read_at(...);
    std::pair<uint32_t, FsStatus> write_at(...);
    FsStatus lseek(...);

private:
    BlockDevice<Backend> device_;
    superblock_t         superblock_{};
    mount_state_t        mount_state_{};

    Fat<Backend>         fat_;
    InodeTable<Backend>  inode_table_;
    OpenFileTable        open_files_;
};
```

実際のコンストラクタでは`device_`を先に構築し、その参照を`fat_`等へ渡す。C++のメンバ初期化順序に従うため、宣言順も依存関係に合わせる。

#### 6.6.3 `Fat<Backend>`

FAT層は独立したクラスとする。特にFATキャッシュは明確な内部状態を持つため、`Filesystem`本体の単なる自由関数群にはしない。

```cpp
template<typename Backend>
class Fat {
public:
    FsStatus get(uint32_t cluster, uint16_t* out);
    FsStatus set(uint32_t cluster, uint16_t value);
    std::pair<uint32_t, FsStatus> alloc();
    FsStatus free_chain(uint32_t head_cluster);

    std::pair<uint32_t, FsStatus> walk_to_index(uint32_t head, uint32_t index);
    std::pair<uint32_t, FsStatus> extend(uint32_t tail_cluster);
    std::pair<uint32_t, FsStatus> find_tail(uint32_t head_cluster);

    FsStatus sync();

private:
    BlockDevice<Backend>& device_;
    const superblock_t& superblock_;
    mount_state_t& mount_state_;

    // FAT 1ブロックキャッシュ
    bool valid_ = false;
    uint32_t block_no_ = 0;
    bool dirty_ = false;
    uint8_t cache_[FS_BLOCK_SIZE]{};
};
```

`Filesystem::sync()`は`fat_.sync()`を呼び出す。FATキャッシュのdirty状態は`Filesystem`外部から直接操作させない。

#### 6.6.4 `InodeTable<Backend>`

inodeテーブルも独立した状態管理クラスとする。inodeテーブルがFATチェーンを利用するという構造は`InodeTable`内部に隠蔽する。

```cpp
template<typename Backend>
class InodeTable {
public:
    FsStatus get(uint32_t inode_num, disk_inode_t* out);
    FsStatus put(uint32_t inode_num, const disk_inode_t& in);
    // alloc()の返却形は既存宣言に合わせる(14.4節)。

    FsStatus inc_nlink(uint32_t inode_num);
    FsStatus dec_nlink(uint32_t inode_num);
    FsStatus finalize_inode(uint32_t inode_num);

private:
    BlockDevice<Backend>& device_;
    Fat<Backend>& fat_;
    const superblock_t& superblock_;
    mount_state_t& mount_state_;
};
```

`inode_num_to_byte_offset()`や`offset_to_cluster()`等の変換処理は、必要に応じてprivateメンバまたは小さな自由関数として実装する。外部の呼び出し側はinodeテーブルの物理配置を意識しない。

#### 6.6.5 `OpenFileTable`

`OpenFileTable`は、Filesystem側でopen中のinodeを追跡するRAM上の固定長handle表として保持する。既存のFilesystem単体APIとの互換性のため`open_file_t`には`offset`と`ref_count`を残すが、OSのFD共通基盤へ統合する経路では、これらを共有open file descriptionの正本として扱わない。OS全体の共有状態(file position、open flags、fd参照数)は上位の`FileDescription`が保持する。

```cpp
class OpenFileTable {
public:
    FsStatus allocate(...);
    FsStatus close(...);
    open_file_t* get(...);
    bool inode_is_open(uint32_t inode_num) const;

private:
    open_file_t files_[MAX_OPEN_FILES];
};
```

Filesystem単体APIでは従来どおり`dup_file()`が`open_file_t::ref_count`を増やし、`read()` / `write()` / `lseek()`が`open_file_t::offset`を利用してよい。一方、FD統合経路では次を守る。

- 1回の上位`open()`につき`Filesystem::open_file()`を1回だけ呼び、得た`open_file_t*`を1個の`RegularFile`が保持する。
- `dup()`や`spawn()`によるfd共有では`Filesystem::dup_file()`を呼ばず、同じ`FileDescription` / `RegularFile` / `open_file_t*`を共有する。
- file positionとfd参照数の正本は`FileDescription`とし、通常I/Oは`RegularFile`から`read_at()` / `write_at()`へ明示offsetを渡す。
- `FileDescription`の最後の参照が解放された時だけ、`RegularFile`が`Filesystem::close(open_file_t*)`を1回呼ぶ。
- open-but-unlinked判定では`OpenFileTable::inode_is_open()`を使用する。fdがdupされてもFS handleは増えないが、そのhandle自体が最後のfd closeまで存続するため判定は正しい。

プロセスごとのfdテーブルはプロセス層の責務であり、`OpenFileTable`には持ち込まない。クラス名`OpenFileTable`と型名`open_file_t`は既存API互換のため当面維持するが、FD統合時にはPOSIXのopen file descriptionそのものとは解釈しない。

#### 6.6.6 クラス化しないもの

以下は原則としてクラス化しない。

| 対象 | 実装形態 | 理由 |
|---|---|---|
| `disk_inode_t` | `struct` | オンディスク形式そのものを表現するため |
| `dirent_t` | `struct` | 同上 |
| `superblock_t` | `struct` | 同上 |
| `inode_encode/decode` | 自由関数 | 状態を持たないシリアライズ処理 |
| `dirent_encode/decode` | 自由関数 | 同上 |
| `byteswap`/`buf_reader`/`buf_writer` | 自由関数・軽量型 | 汎用プリミティブであり状態管理主体ではない |
| `resolve`系 | `Filesystem`のメンバ | pathname解決自体は永続状態を持たない |

#### 6.6.7 継承・virtualの扱い

- FS内部のサブシステム間で継承関係は設けない。
- `BlockDevice<Backend>`を含め、原則として**静的多相(テンプレート)**を使用する。
- `virtual`/vtableはCortex-M0向けコードサイズと実装の単純性を優先して採用しない。
- `Filesystem<Backend>`自身を基底クラス化する設計も行わない。

#### 6.6.8 公開APIと内部APIの境界

`Filesystem<Backend>`のpublicメンバは、ファイルシステム利用者が必要とする操作を中心にする。FATやinodeテーブルの物理操作はprivateまたはサブシステムクラス経由とし、呼び出し側がディスクレイアウトを直接操作することを禁止する。

```text
アプリケーション / syscall層
          │
          ▼
Filesystem<Backend>  ← ファイルシステムの公開窓口
          │
     ┌────┼──────────┐
     ▼    ▼          ▼
   Fat  InodeTable  OpenFileTable
     │    │
     └────┴──────┐
                 ▼
       BlockDevice<Backend>
                 │
                 ▼
             Backend
```

この構造により、ディスクイメージ、SDカード、内蔵flash等のBackend差異は`BlockDevice`以下に閉じ、FAT/inode/ディレクトリ等のファイルシステムロジックは共通化する。



---

## 7. FATチェーン操作層の設計

### 7.1 位置づけと実装方式: `Fat<Backend>`クラス

**6.6節のC++構成方針に従い、FAT層は独立した`Fat<Backend>`クラスとして実装する。** `Filesystem<Backend>`が`Fat<Backend>`を所有し、FATキャッシュ・allocator状態をファイルシステムインスタンスごとに分離する。SDカードと内蔵flash ROMを同時にマウントしても状態が混線しない。

```
呼び出し側(inode層 / ディレクトリ層 / ファイルread/write層)
        │
        ▼
Fat<Backend>::get/set/alloc/free_chain/walk_to_index/extend
        │
        ▼
Fat<Backend>::device_ (= BlockDevice<Backend>)
```

```cpp
template<typename Backend>
class Fat {
public:
    FsStatus get(uint32_t cluster, uint16_t* out);
    FsStatus set(uint32_t cluster, uint16_t value);
    std::pair<uint32_t, FsStatus> alloc();
    FsStatus free_chain(uint32_t head_cluster);
    std::pair<uint32_t, FsStatus> walk_to_index(uint32_t head, uint32_t index);
    std::pair<uint32_t, FsStatus> extend(uint32_t tail_cluster);
    std::pair<uint32_t, FsStatus> find_tail(uint32_t head_cluster);
    FsStatus sync();

private:
    BlockDevice<Backend>& device_;
    const superblock_t& superblock_;
    mount_state_t& mount_state_;
    bool valid_ = false;
    uint32_t block_no_ = 0;
    bool dirty_ = false;
    uint8_t cache_[FS_BLOCK_SIZE]{};
};
```

**同時マウント**: SDカードと内蔵flashごとに独立した`Filesystem<Backend>`インスタンスを作る。各mountにはBackend固有の引数に加え、デバイス容量と外部指定の`fs_start_block`を与える。FAT cache、mount state、superblockは共有しない。

`mkfs`ツールは、まだmountしていないフォーマット前イメージに構造を書き込む一回限りの処理であり、恒常的な状態(FATキャッシュ等)が不要なため、`Filesystem<Backend>`を経由せず**`BlockDevice<Backend>`だけを直接使う独立ツール**として設計する(3.4節)。呼び出し側は「クラスタ番号→FATブロックの何バイト目か」「クラスタ番号→データ領域のどのブロックか」といった換算やチェーン走査ロジックを一切意識しない、という原則自体はテンプレート化の前後で変わらない。

### 7.2 内部状態

| 状態 | 保持場所 |
|---|---|
| `data_blocks`(スーパーブロック由来、不変) | `Filesystem<Backend>::superblock_`(インスタンスメンバ) |
| `next_free_cluster_hint`(動的帳簿、mount時に再構築) | `Filesystem<Backend>::mount_state_`(インスタンスメンバ、13節) |
| **FATブロック単位の1ブロックキャッシュ**(ブロック番号+dirtyフラグ+内容) | `Fat<Backend>`のインスタンスメンバ(5節A種の固定サイズ方針に沿う) |

1FATブロックには256エントリ入る。bumpアロケータでのクラスタ連続確保(ファイル拡張・inodeテーブル成長のたび)は多くの場合「直前と同じFATブロック」への書き込みになるため、都度read-modify-writeすると256回中255回が無駄なI/Oになる。**FAT層専用の決め打ち1エントリキャッシュ**(`sync()`で dirty なら書き戻す)を持つことでこれを回避する。これは6.2節で触れた「将来のバッファキャッシュ層」の最小限の先行実装とみなせる。デバイスごとに`Filesystem<Backend>`インスタンスが分かれるため、このキャッシュも自動的にデバイスごとに独立する。

### 7.3 提供するメンバ関数

`03-fat.md`を各操作の契約とする。クラスタ番号を返す操作は`std::pair<uint32_t, FsStatus>`、取得・更新・同期等は`FsStatus`と出力引数を使用する。

| 関数 | 契約 |
|---|---|
| `get(cluster, out)` / `set(cluster, value)` | cluster範囲とFAT位置のchecked計算を行い、2バイトLE entryを取得・更新する。領域外計算は`ECORRUPT`とする |
| `alloc()` | hint位置のentryがFREEであることを確認し、EOF設定成功後だけhintを進める。枯渇は`ENOSPC`、想定外の非FREEは安全停止する |
| `walk_to_index(head, index)` | index 0はhead。各hopの範囲、FREE、必要位置前のEOF、過剰hopを検査し、破損は`ECORRUPT`で停止する |
| `find_tail(head)` | EOFを持つ末尾を返す。範囲外・FREE・過剰hopを`ECORRUPT`で停止する |
| `extend(tail)` | 有効なEOF tailを確認し、新clusterをallocしてからtailへ連結する。途中失敗で既存chainを壊さない |
| `free_chain(head)` | phase 1ではheadが範囲外なら`ECORRUPT`、有効なら`OK`。I/Oもchain走査もFAT更新も行わない |
| `sync()` | dirty blockを書き戻す。失敗時はdirty状態を維持する |

cache切替前にdirty blockを書き戻し、失敗時は新blockへ切り替えない。read失敗時にvalidを立てない。`walk_to_index`はinode-table位置計算と通常ファイルの位置計算で共有する。`CLUSTER_NONE`は未割当のinode表現であり、`free_chain()`へそのまま渡さない。

### 7.4 フェーズ1/フェーズ2の差分が閉じる範囲

| 関数 | フェーズ1(bump) | フェーズ2(ビットマップ回収) | 呼び出し側への影響 |
|---|---|---|---|
| `fat_alloc` | hintのFREEを確認し、EOF設定成功後だけhintを進める | 空きビットマップを探索 | なし |
| `fat_free_chain` | head範囲を検査し、I/O・走査・更新を行わない | チェーンを辿り各クラスタを解放・ビットマップ更新 | なし |
| その他全関数 | 変更なし | 変更なし | なし |

呼び出し側(inode層・unlink実装)は最初からフェーズ2を見据えた正しい呼び出し方で実装しておくことで、フェーズ2移行時の変更をFAT層内部だけに閉じ込める。

**unlink実装への制約(13節の再構築アルゴリズムの前提条件)**: フェーズ1のunlinkは、`nlink`が0になっても**inodeの`mode`フィールドを`S_IFREE`へ戻してはならない**(非openのorphanをfinalizeした時点で`S_IUNUSED`にする)。「現在生きているか」は`(mode != S_IFREE) && (mode != S_IUNUSED)`で判定し、「一度でも使われたか」は`mode != S_IFREE`のみで判定する。これはFATの`fat_free_chain`と同じ「回収相当の操作を実施しない」思想をinode層にも一貫させたものであり、13節のmount時再構築(inodeテーブルの高水準点探索)が正しく動作するための前提条件となる。

### 7.5 末尾クラスタcache

通常ファイルのinodeには末尾clusterを永続化しない。将来必要になればRAM専用のopen-file entryへcacheを追加する(14.7節)。

---

## 8. 実装順序

`00-overview.md`の順序に従う。

1. シリアライズ(4節、`01-serialization.md`)
2. ブロックデバイス(6節、`02-block-device.md`)
3. FAT(7節、`03-fat.md`)
4. mountとsuperblock(13節、`04-mount-superblock.md`)
5. inodeテーブル(14節、`05-inode-table.md`)
6. ディレクトリ(15節、`06-directory.md`)
7. creat/write(16節、`07-creat-write.md`)
8. read/lseek(17節、`08-read-lseek.md`)
9. unlink/close(18節、`09-unlink-close.md`)
10. pathname/link(19〜20節、`10-pathname-link.md`)
11. mkdir/rmdir (21節、`11-mkdir.md`)
12. pathname rename (12節、`12-rename.md`)
13. syscall接続

`mkdir()`と`rename()`は上記のinode・directory・link count・rollback経路を組み合わせるphase 1機能として、syscall接続前にそれぞれ21節/`11-mkdir.md`、`12-rename.md`を満たす。各段階で下位層の単体テストを完了してから上位層へ進み、上位層が下位層のディスクレイアウトを直接操作しないことを確認する。

## 9. テスト方針

- 最初からカーネルに組み込まず、**ホストOS上で動くC++ライブラリ**として実装(`fs_open`, `fs_read`, `fs_write`, `fs_creat`, `fs_unlink`, `fs_readdir`等)。
- ライブラリを呼ぶだけの簡単なCLIツール(`myls`, `mycat`, `mycp`)でまず検証。
- ある程度動いたら**FUSEでマウント**し、本物の`ls`/`cat`/`cp`/`echo >>`から自作FSを触れるようにする。これが最初の大きな山場。
- エンディアン/アラインメント検証用のダンプ・比較には、4.3節の「テストハーネス専用」iostreamベースのバイナリマニピュレータ(`binary_out`/`binary_in`)を利用してよい(本体には組み込まない)。

---

## 10. マイルストーン

| # | 内容 |
|---|---|
| M1 | `mkfs`でイメージ作成 → ライブラリ経由で"hello.txt"を書いて、バイト完全一致で読み戻せる |
| M2 | 複数ファイルの作成・`readdir`でのディレクトリ一覧が動く |
| M3 | 逐次チェーン延長で数百KBのファイルへの追記が正しく動く(境界値:ちょうど確保サイズを超える瞬間) |
| M4 | `unlink`後、(フェーズ2ビットマップ化していれば)本当に領域が再利用されることを確認 |
| M5 | FUSEマウントして、実際の`ls`/`cat`/`cp`/`echo hello >> file`が動く |

---

## 11. 未確定・保留事項一覧

phase 1の各層契約は`00-overview.md`〜`12-rename.md`と本書4〜21節にまとめる。FAT hint再構築、pathname、orphan lifetime、部分write、`mkdir()`、限定renameは保留事項ではない。

- phase 2のcluster/inode回収方式とmount再構築方式は、phase 2着手時に再検討する。
- FAT8は現スコープ外の検討事項とする。
- ユーザプロセスのヒープは`brk`/`sbrk`を扱う段階まで保留する。
- 通常ファイルの末尾cluster cacheは必要性が判明した時点で、FD統合経路では`RegularFile`、Filesystem単体利用では`open_file_t`等のRAM専用open単位状態へ追加する。inodeには永続化しない。
- open-but-unlinked状態でクラッシュすると、RAMのFilesystem側open handle表が失われ、`nlink == 0`のlive inodeが残留し得る。phase 1ではmount中に推測で回収しない。
- truncateはphase 1で旧clusterを再利用可能にしないため、繰り返すと枯渇を早め得る。
- VFS、統合mount名前空間、cross-device link、symlink、fsckは現スコープ外とする。`rename()`はphase 1の同一Filesystem・destination置換なしの限定APIとして`spec/filesystem/fs_impl_spec/fs_impl_spec_12-rename.md`に定める。`rmdir()`はphase 1対象で、詳細は`spec/kernel/mkdir_rmdir_implimentation_spec.md`を参照する。将来のVFS接続の検討箇所は19.5節を参照する。

破損入力とI/O故障注入のテストは保留せず、各層の必須テストとして実施する。`creat()`、`link()`、`mkdir()`のrollbackは18.2節の共通helperを使い、最初の失敗statusを保持してdangling nameよりリークを優先する。

---

## 11.1 仕様レビューで追加した境界条件・運用規約

この節の事項は、外部レビューとその後の設計検討を反映したphase 1の確定仕様である。

### 11.1.1 FATクラスタ番号の上限

FATエントリは`uint16_t`、`FAT_FREE=0x0000`、`FAT_EOF=0xFFFF`であるため、`0xFFFF`をクラスタ番号として使用してはならない。実装では有効クラスタ番号の上限を明示的に定義し、`data_blocks`がその範囲を超えないよう`mkfs`で検証する。

### 11.1.2 最大ファイルサイズと整数オーバーフロー

`offset + len`、`inode_num * inode_size`、`(size + block_size - 1)`等の計算で整数オーバーフローを起こしてはならない。ファイルサイズ・オフセットはファイルシステム容量および使用する整数型の範囲内に制限し、範囲外は`FsStatus::ERANGE`または`FsStatus::EINVAL`として拒否する。`lseek`で負の結果となる指定は拒否し、offsetを変更しない。

### 11.1.3 inode番号・inodeテーブル容量

inode番号は`uint32_t`で表現するが、実際に割り当て可能な最大inode番号は、`inode_size`とinodeテーブルの最大容量およびファイルシステム容量から決まる。inode番号→バイトオフセット変換でオーバーフローしないことを`get_inode`/`alloc_inode`の境界条件として保証する。inode 0は予約、inode 1はroot固定。

### 11.1.4 directory inodeの`size`

通常ファイルでは`size`は論理的なファイルデータ長を表す。ディレクトリでは、固定長direntを格納するために**確保済みディレクトリ領域の容量**を表す。この意味の違いは仕様として明示する。

### 11.1.5 FAT cacheのI/Oエラー

`sync()`でdirtyなFATキャッシュを書き戻す際にI/Oエラーが発生した場合、エラーを呼び出し側へ返し、書き戻しに成功するまでdirty状態を失わない。`unmount()`の最終`sync()`が失敗した場合はアンマウント成功とはしない。

### 11.1.6 クラッシュ整合性

フェーズ1ではジャーナリング・トランザクション・完全なfsckによる自動復旧を提供しない。電源断・クラッシュの更新途中にファイルシステムが不整合になる可能性を許容する。整合性が保証できない場合は、参照されないリソースが残る側を優先し、既存データを別ファイルとして誤認するような状態を避ける設計を原則とする。

### 11.1.7 排他・リエントラント性

FS内部状態(`fat_cache_`、`mount_state_`、Filesystem側open handle表(`open_file_t`テーブル)等)への同時アクセスは、FS操作をシリアライズすることで保護する。マルチスレッド/割り込みから同時にFS APIを呼び出すことはフェーズ1ではサポートしない。将来並行実行を導入する場合はFilesystemインスタンス単位の排他を追加する。

### 11.1.8 nlinkの上限

`nlink`は`uint16_t`であるため、`UINT16_MAX`の状態でさらに増加させる操作は拒否し、`FsStatus::EMLINK`を返す。減算時のアンダーフローも許可しない。

### 11.1.9 FATチェーン破損時の安全停止

`chain_walk_to_index`、`chain_find_tail`等のチェーン走査では、クラスタ番号の範囲外参照およびチェーン長が`data_blocks`を超える状態を検出し、`FsStatus::ECORRUPT`等のエラーで停止する。循環チェーンによって無限ループしてはならない。

### 11.1.10 pathname grammar

`PATH_MAX=128`、`NAME_MAX=27`を上限とする。先頭`/`はrootから開始し、連続する`/`は空コンポーネントとして無視する。末尾`/`やファイル名部分を持たないパスを`resolve_parent`へ渡した場合はエラーとする。`.`と`..`は通常のdirentとして解決し、rootの`..`はroot自身を指す。空文字列は`FsStatus::EINVAL`とする。

### 11.1.11 read/write offset契約

`read_at`/`write_at`はoffsetを変更しない。両者とも`{処理バイト数, FsStatus}`を返す。Filesystem単体APIの`read(open_file_t*)` / `write(open_file_t*)`は互換APIとして、実際に処理したバイト数だけ`open_file_t.offset`を進める。エラー時も、既に正常に処理されたバイト数があればその分だけoffsetを進める。

OSのFD統合経路では`open_file_t.offset`を使用せず、上位`FileDescription::offset_`をfile positionの正本とする。`RegularFile`は上位から渡されたoffsetで`read_at` / `write_at`を呼び、転送成功数に応じたoffset更新は`FileDescription`側で行う。

### 11.1.12 `open_file_t`の参照とFD統合

Filesystem単体APIでは、`dup_file()`により同一`open_file_t`の`ref_count`を増加させ、`offset`を共有する従来契約を維持する。

OSのFD統合経路では、複数fdの共有単位は上位`FileDescription`である。`dup()` / `spawn()`では`open_file_t::ref_count`を増加させず、同じ`FileDescription` / `RegularFile` / `open_file_t*`を共有する。したがって`open_file_t::ref_count`と`offset`はFD統合経路の共有状態の正本ではない。最後のfd参照が外れたときにのみ`Filesystem::close(open_file_t*)`を1回呼び、その時点でunlink済みならFilesystem側のorphan finalize判定を行う。

### 11.1.13 mkfsの容量計算

FATはデータcluster1個につき2バイトentryを必要とする。superblock後の利用可能領域を`available_blocks`として、**最小の十分なFAT長**を直接求める。

```text
available_blocks = total_blocks - fs_start_block - 1
fat_blocks = available_blocks / 257 + (available_blocks % 257 != 0)
data_start_block = fs_start_block + 1 + fat_blocks
data_blocks = available_blocks - fat_blocks
```

全加算・減算・乗算は評価前にoverflow/underflowを検査する。FAT容量条件は`data_blocks <= fat_blocks * 256`であり、等式による固定点反復は使わない。反復方式は容量によってFAT長が振動するためである。

inodeテーブルとroot directoryに最低2 data clusterを必要とする。最小構成はsuperblock 1、FAT 1、data 2の計4ブロックであり、`total_blocks >= fs_start_block + 4`をchecked計算で確認する。`data_blocks <= 65535`を要求し、`0xFFFF`をdata cluster番号に使わない。

### 11.1.14 fsckの将来仕様

フェーズ1ではfsckを実装しないが、将来の検査項目として以下を維持する: FATクラスタ範囲、FATチェーンの循環、inodeの`first_cluster`、ファイルサイズとチェーン長、dirent→inode参照、nlink、`.`/`..`、inodeテーブルチェーンの整合性。

### 11.1.15 境界値テスト

少なくとも、ファイルサイズ0/1/511/512/513/1023/1024、EOFおよびEOF+1へのseek、負のseek結果、EOFを越えた位置へのwrite、inodeテーブルのクラスタ境界、dirent境界、hard link/unlink/open中unlink、FAT枯渇、最大クラスタ・最大inode付近をテストする。特に`write_at()`については、`offset=0,len=1`、`offset=511,len=2`、`offset=512,len=1`、`offset=512,len=512`、`offset+len`のuint32_t境界付近を含める。

### 11.1.16 ファイル名の文字集合

phase 1ではファイル名を**NUL終端された不透明なバイト列**として扱う。ファイルシステムは文字コードを解釈・正規化せず、NUL以外の各バイトをそのまま保持・比較する。`/`はpathname区切りとしてpathname層で処理し、NULはpathname終端なので名前の一部として表現しない。`NAME_MAX=27`は文字数ではなくバイト数で判定する。使用中direntの空名はdirectory層で破損として扱う。

### 11.1.17 `now()`の抽象化

ファイルシステム内部から現在時刻を直接取得する実装には依存しない。`now()`は将来、RTC等を持たないCortex-M0/68000環境にも移植できるよう、**time provider / HALの抽象化ポイント**として扱う。ホスト実装ではホスト時刻を返すproviderを用い、実機ではRTC等に対応するproviderへ差し替える。

### 11.1.18 directory `read()`のdirent分断契約

phase 1ではV7流にdirectoryを通常の`read()`で生の`dirent_t`配列として読み出す。ユーザランドはserialized sizeである`DIRENT_SIZE(32)`単位で読み出し、**1回のreadでdirentの途中を切り出して次のreadへ持ち越すことを前提としない**。FS内部の`read_at()`自体は通常ファイルと同じバイト列読み出しとして動作し、dirent境界を強制しない。

---

## 12. 次のステップ

実装・検証は8節の順序と各層仕様書の完了条件に従う。inodeは14節、directoryは15節、creat/writeは16節、read/lseekは17節、unlink/closeは18節、pathnameは19節、linkは20節、mkdirは21節を参照する。

仕様の確定と実装・テストの完了は区別する。ホスト単体テストで境界値・破損・故障注入を確認し、phase 1のAPIとlifetimeを満たした後にsyscall層へ接続する。VFSや回収を追加する際は別途設計する。

---

## 13. スーパーブロックの動的帳簿とmount時再構築

### 13.0 `mount()`時のsuperblock検証

`mount()`では、superblockの内容を使って`mount_state`を再構築する前に、**superblock自身の妥当性を検証する**。未検証の値をクラスタ数・ブロック位置・inodeサイズ等として使用してはならない。

最低限、以下を検証する。

1. superblockを指定された`fs_start_block`から読み込む。
2. `magic`が本FSの値と一致すること。
3. `block_size == FS_BLOCK_SIZE == 512`であること。
4. superblockの`fs_start_block`がmount時に外部から指定された値と一致すること。
5. `total_blocks`がデバイス容量を超えないこと。
6. `fat_start_block`、`fat_blocks`、`data_start_block`、`data_blocks`がオーバーフローなく計算可能で、互いに重複せず、FS領域内に収まること。
7. FATが`data_blocks`個のエントリを収容できること。
8. `data_blocks <= 65535`であること。
9. `inode_size != 0`、`block_size % inode_size == 0`、`inode_size == INODE_SIZE(64)`であること。phase 1のdecode契約より小さいレコード長を受け入れない。
10. `inode_table_start_cluster`が有効なクラスタ番号であり、phase 1の規定値0と一致すること。
11. `root_inode == 1`であること。

いずれかの検証に失敗した場合、`mount()`は`FsStatus::ECORRUPT`を返し、`mount_state`の再構築および通常のFS操作へ進まない。範囲計算に使用する整数のオーバーフローも不正FSとして扱う。

検証後に初めてFAT・inode tableを読み、13.3節の再構築を行う。全処理成功後だけmounted状態へ遷移する。mount失敗時は通常APIを利用可能にせず、open済みBackendを適切にcloseして途中cache/stateを破棄する。

`04-mount-superblock.md`に従い、unmountは次の順序とする。

1. mounted状態を確認する。
2. FAT cache等のdirty状態をsyncする。
3. sync失敗時はBackendをcloseせず、mounted/open状態を維持してstatusを返す。この場合だけunmountを再試行できる。
4. sync成功後に`BlockDevice::close()`を1回呼ぶ。
5. closeの成否にかかわらずmounted状態を解除し、runtime stateを無効化する。close失敗時はそのstatusを返すが、unmount/Backend closeは再試行しない。

superblockはunmountでも書き戻さない。

### 13.1 問題意識

当初案(3.3節旧版)ではスーパーブロックに「不変な仕様情報」と「操作のたびに変わる動的な帳簿」(`next_free_cluster_hint`, `next_free_inode`, `inode_table_last_cluster`, `inode_table_capacity`)が同居しており、これをディスクへ永続化しようとすると、ファイル作成・拡張のたびにスーパーブロック全体(またはdirtyフラグ管理)を書き戻す必要が生じ、以下のトレードオフが避けられなかった:

- 毎回同期書き込み → I/O回数が増える。
- 遅延書き込み(dirty+`sync()`) → 電源断・異常終了時に更新が失われるリスク。

### 13.2 解決方針: 動的帳簿を「持たない」

動的な帳簿4フィールドは、いずれも**FAT・inodeテーブルという「唯一の真実の情報源」から再計算可能**であるため、ディスクには持たず、**mount時に一度だけ再構築**する。これにより:

- スーパーブロック(`fs_start_block`)はmkfs時に一度書いたら運用中は一切書き込まれない。
- 「dirty書き戻し漏れ」というバグのクラスが構造的に存在しなくなる。
- フェーズ1の思想(信頼できる情報源はFAT/inodeテーブル本体のみ、帳簿はあくまで派生キャッシュ)と一貫する。

再構築結果はRAM専用の`mount_state_t`として`Filesystem<Backend>`インスタンスが保持する(7.1節)。

```cpp
struct mount_state_t {          // RAM専用。mount()時に再構築。ディスクには一切書かない
    uint32_t next_free_cluster_hint;
    uint32_t next_free_inode;
    uint32_t inode_table_last_cluster;
    uint32_t inode_table_capacity;
};
```

### 13.3 再構築アルゴリズム(phase 1確定)

**(a) `next_free_cluster_hint`: FAT全体の前方走査**

`04-mount-superblock.md`に従い、最初のFREE位置を求めると同時に、その後がすべてFREEであることを検証する。使用済みclusterは0から連続するprefixでなければならない。

```text
first_free = data_blocks
for i in [0, data_blocks):
    statusを確認して v = FAT[i] を取得する
    if first_free == data_blocks:
        if v == FAT_FREE: first_free = i
    else:
        if v != FAT_FREE: return ECORRUPT
next_free_cluster_hint = first_free
```

境界後にusedが混入していれば`ECORRUPT`、FAT read失敗はそのstatusを返す。全使用済みならhintは`data_blocks`となり、次のallocは`ENOSPC`を返す。1ブロックcacheを使い、FAT全体の走査で各物理FAT blockのreadは最大1回とする。境界推定のためだけの二分探索は行わない。

**(b) inodeテーブル: 1回の前方走査**

1. `inode_table_start_cluster`から末尾まで走査し、各clusterの範囲、FREE混入、cycle/過剰hopを検査する。
2. 各blockのreadと全inodeレコードのdecodeのstatusを確認する。
3. `mode != S_IFREE`の最大indexを`highest_ever_used_index`とする。`S_IUNUSED`も候補に含める。
4. 最終clusterを`inode_table_last_cluster`とする。
5. `inode_table_capacity = cluster_count * (512 / inode_size)`をchecked計算する。
6. `next_free_inode = highest_ever_used_index + 1`をchecked計算する。

範囲外、cycle、FREE混入は`ECORRUPT`で停止する。live判定は`mode != S_IFREE && mode != S_IUNUSED`であり、「一度でも使われたか」の判定とは区別する。再構築成功前の途中stateを通常APIへ公開しない。

### 13.4 前提条件: unlinkの実装制約

上記(b)の`highest_ever_used_index`計算が正しく動作するためには、**「一度使われたinodeスロット」と「一度も使われていないinodeスロット」が、unlink後も区別できる**必要がある。7.4節で明記した通り、フェーズ1のunlinkは`nlink`が0になっても`mode`を`S_IFREE`へ戻してはならない(`mode != S_IFREE`で「一度でも使われたか」を判定する)。これはFATの`fat_free_chain`スタブと同じ「回収相当の操作を実施しない」思想をinode層にも一貫させたものである。

### 13.5 副次的なメリット

「電源断でsuperblockの更新が失われるリスク」は、そもそも書き込まないので発生し得ない。フェーズ1の「壊れたらmkfsし直す」運用とも思想的に一貫している。

---

## 14. inode層の設計

### 14.1 位置づけ・アクセス経路

inodeテーブル自体が「FATチェーンを持つ特殊ファイル」(2.3節)である以上、inode層の主要な仕事は**「inode番号→バイトオフセット→クラスタ番号」の変換**と、**inodeテーブル自身の成長**の2つ。どちらも既存の`Fat<Backend>`(7節)のプリミティブをそのまま再利用する。`InodeTable<Backend>`クラスのメンバとして実装する。

```
呼び出し側(ディレクトリ層 / creat / unlink / read・write)
        │
        ▼
InodeTable<Backend>::get / put / alloc / dec_nlink
        │
        ▼
InodeTable<Backend>::fat_ / Fat<Backend>::walk_to_index / extend / get / set  (7節、既存)
        │
        ▼
Filesystem<Backend>::device_ (= BlockDevice<Backend>)
```

### 14.2 バイトオフセット→クラスタ変換

inode位置は次の式で求める。全乗算・加算・narrowingは評価前に検査する。

```text
byte_offset = inode_num * inode_size
chain_index = byte_offset / FS_BLOCK_SIZE
offset_in_cluster = byte_offset % FS_BLOCK_SIZE
```

`Fat::walk_to_index(inode_table_start_cluster, chain_index)`のstatusを確認して対象clusterを得る。`offset_in_cluster + inode_size <= 512`を保証し、物理blockへの変換も範囲検査後に行う。通常ファイルのbyte offsetも同じFAT走査プリミティブを使い、位置変換に失敗した場合に番兵を物理位置として使わない。

### 14.3 `get_inode` / `put_inode`

`05-inode-table.md`の`InodeTable::get()` / `put()`を使用する。

```cpp
// inode-tableの物理位置はこの層に隠蔽する。
FsStatus get(uint32_t inode_num, disk_inode_t* out);
FsStatus put(uint32_t inode_num, const disk_inode_t& in);
```

getは引数・inode番号範囲を検査し、checked位置計算、FAT走査、block read、decode・意味検証の順に実施する。すべて成功後に出力を確定し、各失敗statusを伝播する。`nlink == 0`のlive inodeはopen-but-unlinkedやクラッシュ後残留で現れ得るため、その状態だけで破損扱いしない。

putはinode値と位置を検証し、blockをreadして一時buffer内の対象レコードだけをencodeし、block全体をwriteする。他inodeを変更しない。phase 1ではinode専用cacheを持たない。

### 14.4 `alloc_inode`(inodeテーブル自身の成長)

1. `next_free_inode < inode_table_capacity`なら、その番号を返してhintを進める。
2. 容量不足なら新容量の計算をoverflow検査し、FAT経由で末尾へclusterを1個extendする。
3. 新clusterを全ゼロでwriteする。
4. zero-fill成功後だけ`inode_table_last_cluster`と容量を更新する。
5. 新規番号を返し、呼び出し側が全フィールドを初期化してputする。

allocation枯渇は`ENOSPC`、途中のI/O等はそのstatusを返す。途中失敗で成功と誤認するhint更新を行わない。返却形の細部は既存宣言に合わせるが、inode番号0や`CLUSTER_NONE`だけで失敗理由を置き換えない。

### 14.5 link countと最終状態への遷移

`InodeTable`はFilesystem側open handle表を参照しない。永続inode操作を次の3操作に分ける。

- `inc_nlink(inode_num)`: getしてliveを確認する。`nlink == UINT16_MAX`なら変更せず`EMLINK`を返す。1増加し`ctime = now()`としてputする。
- `dec_nlink(inode_num)`: getしてliveを確認する。`nlink == 0`なら減算せず`ECORRUPT`を返す。1減少し`ctime = now()`としてputする。0に到達しても`mode`・`size`・`first_cluster`を変えず、finalizeもしない。
- `finalize_inode(inode_num)`: liveかつ`nlink == 0`を確認する。条件不成立は`ECORRUPT`。非open確認は呼び出し側の事前条件とする。`first_cluster != CLUSTER_NONE`なら`Fat::free_chain()`のstatusを確認し、`size = 0`、`first_cluster = CLUSTER_NONE`、`mode = S_IUNUSED`、`ctime = now()`としてputする。

`S_IFREE`へ戻さない。open状態を調べて減算とfinalizeを組み合わせる責務は、`Filesystem`の`drop_link()` / `finalize_if_orphaned()`に置く(18.2節)。

### 14.6 ファイル拡張戦略

**結論: `chain_extend`ベースの単純な逐次延長のみを採用し、倍々確保+コピー方式は不採用とする。**

理由:
- FATチェーンは連結リストであり、「末尾に1クラスタ追加する」操作(`chain_extend`)はO(1)。`std::vector`が抱える「確保容量を超えたら全体を再確保してコピーする」問題は、配列ではなく連結リストだからこそ発生しない。
- 倍々拡張方式が必要になるのは「連続領域でなければならない」制約がある場合(配列、実メモリ上のヒープ)であり、FATチェーンにその制約はない。
- 結果としてinodeには「実サイズ(size)」だけを持たせれば十分で、「確保済みサイズ」フィールドは不要(3.3節のinode構造体は既にsizeのみで、この結論と整合している)。

### 14.7 通常ファイルの末尾クラスタキャッシュ

**結論: inode自体には末尾クラスタを持たせない(ディスクへの永続化は不要)。将来必要になれば、1回のopenに対応するRAM上の`RegularFile`またはFilesystem側open handleにキャッシュを持たせる。**

理由:
- inodeにディスク上の永続フィールドとして持たせると、`write`のたびにinodeも書き戻す必要が生じ、13節で解消したはずの「動的帳簿の永続化問題」をinodeレベルで再発させてしまう。
- 「同じファイルへの連続書き込み」で末尾探索コストが効いてくる典型的な場面は、1つのopenセッション中に何度もwrite/appendするケースである。FD統合経路では1回のopenごとに`RegularFile`が1個存在し、`dup()` / `spawn()`時にも同じ`RegularFile`を共有するため、このRAM専用オブジェクトにcacheを置けばopen単位の共有状態として自然に扱える。
- Filesystem単体利用だけを考える場合は`open_file_t`へcacheを追加してもよい。いずれの場合もinodeへ永続化しない。末尾cache自体は将来の最適化であり、phase 1では必須としない。

---

## 15. ディレクトリ層の設計

### 15.1 前提: ディレクトリは「中身がdirentの配列である、普通のファイル」

ディレクトリの実体は、`mode = S_IFDIR` のinodeが指す通常のFATチェーン。中身が「32バイト固定長の`dirent_t`の並び」であるという点だけが特殊で、クラスタ確保・成長のメカニズムは通常ファイルと完全に同一にする。ディレクトリ層は「dirent配列としての解釈」だけを担当し、チェーン操作は7節・14節の既存プリミティブに委譲する。

### 15.2 FAT末尾探索の利用

通常ファイルとdirectoryの末尾追加は`Fat::find_tail()`を使用する。戻り値は`std::pair<uint32_t, FsStatus>`であり、status確認後だけ得たclusterを使用する。範囲外、FREE、cycle/過剰hopを`ECORRUPT`で停止する。計算量はO(chain長)であり、inodeに末尾clusterを永続化しない。

### 15.3 ファイルへのcluster追加と初期化

空の未割当ファイルは`Fat::alloc()`、既存chainは末尾探索後に`Fat::extend()`を使う。FAT操作とzero-fillのstatusを確認し、新clusterを適切に初期化してから上位層へ公開する。初期化失敗をallocation枯渇へ置き換えず、未初期化データをユーザへ読ませない。途中失敗では既存chainを壊すよりリークを優先する。

inodeテーブルの成長も同じFAT延長経路を使うが、inode-table末尾と容量はzero-fill成功後だけ更新する(14.4節)。通常ファイルは論理size、directoryは確保済み容量を保持する。

### 15.4 ディレクトリ層が提供する関数

```cpp
// directoryの内容だけを操作し、対象inodeのlink countは変更しない。
FsStatus dir_lookup(uint32_t dir_inode_num, const char* name, uint32_t* out_inode_num);
FsStatus dir_add_entry(uint32_t dir_inode_num, const char* name, uint32_t inode_num);
FsStatus dir_remove_entry(uint32_t dir_inode_num, const char* name, uint32_t* out_inode_num);
```

全APIは対象inodeがlive directoryであることを確認する。pathname由来の名前は検証済みの1〜27バイトcomponentとして受け取り、文字コードを解釈しない。`.`と`..`も通常direntである。block I/O、FAT走査、decode/encode、inode更新のstatusを伝播する。

**lookup**: `inode.size`内を`DIRENT_SIZE(32)`単位で線形走査し、空きentryを飛ばして名前をbyte列で完全一致比較する。発見時にinode番号を返し、未発見は`ENOENT`。無効なinode番号、NUL終端不正、使用中direntの空名・`/`、size不整合は`ECORRUPT`とする。

**add_entry**: 重複名検査は呼び出し側の責務とし、既存の空きslotを優先する。空きがなければchainを1cluster延長し、全ゼロ初期化後に`dirent_encode()`でentryを書く。この場合だけdirectory sizeを512増やす。内容変更時にdirectoryの`mtime`・`ctime`を更新し、`atime == mtime`を維持する。inode更新を含む失敗時はdangling nameよりリーク側を優先する。

**remove_entry**: lookup後、対象slotをゼロ化し、chainやsizeは縮小しない。削除対象inode番号を出力する。directoryの`mtime`・`ctime`を更新し、`atime == mtime`を維持する。対象inodeの`nlink`は変更せず、unlinkがremove成功後に`drop_link()`する(18.1節)。

### 15.5 ルートディレクトリとの整合性

3.4節のmkfs手順で、ルートディレクトリには`.`と`..`の2エントリが既に書き込まれている。この2つも`dir_add_entry`と全く同じ形式(`dirent_t`)であるため、mkfs時点で特別扱いする必要はなく、`dir_lookup(root, ".", ...)`は普通に動作する。新しいディレクトリをphase 1の`mkdir`(21節)で作る際も、mkfsのルート初期化手順と全く同じ2エントリ書き込みロジックを再利用できる。

### 15.6 新規エラーコード

| コード | 用途 |
|---|---|
| `FsStatus::ENOENT` | 指定名のエントリが見つからない(`dir_lookup`失敗) |
| `FsStatus::EEXIST` | `creat`の通常パス(16.1節)では未使用(既存ファイルはtruncateして使い回すため)。**`link`(20節)と`mkdir`(21節)では対象名が既に存在する場合に使用する** |
| `FsStatus::EISDIR` | `creat`の対象が既にディレクトリだった場合(16.1節)。truncate対象になり得ないための拒否。`write(2)`をディレクトリに対して行った場合も拒否する(17.3節) |
| `FsStatus::EINVAL` | `lseek`の結果が負のoffsetになった場合(17.1節) |
| `FsStatus::ESPIPE` | seek不能なfd(pipe等)に対する`lseek`(17.1節) |

`dir_add_entry`は現時点では同名エントリの重複チェックを意図的に省略している。重複禁止のポリシーは`creat`/`mkdir`側の責務(呼び出し前に`dir_lookup`で存在確認する)とし、ディレクトリ層自体はダムな配列操作に徹する、という役割分担にしている。`link`も呼び出し前に存在確認する。

### 15.7 dirent_tのフィールド順序

3.3節の通り、`dirent_t`は`inode_num`(固定長)を先頭、`name`を末尾に配置する。理由: `name`は将来27バイトを超える可変長化の可能性があり、可変長部分を末尾に置くことで、固定長ヘッダ部分だけを見て次エントリへスキップする、といった処理と相性を良くしておくため。本節の空きスロット判定は`inode_num == INODE_NUM_NONE`とする。正常な未使用slotはゼロ化され`name[0] == '\0'`も満たすが、使用中direntの空名を空きとして扱わず、破損として拒否する。

---

## 16. creat/write層の設計

### 16.1 `creat`:既存ファイルと新規作成

`07-creat-write.md`に従い、pathname解決後の親directory inodeと単一名を受け取る。

```cpp
// 名前公開前にinode全体を初期化する。
FsStatus creat(uint32_t dir_inode_num, const char* name, uint16_t mode, uint32_t* out_inode_num);
```

名前と親directoryを変更前に検証する。`mode`の許可9ビットを保持し、phase 1ではumaskを適用しない。将来のumaskはprocess/syscall層で適用し、FS内部状態にはしない。

1. `dir_lookup()`が`OK`なら対象inodeを取得する。通常ファイルは同じinodeをtruncateして再利用し、directoryは`EISDIR`で拒否する。`EEXIST`は返さない。
2. truncateでは旧clusterを再利用可能に戻さず、論理sizeを0、`first_cluster`を`CLUSTER_NONE`とし、`mtime`・`ctime`と同値の`atime`を更新する。未割当なら`free_chain()`を呼ばず、呼ぶ場合はstatusを確認する。既存のpermission・nlinkは保持する。
3. lookupが`ENOENT`の場合だけ新規作成へ進む。他の失敗はそのまま返す。
4. inode番号をallocし、`mode = S_IFREG | (mode & 0777)`、`nlink = 1`、`uid = gid = 0`、`size = 0`、`first_cluster = CLUSTER_NONE`、provider由来の3時刻を設定してputする。
5. 初回put失敗時はdirentを作らず、そのstatusを返す。live inodeの永続化に成功していないため`dec_nlink()` / `drop_link()`は呼ばない。
6. put成功後に`dir_add_entry()`で名前を公開する。add失敗時は18.2節の`drop_link()`でrollbackし、操作を停止させた最初のstatusを返す。
7. 全成功後だけ`*out_inode_num`を確定する。

#### 16.1.1 孤児inodeのrollback

新規inodeの永続化後に名前追加が失敗した場合、`drop_link()`がlink count減算と非open orphanのfinalizeを調停する。この時点ではopenされていないため、正常なrollbackでは`S_IUNUSED`へ遷移する。`S_IFREE`へ戻さず、phase 1でclusterやinode slotを再利用しない。

rollback失敗時は名前を公開せず、inode/link countの残留を許容する。名前が未初期化またはfinalize済みinodeを指す状態を作らない。共通helperは`link()`と`mkdir()`でも使う。

#### 16.1.2 エラーコード

directoryに対するcreatは`EISDIR`。既存の通常ファイルをtruncateするため、通常creatは`EEXIST`を使わない。`EEXIST`はlinkとmkdirの既存名拒否で使用する。

### 16.2 `write_at`:位置計算・cluster確保・inode更新

```cpp
// offsetは呼び出し側が渡し、open-file offsetは変更しない。
std::pair<uint32_t, FsStatus> write_at(
    uint32_t inode_num, uint32_t offset, const uint8_t* src, uint32_t len);
```

`07-creat-write.md`に従う。通常ファイルのsizeは論理長であり、directoryのsizeは確保済み容量である。失敗により残ったchainの余分なclusterを正常なデータ長として数えない。

1. 長さ0は`{0, OK}`とする。長さ0以外のnull bufferは拒否する。`offset + len`、最大ファイルサイズ/FS容量をallocation・I/O・状態変更前に検査し、範囲外要求は`ERANGE`とする。
2. inodeを取得し、必要位置をchecked計算する。開始cluster indexは`offset / 512`、cluster内位置は`offset % 512`とする。
3. 未割当ファイルは先頭clusterをallocし、不足分は必要なだけ末尾へ逐次extendする。新clusterを適切にzero-fillし、EOF後の論理gapがゼロとして読めるようにする。倍々確保・chain全体コピーは行わない。
4. 各転送位置は`position = offset + transferred`をchecked計算し、`cluster_index = position / 512`、`cluster_offset = position % 512`から求める。ブロックを跨いだ後も最初のcluster内offsetを再利用しない。
5. 部分blockはread-modify-writeする。完全上書きblockは不要なreadを省略してよい。必要位置前のEOF、範囲外cluster、FREE、cycleは`ECORRUPT`で停止する。
6. 成功したblock/byteだけを転送数へ加える。I/O失敗後は次blockへ進まず、最初の停止statusを保持する。FAT枯渇は可能な範囲の部分writeと`ENOSPC`を返す。
7. inode更新候補のsizeは`max(old_size, offset + transferred)`をchecked計算する。内容またはsizeが変わった場合は`mtime`・`ctime`を更新し、`atime == mtime`を維持する。
8. inodeのsize/cluster情報を更新できなかったbyteを成功数へ含めない。データwrite成功数と、APIとして永続状態から利用できる成功数を混同しない。inode更新失敗の境界をfault injectionで確認する。
9. 成功数と停止理由を返す。inode更新や後始末の結果で先行する失敗statusを上書きしない。

ユーザ向けdirectory writeはsyscall/open-file層で`EISDIR`とする。低レベル`write_at()`はdirectory内部操作にも使えるbyte-range primitiveとする。

### 16.3 部分writeとエラー結果

all-or-nothingではなく、永続状態へ反映できた範囲の部分writeを返す。

| count | status | 意味 |
|---:|---|---|
| 要求長 | `OK` | 全要求を反映した |
| 0 < count < 要求長 | `ENOSPC` | 成功分の後に容量不足で停止した |
| 0 < count < 要求長 | `EIO`等 | 成功分の後にI/O等で停止した |
| 0 | `ENOSPC` / `EIO`等 | APIとして成功を数えられるbyteがない |

countは実際に永続化できたbyte数であり、inodeのsize/cluster更新に失敗して利用できない拡張部分を含めない。既存範囲への上書きと新規拡張を区別して、data write・inode更新の失敗境界を検証する。Filesystem単体APIの`write()`は返されたcountだけ`open_file_t.offset`を進める。FD統合経路では上位`FileDescription`が返されたcountだけ自身のoffsetを進める。

phase 1はクラッシュ整合性を保証しないが、それを理由にI/O失敗や破損を無視しない。未参照cluster等のリークを許容し、正常な部分成功、停止status、未反映部分を区別する。

---

## 17. read/lseek層の設計

### 17.0 Filesystem側open handleとFD共通基盤

Filesystem単体APIでは、固定配列`OpenFileTable`の`open_file_t`をopen状態として使用し、`read()` / `write()` / `lseek()` / `dup_file()`の互換APIを提供する。

一方、OSへ統合した経路ではprocess fd tableはFilesystemのentryを直接参照しない。process fd tableは上位`FileDescription`を参照し、`FileDescription`が`RegularFile`を介して1個の`open_file_t*`を保持する。共有open file descriptionの正本は`FileDescription`であり、file position、open flags、fd参照数をそこで管理する。

```cpp
// Filesystem単体API互換およびopen中inode追跡用のRAM handle。
struct open_file_t {
    uint32_t inode_num;
    uint32_t offset;     // Filesystem単体API用。FD統合経路では正本にしない
    uint32_t ref_count;  // Filesystem単体API用。FD統合経路では通常1のまま
    bool is_pipe;        // 既存互換。FD統合経路のpipe判定には用いない
};
```

FD統合経路では1回の`open()`につき`open_file()`を1回呼ぶ。`dup()` / `spawn()`では`dup_file()`を呼ばず同じ`FileDescription`を共有するため、`open_file_t`は増減しない。最後の`FileDescription`参照が解放されたときだけFilesystemの`close(open_file_t*)`を呼ぶ。

processのfd表はFSへ持ち込まないという原則は維持する。Filesystemはfd番号やプロセスを認識せず、inodeと自身のopen handleだけを管理する。

### 17.1 `lseek`

```cpp
// signedな位置計算を検証してから共有offsetを確定する。
FsStatus lseek(open_file_t* file, int32_t offset,
               Whence whence, int32_t* out_new_offset);
```

`08-read-lseek.md`に従い、`SEEK_SET`、`SEEK_CUR`、`SEEK_END`を扱う。pipe等のseek不能対象は`ESPIPE`。END起点はinode.sizeを取得する。ここで定義する`Filesystem::lseek(open_file_t*)`はFilesystem単体API用の互換wrapperである。FD統合経路ではfile positionの正本が`FileDescription`にあるため、このwrapperで`open_file_t::offset`を更新しない。FD層の`lseek`実装時には`RegularFile` / Filesystemから必要なinode sizeを取得し、検証後に`FileDescription::offset_`を更新する。

保存済みoffsetやsizeを検査せず`int32_t`へnarrowingしない。signedな中間値で、加算前にoverflowを検査する。負の結果は`EINVAL`、表現範囲外は`ERANGE`または共通算術規約に従うstatusで拒否し、失敗時は古いoffsetを維持する。結果検証後だけ`file->offset`と出力値を確定する。

EOFより先へのseekは許可するが、整数の表現範囲検査を省略しない。後のwriteは16.2節のgap初期化と容量上限に従う。

### 17.2 `read_at()`と`read()`

```cpp
// 論理EOFまでのbyte列を読み、成功数と停止理由を返す。
std::pair<uint32_t, FsStatus> read_at(
    uint32_t inode_num, uint32_t offset, uint8_t* dst, uint32_t len);
```

1. inodeを取得し、live状態を確認する。
2. `len == 0`または`offset >= inode.size`なら`{0, OK}`を返す。
3. `min(len, inode.size - offset)`をunderflow/overflowなく読込長とする。
4. `Fat::walk_to_index()`のstatusを確認して開始clusterを得る。
5. blockを順に読み、要求範囲だけcopyする。各clusterの範囲、FREE、cycle/過剰hopを検査し、論理EOF前にchainが終われば`ECORRUPT`とする。
6. I/O等の失敗時は既にcopyしたbyte数と停止statusを返す。全要求完了または論理EOF到達は`OK`とする。

read_atはopen-file offsetを変更しない。Filesystem単体APIのread wrapperは`open_file_t.offset`から呼び出し、部分成功にerrorが伴っても実読込byte数だけoffsetを進める。FD統合経路では`RegularFile`が`FileDescription`から渡されたoffsetで`read_at`を呼び、`FileDescription`が実読込byte数だけ自身のoffsetを進める。directoryも同じbyte列primitiveで読み、readからinode時刻の書込みを発生させない。

#### 17.2.1 atime: 3.3節の既存規定(mtimeの別名)に従う

3.3節のオンディスク構造体定義には、当初から`atime`の欄に「(予約，現在はmtimeと同一値)」と明記されている。つまり「atimeをmtimeの完全な別名として扱う」というのは今回新たに決めた事項ではなく、**既存の仕様**である。

`write_at`は内容またはsizeの変更時に`mtime`・`ctime`を更新し、`atime`を同じ`mtime`値へ更新する(16.2節)。`read_at`では3つの時刻をいずれも変更しない。

事後的に見て、この規定(POSIX本来の「最終アクセス時刻」の意味を意図的に簡略化する)には以下のような合理性がある(3.3節の設計時点での意図の推測、または追認できる根拠):

1. Linuxの実運用自体が`relatime`/`noatime`マウントオプションへ移行しており、「読み込みのたびにatimeを書く」がコストに対して価値の低い設計だという認識は業界的にも既に確立している。
2. 今回のLv2スコープ(パイプ・リダイレクトが効くB-shell)には`ls -u`のようなatime依存機能が含まれない。
3. `read_at`が`put_inode`を一切呼ばずに完結するのは、6節の複数デバイス対応(SDカードだけでなく内蔵flash ROMの同時マウント)を考えると実利がある。読み込みのたびに書き込みが発生しない、という性質は書き込み耐性が限られるflashバックエンドで重要になる。

これにより`disk_inode_t`の3つの時刻フィールドの意味は以下の通り確定する(`ctime`/`mtime`の更新タイミングは3.3節では明記されていなかったため、本節・16節での決定として記録する)。

| フィールド | 更新タイミング |
|---|---|
| `ctime` | 生成、nlink変更、内容またはsizeの変更、finalize時 |
| `mtime` | ファイル内容が変更されたとき |
| `atime` | 現在は`mtime`と同一値。`read_at`では更新しない |

`atime`フィールド自体は削除せず残す。将来`stat(2)`が`st_atime`を返す際に、フィールド追加なしで(意味は簡略化した形のまま)対応できるようにするため。

### 17.3 ディレクトリに対する`read`:v7流の生読み込みを採用

比較した方式:

| 方式 | 挙動 |
|---|---|
| v6/v7 | ディレクトリを`open`+`read`すると、生の`dirent_t`配列がそのままバイト列として返る。`ls(1)`はこれを自前でパースしていた(`readdir(3)`は存在しない) |
| POSIX/Linux | ディレクトリに対する`read(2)`は禁止(`EISDIR`)。専用の`getdents(2)`(Linux)や`readdir(3)`経由でしか列挙できない |

対象20syscallに`getdents`/`readdir`は含まれていないが、`ls`が動かないと実用的なシェルとして成立しないため、**v7流(ディレクトリの生読み込みを許可)を採用する**。

利点: `read_at`は無変更でそのまま使える。ディレクトリのデータもクラスタチェーン上に`dirent_t`の配列として乗っているだけなので(15節)、「通常ファイルかディレクトリか」で`read_at`内部が分岐する必要が一切ない。ユーザランドの`ls`は返ってきたserialized byte列を`DIRENT_SIZE(32)`単位で`dirent_decode()`し、メモリ上のdirentとして解釈する。ディスク表現をC++構造体配列へ直接キャストしない。

非対称な制約として、**ユーザ向けwriteはディレクトリに対して常に禁止**(`EISDIR`)する。これはv6/v7/POSIX/Linuxすべてで一貫している例外なしのルール(ディレクトリの内容を書き換えられるのは`dir_add_entry`/`dir_remove_entry`のようなFS内部関数だけ)。この`S_IFDIR`判定は低レベル`write_at`自身には置かない。Filesystem単体APIでは`write(open_file_t*)` wrapperが判定し、FD統合経路では`RegularFile`等の通常ファイルadapter境界で判定する。syscall層はinode種別を直接検査せず、薄いラッパに保つ。`write_at`はあくまでinodeとoffsetを受け取るbyte-range primitiveのままとする。

---

## 18. unlink/close層の設計

### 18.1 `unlink`:名前削除とlink count更新

```cpp
// 名前を先に消し、成功後だけlink countとorphan lifetimeを更新する。
FsStatus unlink(uint32_t dir_inode_num, const char* name);
```

1. `dir_lookup()`でtargetを得る。
2. target inodeを取得する。
3. directoryなら`EISDIR`を返す。
4. `dir_remove_entry(dir_inode_num, name, &removed_inode)`で名前を先に消す。
5. remove成功後だけ`drop_link(target)`し、そのstatusを返す。

逆順は禁止する。名前削除後のdrop失敗ではinodeやlogical storageのリークが残り得るが、live nameがfinalize済みinodeを指す状態より安全である。phase 1ではdirectoryに対する`unlink()`は常に拒否する。空directoryの削除は`rmdir()`経路で行い、cwd/open参照とorphan寿命は`spec/kernel/mkdir_rmdir_implimentation_spec.md`に従う。

### 18.2 open-but-unlinkedと共通helper

`09-unlink-close.md`に従い、open状態の判定は`Filesystem`が所有するopen handle表全体を走査する。特定entryの`ref_count`やprocess-local fd表だけで判定しない。FD統合経路ではdupされた複数fdが同じ1個のFS handleを共有するため、handleが存在する限りinodeはopenと判定できる。`InodeTable`はopen handle表を参照しない。

```cpp
// FS所有のopen状態と永続inode操作を調停する。
FsStatus finalize_if_orphaned(uint32_t inode_num);
FsStatus drop_link(uint32_t inode_num);
```

**`finalize_if_orphaned()`**:

1. inodeを取得し、失敗statusを伝播する。
2. `nlink != 0`ならinodeを変更せず`OK`を返す。
3. `open_files_.inode_is_open(inode_num)`ならinodeを変更せず`OK`を返す。
4. `inode_table_.finalize_inode(inode_num)`を呼び、そのstatusを返す。

**`drop_link()`**:

1. `inode_table_.dec_nlink(inode_num)`を呼ぶ。
2. 失敗した場合はそのstatusを返す。
3. `finalize_if_orphaned(inode_num)`を呼び、そのstatusを返す。

dec成功後にfinalizeが失敗しても、減算済みlink countを巻き戻さず、`nlink == 0`のlive orphan残留を許容する。`unlink()`、`creat()`の名前追加失敗、`link()`のrollbackはこの共通経路を使う。mkdirの新規inodeもこの経路で2つのlinkを減らす(21節)。

#### 18.2.1 Filesystem側`close(open_file_t*)`の順序

Filesystem単体APIでは既存契約を維持する。

1. 有効entryを確認し、`ref_count` underflowを防ぐ。
2. `ref_count`を1減らし、まだ0でなければ`OK`を返す。
3. 最後の参照ならinode番号を保持し、entryを解放する。
4. `finalize_if_orphaned()`を呼び、そのstatusを返す。

FD統合経路では`dup()` / `spawn()`時に`dup_file()`を呼ばないため、通常このFS handleの`ref_count`は1のままである。上位`close(fd)`は`FdTable::detach()`と`FileDescription`参照解放を行い、まだ他fdが同じ`FileDescription`を参照している場合はFilesystemへ`close()`しない。最後の`FileDescription`参照が外れた時だけ`RegularFile`から本関数を1回呼ぶ。

entryを先に解放し、自分自身をopenと数えない。同一inodeを参照する別のFS handleが残っていればfinalizeしない。Filesystem側closeはnlinkを変更しない。finalizeが失敗してもFS handleは解放済みとし、復活させず、同じ`open_file_t*`への二度目のcloseを拒否する。

#### 18.2.2 所有関係と永続状態

`Filesystem<Backend>`は固定配列の`OpenFileTable`を所有するが、これはFS側open handleの格納領域であり、OS全体の共有open file descriptionではない。process fd tableは上位`FileDescription`だけを参照し、Filesystemのentryを直接保持しない。`RegularFile`が1個のFS handleを保持し、`FileDescription`から参照される。

`dup()` / `spawn()`では同じ`FileDescription`と`RegularFile`を共有するため、file positionも`FileDescription::offset_`として共有される。FS handleは増えない。別々の`open()`では別々の`FileDescription` / `RegularFile` / FS handleを生成する。

open中unlinkされたinodeは`nlink == 0`になっても`mode`・`size`・`first_cluster`を維持し、既存fdからread/writeできる。対象inodeを指すすべてのFS handleが閉じられた時点でのみorphanのfinalizeを行う。

#### 18.2.3 クラッシュ時の残留

open-but-unlinked状態でクラッシュするとRAMのFilesystem側open handle表は失われ、`nlink == 0`のlive inodeが残留し得る。phase 1ではこの残留を許容し、mount中に推測で回収しない。getや意味検証も、この状態だけで`ECORRUPT`とはしない。回収はfsckまたはphase 2の課題とする。

### 18.3 pathname層への接続

creat、mkdir、unlinkは親directory inodeと単一名を受け取る低レベル操作とする。フルパスを受け取るsyscall層は19節の`resolve_parent()`で親と名前へ変換してから呼ぶ。

---

## 19. パス解決(namei相当)層の設計

### 19.0 設計方針の要点

- 既存の`creat`/`unlink`(16節・18節)は「親dirのinode番号+単一名」を受け取る形のままにし、パス文字列からその形へ変換する層を上に被せる、という構成にする。これにより16節・18節の実装は変更不要。
- `open`(未設計)・`stat`/`access`(未設計)・`chdir`は「最後まで解決してtarget inodeそのものが欲しい」パターンなので、両対応できるよう2つのモード(「親までで止める」/「最後まで解決する」)を持つ1つの内部関数として実装する。
- シンボリックリンクは`00-overview.md`の通り対象外とし、symlink展開を行わない。FAT chainのcycle検査とは区別する。
- 現在のcwd値はProcessが所有し、pathname関数は`cwd_inode_num`を呼び出し側から明示的に受け取る。Filesystemはcwdそのものを保持しないが、`rmdir()`後のorphan寿命判断のためcwd参照数を固定長表で追跡する。詳細は`spec/kernel/mkdir_rmdir_implimentation_spec.md`を参照する。

### 19.1 中核関数`resolve`

`10-pathname-link.md`のAPIに統一する。

```cpp
// component上限と全体長を検査し、単一FS内で解決する。
enum class ResolveMode { Target, Parent };
FsStatus resolve(uint32_t cwd_inode, const char* path, ResolveMode mode,
                 uint32_t* out_inode, char out_name[NAME_MAX + 1]);
FsStatus resolve_path(uint32_t cwd_inode, const char* path, uint32_t* out_inode);
FsStatus resolve_parent(uint32_t cwd_inode, const char* path,
                        uint32_t* out_dir_inode, char out_name[NAME_MAX + 1]);
```

共通のresolveをwrapperから呼ぶ。Targetでは最終targetを、Parentでは親inodeと最後のcomponentを返す。

1. 空文字列は`EINVAL`、NUL終端までの長さが128バイト以上なら`ENAMETOOLONG`を返す。上限を検査せずcopy/走査しない。
2. absolute pathはroot inode 1、relative pathは呼び出し側指定cwdから開始する。cwdの保管はprocess層の責務とする。
3. componentは`/`または終端NULまでの1〜27バイトとして扱う。copy前に長さを検査し、文字コードは解釈しない。連続slashは空componentとして無視する。
4. 現在inodeを取得してdirectoryであることを確認し、通常の`dir_lookup()`で次へ進む。non-directoryは`ENOTDIR`、その他の取得/lookup失敗はそのstatusを返す。
5. Parentでは最終componentのlookup前に止まるが、返す親inodeのdirectory確認を省略しない。最終名の存在確認は呼び出し側操作が行う。
6. Parentへの`/`だけのpath、`foo/`等の末尾slashは`EISDIR`で拒否する。最終componentを判定する前に末尾slashを確認し、slashを飛ばして成功させない。
7. `.`と`..`は通常direntとしてlookupする。rootの`..`はroot自身を指す。symlink展開、cross-filesystem traversalは行わない。

成功後だけ各出力を確定する。creat、mkdir、unlinkはParent、open、stat、access、chdirはTargetを使用する。

### 19.2 syscall層からの呼ばれ方(例)

```cpp
fs::FsStatus sys_creat(uint32_t cwd_inode, const char* path, uint16_t mode, uint32_t* out_inode_num) {
    auto& fs = kernel::root_filesystem();   // root filesystem の Filesystem 実体(kernel_core_spec 2.8)
    uint32_t dir_inode; char name[fs::NAME_MAX + 1];
    fs::FsStatus st = fs.resolve_parent(cwd_inode, path, &dir_inode, name);
    if (st != fs::FsStatus::OK) return st;
    return fs.creat(dir_inode, name, mode, out_inode_num);   // 16.1節
}

fs::FsStatus sys_unlink(uint32_t cwd_inode, const char* path) {
    auto& fs = kernel::root_filesystem();
    uint32_t dir_inode; char name[fs::NAME_MAX + 1];
    fs::FsStatus st = fs.resolve_parent(cwd_inode, path, &dir_inode, name);
    if (st != fs::FsStatus::OK) return st;
    return fs.unlink(dir_inode, name);   // 18.1節
}

fs::FsStatus sys_chdir(uint32_t cwd_inode, const char* path, uint32_t* out_new_cwd) {
    auto& fs = kernel::root_filesystem();
    uint32_t target;
    fs::FsStatus st = fs.resolve_path(cwd_inode, path, &target);
    if (st != fs::FsStatus::OK) return st;
    fs::FileStat info{};
    st = fs.stat(target, &info);
    if (st != fs::FsStatus::OK) return st;
    if ((info.mode & fs::S_IFMT) != fs::S_IFDIR) return fs::FsStatus::ENOTDIR;
    *out_new_cwd = target;   // 呼び出し側(プロセス層)がこれをcwdとして保存する
    return fs::FsStatus::OK;
}
```

上記Filesystem利用例はtarget inodeの解決・種別確認までを示す。実際のsyscallは[`spec/kernel/chdir_implementation_spec.md`](../kernel/chdir_implementation_spec.md)に従い、Process cwd参照をretainしてからProcess管理APIでcwdを交換し、旧参照をreleaseする。cwd値の所有者はProcessであり、Filesystemはorphan寿命判定用の参照数だけを管理する。

### 19.3 `..`の扱いが特別扱いなしで正しく動く

`resolve`は`.`/`..`を通常のdirentとしてlookupする。3.4節・15.5節で確定した通り、mkfsがルートに`.`(自分自身)と`..`(ルートの場合は自分自身)の2エントリを`dirent_t`として書き込んでおり、phase 1の`mkdir`も自身と親を指す2エントリを書き込む(21節)。したがって`..`は単なる`dir_lookup(cur, "..", ...)`として素朴に処理するだけで、「ルートより上には行けない」(`/..` は `/` のまま)という挙動が副作用として自然に成立する。namei層に特別なガードを書く必要がないのは、15.5節までの設計が正しく効いている証拠と言える。

### 19.4 新規エラーコード

| コード | 用途 |
|---|---|
| `FsStatus::ENOTDIR` | パスの途中のコンポーネントがディレクトリでなかった場合 |
| `FsStatus::ENAMETOOLONG` | 1コンポーネントが27バイトを超える、または全体が`PATH_MAX`(128バイト)以上の場合 |

`FsStatus::EINVAL`(17.1節で追加済み)は空文字列パスに、`FsStatus::EISDIR`(16.1.2節で追加済み)は`resolve_parent`が「ファイル名部分を持たないパス」(`"/"`や`"foo/"`)を渡された場合に、それぞれ流用する。

### 19.5 将来のVFSとmountモデル

phase 1ではデバイスごとに独立した`Filesystem<Backend>`インスタンスを使い、各インスタンスのパスは交わらない。rootの`..`はそのFSのroot自身を指す。VFS、統合mount名前空間、`mount(2)`によるcross-filesystem traversalは未設計・対象外とする。

将来統合名前空間を設計する際は、個々のFSのdirent形式とは別に、VFSが次の境界を扱う。

1. 順方向: component lookupで得たinodeがmount pointなら、別FSのrootへ移る。
2. 逆方向: mountされたFSのrootで`..`を解決する際に、親mount側へ戻る。

これらは将来のVFSの検討箇所であり、現在のresolveが変更なしに外部hookを受け入れるというAPI保証ではない。FS内部のresolveを複数インスタンス間で自律的に走査する形へ拡張せず、mount名前空間の責務をVFSへ分離する。

---

## 20. `link(2)`の設計

### 20.1 APIと更新順序

```cpp
// 新しい名前の公開より先にsourceのlink countを増やす。
FsStatus link(uint32_t cwd_inode, const char* oldpath, const char* newpath);
```

`10-pathname-link.md`に従い、次の順序を変更しない。

1. `resolve_path(cwd_inode, oldpath, &old_inode)`でsourceを得る。
2. source inodeを取得し、directoryなら`EISDIR`で拒否する。
3. `resolve_parent()`でnewpathの親と名前を得る。
4. `dir_lookup()`が`OK`なら`EEXIST`で拒否する。`ENOENT`の場合だけ続け、その他の失敗は伝播する。
5. `inode_table_.inc_nlink(old_inode)`を先に行う。上限`UINT16_MAX`では`EMLINK`で拒否する。
6. `dir_add_entry()`で新名を追加する。
7. add失敗時は`drop_link(old_inode)`でrollbackし、操作を停止させた最初のstatusを返す。

rollback成功時は元のnlinkに戻り、通常はnlinkが正なのでfinalizeはinode内容を変えない。rollback失敗時は新しいdirentを残さず、増加したlink countの残留を許容する。成功とrollbackのどちらもctime規約に従う。

### 20.2 orderingと対象範囲

dirent追加より先にlink countを増やし、公開済みの名前に対する過少countを防ぐ。add失敗時は共通helperで増分を戻し、失敗してもdangling nameよりリークを優先する。

creatは既存通常ファイルをtruncateするが、linkは既存destinationを`EEXIST`で拒否する。directory sourceはcreat/write/unlinkのdirectory拒否と合わせて`EISDIR`とする。cross-device linkは対象外であり、将来必要となる`EXDEV`等の調停はVFSの設計で扱う。

---

## 21. `mkdir()`の設計

### 21.1 phase 1の範囲とAPI

`11-mkdir.md`に従い、phase 1でsubdirectoryを作成する。低レベルAPIは親directory inodeと単一名を受け取り、フルパスの呼び出し側は`resolve_parent()`後に呼ぶ。

```cpp
// 子directoryを完全に初期化してから親に名前を公開する。
FsStatus mkdir(uint32_t dir_inode_num, const char* name,
               uint16_t mode, uint32_t* out_inode_num);
```

rename、空間回収、fsckは対象外とする。空directoryは`rmdir()`で削除でき、`unlink()`はdirectoryを常に`EISDIR`で拒否する。modeは許可9ビットだけ保持し、umaskを適用しない。出力inode番号は全成功後だけ確定する。

### 21.2 事前検査とstatus

状態変更前に次を検査する。

| 条件 | status |
|---|---|
| 未mount | `ENODEV` |
| null name / null出力、空名、`/`を含む名前 | `EINVAL` |
| 27バイトを超える名前 | `ENAMETOOLONG` |
| 親が通常ファイル | `ENOTDIR` |
| 親が未使用・削除済みinode、dangling dirent、directory構造不整合 | `ECORRUPT` |
| 同名の通常ファイル/directory、正常な親の`.` / `..` | `EEXIST` |
| 親のnlinkが`UINT16_MAX` | `EMLINK` |
| inode/cluster/directory領域の枯渇 | `ENOSPC` |
| block I/O失敗 | `EIO`またはBackend由来のstatus |

親はlive directoryでなければならない。lookupの`ENOENT`だけを新規作成の開始条件とし、その他の失敗を伝播する。

### 21.3 成功時の永続状態

| フィールド | 値 |
|---|---|
| mode | `S_IFDIR | (mode & 0777)` |
| nlink | 2(親からの名前と自身の`.`) |
| uid / gid | 0 |
| size | `FS_BLOCK_SIZE(512)` |
| first_cluster | 新規割当済みdirectory data cluster |
| atime / ctime / mtime | 作成時のprovider由来`now()` |

data clusterは全ゼロ初期化し、先頭の`.`は自身、2番目の`..`は親inodeを指す。残りのslotは全ゼロとする。親のnlinkは子の`..`参照を反映して1増える。

### 21.4 更新順序

1. 引数、名前、親inodeを検証する。
2. 同名が存在しないことをlookupで確認する。
3. 親nlinkの増加が可能であることを確認する。
4. 新規inode番号を単調allocする。
5. 全フィールドを初期化した未公開directory inodeを、`nlink = 2`、`size = 0`、`first_cluster = CLUSTER_NONE`で永続化する。
6. directory data clusterを1個allocし、全ゼロで初期化する。
7. directory層のhelperまたは同等の内部経路で`.`と`..`をencodeし、sizeとfirst_clusterを含む完成状態を永続化する。
8. `inc_nlink(parent)`で親のlink countを先に増やす。
9. `dir_add_entry(parent, name, new_inode)`で作成済みdirectoryを公開する。
10. 全成功後だけ出力を確定する。

ユーザ向けwriteでdirectory内容を変更せず、上位層からFAT/inode-table offset/物理blockを直接操作しない。初期化途中の子を親の名前から到達可能にしない。各失敗は次のrollbackを行う。

### 21.5 rollback

- 初回inode永続化に成功する前の失敗ではdirentを作らず、drop_linkを呼ばない。
- nlink 2のinode永続化後に初期化が失敗した場合は、`drop_link(new_inode)`を2回呼び、正常時は最後に`S_IUNUSED`へfinalizeする。1回目が失敗したら2回目を行わない。
- 親のinc失敗時も新規directoryを同じ2回のdropでfinalizeする。
- 親inc後のadd失敗では、先に新規directoryを2回のdropでfinalizeする。その成功後だけ`dec_nlink(parent)`で増分を戻す。
- 新規directoryのrollback失敗時は、新規inodeと親link count増分のリークを許容する。親を先に減算して過少countを残さない。
- 親decのrollback失敗時は過剰countを残し、他metadataを変更しない。
- rollbackのstatusで、操作を停止させた最初のstatusを上書きしない。inodeを`S_IFREE`へ戻さず、clusterとinode slotを再利用しない。

この順序を共通helperに集約し、呼び出し側ごとに部分rollbackを複製しない。

### 21.6 必須検証と完了条件

基本作成、permission、nlink 2、size 512、時刻、`.`/`..`、親nlink増分、空slotゼロ、子の下でのcreat/mkdir、absolute/relative解決を確認する。

null/空名/slash/27・28バイト名、不正な親inode、既存名と`.`/`..`、親nlink上限、inode-table境界と拡張、FAT/inode枯渇、親の空slot再利用と拡張を確認する。

dangling direntとchain破損を停止し、inode永続化、alloc、zero-fill、`.`/`..`、親inc、名前追加、子/親rollbackの各境界へread/write失敗を注入する。失敗時の出力不変、未初期inodeへの到達不能、成功したrollbackでの`S_IUNUSED`、最初のstatus保持とリーク側停止を確認する。これらを`11-mkdir.md`の完了条件として満たす。
