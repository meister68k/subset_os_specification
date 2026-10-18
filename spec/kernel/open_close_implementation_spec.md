# `creat` / `open` / `close` システムコール実装仕様

## 1. 目的

本書は、Ancient UNIX風サブセットOSにおける通常ファイル用`creat()` / `open()`と共通`close()`の実装指示を規定する。AIエージェントは本書を直接の実装契約として扱い、既存のFD共通基盤、`FileDescription`、`FdTable`、Filesystem APIを壊さず接続すること。

本段階では、通常ファイルを用いた`open()`テストに必要なファイルを作成するため、`creat()`を`open()`より先に実装する。`creat()`専用の資源管理経路は作らず、後続の`open(O_CREAT | O_TRUNC)`と共有できる内部処理を先行実装すること。

本段階のゴールは、疑似ユーザープログラムから次の経路を成立させることである。

```text
creat(path, mode)
  -> pathname resolve/create
  -> Filesystem open handle
  -> RegularFile
  -> FileDescription(O_WRONLY)
  -> process FdTable

open(path, flags, mode)
  -> pathname resolve/create
  -> Filesystem open handle
  -> RegularFile
  -> FileDescription
  -> process FdTable

read/write(fd, ...)
  -> FdTable
  -> FileDescription (flags)
  -> FileInterface::read()/write()
  -> RegularFile (offset)
  -> Filesystem::read_at()/write_at()

close(fd)
  -> FdTable::detach()
  -> release_file_description()
  -> 最終参照の場合のみ RegularFile::release()
  -> Filesystem::close()
```

## 2. 前提と設計原則

- fdはプロセスごとの`FdTable`で管理する。
- システム全体のopen file descriptionはFD層の`FileDescription`を唯一の正本とする。
- `FileDescription`はopen flagsとfd参照数を保持し、I/O対象への共通参照を管理する。
- 通常ファイルのfile positionは`RegularFile`が保持する。`dup()`や将来の`spawn()`で同一`FileDescription`を共有する場合は同一`RegularFile`も共有されるため、file positionも共有される。
- Filesystem既存の`open_file_t`は当面API互換のため残すが、FD統合経路ではその`offset`と`ref_count`を共有状態の正本として使用しない。
- 1回の`open()`につきFilesystemの`open_file()`を1回呼び、そのhandleを1個の`RegularFile`が保持する。
- `dup()`や将来の`spawn()`によるfd継承ではFilesystemの`dup_file()`を呼ばない。同一`FileDescription`を共有する。
- 動的メモリ確保を新規に導入しない。`RegularFile`は固定長テーブルから確保する。
- syscall層は薄いラッパとし、fd表、`FileDescription`、Filesystem内部状態を直接破壊しない。
- RTTI、`dynamic_cast`、`typeid`は使用しない。


### 2.1 openフラグの値

本仕様で使用する`open()`フラグはOS固有ABIとして次の値に固定する。

```cpp
constexpr uint32_t O_RDONLY  = 0x0000u;
constexpr uint32_t O_WRONLY  = 0x0001u;
constexpr uint32_t O_RDWR    = 0x0002u;
constexpr uint32_t O_ACCMODE = 0x0003u;
constexpr uint32_t O_CREAT   = 0x0040u;
constexpr uint32_t O_TRUNC   = 0x0200u;
```

- `O_RDONLY` / `O_WRONLY` / `O_RDWR`は相互排他的なアクセスモードであり、`flags & O_ACCMODE`で判定する。
- `O_RDONLY == 0`であるため、`flags & O_RDONLY`による判定は禁止する。
- `O_CREAT` / `O_TRUNC`はアクセスモードとは独立したビットフラグであり、`flags & O_CREAT`、`flags & O_TRUNC`で判定する。値はLinuxのasm-generic系ABIに合わせ、`O_CREAT = 0x0040`、`O_TRUNC = 0x0200`とする。
- 未定義ビットが立っている場合は黙って無視せず`-1`とする。
- これらの定数はユーザ空間とカーネルで共有する共通ヘッダに置き、syscall実装内へ数値を直書きしない。

