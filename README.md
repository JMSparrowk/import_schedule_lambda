# Import Morning Schedule Lambda

S3 にアップロードされた朝礼スケジュール Excel（`.xlsx`）を読み込み、
担当者を `Users` テーブルと照合したうえで、DynamoDB の `Calendar` テーブルへ保存する AWS Lambda です。

## Flow

```text
Excel Upload
    ↓
S3
    ↓ ObjectCreated
Lambda 実行
    ↓
Excel のヘッダー / スケジュールを読み込み
    ↓
Users テーブルを参照
    ↓
担当者名 → userId をマッピング
    ↓
Calendar データを生成
    ↓
DynamoDB に保存
