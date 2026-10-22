# `dup()` システムコール実装仕様

## 1. 目的と範囲

本書は、呼出し元Processのfdを複製する`dup()` syscallのABI、参照管理、失敗時の不変条件を定める。複製先は呼出し元Processの`FdTable`内にある最小の未使用fd番号とする。`dup2()`、fd close-on-exec属性、FDテーブル間の直接コピーは対象外とする。

`dup()`は新しい`FileDescription`やI/O対象を作らない。複製元と複製先は同一`FileDescription`を参照し、flags、共有file position、対象I/O状態を共有する。

## 2. syscall ABI

```cpp
int32_t sys_dup(int32_t fd);
```

- 成功時は複製先fd番号を返す。
- 失敗時は`-1`を返す。errno機構やOSのglobal `errno`は追加しない。
- `fd`は呼出し元Processの`FdTable`で有効なfd番号でなければならない。
- syscall番号の割当は本書では行わない。ユーザー側`btx::sys_dup()`、共通syscall番号定義、dispatcher生成定義、`kernel::sys_dup()`で本シグネチャと引数型を一致させる。

## 3. 処理経路と責務

```text
btx::sys_dup(fd)
  -> syscall dispatcher
  -> kernel::sys_dup(fd)                 // syscall ABIの薄いadapter
  -> kernel::kernel_dup(current Process&, fd)
       -> FdTable::get(fd)
       -> retain_file_description(desc)
       -> FdTable::alloc(desc)            // 最小の空きfd
```

概念上の内部API:

```cpp
int32_t kernel_dup(Process& process, int32_t fd);
```

実際の命名とProcessManagerへの依存方法は既存実装規約に合わせる。`sys_dup()`は現在Processと引数を取得して内部APIへ委譲し、FD資源管理をdispatcherへ重複実装しない。

## 4. 成功時の処理順序

1. 現在Processの`FdTable::get(fd)`で複製元を確認する。無効・未使用fdなら失敗する。
2. 対象の`FileDescription`に対して`retain_file_description()`を呼ぶ。これにより複製先fdに対応する参照を先に確保する。
3. `FdTable::alloc(desc)`で最小の未使用fdへ同じ`FileDescription`を登録する。
4. 登録したfd番号を返す。

`FdTable`は参照数を変更しないため、複製処理は明示的なretainを必ず伴う。retain成功後にfd登録が成功した時点で、その参照は複製先fdに対応する。`FileDescription`の内容、open flags、target、positionは書き換えない。

複製元fd自体が最小空き番号より小さいので、正常な`FdTable`では返却fdは複製元fdと異なる。

## 5. 失敗とrollback

処理失敗時は`-1`を返し、複製元fdとそのI/O状態を維持する。

| 失敗条件 | 動作 |
|---|---|
| `fd < 0`、`fd >= MAX_FD`、または未使用fd | `-1`。参照数・FdTable不変 |
| `retain_file_description()`失敗（不正なdescription、参照数上限等） | `-1`。FdTable不変 |
| `FdTable::alloc()`失敗（空きfdなし等） | retain済み参照を`release_file_description()`で1つ戻し、`-1` |

`alloc()`が失敗した場合、複製先fdは登録されていない。rollbackでreleaseする参照は元のfdが保持する参照とは別の追加分であり、複製元fdをdetachしてはならない。元fdが有効なためrollback後もFileDescriptionには少なくとも元fdの参照が残り、最終cleanupは発生しない。予期しないrelease失敗は内部不変条件違反として扱い、失敗結果を成功fdへ置き換えない。

`FdTable::alloc()`が失敗を返したのにエントリを登録する実装は禁止する。したがってrollback後に一部登録が残る状態はない。

## 6. 共有意味論とcloseとの関係

- 複製後の2 fdは同じ`FileDescription`を指し、通常ファイルの`RegularFile::offset_`を共有する。片方からの`read()`、`write()`、`lseek()`、`getdents()`によるposition変化は他方からも観測される。
- read/write access mode等の`FileDescription` flagsは共有され、dup時に変更・再検査・複製しない。
- console、pipe、通常ファイルを含む全I/O対象で同じFD共通処理を使う。対象種別ごとの複製処理やFilesystem `dup_file()`は呼ばない。
- 複製元または複製先の一方を`close()`しても、他方のfdは有効なまま残る。`FileDescription`の最終参照解放時に限り、既存のrelease規則でtarget cleanupが行われる。
- `dup()`は新しいopenやFilesystem handleを作らず、inode metadata、file offset、pipe内容、device状態を直接変更しない。
- `spawn()`によるFD継承も同じFileDescription共有原則を使うが、子Processへの参照登録とrollbackはProcess管理仕様に従う。

## 7. syscall境界と安全性

- syscallはユーザー指定`fd`を`int32_t`として受け取り、FdTableの有効範囲を通じて検証する。範囲外値を配列添字へ直接変換してはならない。
- `kernel_dup()`はProcessの公開`fd_table()` API、`FdTable`公開API、FileDescription参照管理APIを使う。`entries_`や`ref_count_`を直接操作しない。
- `FdTable::get()`が返す参照を、refcount追加前に複製先へ登録してはならない。
- refcount上限は`retain_file_description()`で検査し、整数wraparoundを起こさない。
- 並行実行段階では、複製元の有効性確認、retain、空きfd登録をFD/Process管理で定める同期単位内で不可分に扱う。ロックや割り込み禁止の具体策は既存のFD同期仕様に従い、本書だけで新たな部分ロックを導入しない。現行段階が協調実行である場合はその実行モデルに従う。

## 8. 受入確認項目

- fd 0、fd 1、および通常ファイルfdを複製すると、最小の未使用fd番号が返る。
- fdが負、`MAX_FD`以上、または未使用の場合は`-1`で、他fdと参照数を変更しない。
- FdTable満杯時は`-1`を返し、追加retainがrollbackされ、全エントリが呼出し前の状態を保つ。
- FileDescription参照数上限時にretainが失敗し、fd登録が行われない。
- 複製した通常ファイルfd間でread/write/seek/getdentsのpositionが共有される。
- 複製元だけ、複製先だけをcloseした各場合に、残るfdが利用可能である。
- 全複製fdをcloseしたときにのみ最終releaseとtarget cleanupが1回実行される。
- consoleおよびpipeでも同一FileDescriptionを共有し、dup自体がI/O target固有APIを呼ばない。
- dispatcherの引数取得型、ユーザーwrapper、kernel handlerが`int32_t`で一致する。

## 9. 関連仕様

- FDテーブル、FileDescription、参照API、共有position: [`open_file_description_spec.md`](open_file_description_spec.md)
- `open()` / `close()`と最終release: [`open_close_implementation_spec.md`](open_close_implementation_spec.md)
- spawn時のFD継承: [`spawn_wait_implimentation_spec.md`](spawn_wait_implimentation_spec.md)
- syscall入口・番号・wrapper/dispatcher生成: [`syscall_entry_spec.md`](syscall_entry_spec.md)、[`syscall_generator_spec.md`](syscall_generator_spec.md)
- `lseek()`共有position: [`lseek_implementation_spec.md`](lseek_implementation_spec.md)