本段階では`O_APPEND`等の未実装フラグは定義しない。必要になった時点で、既存値と衝突しない別ビットを追加する。

## 3. `FileInterface` の追加契約

共通I/O境界には、通常ファイル固有のfile positionを持ち込まない。
`FileInterface`はbyte streamとしての`read()` / `write()`を基本操作とし、
seek可能な対象だけが`seek()`をoverrideする。

```cpp
class FileInterface {
public:
    virtual ~FileInterface() = default;

    virtual void init() {}
    virtual int32_t read(void* buf, uint32_t count) = 0;
    virtual int32_t write(const void* buf, uint32_t count) = 0;
    virtual int32_t seek(int32_t offset, uint32_t whence) { return -1; }
    virtual int32_t release() { return 0; }
};
```

`read()` / `write()`にはoffset引数を持たせない。通常ファイルのfile positionは
`RegularFile`内部で管理し、Filesystemとの境界でのみ
`Filesystem::read_at()` / `write_at()`へ明示的なoffsetを渡す。

`seek()`はこの段階で共通virtual hookとして定義する。
seek不能な対象は既定実装の`-1`をそのまま使用してよい。
`ConsoleDevice`や将来の`Pipe`は原則としてoverrideしない。
`RegularFile`は通常ファイル用のfile positionを変更するため`seek()`をoverrideする。

`release()`は当該`FileDescription`の最終参照が外れた時に1回だけ呼ばれる
終端cleanup hookである。常駐デバイスでは既定の空実装を利用できる。

`FileDescription::read()` / `write()`はopen flagsによるアクセス可否を検査した後、
対象の`FileInterface::read()` / `write()`へ委譲する。
`FileDescription`自身はfile positionを保持・更新しない。

## 4. `RegularFile`

### 4.1 役割

`RegularFile`は1回のFilesystem openを`FileInterface`へ接続するアダプタであり、
通常ファイル固有のfile positionの正本を保持する。

概念上、最低限以下の状態を持つ。

```cpp
class RegularFile final : public FileInterface {
public:
    int32_t read(void* buf, uint32_t count) override;
    int32_t write(const void* buf, uint32_t count) override;
    int32_t getdents(void* buf, uint32_t count) override;
    int32_t seek(int32_t offset, uint32_t whence) override;
    int32_t release() override;

private:
    FilesystemType* fs_ = nullptr;      // 実際の型に合わせる
    fs::open_file_t* file_ = nullptr;
    uint32_t offset_ = 0;
    bool active_ = false;
};
```

実際のFilesystem型がテンプレート等で異なる場合は既存コードに合わせること。
不要な汎用化を追加しない。

### 4.2 固定長テーブル

`RegularFile`実体は固定長テーブルで保持する。空き判定と確保・解放用の小さな内部APIを設ける。

- 空きが無ければ`open()`は`-1`。
- 同じslotをACTIVEのまま再利用してはならない。
- `RegularFile`への参照を`FileDescription`へ渡した後、その寿命は最終`release_file_description()`まで保証する。
- 新規open時の`offset_`は0とする。

### 4.3 I/Oとfile position

`RegularFile::read()`は保持する`open_file_t`の`inode_num`と現在の`offset_`を使って、
Filesystemの`read_at(inode_num, offset_, ...)`を呼ぶ。

`RegularFile::getdents()`は`getdents_implementation_spec.md`に従い、同じ`offset_`を使って
Filesystem公開APIからdirectory属性を確認し、32-byte dirent slotを列挙する。

`RegularFile::write()`も同様に、
Filesystemの`write_at(inode_num, offset_, ...)`を呼ぶ。

Filesystemから`{n, status}`が返った場合、次の規則で`int32_t`へ変換する。

- `n > 0`: `offset_`を`n`だけ進め、`n`を返す。statusがエラーでも成功済みbyte数を優先する。
- `n == 0 && status == OK`: `0`を返し、`offset_`は変更しない。
- `n == 0 && status != OK`: `-1`を返し、`offset_`は変更しない。

