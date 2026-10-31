# sleep(1)

## 名前
**sleep** — 指定秒数だけ呼出しProcessを待機させる

## 書式
`sleep seconds`

## 説明
ASCII数字だけからなる非負整数秒を1個受け取り、1000 tick/秒に換算して`sleep` syscallを1回呼びます。busy-waitは行いません。最大値は`UINT32_MAX / 1000`です。0秒は協調Scheduler有効時にyieldします。

## 終了ステータス
- `0`: syscall成功
- `1`: 引数、範囲、またはsyscallエラー

## 使用例
```sh
sleep 1
sleep 0
```
