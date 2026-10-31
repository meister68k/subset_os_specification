# ln(1)

## 名前
**ln** — hard linkを作成する

## 書式
`ln source destination`

## 説明
`link()`を1回呼び、destinationをsourceと同じinodeへ結び付けます。内容のコピーや事前stat/openは行いません。destinationが存在すれば失敗します。directory link、symlink、複数sourceは非対応です。相対pathはcwd基準です。

## 終了ステータス
- `0`: link作成成功
- `1`: 引数またはsyscallエラー

## 使用例
```sh
ln original alias
```