directoryへのwrite拒否など、Filesystemの上位`write(open_file_t*)`だけが行っている検査を
`write_at()`が行わない場合は、`RegularFile`側で同等の検査を行うこと。
Filesystem内部層へ直接侵入してはならず、公開APIまたは既存の公開アクセサで
実現できない検査が必要なら、実装を推測せず仕様矛盾として報告すること。

### 4.4 `seek()`

`RegularFile::seek()`は通常ファイルの`offset_`を変更するための共通I/O境界である。
この段階ではvirtual interfaceを確定し、完全な`lseek()` semanticsの実装は後続段階としてよい。

少なくとも、seek不能な対象では`FileInterface`の既定実装が`-1`を返し、
通常ファイルだけがoverrideする構造を維持する。

`SEEK_SET` / `SEEK_CUR` / `SEEK_END`の詳細な境界条件、負offset、EOF超過の扱いは
[`lseek_implementation_spec.md`](lseek_implementation_spec.md)に従う。

### 4.5 `dup()` / `spawn()`時のoffset共有

`dup()`や将来の`spawn()`は新しい`RegularFile`を生成せず、
同一`FileDescription`を共有する。

同一`FileDescription`は同一`RegularFile`を参照するため、
通常ファイルの`offset_`も共有される。

一方、同じpathnameを別々に`open()`した場合は別々の`RegularFile`を生成し、
それぞれ独立した`offset_`を持つ。

### 4.6 `release()`

`RegularFile::release()`は保持するFilesystem handleに対して
Filesystemの`close(open_file_t*)`をちょうど1回呼ぶ。

その後は結果にかかわらず`file_`等を無効化し、
`offset_`を0へ戻し、RegularFile slotを再利用可能に戻す。
Filesystem仕様上、`close()`がエラーを返した場合でもopen entry自体は解放済みであり、
同じhandleへ二度目の`close()`を行ってはならない。

Filesystem `close()`が`OK`なら`0`、それ以外なら`-1`を返す。

## 5. `release_file_description()`

シグネチャを次の形とする。

```cpp
int32_t release_file_description(FileDescription& desc);
```

- 参照数が2以上なら1減らして`0`を返し、`target_->release()`を呼ばない。
- 最終参照なら一時的にRELEASINGへ遷移し、`target_->release()`を1回だけ呼ぶ。
- cleanup後はtargetとflagsを初期化しFREEへ戻す。通常ファイルのoffsetは`RegularFile::release()`側で初期化する。
- targetの`release()`が失敗した場合もfd/open状態は復活させない。`-1`を返す。

この戻り値を`close()` syscallの結果へ反映する。

## 6. `close()` 実装

### 6.1 処理

カーネル内部のclose処理は次の順序とする。

```text
1. current processのFdTableを取得
2. fd_table.detach(fd)
3. nulloptなら -1
4. release_file_description(desc)
5. その戻り値を0/-1として返す
```

syscall層は可能な限りこの内部APIを呼ぶだけとする。

### 6.2 必須セマンティクス

- 無効fd、範囲外fd、未使用fdは`-1`。
- 成功時は`0`。
- `detach()`後は、最終cleanupが`-1`でもfdは閉じられた状態である。
- 二重closeは`-1`。
- 同じ`FileDescription`を複数fdが共有している場合、一方のcloseではFilesystem handleを閉じない。
- 最後のfdをcloseした時だけ`RegularFile::release()`→Filesystem `close()`が発生する。
- fd 0/1/2のConsoleDeviceも同じ経路でcloseできる。ConsoleDeviceの`release()`は何もしないため、グローバルなコンソール実体は破棄しない。

## 7. `creat()` 実装

### 7.1 syscall外部仕様

```cpp
int32_t creat(const char* path, uint32_t mode);
```

成功時は書込み用にopenされた未使用の最小fd番号、失敗時は`-1`を返す。意味上は次と等価とする。

```cpp
open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)
```

ただし実装順序上は`creat()`を先に完成させる。後から`open()`を実装するときに`creat()`の処理を複製してはならない。

### 7.2 pathname解決と作成

