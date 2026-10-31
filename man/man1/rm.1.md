# rm(1)

## 名前
**rm** — pathnameのdirectory entryを削除する

## 書式
`rm file ...`

## 説明
各pathnameに`unlink()`を順に呼びます。directoryは削除せず、directoryには`rmdir`を使います。hard linkが残るfileは他の名前から引き続き参照できます。open中fileの寿命はOSに委ねます。失敗後も次のpathnameを処理します。

再帰、確認、強制削除optionはありません。`-r`等はpathnameとして扱われます。

## 終了ステータス
- `0`: 全pathname成功
- `1`: 引数なし、または1件以上の失敗

## 使用例
```sh
rm old-file
```
