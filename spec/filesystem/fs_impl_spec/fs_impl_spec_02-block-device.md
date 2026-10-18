# 02 — ブロックデバイス

> 型・エラー・排他・算術の共通規約は`AGENTS.md`を参照する。全エラーの定義は`errors.md`を参照する。

## 1. 目的

- ファイルイメージ、RAM、SDカード等を同期512バイトブロックI/Oへ統一する。
- Backend差分をテンプレートでコンパイル時に吸収する。
- 境界検査とopen状態検査を共通ラッパーへ集約する。

## 2. インターフェース

```cpp
template<class Backend>
class BlockDevice {
public:
    FsStatus open(typename Backend::OpenArg arg, uint32_t total_blocks);
    FsStatus read(uint32_t blkno, uint8_t* buf);
    FsStatus write(uint32_t blkno, const uint8_t* buf);
    FsStatus close();

private:
    Backend backend_;
    uint32_t total_blocks_ = 0;
    bool is_open_ = false;
};
```

- 実引数名は既存コードに合わせてよいが、契約は変えない。
- 1回のread/writeは必ず1ブロック、512バイトとする。
- 部分転送は提供しない。
- I/Oは同期完了とする。
- 読み込み先は呼び出し側所有の`uint8_t*`とする。
- 書き込み元は呼び出し側所有の`const uint8_t*`とする。

## 3. Backend契約

Backendは少なくとも次の操作を提供する。

```cpp
struct Backend {
    using OpenArg = /* backend固有 */;
    FsStatus open(OpenArg arg);
    FsStatus read(uint32_t blkno, uint8_t* buf);
    FsStatus write(uint32_t blkno, const uint8_t* buf);
    FsStatus close();
};
```

- Backendは物理媒体またはホストファイル固有の処理だけを持つ。
- FSレイアウト、FAT、inodeをBackendへ持ち込まない。
- Backendは例外を送出しない。
- BackendのOS/デバイスエラーは`FsStatus`へ変換する。
- Backendの`open()`は、失敗した場合に自身で途中状態を後始末し、閉状態へ戻る。
- Backendの`close()`は終端操作とする。エラーを返した場合も、呼び出し側は再度`close()`してはならない。

## 4. 操作契約

### open

- `is_open_`を先に検査する。
- 二重openは`EBUSY`とし、Backendの`open()`を呼ばない。
- 閉状態の場合だけBackendの`open()`を1回呼ぶ。
- デバイス容量または呼び出し側指定容量を`total_blocks_`へ保持する。
- Backendの`open()`に失敗した場合、そのstatusを失わず返し、`is_open_ == false`、`total_blocks_ == 0`を維持する。
- Backendの`open()`に成功した場合だけ、`total_blocks_`を設定して`is_open_ = true`とする。

### read/write

- `is_open_`を先に検査する。
- 未openなら`ENODEV`とする。
- nullバッファなら`EINVAL`とする。
- `blkno >= total_blocks_`なら`ERANGE`とする。
- 検査順はopen状態、nullバッファ、block範囲の順とし、複数条件に違反する場合は先に検出したstatusを返す。
- 検査成功後だけBackendを呼ぶ。
- Backend結果を失わず返す。

### close

- 未openなら`ENODEV`とし、Backendの`close()`を呼ばない。
- open中ならBackendの`close()`をちょうど1回呼ぶ。
- Backendの`close()`の成否にかかわらず、呼び出し後は`is_open_ = false`、`total_blocks_ = 0`とする。
- Backendの`close()`が返したstatusを失わず返す。
- close失敗後もclose済みとして扱い、Backendの`close()`を再試行しない。
- mount時にopenし、unmount時にcloseする。
- 個々のFS操作でopen/closeしない。

## 5. 完了条件

- 未open時のread/writeを拒否できる。
- ブロック0と最終有効ブロックを読み書きできる。
- `blkno == total_blocks_`を拒否できる。
- Backendのopen/read/write/close失敗を伝播できる。
- 二重openを`EBUSY`で拒否し、Backendのopen回数と状態が変化しない。
- 未open closeを`ENODEV`で拒否し、Backendのclose回数が変化しない。
- close成功時と失敗時のどちらも閉状態へ遷移し、Backendのcloseを二重に呼ばない。
- 2つの`BlockDevice`インスタンスの状態が混線しない。
- fake Backendで呼び出し回数と引数を検証できる。