1. current processのcwdを取得する。cwd管理が未完成の場合の暫定`ROOT_INODE`使用規則は後述の`open()`と同じ1か所に局所化する。
2. `resolve_parent(cwd_inode, path, &parent_inode, name)`で親directoryと最終componentを得る。
3. `Filesystem::creat(parent_inode, name, mode, &inode)`を呼ぶ。
4. Filesystem側`creat()`の契約に従い、不存在なら新規作成、既存通常ファイルならtruncate、directory等はエラーとする。
5. 成功後、得られたinodeを通常ファイルfd化の共通内部処理へ渡す。

`Filesystem::creat()`が成功した後に後段のfd資源確保が失敗しても、作成またはtruncateはrollbackしない。永続FS変更とRAM上のopen資源確保は分離する。


### 7.3 openを伴わない作成内部API

`creat()`実装の途中段階として、pathname解決からinode作成・directory entry新設までを行い、open処理を行わないカーネル内部APIを用意する。

概念上のシグネチャは次の形とする。

```cpp
FsStatus kernel_creat_file(
    Process& process,
    const char* path,
    uint32_t mode,
    uint32_t* out_inode);
```

実際の命名・namespaceは既存コード規約に合わせてよいが、責務は次に限定する。

```text
current processのcwd取得
    -> Filesystem::resolve_parent()
    -> Filesystem::creat()
    -> inode番号を返す
```

このAPIは次を行ってはならない。

- `Filesystem::open_file()`
- `RegularFile` slot確保
- `FileDescription`確保
- `FdTable`登録

`Filesystem::creat()`の契約をそのまま利用するため、不存在なら新規作成し、既存通常ファイルならtruncateし、directory等はエラーとする。

`out_inode == nullptr`などFilesystem側契約で拒否される条件は、そのstatusをそのまま返してよい。syscall形式の`-1`への変換は上位の`kernel_creat()` / `sys_creat()`で行う。

このAPIの目的は、

```text
pathname解決
-> inode作成または既存通常ファイルtruncate
-> directory entry作成/維持
```

までを、fd化処理から独立して検証可能にすることである。

`kernel_creat_file()`成功後に、後段のopen資源確保が失敗してもFilesystem上の作成/truncateをrollbackしない。永続FS変更とRAM上open資源管理の境界はこのAPIと後段fd化処理の間に置く。

なお、このAPIは`creat(2)` semanticsを持つ。将来の`open(O_CREAT)`一般処理に無条件で再利用してはならない。`open(path, O_CREAT)`は既存ファイルをtruncateしないため、`open()`側は別途flagsに応じて作成/既存利用/truncateを分岐すること。

### 7.4 fd化の共通内部処理

`creat()`と後続の`open()`は、inode確定後の処理を共通化する。概念上、次のような内部APIを用意してよい。

```cpp
int32_t open_regular_inode(
    Process& process,
    uint32_t inode,
    uint32_t flags);
```

実際の命名・namespaceは既存コード規約に合わせる。重要なのは、次の処理を`sys_creat()`と`sys_open()`へ重複実装しないことである。

```text
A. Filesystem::open_file(inode)
B. RegularFile slot確保
C. alloc_file_description(regular_file, flags)
D. FdTable::alloc(desc)
```

`creat()`からは`flags = O_WRONLY`としてこの共通処理を呼ぶ。`O_CREAT` / `O_TRUNC`はpathname側の作成・truncateを指示するsyscall入力flagであり、作成完了後の`FileDescription`に必要なアクセスモードは`O_WRONLY`である。

### 7.5 rollback

fd化段階のrollbackは`open()`と完全に共通とする。

- A失敗: 何も解放せず`-1`。
- B失敗: Aで得たhandleをFilesystem `close()`して`-1`。
- C失敗: `RegularFile::release()`でFilesystem handleまで解放して`-1`。
- D失敗: `release_file_description(desc)`のみ呼ぶ。追加の`RegularFile::release()` / Filesystem `close()`は禁止。

### 7.6 syscall層

