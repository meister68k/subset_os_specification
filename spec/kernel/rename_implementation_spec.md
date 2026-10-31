# `rename()`システムコール実装仕様

## 1. 目的と範囲

pathnameを変更または移動する`rename()` syscallのABIとkernel/Filesystem境界を定める。Filesystem更新規則は[`fs_impl_spec_12-rename.md`](../filesystem/fs_impl_spec/fs_impl_spec_12-rename.md)に従う。

## 2. syscall ABI

```cpp
int32_t sys_rename(const char* oldpath, const char* newpath);
```

- syscall番号70を使用する。V7 syscall番号範囲外の本OS固有番号であり、番号54の`ioctl`とは別である。
- 成功時0、失敗時-1。errno機構や詳細エラー取得APIを追加しない。
- syscall失敗時のFilesystem状態はFilesystem rename仕様に従い、mutation開始後の失敗は部分変更を残す可能性がある。

## 3. 処理経路

```text
btx::sys_rename(oldpath, newpath)
  -> syscall dispatcher (70)
  -> kernel::sys_rename(oldpath, newpath)   // ABI変換
  -> kernel::kernel_rename(current, oldpath, newpath) // Process/copy
  -> Filesystem::rename(cwd_inode, oldpath, newpath)
```

概念上のkernel内部API:

```cpp
fs::FsStatus kernel_rename(Process& process,
                           const char* oldpath,
                           const char* newpath);
```

- handlerはpath、inode、directory entryを独自処理せず、kernel内部APIへ委譲する。
- 両ユーザーpointerを検証し、各pathを`PATH_MAX`未満のNUL終端bufferへcopyする。片方でも不正ならFilesystemを呼ばず失敗する。
- 相対pathはcurrent Processのcwd、絶対pathはFilesystem rootを基準とする。
- Filesystem `FsStatus`はsyscall境界で0/-1へ変換する。

## 4. Filesystem・Processとの契約

- syscallは[`filesystem_external-api_spec.md`](../filesystem/filesystem_external-api_spec.md)の`Filesystem::rename(cwd_inode, oldpath, newpath)`を呼ぶ。
- syscallはProcess cwd、FD table、FileDescription、file offsetを変更しない。
- rename成功後も、移動対象をopen中のFDやcwd参照は同じinodeを参照し続ける。
- destination置換なし、source/destination同一inodeのno-op、directory移動時の`..`・親link count更新などの意味はFilesystem API仕様を正本とする。

## 5. 確認項目

1. syscall番号70が2つのpathname引数を正しい順序で受ける。
2. null、未終端、長さ超過の各pointer/pathをFilesystem mutation前に拒否する。
3. relative pathにcurrent Process cwdを渡し、absolute pathをroot基点で解決する。
4. 成功時0、Filesystem/API失敗時-1となる。
5. syscall handlerがdirent、inode、link countを直接変更しない。
6. mutation後failureの可能性を`mv`に正しく伝え、隠れたcleanup/rollbackを行わない。
