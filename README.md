# Import Morning Schedule Lambda

````markdown

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
````

## Lambda 設定

* Runtime: Python 3.12
* Memory: 256 MB
* Timeout: 30 seconds
* Handler: `lambda_function.lambda_handler`

## S3 Trigger

```text
Event type: ObjectCreated
Suffix: .xlsx
```

S3 に `.xlsx` ファイルがアップロードされると Lambda が実行されます。

## Users Table

```text
PK: userId

cognitoSub
name
email
employeeId
emailNotification
```

Excel には担当者名のみが記載されているため、
Lambda は `Users.name` を基準に `userId` を取得します。

例:

```text
Excel 担当者: ジョン

Users:
name   = ジョン
userId = USER#002
```

マッピング結果:

```text
ジョン → USER#002
```

### 担当者が見つからない場合

Excel に記載されている担当者名が `Users` テーブルに存在しない場合、
Lambda はエラーとして終了します。

例:

```text
Excel:
ジョン

Users:
ジョン が存在しない
```

結果:

```text
担当者「ジョン」がUsersテーブルに存在しません。
```

不正なユーザー情報が `Calendar` に保存されることを防ぐためのチェックです。

## Calendar Table

```text
PK: month
SK: date

originalUserId
originalUserName

assignedUserId
assignedUserName

holiday
holidayName

jwd
holidayWorkRequest
```

Excel から初回登録する時点では、以下の値は同一になります。

```text
originalUserId   = assignedUserId
originalUserName = assignedUserName
```

例:

```text
originalUserId   = USER#002
originalUserName = ジョン

assignedUserId   = USER#002
assignedUserName = ジョン
```

今後スケジュール変更を行う場合は、
`originalUser` は変更せず、`assignedUser` のみ更新します。

## Excel Format

必須カラム:

```text
日付
担当者
```

任意カラム:

```text
曜日
JWD
休出申請
備考
```

処理ルール:

```text
JWD = 〇
→ jwd = true

休出申請 = 〇
→ holidayWorkRequest = true

備考に値あり
→ holiday = true
→ holidayName = 備考

備考が空
→ holiday = false
```

現在は最初の Sheet のみを読み込みます。

そのため、現在の運用では対象月の Sheet のみを残した `.xlsx` ファイルをアップロードします。

## コード処理フロー

```text
1. S3 イベントから bucket / file key を取得

2. S3 の Excel ファイルを Lambda の /tmp にダウンロード

3. openpyxl で Excel を開く

4. 最初の Sheet を選択

5. 上から10行以内で
   日付 / 担当者 ヘッダーを検索

6. Users テーブルを取得

7. 名前 → userId の Map を作成

   例:
   ジョン → USER#002

8. Excel を1行ずつ読み込む

9. 担当者が存在する場合は Users と照合

10. JWD / 休出申請 / 備考 を変換

11. Calendar Item を生成

12. 全データのチェック完了後、
    DynamoDB の Calendar に保存
```

例えば Excel が以下の場合:

```text
日付        担当者   JWD   休出申請   備考
2026/10/08  ジョン
```

Calendar には以下のように保存されます。

```text
month = 2026-10
date  = 2026-10-08

originalUserId   = USER#002
originalUserName = ジョン

assignedUserId   = USER#002
assignedUserName = ジョン

jwd = false
holidayWorkRequest = false
holiday = false
```

休日の場合:

```text
日付        担当者   JWD   休出申請   備考
2026/10/12                             スポーツの日
```

↓

```text
month = 2026-10
date  = 2026-10-12

holiday = true
holidayName = スポーツの日
jwd = false
holidayWorkRequest = false
```

## Important

Excel の担当者名は `Users.name` と完全に一致している必要があります。

```text
Excel: ジョン
Users: ジョン
```

### 同一月の再アップロード

同じ月の Excel を再アップロードした場合、
同一の `month + date` を持つ既存データは上書きされます。

`PutItem` は部分更新ではなく Item 全体を置き換えるため、
Web 上で変更した `assignedUser` などの情報も Excel の内容で上書きされる可能性があります。

そのため、運用時には以下の対策を追加する予定です。

```text
すでに存在する month の再アップロードを禁止する

## Local Build

```bash
python3.12 -m pip install openpyxl -t .
zip -r function.zip .
```

ZIP の最上位には、必ず以下のファイルが存在する必要があります。

```text
lambda_function.py
```
