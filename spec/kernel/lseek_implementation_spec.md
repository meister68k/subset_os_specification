# `lseek()` システムコール実装仕様

## 1. 目的と範囲

本書はopen済みfdのfile positionを変更する`lseek()` syscallを定める。syscallはfdと`whence`を検証し、`FileDescription`から`FileInterface`のseek hookへ委譲する。通常ファイル・directoryのpositionは`RegularFile`が所有する。

phase 1ではbyte単位の32-bit signed positionを扱う。EOFより後へのseekを許可する。pipe、console等、seek不能なI/O対象へのseekは失敗する。hole punching、extent操作、directory固有の再配置規則は設けない。

## 2. ABI

```cpp
int32_t sys_lseek(int32_t fd, int32_t offset, int32_t whence);
```

- `whence`は`SEEK_SET`、`SEEK_CUR`、`SEEK_END`の値を受け取る。
- 成功時は新しいpositionを`int32_t`で返す。結果は`0..INT32_MAX`。
- 失敗時は`-1`を返す。現行syscall ABIにerrno設定はないため、新たなglobal errno機構を追加しない。
- syscall番号や、未登録のユーザーラッパ番号は本書では割り当てない。
- Filesystem、FD層内部では既定の型/result表現を使い、OS `errno`はsyscall境界以外で使用しない。

## 3. 処理経路

```text
btx::sys_lseek(fd, offset, whence)
  -> syscall dispatcher
  -> kernel::sys_lseek(fd, offset, whence)   // ABI引数と結果の変換
  -> current Process::FdTable::get(fd)
  -> FileDescription::seek(offset, whence)
  -> FileInterface::seek(offset, whence)
  -> RegularFile::seek(offset, whence)
       -> Filesystem公開APIで必要なinode sizeを取得
       -> positionを検証してRegularFile::offset_を更新
```

syscall handlerは`RegularFile`の私有状態、Filesystemの`open_file_t::offset`、inode table、FATを直接操作しない。無効fdはFD層の通常規則で失敗する。

## 4. whenceとposition計算

| `whence` | 基準位置 |
|---|---:|
| `SEEK_SET` | 0 |
| `SEEK_CUR` | 呼出し前の共有file position |
| `SEEK_END` | 対象inodeの現在size |

提案positionを`base + offset`で計算する。符号拡張した十分な幅の中間値を使用し、加算前後にoverflowを検査する。検証する順序:

1. fdが有効であることを確認する。
2. `whence`が3つの既知値のいずれかであることを確認する。
3. `SEEK_END`の場合はFilesystem公開APIから現在のsizeを取得する。inode tableやディスク形式へ直接アクセスしない。
4. 中間値を計算し、`0 <= proposed <= INT32_MAX`を確認する。
5. 検証成功後に限り、`RegularFile::offset_`を`proposed`へ更新し、その値を返す。

syscallの`int32_t whence`は既知の値との一致確認後だけ`FileInterface::seek()`の`uint32_t whence`へ変換する。`RegularFile`は受け取った値からseek基準を明示的に選び、未知値を無検証castしない。

負の`offset`は一律に不正ではない。`SEEK_CUR` / `SEEK_END`では結果positionが非負なら許可する。`SEEK_SET`で負の提案positionとなる値は拒否する。

`proposed < 0`は`EINVAL`相当、`proposed > INT32_MAX`は`ERANGE`相当として失敗し、positionを変更しない。EOFより後のpositionは許可する。後続readはEOFを返し、後続writeは既存write仕様のzero gap規則に従う。

未知`whence`、size取得失敗、無効fd、seek不能対象もpositionを変更しない。Filesystemの`lseek(open_file_t*)`はFilesystem単体利用APIとして既存契約を維持するが、FD統合経路のposition正本は`RegularFile::offset_`であり、FD統合経路からFS handle offsetへ同期・コピーしない。

## 5. FD層と共有position

- `FileDescription::seek()`は対象`FileInterface`のseek hookへ委譲する。
- `FileInterface`の既定seek hookは失敗し、`RegularFile`だけが通常ファイル・directory向けに実装する。RTTIや`dynamic_cast`で対象種別を判定しない。
- `RegularFile::offset_`は同一FileDescriptionを共有するfdの唯一のpositionである。`dup()`、`spawn()`で継承されたfdは同じ位置を観測する。
- 別々にopenした同一inodeは別々の`RegularFile`を持ち、positionは独立する。
- `read()`、`write()`、`getdents()`も同じ`offset_`を使う。成功したread/write分だけ各APIがpositionを進め、getdentsはその仕様どおりslotを走査する。
- seekはopen flagsのread/write許可に依存しない。`O_RDONLY`や`O_WRONLY`でも有効なfdならseekできる。
- directoryも通常のbyte positionを持つためseek可能とする。`getdents()`が要求するrecord境界の制約は`getdents_implementation_spec.md`に従い、`lseek()`自体は32-byte alignmentを要求しない。
- 同一FileDescriptionの並行position操作はFD/Filesystemの既定直列化方針に従い、部分的な並行更新を許さない。

## 6. エラーとsyscall戻り値

| 条件 | 内部status相当 | syscall戻り値 |
|---|---|---:|
| 成功 | `OK` | 新position |
| 無効/未使用fd | FD層エラー | `-1` |
| 未知`whence` | `EINVAL` | `-1` |
| 提案positionが負 | `EINVAL` | `-1` |
| 提案positionが`INT32_MAX`を超過 | `ERANGE` | `-1` |
| pipe等のseek不能target | `ESPIPE`相当 | `-1` |
| `SEEK_END`でinode size取得失敗 | Filesystem由来status | `-1` |

失敗時に古いpositionを維持し、成功値を返さない。errnoを追加する場合のstatus-to-errno対応は共通ABI仕様で決定する。本書では全失敗を`-1`に変換する既存規約を適用する。

## 7. 必須検証

- `SEEK_SET`で0、中間位置、EOF、EOF超過へ移動する。
- `SEEK_CUR`で正負offset、現position未満となる負結果、0 offsetを検証する。
- `SEEK_END`でファイル末尾、末尾より前、末尾より後へ移動する。
- positionが負になる場合に`EINVAL`相当、`INT32_MAX`を超える場合に`ERANGE`相当となり、positionが不変である。
- 未知`whence`、無効fd、pipe/console等のseek不能targetで失敗し、positionが変化しない。
- read/write/getdentsとのposition共有、dup/spawn後の共有、別々にopenしたhandle間の独立性を検証する。
- directoryに対し任意byte位置へのseekを許し、その後のgetdentsが未整列positionを拒否することを確認する。
- `SEEK_END`のsize取得がFilesystem公開APIを通り、FD syscallがinode/FAT内部を直接操作しない。

## 8. 関連仕様

- Filesystem単体APIと`Whence`: `spec/filesystem/filesystem_external-api_spec.md` §2.5・§11、`spec/filesystem/fs_impl_spec/fs_impl_spec_08-read-lseek.md`
- FD positionの所有と共有: `spec/kernel/open_close_implementation_spec.md`、`spec/kernel/open_file_description_spec.md`
- directory offset規則: `spec/kernel/getdents_implementation_spec.md`
- syscall ABI: `spec/kernel/syscall_entry_spec.md`、`spec/kernel/syscall_generator_spec.md`
