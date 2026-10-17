# PostgreSQLバックアップ・復元実験記録

## 1. 目的

RTO（目標復旧時間）・RPO（目標復旧時点）の考え方を理解するため、Windows上のPostgreSQL18.6でバックアップと復元を実施。

確認した内容： 
- `pg_dump` によるバックアップ取得時間とファイルサイズ
- バックアップ後に追加したデータが復元結果に含まれるか
- `pg_restore`の実行時間 - 復元・接続・データ件数確認までの時間

> **検証範囲:**  
> 少量のテストデータを別データベースへ復元したローカル実験(大量データは大変だったので少々で、、、)。  
> 障害検知、復旧判断、アプリケーションの正常性確認、業務再開までを含むシステム全体のRTOは測定してない。  
> 業務上のRPO目標を決定したものじゃない。  

## 2. 実行環境

| 項目 | 内容 |
|---|---|
| OS | Windows |
| PostgreSQL | 18.6、64-bit |
| 接続先 | `localhost:5432` |
| 接続ユーザー | `postgres` |
| 実験用DB | `rto_rpo_lab` |
| バックアップ形式 | `pg_dump -Fc`（カスタム形式） |
| バックアップファイル | `recovery_experiment\rto_rpo_before.dump` |

PostgreSQLのポータブル版を使用。  
別の記事で使用していた既存の `postgres` データベースと  
`public.products` テーブルは変更せず、別の実験用データベースを作成。  

PowerShellで使用した実行ファイルのパス設定：

``` powershell
$root = "<SANDBOX>\blog_sandbox\postgres-lock-test"
$psql = "$root\postgresql-18.6-4-windows-x64-binaries\pgsql\bin\psql.exe"
$pgDump = "$root\postgresql-18.6-4-windows-x64-binaries\pgsql\bin\pg_dump.exe"
$pgRestore = "$root\postgresql-18.6-4-windows-x64-binaries\pgsql\bin\pg_restore.exe"
```

## 3. 実験記録

### 3.1 PostgreSQLへの接続確認

``` powershell
& $psql -h localhost -p 5432 -U postgres -d postgres -c "SELECT version();"
```

結果：PostgreSQL 18.6が応答することを確認。

### 3.2 既存テーブルの確認

``` powershell
& $psql -h localhost -p 5432 -U postgres -d postgres -c "SELECT schemaname, tablename FROM pg_tables WHERE schemaname NOT IN ('pg_catalog', 'information_schema') ORDER BY schemaname, tablename;"
```

結果：`public.products`  
の存在を確認した。既存テーブルは変更しない方針。

### 3.3 実験用データベースとテーブルの作成

``` sql
CREATE DATABASE rto_rpo_lab;
```

`rto_rpo_lab` に以下のテーブルを作成。  

``` sql
CREATE TABLE recovery_events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);
```

### 3.4 バックアップ前のデータを3件登録

``` sql
INSERT INTO recovery_events (event_name)
VALUES
    ('backup_before_event_1'),
    ('backup_before_event_2'),
    ('backup_before_event_3');
```

登録後に  
`SELECT id, event_name, created_at FROM recovery_events ORDER BY id;`  
を実行し、3件を確認。  

### 3.5 バックアップ取得

``` powershell
$backupDir = "$root\recovery_experiment"
New-Item -ItemType Directory -Path $backupDir -Force | Out-Null
$backupFile = "$backupDir\rto_rpo_before.dump"

$backupTime = Measure-Command {
    & $pgDump -h localhost -p 5432 -U postgres -d rto_rpo_lab -Fc -f $backupFile
    if ($LASTEXITCODE -ne 0) {
        throw "pg_dump failed. Exit code: $LASTEXITCODE"
    }
}

"Backup duration: $($backupTime.TotalSeconds) seconds"
Get-Item $backupFile | Select-Object FullName, Length, LastWriteTime
```

結果： 
- バックアップ取得時間：`0.2460939` 秒（約0.246秒）
- ファイルサイズ：`2,488` バイト
- ファイル更新時刻：`2026-10-17 22:49:45.317`（ローカル時刻）

### 3.6 バックアップ後にデータを2件追加

``` sql
INSERT INTO recovery_events (event_name)
VALUES
    ('after_backup_event_1'),
    ('after_backup_event_2');
```

元データベースを確認した結果、合計5件が存在。  

| ID | イベント名 | 登録時刻（+09） |
|---:|---|---|
| 1 | `backup_before_event_1` | 2026-10-17 14:41:32.931501 |
| 2 | `backup_before_event_2` | 2026-10-17 14:41:32.931983 |
| 3 | `backup_before_event_3` | 2026-10-17 14:41:32.931987 |
| 4 | `after_backup_event_1` | 2026-10-17 22:51:26.988082 |
| 5 | `after_backup_event_2` | 2026-10-17 22:51:26.988274 |

