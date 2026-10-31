# mv(1)

## 名前
**mv** — pathnameを変更または移動する

## 書式
`mv source destination`

## 説明
`rename()` syscallを1回呼びます。destinationが別inodeとして存在すれば失敗し、置換しません。同一inodeへのrenameはno-op成功です。copy/unlinkやlink/unlinkによる代替は行いません。Filesystem error後の状態はrollbackされるとは限りません。

複数source、directory destinationへの配置、cross-filesystem move、symlink、force optionは非対応です。

## 終了ステータス
- `0`: rename成功
- `1`: 引数またはsyscallエラー

## 使用例
```sh
mv old-name new-name
```
