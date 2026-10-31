# mkdir(1)

## 名前
**mkdir** — directoryを作成する

## 書式
`mkdir directory ...`

## 説明
各pathnameに`mkdir(path, 0777)`を順に呼びます。親directoryの再帰作成はしません。既存名や失敗したpathnameを診断し、残りの引数も処理します。相対pathはcwd基準です。optionはなく、`-p`等もpathnameとして渡されます。

## 終了ステータス
- `0`: 全directory作成成功
- `1`: 引数なし、または1件以上の失敗

## 使用例
```sh
mkdir tmp work
```