### 3.7 別データベースへ復元

元データを上書きしないよう、`rto_rpo_restore` を作成。  

``` sql
CREATE DATABASE rto_rpo_restore;
```

復元と時間計測：  

``` powershell
$restoreTime = Measure-Command {
    & $pgRestore -h localhost -p 5432 -U postgres -d rto_rpo_restore $backupFile
    if ($LASTEXITCODE -ne 0) {
        throw "pg_restore failed. Exit code: $LASTEXITCODE"
    }
}

"Restore duration: $($restoreTime.TotalSeconds) seconds"
```

結果：`pg_restore` の実行時間は `0.1929146` 秒（約0.193秒）。  

次のクエリで復元内容を確認。  

``` sql
SELECT id, event_name, created_at
FROM recovery_events
ORDER BY id;

SELECT COUNT(*) AS restored_count
FROM recovery_events;
```

結果：復元先にはバックアップ前の3件だけが存在し、`restored_count` は `3`だった。  
バックアップ後に追加した2件は復元先に含まれなかった。  
元の`rto_rpo_lab` は5件のまま保持された。  

### 3.8 復元・接続・件数確認までの時間を測定

追加で `rto_rpo_restore_timed` を作成し、  
復元後に新しい `psql`接続で件数を確認するまでを計測した。  

``` powershell
$timedRestoreDb = "rto_rpo_restore_timed"

$endToEndTime = Measure-Command {
    & $pgRestore -h localhost -p 5432 -U postgres -d $timedRestoreDb $backupFile
    if ($LASTEXITCODE -ne 0) {
        throw "pg_restore failed. Exit code: $LASTEXITCODE"
    }

    & $psql -h localhost -p 5432 -U postgres -d $timedRestoreDb -v ON_ERROR_STOP=1 -c "SELECT COUNT(*) AS restored_count FROM recovery_events;"
    if ($LASTEXITCODE -ne 0) {
        throw "Validation failed. Exit code: $LASTEXITCODE"
    }
}

"Restore + connection + validation: $($endToEndTime.TotalSeconds) seconds"
```

結果： 
- 復元・接続・件数確認の合計：`0.1002148` 秒（約0.100秒）
- 件数確認：`3`

この値は別の実行で計測した `pg_restore`単独の時間より短かった。  
測定条件や処理時間のばらつきがある?ため、両者を厳密に比較できる値とは扱わないこととする。  

## 4. 結果一覧

| 項目 | 結果 |
|---|---:|
| バックアップ前のデータ件数 | 3件 |
| バックアップ後の元DBのデータ件数 | 5件 |
| 復元先のデータ件数 | 3件 |
| バックアップ取得時間 | 約0.246秒 |
| バックアップファイルサイズ | 2,488バイト |
| `pg_restore` 単独の実行時間 | 約0.193秒 |
| 復元・接続・件数確認の合計時間 | 約0.100秒 |
| バックアップファイルの更新時刻 | 2026-10-17 22:49:45.317 |
| 最後の追加データの登録時刻 | 2026-10-17 22:51:26.988 |

バックアップファイルの更新時刻と最後の追加データの登録時刻には約101.7秒（約1分42秒）の差があった。  
ただし、ファイルの更新時刻は、バックアップに含まれるデータの厳密な取得時点を示すとは限らないので、  
この差を正確なRPOの測定値とはみなさないようにする。  

## 5. 分かったこと

### 確認できたこと

1.  `pg_dump` でバックアップを取得できた。
2.  別のデータベースへ `pg_restore` で復元できた。
3.  バックアップ後に追加した2件は復元先に含まれず、バックアップ前の3件が復元されたことを確認した。
4.  この環境・データ量で、バックアップ取得時間と復元処理時間を計測できた。

### この実験だけでは分からないこと

-   障害発生から業務再開までのシステム全体のRTO
-   業務上許容できるデータ損失時間としてのRPO目標
-   WALやPITR（ポイントインタイムリカバリ）を使った復旧結果
-   大量データや本番相当の負荷における性能
-   障害検知、復旧判断、アプリケーションの正常性確認、業務再開に必要な時間

バックアップ方式によって復元されるデータの時点が異なることと、  
復元処理にかかる時間は確認できた。  
データも少量なので、意味があるかと言われると微妙だが、自分がそもそもバックアップリストアをしらなかったので  
勉強にはなったということで、、、  