`sys_creat()`は`kernel_creat()`を呼ぶ薄いラッパとする。`kernel_creat()`は、前節の`kernel_creat_file()`とinode確定後の共通fd化内部APIを組み合わせる。`FdTable`、`FileDescription::ref_count_`、Filesystem内部表をsyscall層から直接操作しない。

概念上は次の形とする。

```cpp
int32_t kernel_creat(Process& process, const char* path, uint32_t mode)
{
    uint32_t inode = 0;
    const FsStatus st = kernel_creat_file(process, path, mode, &inode);
    if(st != FsStatus::OK) {
        return -1;
    }

    return open_regular_inode(process, inode, O_WRONLY);
}

int32_t sys_creat(const char* path, uint32_t mode)
{
    return kernel_creat(current_process(), path, mode);
}
```

既存のProcessManager等に同じ責務の内部APIがある場合は、新しい並行APIを増やさず既存構造へ実装する。

### 7.7 実装順序

本仕様での推奨順序は次の通り。

1. `close()`を完成させる。
2. `kernel_creat_file()`を実装し、pathname解決・inode作成・directory entry更新までを単独で検証する。
3. `RegularFile`と通常ファイルfd化の共通内部処理を実装する。
4. `kernel_creat()` / `sys_creat()`を接続し、`creat -> write -> close`を通す。
5. `open()`を同じ共通fd化内部処理へ接続する。
6. `open -> read -> close`を通し、作成した内容を読み戻す。

## 8. `open()` 実装

### 8.1 syscall外部仕様

```cpp
int32_t open(const char* path, uint32_t flags, uint32_t mode);
```

成功時は未使用の最小fd番号、失敗時は`-1`を返す。

`mode`は`O_CREAT`で新規作成する場合のみ意味を持つ。

### 8.2 flags

少なくとも既存ヘッダで定義済みの以下を扱う。

- `O_RDONLY`
- `O_WRONLY`
- `O_RDWR`
- `O_CREAT`
- `O_TRUNC`

アクセスモードが不正な組み合わせなら`-1`。

`O_TRUNC`は書込み可能なopenでのみ許可する。その他の未対応flagが立っている場合は黙って無視せず`-1`にする。

### 8.3 pathnameとcwd

pathname解決にはFilesystemの`resolve_path(cwd_inode, path, &inode)`を使用する。

完全なprocess cwd管理がまだ無い段階では、現在の疑似プロセスで採用しているcwd値を使用する。cwdがまだ保持されていない場合のみ、暫定的に`ROOT_INODE`を使う。この暫定処理は1か所に局所化し、後のProcess cwd導入で差し替え可能にする。

### 8.4 既存ファイルを開く場合

1. `resolve_path()`でinodeを取得する。
2. `O_TRUNC`が無ければそのinodeを使用する。
3. `O_TRUNC`がある場合は`resolve_parent()`で親inodeと最終名を取得し、Filesystem `creat()`を使って既存通常ファイルをtruncateし、返されたinodeを使用する。
4. directoryに対するtruncate等、Filesystemが拒否する条件は`-1`へ変換する。

### 8.5 `O_CREAT`

`resolve_path()`が`ENOENT`の場合のみ、`resolve_parent()`で親と最終名を取得し、Filesystem `creat(parent, name, mode, &inode)`で作成する。

既存ファイルに`O_CREAT`だけが指定された場合はtruncateしない。`O_CREAT | O_TRUNC`なら前項のtruncate経路を使う。

`resolve_path()`が`ENOENT`以外のエラーを返した場合は作成へ進まず`-1`。

### 8.6 resource確保順序

inode確定後は必ず次の順序で確保する。

```text
A. Filesystem::open_file(inode)
B. RegularFile slot
C. alloc_file_description(regular_file, flags)
D. FdTable::alloc(desc)
```

別々の`open()`は同じpathnameでもA〜Dを別々に実行する。

### 8.7 rollback

途中失敗時は次の規則を厳守する。

