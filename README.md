# iaeon_receipt

iAEON アプリの電子レシートを取得し、画像として保存する Python ライブラリ。

mitmproxy 等で取得した通信データから認証情報を抽出し、電子レシート API を操作する。

## セットアップ

```bash
pip install requests Pillow
```

日本語フォント (NotoSansCJK) がシステムにインストールされていると、レシート画像が正しくレンダリングされる。

```bash
# Debian/Ubuntu
sudo apt install fonts-noto-cjk
```

## 必要な認証情報の取得

mitmproxy でiAEONアプリの HTTPS 通信をキャプチャし、以下の2つの値を取得する。

### 1. access_token

任意の `aeonapp.aeon.com` 宛リクエストの `Authorization` ヘッダーから取得。

```
Authorization: Bearer EMLBl6dengx27u7f6AxFQ0hNdA0MZwmn_00000000000000000000000000000000_4300077585681162
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                      この部分が access_token
```

### 2. receipt_account_id

電子レシート画面を開いた際に発生する `/api/aeonapp/1.0/receipt/members/auth` へのPOSTリクエストボディから取得。

```json
{
  "accountId": "iighiqrqusuxrsyv",  // <-- この値
  "accessToken": "..."
}
```

## 使い方

### 基本的な流れ

```python
from iaeon_receipt import IAEONReceiptClient

client = IAEONReceiptClient(
    access_token="Bearer以降の値",
    receipt_account_id="accountIdの値",
)

# レシート一覧取得 (デフォルト: 今月)
receipts = client.list_receipts()

for r in receipts:
    print(f"{r.datetime}  {r.store_name}  ¥{r.total}")

    # レシート詳細取得
    detail = client.get_receipt_detail(r.receipt_id)

    # 画像として保存
    path = client.save_receipt_image(detail, output_dir="receipts")
    print(f"保存: {path}")
```

### 日付範囲を指定

```python
receipts = client.list_receipts(from_date="20260101", to_date="20260131")
```

### レシートのテキスト行を取得

```python
detail = client.get_receipt_detail(receipt_id)
for line in detail.lines:
    print(line)
```

出力例:

```
PrintBitmap(0, 'Electro.bmp', -11, -2)RCみらい長崎ココウォーク店
TEL095-847-0540 FAX095-847-0541
ﾚｼﾞ0144   2026/ 2/12(木)   12:26
クランキーエクセレント   428※
ジャ-ジ-ニュウ           238※
  割引        10%        -24
大粒ラムネ               118※
________________________________
合  計                  PrintDouble('\820', 2)
```

### 埋め込みロゴ画像を保存

```python
logos = client.save_embedded_images(detail, output_dir="receipts/logos")
```

### 店舗情報を取得

```python
store = client.get_store_info("39029")
print(store["name"])       # レッドキャベツみらい長崎ココウォーク店
print(store["prefecture"]) # 長崎県
```

### レンダリングオプション

```python
img = IAEONReceiptClient.render_receipt_image(
    detail,
    font_size=18,     # フォントサイズ (default: 20)
    width=580,        # 画像幅 px (default: 640)
    padding=20,       # 余白 px (default: 20)
    bg_color="white", # 背景色
    text_color="black", # 文字色
)
img.save("receipt.png")
```

## API リファレンス

### IAEONReceiptClient

| メソッド | 説明 |
|---|---|
| `auth_receipt()` | レシートサービス認証。JWT を返す。通常は自動呼出し。 |
| `get_user_receipt_info()` | レシートアカウント情報 (`receipt_account_id`, `use_receipt`) を返す |
| `list_receipts(from_date?, to_date?)` | レシート一覧を `ReceiptSummary` のリストで返す |
| `get_receipt_detail(receipt_id)` | レシート詳細を `ReceiptDetail` で返す |
| `get_store_info(store_code)` | 店舗情報の dict を返す |
| `render_receipt_image(detail, ...)` | レシートを `PIL.Image` にレンダリング (staticmethod) |
| `save_receipt_image(detail, output_dir, ...)` | レシート画像を PNG で保存。Path を返す |
| `save_embedded_images(detail, output_dir)` | 埋め込みBMP画像を個別保存。Path リストを返す |

### データクラス

**ReceiptSummary**

| フィールド | 型 | 説明 |
|---|---|---|
| `receipt_id` | str | レシートID |
| `store_name` | str | 店舗名 |
| `store_code` | str | 店舗コード |
| `datetime` | str | 日時 (`2026-02-12T12:26:13`) |
| `total` | str? | 合計金額 |
| `workstation_id` | str? | レジ番号 |

**ReceiptDetail**

| フィールド | 型 | 説明 |
|---|---|---|
| `receipt_id` | str | レシートID |
| `lines` | list[str] | レシートテキスト行 |
| `images` | dict[str, bytes] | 埋め込み画像 (名前 -> BMPバイナリ) |
| `raw` | dict? | API レスポンス生データ |

## iAEON 電子レシート API エンドポイント

| メソッド | エンドポイント | 説明 |
|---|---|---|
| GET | `/api/iaeon/auth/1.0/account/token` | 認証トークン取得 |
| POST | `/api/iaeon/auth/1.0/account/access_token` | アクセストークン取得 |
| GET | `/api/iaeon/user/1.0/account/information` | ユーザー情報取得 |
| POST | `/api/aeonapp/1.0/receipt/members/auth` | レシートサービス認証 |
| POST | `/api/aeonapp/1.0/receipt/receipts` | レシート一覧 |
| POST | `/api/aeonapp/1.0/receipt/receipts/stringArray` | レシート詳細 |
| POST | `/api/storelist/v2/stores/{code}` | 店舗情報 |

## レシートテキストの特殊コマンド

レシートの `lines` には POS プリンター制御コマンドが含まれる。

| コマンド | 説明 |
|---|---|
| `PrintBitmap(0, 'name.bmp', ...)` | 埋め込みBMP画像の描画位置 |
| `PrintDouble('text', 2)` | テキストを倍角 (太字) で描画 |
| `PrintBarCode('code', ...)` | バーコードの描画 |

`render_receipt_image()` はこれらを自動的に解釈してレンダリングする。

## mitmproxy でのキャプチャ手順

1. PC で mitmproxy を起動
   ```bash
   mitmproxy --listen-port 8080
   ```
2. Android 端末の Wi-Fi プロキシを PC の IP:8080 に設定
3. ブラウザで `http://mitm.it` を開き、CA 証明書をインストール
4. iAEON アプリを起動し、電子レシート画面を開く
5. mitmproxy でフローを保存
   ```bash
   mitmdump -r flows -n --set flow_detail=2 | grep receipt
   ```

## ファイル構成

```
iaeon_receipt/
├── __init__.py    # パッケージエクスポート
└── client.py      # IAEONReceiptClient, ReceiptSummary, ReceiptDetail
example.py         # 使い方サンプルスクリプト
```
