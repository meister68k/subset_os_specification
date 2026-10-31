# rmdir(1)

## 名前
**rmdir** — 空directoryを削除する

## 書式
`rmdir directory ...`

## 説明
各pathnameに`rmdir()`を順に呼びます。再帰削除はせず、空でないdirectoryは失敗します。root、`.`、`..`も削除できません。失敗を診断して残りのpathnameを処理します。

## 終了ステータス
- `0`: 全pathnameの削除成功
- `1`: 引数なし、または1件以上の失敗

## 使用例
```sh
rmdir empty-dir
```