- A失敗: 何も解放せず`-1`。
- B失敗: Aで得たhandleをFilesystem `close()`して`-1`。
- C失敗: RegularFileがまだFileDescriptionに所有されていないため、RegularFileの`release()`でA/Bをまとめて解放して`-1`。
- D失敗: `release_file_description(desc)`のみ呼ぶ。これがRegularFileとFilesystem handleまで解放する。追加で`RegularFile::release()`やFilesystem `close()`を呼んではならない。

作成またはtruncateそのものが成功した後、後段のresource確保に失敗しても、ファイル作成/truncateはrollbackしない。`open()`のresource確保失敗とFilesystem上の永続変更は分離する。

## 9. syscall層の薄さ

`sys_open()` / `sys_close()`にはresource管理ロジックを重複させない。可能なら、process/fd/filesystemを扱う内部APIへ委譲する。

例:

```cpp
int32_t kernel_open(Process& process, const char* path,
                    uint32_t flags, uint32_t mode);
int32_t kernel_close(Process& process, int32_t fd);
```

実際の命名・namespaceは既存コード規約に合わせる。既存の`ProcessManager`内部APIが同じ責務を持つ場合は新規の並行APIを作らずそこへ実装する。

## 10. 必須テスト

少なくとも以下を自動テストする。

1. `close(1)`成功後、`write(1, ...)`が失敗する。
2. 同じfdの二重closeが失敗する。
3. 共有`FileDescription`を2fdから参照し、一方をcloseしても他方が有効。
4. 最後のcloseでのみ`RegularFile::release()`が1回呼ばれる。
5. 既存ファイルを`O_RDONLY`でopenしreadできる。
6. 既存ファイルを`O_WRONLY`でopenしwriteできる。
7. read-only fdへのwrite、write-only fdへのreadが失敗する。
8. 同じファイルを2回openすると別々の`FileDescription`となりoffsetが独立する。
9. `O_CREAT`で不存在ファイルを作成できる。
10. 既存ファイルへの`O_CREAT`だけではtruncateしない。
11. `O_TRUNC`で既存通常ファイルがsize 0になる。
12. RegularFile表、FileDescription表、fd表の各枯渇時にリークしない。
13. fd割当失敗時にFilesystem handleが1回だけcloseされる。
14. `kernel_creat_file()`で不存在ファイルを作成でき、inode番号を取得できる。
15. `kernel_creat_file()`で作成したファイルをdirectory lookupで確認できる。
16. `kernel_creat_file()`で既存通常ファイルを指定するとtruncateされる。
17. `kernel_creat_file()`実行だけでは`Filesystem::open_file()`、`RegularFile`、`FileDescription`、`FdTable`を消費しない。
18. `creat()`で不存在ファイルを作成し、返されたfdへ`write()`できる。
19. `creat()`で既存通常ファイルを開くとtruncateされる。
20. `creat()`が返す`FileDescription`は`O_WRONLY`であり、`read()`は失敗する。
21. `creat()`後のfd資源確保失敗時、作成/truncate済みFilesystem状態をrollbackせず、RAM上のopen資源だけを漏れなく解放する。
22. `creat()`と`open()`がinode確定後のfd化処理を同じ内部APIで共有している。
23. ConsoleDevice経路の既存`read/write`テストが回帰しない。


## 11. 実装完了条件

疑似ユーザープログラムから最低限次が通ること。

```cpp
int32_t fd = btx::sys_creat("/test.txt", 0644);
// fd >= 0

btx::sys_write(fd, "Hello, World\n", 13);
// 13

btx::sys_close(fd);
// 0

fd = btx::sys_open("/test.txt", O_RDONLY, 0);
// fd >= 0

char buf[32];
int32_t n = btx::sys_read(fd, buf, sizeof(buf));
// n == 13, 内容を確認

btx::sys_close(fd);
// 0

btx::sys_read(fd, buf, sizeof(buf));
// -1
```

これにより、`creat -> write -> close -> open -> read -> close`の一連の通常ファイル経路を検証する。

この時点で、標準入出力と通常ファイルが同じ`FdTable -> FileDescription -> FileInterface`経路を通り、通常ファイルだけが`RegularFile -> Filesystem`へ接続されていること。
