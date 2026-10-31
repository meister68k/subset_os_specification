# cp(1)

## 名前
**cp** — 通常ファイルを複製する

## 書式
`cp source destination`

## 説明
通常ファイル1個をbyte単位でコピーします。sourceとdestinationが同じinode（hard linkを含む）なら、destinationを変更する前に失敗します。既存destinationはtruncateし、新規作成時のmodeは`0666`です。内容・owner・時刻等はコピーしません。失敗後の部分destinationはrollbackしません。

directory、複数source、標準入出力、recursive copyは非対応です。`-r`等はoptionでなくpathnameとして扱います。相対pathはcwd基準です。

## 終了ステータス
- `0`: コピーとcloseが成功
- `1`: 引数、stat/open/read/write/close等の失敗

## 使用例
```sh
cp source backup
```
