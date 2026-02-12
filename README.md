# aeon

iAEON アプリの認証・電子レシート取得を行う Python ツールキット。

`login.py` で iAEON にログインしてアクセストークンを取得し、電子レシート API を操作できる。

## セットアップ

```bash
pip install -r requirements.txt
```

日本語フォント (NotoSansCJK) がシステムにインストールされていると、レシート画像が正しくレンダリングされる。

```bash
# Debian/Ubuntu
sudo apt install fonts-noto-cjk
```

## ログイン

`login.py` でiAEONアカウントにログインし、アクセストークンを `.env` に保存する。

```bash
# 対話式 (電話番号・パスワードを入力)
python login.py

# 引数で指定
python login.py --phone 09012345678 --password yourpassword

# デバイスIDを指定 (同じIDを使い続けるとSMS認証をスキップできる場合がある)
python login.py --phone 09012345678 --password yourpassword --device-id <UUID>
```

認証フロー:
1. 電話番号 + パスワードでログイン
2. SMS認証コードの送信・検証 (新デバイス時)
3. Bearerトークン取得
4. アクセストークン取得 → `.env` に `ACCESS_TOKEN` と `DEVICE_ID` を保存

### receipt_account_id の取得

電子レシート機能を使うには `RECEIPT_ACCOUNT_ID` も必要。
mitmproxy で `/api/aeonapp/1.0/receipt/members/auth` へのPOSTリクエストボディから取得する。

```json
{
  "accountId": "iighiqrqusuxrsyv",  // <-- この値
  "accessToken": "..."
}
```

### .env ファイル

```
ACCESS_TOKEN="..."
DEVICE_ID="bf533bf5-..."
RECEIPT_ACCOUNT_ID="iighiqrqusuxrsyv"
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

### IAEONAuth

| メソッド | 説明 |
|---|---|
| `login(phone, password)` | ログイン。SMS認証が必要な場合 code `10021`/`10008` を返す |
| `request_sms()` | SMS認証コード送信リクエスト |
| `verify_sms_code(auth_code)` | SMS認証コード検証 |
| `login_token()` | Bearerトークン取得。`_access_token` に保存される |
| `get_access_token(client_id?)` | サービス用アクセストークン取得 |
| `full_login(phone, password)` | 完全なログインフロー (対話式、SMS入力あり) |
| `get_service_token()` | サービス用 (client_id=...0003) アクセストークン取得 |

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

## API エンドポイント

### 認証

| メソッド | エンドポイント | 説明 |
|---|---|---|
| POST | `/api/iaeon/auth/1.0/login` | ログイン (電話番号 + パスワードハッシュ) |
| PUT | `/api/iaeon/auth/1.0/sms` | SMS認証コード送信 |
| POST | `/api/iaeon/auth/1.0/auth_code` | SMS認証コード検証 |
| POST | `/api/iaeon/auth/1.0/login/token` | Bearerトークン取得 |
| POST | `/api/iaeon/auth/1.0/account/access_token` | サービス用アクセストークン取得 |

### 電子レシート

| メソッド | エンドポイント | 説明 |
|---|---|---|
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
iaeon_auth.py      # iAEON 認証モジュール (IAEONAuth)
login.py           # ログインCLIスクリプト
example.py         # レシート取得サンプルスクリプト
requirements.txt   # 依存パッケージ
iaeon_receipt/
├── __init__.py    # パッケージエクスポート
└── client.py      # IAEONReceiptClient, ReceiptSummary, ReceiptDetail
```
