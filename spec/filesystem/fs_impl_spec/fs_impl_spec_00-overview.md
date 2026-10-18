# 00 — 概要とオンディスクレイアウト

> 共通規約はルートの `AGENTS.md` を参照する。本書はレイヤー固有事項のみを定義する。矛盾時は `AGENTS.md` を優先し、仕様矛盾として報告する。

## 1. 目的と範囲

- UNIX V7風の最小ファイルシステムを、まずホストOS上のC++ライブラリとして実装する。
- 最終ターゲットはCortex-M0またはMotorola 68000を想定する。
- ブロックデバイス、FAT、inode、ディレクトリ、ファイル操作を独立レイヤーとして実装する。
- 実装はすべて`namespace fs`内に置く(`fs/src/*.h`)。各層仕様のクラス定義・メンバ宣言・擬似コード例は`namespace fs`を省略して示し、FSの外側から書く利用例(例: `11-mkdir.md` §7)のみ`fs::`で修飾する。
- phase 1は単純さと検証可能性を優先し、空き領域の再利用やfsckは行わない。
- シンボリックリンク、VFS、複数ファイルシステム間のパス移動は対象外とする。

## 2. 基本モデル

- データクラスタはFAT16相当の単方向チェーンで管理する。
- メタデータはinode、名前はdirentに分離する。
- 複数のdirentが同じinodeを指すことでハードリンクを表現する。
- inodeテーブル自身もFATチェーンを持つ特殊ファイルとする。
- ディレクトリは固定長dirent列を内容とする通常ファイルとして扱う。
- inode番号0は未使用番兵、inode番号1はルート固定とする。
- inodeテーブルの先頭クラスタはクラスタ0固定とする。

## 3. 物理レイアウト

```text
[0 .. fs_start_block-1]                         予約領域（FS仕様外）
[fs_start_block]                                superblock
[fat_start_block .. fat_start_block+fat_blocks) FAT
[data_start_block .. total_blocks)              データクラスタ領域
```

- ブロックとクラスタのサイズは512バイト固定とする。
- 予約領域は0ブロックでもよい。
- MBR等を置く場合、その内容は本仕様の対象外とする。
- `fs_start_block`は`mount()`の呼び出し側が与える。
- データ領域のクラスタ番号0は物理ブロック`data_start_block`に対応する。
- クラスタ番号から物理ブロックへの変換は範囲検査後に行う。

## 4. 基本定数

| 定数 | 値・意味 |
|---|---|
| `FS_BLOCK_SIZE` | 512 |
| `FS_MAGIC` | `0x42545458` (文字列`BTTX`, リトルエンディアンのシステムで数値として扱うときは `0x58545442` となることに注意すること) |
| FATエントリ幅 | 2バイト |
| `FAT_FREE` | `0x0000` |
| `FAT_EOF` | `0xFFFF` |
| `INODE_NUM_NONE` | 0 |
| `ROOT_INODE` | 1 |
| `CLUSTER_NONE` | `0xFFFFFFFF` |
| `NAME_MAX` | 27バイト |
| dirent名領域 | `char[28]` |
| `PATH_MAX` | 128バイト |
| `INODE_SIZE` | 64バイト（phase 1固定） |
| `DIRENT_SIZE` | 32バイト |
| `ROOT_MODE` | `S_IFDIR | 0777` (`040777`) |
| `ROOT_NLINK` | 2 |

## 5. superblockの永続フィールド

| フィールド | 型 | 契約 |
|---|---:|---|
| `magic` | `uint32_t` | フォーマット識別子 |
| `block_size` | `uint32_t` | 512固定 |
| `total_blocks` | `uint32_t` | デバイス先頭から数えた総ブロック数 |
| `fs_start_block` | `uint32_t` | FS開始ブロック |
| `fat_start_block` | `uint32_t` | FAT開始ブロック |
| `fat_blocks` | `uint32_t` | FAT長 |
| `data_start_block` | `uint32_t` | データ領域開始ブロック |
| `data_blocks` | `uint32_t` | データクラスタ数 |
| `inode_size` | `uint32_t` | inodeレコード長 |
| `inode_table_start_cluster` | `uint32_t` | phase 1では0固定 |
| `root_inode` | `uint32_t` | 1固定 |

- superblockは不変なフォーマット情報だけを保持する。
- allocatorのヒント、inodeテーブル末尾、容量等は永続化しない。
- 動的情報はmount時に`mount_state_t`へ再構築する。

## 6. inodeの永続フィールド

| フィールド | 型 | 契約 |
|---|---:|---|
| `mode` | `uint16_t` | 種別・予約・許可ビット |
| `nlink` | `uint16_t` | ハードリンク数 |
| `uid` | `uint16_t` | phase 1では0 |
| `gid` | `uint16_t` | phase 1では0 |
| `size` | `uint32_t` | 通常ファイルは論理長、directoryは確保済み容量 |
| `atime` | 64bit領域 | 下位32bitを使用、上位0 |
| `ctime` | 64bit領域 | 下位32bitを使用、上位0 |
| `mtime` | 64bit領域 | 下位32bitを使用、上位0 |
| `first_cluster` | `uint32_t` | 未割当時`CLUSTER_NONE` |

- 種別値は`S_IFREG=0100000`、`S_IFDIR=0040000`とする。
- 未使用は`S_IFREE=0000000`、削除済みは`S_IUNUSED=0170000`とする。
- modeの予約3ビットは0固定とする。
- 許可9ビットは`creat(mode)`の値を保持し、phase 1ではumaskを適用しない。

## 7. direntの永続フィールド

| フィールド | 型 | 契約 |
|---|---:|---|
| `inode_num` | `uint32_t` | 先頭配置、0なら空き |
| `name` | `char[28]` | NUL終端の不透明なバイト列、最大27バイト |

- 未使用スロットは全ゼロで表現できる。
- `inode_num`を固定長ヘッダとして先頭に置き、名前は末尾に置く。
- 文字コードは規定せず、NUL以外の各バイトをそのまま保持・比較する。
- pathnameはNUL終端文字列なので、NULを名前の一部として表現しない。
- `/`はpathname区切りであり、pathname層が名前componentへ含めない。
- 使用中direntの名前は1〜27バイトとする。空名は未使用slotの表現にのみ使用する。

## 8. 所有関係

```text
Filesystem<Backend>
├── BlockDevice<Backend> device_
├── superblock_t superblock_
├── mount_state_t mount_state_
├── Fat<Backend> fat_
├── InodeTable<Backend> inode_table_
└── OpenFileTable open_files_
```

- 各FSインスタンスは状態を完全に分離する。
- グローバルまたはnamespace staticな可変FS状態を置かない。
- 依存先への参照はコンストラクタで明示的に渡す。

## 9. 実装順序

1. シリアライズ
2. ブロックデバイス
3. FAT
4. mountとsuperblock
5. inodeテーブル
6. ディレクトリ
7. creat/write
8. read/lseek
9. unlink/close
10. pathname/link
11. syscall接続

各段階で下位層の単体テストを完了してから上位層へ進む。
