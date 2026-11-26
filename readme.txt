① SQL Server インスタンス情報

SELECT
    @@SERVERNAME AS ServerName,
    SERVERPROPERTY('MachineName') AS MachineName,
    SERVERPROPERTY('InstanceName') AS InstanceName,
    SERVERPROPERTY('Edition') AS Edition,
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('Collation') AS ServerCollation,
    SERVERPROPERTY('IsIntegratedSecurityOnly') AS WindowsAuthenticationOnly;

確認対象：
- サーバー名・インスタンス名
- SQL Serverのエディション・バージョン
- サーバー照合順序
- 認証モード

② SQL Server ログイン情報
SQL Serverに登録されているログインを確認するSQLです。
SELECT
    name AS LoginName,
    type_desc AS LoginType,
    is_disabled AS IsDisabled,
    create_date AS CreateDate,
    default_database_name AS DefaultDatabase
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY name;


確認対象：
- SQL Server認証ログイン
- Windows認証ログイン
- 有効・無効状態
- デフォルトデータベース

③ サーバーロール・権限
ログインごとのサーバーロール所属を確認するSQLです。
SELECT
    sp.name AS LoginName,
    sp.type_desc AS LoginType,
    sr.name AS ServerRole
FROM sys.server_role_members AS srm
INNER JOIN sys.server_principals AS sp
    ON srm.member_principal_id = sp.principal_id
INNER JOIN sys.server_principals AS sr
    ON srm.role_principal_id = sr.principal_id
ORDER BY
    sp.name,
    sr.name;


確認対象：
- sysadmin
- securityadmin
- serveradmin
- setupadmin
- processadmin
- diskadmin
- dbcreator
- bulkadmin

④ サーバーレベルの権限
ログインに付与されているサーバーレベルの権限を確認します。
SELECT
    sp.name AS LoginName,
    perm.state_desc AS PermissionState,
    perm.permission_name AS PermissionName,
    perm.class_desc AS PermissionClass,
    grantor.name AS GrantorName
FROM sys.server_permissions AS perm
INNER JOIN sys.server_principals AS sp
    ON perm.grantee_principal_id = sp.principal_id
LEFT JOIN sys.server_principals AS grantor
    ON perm.grantor_principal_id = grantor.principal_id
ORDER BY
    sp.name,
    perm.permission_name;


確認対象：
- GRANT（権限付与）
- DENY（権限拒否）
- サーバーレベルの権限
- 権限付与者

⑤ データベース一覧・基本設定
ユーザーデータベースの基本情報を確認します。
SELECT
    name AS DatabaseName,
    state_desc AS State,
    compatibility_level AS CompatibilityLevel,
    collation_name AS CollationName,
    recovery_model_desc AS RecoveryModel,
    page_verify_option_desc AS PageVerifyOption,
    is_auto_close_on AS AutoClose,
    is_auto_shrink_on AS AutoShrink,
    is_read_only AS ReadOnly,
    SUSER_SNAME(owner_sid) AS OwnerName
FROM sys.databases
WHERE database_id > 4
ORDER BY name;


確認対象：
- データベース名
- 互換性レベル
- 照合順序
- 復旧モデル（FULL / SIMPLE / BULK_LOGGED）
- ページ検証設定
- AUTO_CLOSE / AUTO_SHRINK
- 読み取り専用設定
- データベース所有者

⑥ データベースファイル設定（MDF / LDF）
データファイル・ログファイルの配置やサイズ、自動拡張設定を確認します。
SELECT
    DB_NAME(database_id) AS DatabaseName,
    name AS LogicalFileName,
    type_desc AS FileType,
    physical_name AS PhysicalPath,
    size * 8 / 1024 AS SizeMB,
    CASE
        WHEN max_size = -1 THEN -1
        ELSE max_size * 8 / 1024
    END AS MaxSizeMB,
    CASE
        WHEN is_percent_growth = 1
            THEN CAST(growth AS VARCHAR(20)) + '%'
        ELSE CAST(growth * 8 / 1024 AS VARCHAR(20)) + ' MB'
    END AS AutoGrowth
FROM sys.master_files
WHERE database_id > 4
ORDER BY database_id, file_id;


確認対象：
- MDF / NDF / LDF
- 論理ファイル名
- 物理ファイルパス
- 現在のサイズ
- 最大サイズ
- 自動拡張設定

⑩ SQL Server 構成設定（sp_configure）
SQL Serverインスタンスの構成設定を確認します。
SELECT
    name AS SettingName,
    value AS ConfiguredValue,
    value_in_use AS RunningValue,
    minimum AS MinValue,
    maximum AS MaxValue,
    description AS Description
FROM sys.configurations
ORDER BY name;


確認対象：
- 最大サーバーメモリ
- 最小サーバーメモリ
- 最大ワーカー数
- 最大接続数
- 並列処理の最大次数（MAXDOP）
- 並列処理コストしきい値
- Ad Hoc Distributed Queries
- xp_cmdshell
- Database Mail XPs
- リモートアクセス

⑪ サーバー照合順序・文字コード設定
サーバーとデータベースの照合順序を確認します。
SELECT
    SERVERPROPERTY('Collation') AS ServerCollation;

SELECT
    name AS DatabaseName,
    collation_name AS DatabaseCollation
FROM sys.databases
WHERE database_id > 4;


確認対象：
- サーバー照合順序
- データベース照合順序
- 大文字・小文字の区別
- かな・文字幅の区別
- 文字列比較の動作
注意点：
- データベースの照合順序はバックアップ・リストアで引き継がれます。
- サーバー照合順序はインストール時の設定であり、バックアップには含まれません。
- 移行元と移行先で異なると、tempdbとの照合順序の不一致などが発生する場合があります。

⑫ TCP/IP・ネットワーク設定
SQL Serverのネットワーク接続設定を確認します。
SELECT
    ip_address AS IPAddress,
    port AS Port,
    is_ipv4 AS IsIPv4,
    type_desc AS ListenerType,
    state_desc AS State
FROM sys.dm_tcp_listener_states
ORDER BY
    ip_address,
    port;


確認対象：
- TCP/IPリスニング状態
- IPアドレス
- TCPポート番号
- IPv4 / IPv6
- リスナーの状態
注意点：
- このSQLは現在のリスナー情報を確認するものです。
- TCP/IPの有効・無効や固定ポートの設定変更は、SQL Server構成マネージャーで行います。
- これらの設定はデータベースのバックアップには含まれません。

⑬ SQL Serverサービス・起動設定
SQL Server関連サービスの情報を確認します。
SELECT
    servicename AS ServiceName,
    startup_type_desc AS StartupType,
    status_desc AS Status,
    service_account AS ServiceAccount,
    process_id AS ProcessID,
    last_startup_time AS LastStartupTime,
    filename AS ExecutablePath
FROM sys.dm_server_services
ORDER BY servicename;


確認対象：
- SQL Server Database Engine
- SQL Server Browser
- SQL Server Agent
- サービスの起動方式
- サービスアカウント
- 実行ファイルの配置
注意点：
- このSQLはサービス情報の確認用です。
- 起動方式やサービスアカウントはSQL Server構成マネージャーまたはWindowsサービスで設定します。
- ExpressからStandardへの移行では、SQL Server Agentが利用可能になります。

⑭ SQL Server 起動パラメーター
SQL Serverサービスに設定されている起動パラメーターを確認します。
SELECT
    registry_key AS RegistryKey,
    value_name AS ParameterName,
    value_data AS ParameterValue
FROM sys.dm_server_registry
WHERE registry_key LIKE '%MSSQLServer\Parameters%'
ORDER BY value_name;


確認対象：
- -d：masterデータベースのデータファイル
- -e：SQL Serverエラーログ
- -l：masterデータベースのログファイル
- その他の起動オプション
注意点：
- 起動パラメーターはデータベースのバックアップには含まれません。
- 移行先では新PCのフォルダー構成に合わせて設定します。
- 通常はSQL Server構成マネージャーで変更します。

⑮ SQL Server エラーログ設定
エラーログの保存先とローテーション設定を確認します。
EXEC master.dbo.xp_instance_regread
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'ErrorLogFileName';

EXEC master.dbo.xp_instance_regread
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'NumErrorLogs';


確認対象：
- エラーログの保存先
- 保持するエラーログファイル数
- ログファイルの世代管理
注意点：
- エラーログ設定はデータベースのバックアップには含まれません。
- 移行先では保存先と保持数を必要に応じて再設定します。

⑯ SQL Server 監査・トレース設定
SQL Serverで実行されているトレースと監査関連の設定を確認します。
-- 実行中のSQL Serverトレース
SELECT
    id AS TraceID,
    status AS Status,
    path AS TracePath,
    max_size AS MaxSizeMB,
    stop_time AS StopTime,
    is_rowset AS IsRowset,
    is_rollover AS IsRollover
FROM sys.traces
ORDER BY id;

-- SQL Server監査設定（SQL Server 2014で利用可能な範囲）
SELECT
    name AS AuditName,
    audit_guid AS AuditGuid,
    type_desc AS AuditType,
    on_failure_desc AS OnFailure,
    is_state_enabled AS IsEnabled
FROM sys.server_audits
ORDER BY name;


確認対象：
- SQL Serverトレース
- トレースファイルの保存先
- トレースの有効・無効
- SQL Server Auditの設定
- 監査失敗時の動作
注意点：
- トレース設定はインスタンス側の設定です。
- SQL Server Auditの定義もデータベースバックアップには含まれません。
- SQL Server 2014 Expressではエディションによる機能制限があります。

⑰ リンクサーバー（Linked Server）
外部のSQL Serverや他のデータソースへの接続定義を確認します。
SELECT
    name AS LinkedServerName,
    product AS Product,
    provider AS Provider,
    data_source AS DataSource,
    location AS Location,
    catalog AS Catalog,
    is_linked AS IsLinked,
    is_remote_login_enabled AS IsRemoteLoginEnabled,
    is_data_access_enabled AS IsDataAccessEnabled,
    is_rpc_out_enabled AS IsRpcOutEnabled
FROM sys.servers
WHERE server_id <> 0
ORDER BY name;


確認対象：
- リンクサーバー名
- OLE DBプロバイダー
- 接続先サーバー
- カタログ名
- データアクセス設定
- RPC OUT設定
注意点：
- リンクサーバー定義はデータベースバックアップには含まれません。
- 接続先のログイン情報やパスワードは別途移行が必要です。
- SQL Server 2022ではプロバイダーの対応状況も確認します。

⑱ 資格情報（Credentials）
SQL Serverに登録されている資格情報を確認します。
SELECT
    name AS CredentialName,
    credential_identity AS CredentialIdentity,
    create_date AS CreateDate,
    modify_date AS ModifyDate
FROM sys.credentials
ORDER BY name;


確認対象：
- 資格情報名
- 認証に使用するID
- 作成日時
- 更新日時
注意点：
- 資格情報はデータベースバックアップには含まれません。
- パスワードなどの秘密情報は、このSQLでは取得できません。
- 移行先では資格情報を再作成し、パスワードを再設定する必要があります。

⑲ エンドポイント（Endpoints）
SQL Serverに定義されているエンドポイントを確認します。
SELECT
    e.name AS EndpointName,
    e.type_desc AS EndpointType,
    e.state_desc AS EndpointState,
    e.is_admin_endpoint AS IsAdminEndpoint,
    t.port AS Port,
    t.is_dynamic_port AS IsDynamicPort,
    t.ip_address AS IPAddress
FROM sys.endpoints AS e
LEFT JOIN sys.tcp_endpoints AS t
    ON e.endpoint_id = t.endpoint_id
ORDER BY e.name;


確認対象：
- エンドポイント名
- エンドポイントの種類
- 有効・無効状態
- TCPポート番号
- IPアドレス
注意点：
- エンドポイント定義はデータベースバックアップには含まれません。
- 通常のデータベース接続とは別の通信設定です。
- Database Mirroring、Service Brokerなどで利用されます。

# ⑳ Database Mail 設定

## 目的
SQL Serverからメールを送信するための設定を確認する。

## 確認SQL

```sql
-- 1. Database Mail の有効状態
SELECT
    name,
    value,
    value_in_use
FROM sys.configurations
WHERE name = 'Database Mail XPs';

-- 2. メールアカウント
SELECT
    account_id,
    name AS AccountName,
    email_address AS EmailAddress,
    display_name AS DisplayName,
    mailserver_name AS MailServer,
    port AS Port,
    enable_ssl AS EnableSSL
FROM msdb.dbo.sysmail_account;

-- 3. メールプロファイル
SELECT
    profile_id,
    name AS ProfileName,
    description AS Description
FROM msdb.dbo.sysmail_profile;

-- 4. プロファイルとアカウントの関連付け
SELECT
    p.name AS ProfileName,
    a.name AS AccountName,
    pa.sequence_number AS SequenceNumber
FROM msdb.dbo.sysmail_profileaccount AS pa
INNER JOIN msdb.dbo.sysmail_profile AS p
    ON pa.profile_id = p.profile_id
INNER JOIN msdb.dbo.sysmail_account AS a
    ON pa.account_id = a.account_id
ORDER BY
    p.name,
    pa.sequence_number;
```

## 確認対象
- Database Mailの有効・無効
- SMTPサーバー
- ポート番号
- SSL設定
- メールアカウント
- メールプロファイル
- アカウントとプロファイルの関連付け

## 移行方法
Database Mailの設定はユーザーデータベースのバックアップには含まれず、msdbに保存されている。

SQL Server 2022側で個別に再構築する。

SMTPパスワードは別途再設定する。

## 注意事項
- SMTPサーバーへの接続許可を確認する。
- SQL Server 2022側のDatabase Mail XPsを有効にする。
- プロファイルの既定設定やアクセス権も確認する。

# ㉑ SQL Server Agent 設定

## 目的
SQL Server Agentに登録されているジョブ、スケジュール、通知設定を確認する。

## 確認SQL

```sql
-- 1. SQL Server Agent ジョブ一覧
SELECT
    j.name AS JobName,
    j.enabled AS IsEnabled,
    j.description AS Description,
    SUSER_SNAME(j.owner_sid) AS OwnerName,
    j.date_created AS CreatedDate,
    j.date_modified AS ModifiedDate
FROM msdb.dbo.sysjobs AS j
ORDER BY j.name;

-- 2. ジョブステップ
SELECT
    j.name AS JobName,
    s.step_id AS StepID,
    s.step_name AS StepName,
    s.subsystem AS Subsystem,
    s.database_name AS DatabaseName,
    s.command AS Command
FROM msdb.dbo.sysjobsteps AS s
INNER JOIN msdb.dbo.sysjobs AS j
    ON s.job_id = j.job_id
ORDER BY j.name, s.step_id;

-- 3. ジョブスケジュール
SELECT
    j.name AS JobName,
    s.name AS ScheduleName,
    s.enabled AS IsEnabled,
    s.freq_type AS FrequencyType,
    s.freq_interval AS FrequencyInterval,
    s.active_start_date AS StartDate,
    s.active_start_time AS StartTime
FROM msdb.dbo.sysjobschedules AS js
INNER JOIN msdb.dbo.sysjobs AS j
    ON js.job_id = j.job_id
INNER JOIN msdb.dbo.sysschedules AS s
    ON js.schedule_id = s.schedule_id
ORDER BY j.name, s.name;

-- 4. オペレーター一覧
SELECT
    name AS OperatorName,
    enabled AS IsEnabled,
    email_address AS EmailAddress,
    pager_address AS PagerAddress
FROM msdb.dbo.sysoperators
ORDER BY name;

-- 5. ジョブ通知設定
SELECT
    j.name AS JobName,
    o.name AS OperatorName,
    n.notification_method AS NotificationMethod
FROM msdb.dbo.sysnotifications AS n
INNER JOIN msdb.dbo.sysjobs AS j
    ON n.job_id = j.job_id
INNER JOIN msdb.dbo.sysoperators AS o
    ON n.operator_id = o.id
ORDER BY j.name, o.name;
```

## 確認対象
- SQL Server Agentジョブ
- ジョブの有効・無効
- ジョブステップと実行コマンド
- 実行対象データベース
- スケジュール
- オペレーター
- メール通知設定

## 移行方法
SQL Server 2014 ExpressにはSQL Server Agentがないため、通常は移行元にAgentジョブは存在しない。

SQL Server 2022 StandardではSSMSの「ジョブ」から新規作成できる。

ジョブを移行する場合は、SSMSの「スクリプト生成」またはT-SQLで再作成する。

## 注意事項
- ジョブの所有者となるログインを先に作成する。
- ジョブステップ内のファイルパスや実行ファイルの配置を確認する。
- サービスアカウントの権限を確認する。
- メール通知を利用する場合はDatabase Mailの設定が必要。
- Expressで利用していたWindowsタスクスケジューラのバッチ処理も別途確認する。

# ㉒ SQL Server バックアップ設定

## 目的
SQL Serverのバックアップ保存先、バックアップ履歴、バックアップ方式を確認する。

## 確認SQL

```sql
-- 1. データベースのバックアップ履歴
SELECT
    bs.database_name AS DatabaseName,
    bs.type AS BackupType,
    CASE bs.type
        WHEN 'D' THEN 'FULL'
        WHEN 'I' THEN 'DIFFERENTIAL'
        WHEN 'L' THEN 'LOG'
        ELSE bs.type
    END AS BackupTypeName,
    bs.backup_start_date AS BackupStartDate,
    bs.backup_finish_date AS BackupFinishDate,
    bs.backup_size / 1024 / 1024 AS BackupSizeMB,
    bmf.physical_device_name AS BackupFilePath
FROM msdb.dbo.backupset AS bs
INNER JOIN msdb.dbo.backupmediafamily AS bmf
    ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name NOT IN ('master', 'model', 'msdb', 'tempdb')
ORDER BY bs.backup_finish_date DESC;

-- 2. データベースの復旧モデル
SELECT
    name AS DatabaseName,
    recovery_model_desc AS RecoveryModel
FROM sys.databases
WHERE database_id > 4;

-- 3. インスタンスのバックアップ圧縮既定値
SELECT
    name AS SettingName,
    value AS ConfiguredValue,
    value_in_use AS RunningValue
FROM sys.configurations
WHERE name = 'backup compression default';
```

## 確認対象
- 完全バックアップの履歴
- 差分バックアップの履歴
- トランザクションログバックアップの履歴
- バックアップ保存先
- バックアップサイズ
- データベースの復旧モデル
- バックアップ圧縮の既定設定

## 移行方法
バックアップ履歴はmsdbに保存されており、ユーザーデータベースのバックアップには含まれない。

SQL Server 2022側では、必要に応じてバックアップジョブや保存先を再設定する。

今回のデータベース移行では、旧PCで取得した完全バックアップ（.bak）を新PCへコピーして復元する。

## 注意事項
- バックアップ履歴は移行に必須ではない。
- 旧PCと新PCでバックアップ保存先が異なる場合は変更する。
- バックアップファイルの保存先にSQL Serverサービスアカウントの書き込み権限が必要。
- ExpressではSQL Server Agentを利用できないため、Windowsタスクスケジューラによるバックアップ処理も確認する。
- 移行後は定期バックアップの仕組みを新PCで整備する。

# ㉓ サーバートリガー設定

## 目的
SQL Serverインスタンスに登録されているサーバートリガー（DDLトリガー・ログオントリガー）を確認する。

## 確認SQL

```sql
-- 1. サーバートリガー一覧
SELECT
    t.name AS TriggerName,
    t.type_desc AS TriggerType,
    t.is_disabled AS IsDisabled,
    t.create_date AS CreateDate,
    t.modify_date AS ModifyDate,
    t.is_ms_shipped AS IsSystemObject
FROM sys.server_triggers AS t
ORDER BY t.name;

-- 2. サーバートリガー定義
SELECT
    t.name AS TriggerName,
    t.type_desc AS TriggerType,
    t.is_disabled AS IsDisabled,
    sm.definition AS TriggerDefinition
FROM sys.server_triggers AS t
LEFT JOIN sys.server_sql_modules AS sm
    ON t.object_id = sm.object_id
ORDER BY t.name;
```

## 確認対象
- サーバーDDLトリガー
- ログオントリガー
- トリガーの有効・無効状態
- トリガーの作成日時・更新日時
- トリガーの定義内容

## 移行方法
サーバートリガーはユーザーデータベースのバックアップには含まれない。

SQL Server 2022側でCREATE TRIGGER文を使用して再作成する。

無効状態についても移行元と同じ状態に設定する。

## 注意事項
- 暗号化されたトリガーは定義を取得できない場合がある。
- ログオントリガーは接続制御に影響するため、移行前に処理内容を確認する。
- トリガー内で参照しているデータベース名、ログイン名、ファイルパスなどを確認する。
- システムトリガーは移行対象から除外する。
- SQL Server 2022側で再作成後、動作確認を行う。

# ㉔ データ・ログ・バックアップの既定保存先

## 目的
SQL Serverが新規データベースやバックアップファイルを作成するときに使用する既定の保存先を確認する。

## 確認SQL

```sql
SELECT
    SERVERPROPERTY('InstanceDefaultDataPath')
        AS DefaultDataPath,
    SERVERPROPERTY('InstanceDefaultLogPath')
        AS DefaultLogPath,
    SERVERPROPERTY('InstanceDefaultBackupPath')
        AS DefaultBackupPath;
```

## 確認対象
- データファイル（MDF / NDF）の既定保存先
- ログファイル（LDF）の既定保存先
- バックアップファイル（BAK）の既定保存先

## 移行方法
これらの設定はデータベースのバックアップには含まれない。

SQL Server 2022側で、新PCのディスク構成に合わせて保存先を設定する。

## 注意事項
- 旧PCと新PCでドライブ構成が異なる場合は変更する。
- SQL Serverサービスアカウントに対象フォルダーへのアクセス権が必要。
- 既存データベースのファイル配置と、新規作成時の既定保存先は別の設定である。
- SQL Server構成マネージャーやレジストリ設定を利用して変更する。

# ㉕ サーバー構成・リモート接続設定

## 目的
SQL Serverのリモート接続設定とサーバー間通信に関する構成を確認する。

## 確認SQL

```sql
-- 1. リモートアクセス設定
SELECT
    name AS SettingName,
    value AS ConfiguredValue,
    value_in_use AS RunningValue,
    description AS Description
FROM sys.configurations
WHERE name IN (
    'remote access',
    'remote admin connections',
    'remote query timeout'
)
ORDER BY name;

-- 2. サーバー名とネットワーク情報
SELECT
    SERVERPROPERTY('ServerName') AS ServerName,
    SERVERPROPERTY('MachineName') AS MachineName,
    SERVERPROPERTY('InstanceName') AS InstanceName,
    SERVERPROPERTY('ComputerNamePhysicalNetBIOS')
        AS PhysicalNetBIOS;
```

## 確認対象
- リモートアクセスの有効・無効
- リモート管理接続の設定
- リモートクエリのタイムアウト
- SQL Serverのサーバー名
- コンピューター名
- インスタンス名

## 移行方法
これらはデータベースバックアップには含まれない。

SQL Server 2022側で必要な設定を再適用する。

## 注意事項
- `remote access` は主に古いサーバー間リモートストアドプロシージャ機能に関する設定であり、通常のTCP/IP接続設定とは異なる。
- 通常のリモート接続にはTCP/IPとファイアウォール設定も必要。
- `remote admin connections` は専用管理者接続（DAC）に関する設定。
- 新PCではサーバー名・インスタンス名が変わるため、アプリケーションの接続先も確認する。

# ㉖ セキュリティ・認証関連設定

## 目的
SQL Serverのログイン認証、パスワードポリシー、セキュリティ関連の設定を確認する。

## 確認SQL

```sql
-- 1. サーバー認証モード
SELECT
    SERVERPROPERTY('IsIntegratedSecurityOnly')
        AS WindowsAuthenticationOnly;

-- 2. SQL Serverログインのパスワードポリシー
SELECT
    name AS LoginName,
    is_disabled AS IsDisabled,
    is_policy_checked AS IsPolicyChecked,
    is_expiration_checked AS IsExpirationChecked,
    create_date AS CreateDate,
    modify_date AS ModifyDate
FROM sys.sql_logins
ORDER BY name;

-- 3. SQL Serverログインの既定データベース
SELECT
    name AS LoginName,
    default_database_name AS DefaultDatabase,
    default_language_name AS DefaultLanguage
FROM sys.server_principals
WHERE type = 'S'
ORDER BY name;

-- 4. パスワードポリシーに関するサーバー設定
SELECT
    name AS SettingName,
    value AS ConfiguredValue,
    value_in_use AS RunningValue,
    description AS Description
FROM sys.configurations
WHERE name IN (
    'common criteria compliance enabled',
    'contained database authentication',
    'cross db ownership chaining',
    'scan for startup procs'
)
ORDER BY name;
```

## 確認対象
- Windows認証・混合モード
- SQL Serverログインの有効・無効
- パスワードポリシー適用状態
- パスワード有効期限設定
- デフォルトデータベース
- デフォルト言語
- セキュリティ関連のサーバー構成

## 移行方法
認証モードはSQL Server 2022側で混合モードに設定する。

SQL ServerログインはSIDとパスワードハッシュを引き継ぐ方法で再作成する。

パスワードポリシーや各種サーバー設定は、移行元の状態を確認して必要なものを再設定する。

## 注意事項
- Windows認証ログインはWindows側のアカウント・グループも確認する。
- SQL Serverログインのパスワードハッシュは通常のログイン一覧から取得できないため、専用の移行スクリプトが必要。
- パスワードポリシーはWindows側のセキュリティポリシーにも依存する。
- SQL Server 2022では旧バージョンと利用可能なセキュリティ機能が異なる場合がある。
- 不要なセキュリティ機能を無条件に有効化しない。

# ㉗ SQL Server プロキシ設定

## 目的
SQL Server Agentのジョブステップを、SQL Serverサービスアカウント以外のWindowsアカウントで実行するためのプロキシ設定を確認する。

## 確認SQL

```sql
-- 1. SQL Server Agentの資格情報
SELECT
    credential_id AS CredentialID,
    name AS CredentialName,
    credential_identity AS CredentialIdentity,
    create_date AS CreateDate,
    modify_date AS ModifyDate
FROM master.sys.credentials
ORDER BY name;

-- 2. SQL Server Agentのプロキシ一覧
SELECT
    p.proxy_id AS ProxyID,
    p.name AS ProxyName,
    c.name AS CredentialName,
    c.credential_identity AS CredentialIdentity,
    p.enabled AS IsEnabled,
    p.description AS Description
FROM msdb.dbo.sysproxies AS p
LEFT JOIN master.sys.credentials AS c
    ON p.credential_id = c.credential_id
ORDER BY p.name;

-- 3. プロキシに許可されたサブシステム
SELECT
    p.name AS ProxyName,
    s.subsystem AS Subsystem
FROM msdb.dbo.sysproxies AS p
INNER JOIN msdb.dbo.sysproxysubsystem AS ps
    ON p.proxy_id = ps.proxy_id
INNER JOIN msdb.dbo.syssubsystems AS s
    ON ps.subsystem_id = s.subsystem_id
ORDER BY p.name, s.subsystem;

-- 4. プロキシの使用許可ログイン
SELECT
    p.name AS ProxyName,
    sp.name AS LoginName
FROM msdb.dbo.sysproxies AS p
INNER JOIN msdb.dbo.sysproxylogin AS pl
    ON p.proxy_id = pl.proxy_id
INNER JOIN sys.server_principals AS sp
    ON pl.sid = sp.sid
ORDER BY p.name, sp.name;
```

## 確認対象
- SQL Server Agentプロキシ
- 資格情報名
- Windows実行アカウント
- プロキシの有効・無効
- 実行可能なサブシステム
- プロキシの使用を許可されたログイン

## 移行方法
資格情報とプロキシはユーザーデータベースのバックアップには含まれない。

SQL Server 2022側で資格情報を再作成し、プロキシとサブシステムの関連付け、使用許可ログインを再設定する。

## 注意事項
- 資格情報のパスワードはSQLから取得できないため、別途再設定する。
- Windowsアカウントの存在と必要なアクセス権を確認する。
- SQL Server 2014 ExpressにはSQL Server Agentがないため、通常は移行元にAgentプロキシは存在しない。
- SQL Server 2022 StandardでAgentを利用する場合に必要となる設定。

# ㉘ リソース管理設定（Resource Governor）

## 目的
SQL ServerのCPU・メモリなどのリソース制御設定を確認する。

## 確認SQL

```sql
-- 1. Resource Governorの有効状態
SELECT
    is_enabled AS IsEnabled,
    classifier_function_id AS ClassifierFunctionID,
    max_outstanding_io_per_volume
        AS MaxOutstandingIOPerVolume
FROM sys.resource_governor_configuration;

-- 2. リソースプール
SELECT
    pool_id AS PoolID,
    name AS PoolName,
    min_cpu_percent AS MinCPUPercent,
    max_cpu_percent AS MaxCPUPercent,
    min_memory_percent AS MinMemoryPercent,
    max_memory_percent AS MaxMemoryPercent,
    cap_cpu_percent AS CapCPUPercent
FROM sys.resource_governor_resource_pools
ORDER BY pool_id;

-- 3. ワークロードグループ
SELECT
    group_id AS GroupID,
    name AS GroupName,
    pool_id AS PoolID,
    importance AS Importance,
    request_max_memory_grant_percent
        AS RequestMaxMemoryGrantPercent,
    request_max_cpu_time_sec
        AS RequestMaxCPUTimeSec,
    max_dop AS MaxDOP
FROM sys.resource_governor_workload_groups
ORDER BY group_id;
```

## 確認対象
- Resource Governorの有効・無効
- 分類関数
- リソースプール
- CPU使用率の制限
- メモリ使用率の制限
- ワークロードグループ
- クエリの実行制限

## 移行方法
Resource Governorの設定はユーザーデータベースのバックアップには含まれない。

SQL Server 2022側で必要に応じてリソースプール、ワークロードグループ、分類関数を再作成する。

## 注意事項
- SQL Server 2014 ExpressではResource Governorは利用できない。
- 移行元がExpressの場合、通常は既定状態であり、個別移行は不要。
- SQL Server 2022 StandardではResource Governorの利用可否・機能制限を確認する。
- 不要なリソース制限を移行先に設定しない。

# ㉙ tempdb 設定

## 目的
tempdbのデータファイル・ログファイルの構成、サイズ、自動拡張設定、保存先を確認する。

## 確認SQL

```sql
-- 1. tempdbのファイル構成
SELECT
    name AS LogicalFileName,
    type_desc AS FileType,
    physical_name AS PhysicalPath,
    size * 8 / 1024 AS SizeMB,
    CASE
        WHEN max_size = -1 THEN -1
        ELSE max_size * 8 / 1024
    END AS MaxSizeMB,
    CASE
        WHEN is_percent_growth = 1
            THEN CAST(growth AS VARCHAR(20)) + '%'
        ELSE CAST(growth * 8 / 1024 AS VARCHAR(20)) + ' MB'
    END AS AutoGrowth
FROM tempdb.sys.database_files
ORDER BY file_id;

-- 2. tempdbの現在の使用状況
SELECT
    SUM(user_object_reserved_page_count) * 8 / 1024
        AS UserObjectReservedMB,
    SUM(internal_object_reserved_page_count) * 8 / 1024
        AS InternalObjectReservedMB,
    SUM(version_store_reserved_page_count) * 8 / 1024
        AS VersionStoreReservedMB,
    SUM(unallocated_extent_page_count) * 8 / 1024
        AS UnallocatedSpaceMB
FROM tempdb.sys.dm_db_file_space_usage;

-- 3. tempdbの照合順序
SELECT
    name AS DatabaseName,
    collation_name AS CollationName
FROM sys.databases
WHERE name = 'tempdb';
```

## 確認対象
- tempdbのデータファイル数
- データファイル名
- ログファイル名
- ファイル保存先
- 初期サイズ
- 最大サイズ
- 自動拡張設定
- tempdbの使用状況
- tempdbの照合順序

## 移行方法
tempdbはSQL Serverの起動時に再作成されるシステムデータベースであり、通常のデータベースバックアップ・リストアによる移行対象ではない。

SQL Server 2022側でファイル数、サイズ、自動拡張、保存先などを必要に応じて再設定する。

## 注意事項
- 旧PCと新PCではディスク構成が異なるため、保存先は新PCに合わせる。
- tempdbのファイル数はCPU数だけで機械的に決めず、実際の負荷に応じて設定する。
- SQL Server 2014 Expressでは利用可能なCPU・メモリに制限がある。
- tempdbの変更にはSQL Serverサービスの再起動が必要となる場合がある。
- 使用状況は実行時点の情報であり、移行設定値とは区別する。

# ㉚ サーバー依存オブジェクト設定

## 目的
SQL Serverインスタンスに登録されているサーバーレベルのオブジェクトや、外部処理との依存関係を確認する。

## 確認SQL

```sql
-- 1. サーバーレベルのオブジェクト一覧
SELECT
    name AS ObjectName,
    type_desc AS ObjectType,
    create_date AS CreateDate,
    modify_date AS ModifyDate,
    is_ms_shipped AS IsSystemObject
FROM sys.objects
WHERE is_ms_shipped = 0
ORDER BY type_desc, name;

-- 2. サーバーレベルのユーザー定義関数
SELECT
    name AS FunctionName,
    type_desc AS FunctionType,
    create_date AS CreateDate,
    modify_date AS ModifyDate
FROM sys.server_sql_modules AS sm
INNER JOIN sys.server_triggers AS st
    ON sm.object_id = st.object_id
ORDER BY sm.object_id;

-- 3. サーバーに登録されたエンドポイント
SELECT
    name AS EndpointName,
    type_desc AS EndpointType,
    state_desc AS EndpointState
FROM sys.endpoints
ORDER BY name;

-- 4. サーバーロール一覧
SELECT
    name AS RoleName,
    type_desc AS PrincipalType,
    is_fixed_role AS IsFixedRole
FROM sys.server_principals
WHERE type = 'R'
ORDER BY name;
```

## 確認対象
- サーバーレベルのユーザー定義オブジェクト
- サーバートリガー
- エンドポイント
- ユーザー定義サーバーロール
- システムオブジェクト以外の登録情報

## 移行方法
サーバーレベルのオブジェクトはユーザーデータベースのバックアップには含まれない。

必要なオブジェクトを個別にスクリプト化し、SQL Server 2022側で再作成する。

## 注意事項
- データベース内のオブジェクトは通常のバックアップ・リストアで移行されるため、重複して移行しない。
- システムオブジェクトは移行対象から除外する。
- SQL Server 2014と2022で利用可能なオブジェクトや機能が異なる場合がある。
- この項目はサーバーレベルのオブジェクトを対象とし、外部ファイルやWindows側の設定は別途確認する。

# ㉛ 外部接続・OLE DBプロバイダー設定

## 目的
SQL Serverで利用している外部接続プロバイダーと、外部データソースへの接続設定を確認する。

## 確認SQL

```sql
-- 1. 登録済みOLE DBプロバイダー
EXEC master.dbo.xp_enum_oledb_providers;

-- 2. リンクサーバーとプロバイダー
SELECT
    name AS LinkedServerName,
    product AS Product,
    provider AS Provider,
    data_source AS DataSource,
    catalog AS Catalog,
    is_data_access_enabled AS IsDataAccessEnabled,
    is_rpc_out_enabled AS IsRpcOutEnabled
FROM sys.servers
WHERE server_id <> 0
ORDER BY name;

-- 3. OLE DBプロバイダー関連のサーバー設定
SELECT
    name AS SettingName,
    value AS ConfiguredValue,
    value_in_use AS RunningValue,
    description AS Description
FROM sys.configurations
WHERE name IN (
    'Ad Hoc Distributed Queries',
    'DisallowAdhocAccess'
)
ORDER BY name;
```

## 確認対象
- 登録済みOLE DBプロバイダー
- リンクサーバーで使用するプロバイダー
- 外部データソース
- Ad Hoc Distributed Queriesの設定
- 外部接続の利用状況

## 移行方法
OLE DBプロバイダーはSQL Serverのデータベースバックアップには含まれない。

SQL Server 2022側に対応するプロバイダーをインストールし、必要なリンクサーバーや接続設定を再構築する。

## 注意事項
- SQL Server 2022では古いOLE DBプロバイダーが利用できない場合がある。
- 32bit / 64bitの違いに注意する。
- 外部接続先のドライバーや認証情報も確認する。
- プロバイダーのインストール後、SQL Serverサービスの再起動が必要となる場合がある。
- リンクサーバー設定は⑰で確認済みのため、ここではプロバイダーの依存関係を重点的に確認する。
- `DisallowAdhocAccess` はプロバイダー固有のレジストリ設定であり、通常の `sys.configurations` には存在しない。このSQLでは取得できないため、レジストリ側で別途確認する。

# ㉜ Windows タスクスケジューラ設定

## 目的
Windowsタスクスケジューラに登録されているSQL Server関連の自動実行処理を確認する。

## 確認コマンド

以下はSQLではなく、旧PCのPowerShellで実行するコマンド。

```powershell
# 1. 登録済みタスク一覧
Get-ScheduledTask |
    Select-Object `
        TaskPath,
        TaskName,
        State,
        Author,
        Description |
    Format-Table -AutoSize

# 2. タスクの実行アクション
Get-ScheduledTask | ForEach-Object {
    $task = $_

    foreach ($action in $task.Actions) {
        [PSCustomObject]@{
            TaskName = $task.TaskName
            TaskPath = $task.TaskPath
            Execute  = $action.Execute
            Arguments = $action.Arguments
            WorkingDirectory = $action.WorkingDirectory
        }
    }
} | Format-List

# 3. タスクのトリガー
Get-ScheduledTask | ForEach-Object {
    $task = $_

    foreach ($trigger in $task.Triggers) {
        [PSCustomObject]@{
            TaskName = $task.TaskName
            TaskPath = $task.TaskPath
            TriggerType = $trigger.CimClass.CimClassName
            StartBoundary = $trigger.StartBoundary
            Enabled = $trigger.Enabled
        }
    }
} | Format-List
```

## 確認対象
- 登録済みタスク
- タスク名・保存先
- 実行ファイル・バッチファイル
- 実行引数
- 作業フォルダー
- 実行スケジュール
- タスクの有効・無効状態

## 移行方法
WindowsタスクスケジューラのタスクはSQL Serverのバックアップには含まれない。

必要なタスクをXML形式でエクスポートし、新PCのタスクスケジューラにインポートする。

## 注意事項
- 実行ユーザーとパスワードは新PCで再設定する。
- バッチファイルやPowerShellスクリプトも別途コピーする。
- SQL Serverの接続先・インスタンス名・ポート番号を変更する。
- ファイルパスやドライブ構成を新PCに合わせる。
- タスクの実行権限とフォルダーアクセス権を確認する。
- SQL Server関連以外のタスクは、今回の移行対象から除外する。

# ㉝ SQL Server 構成・インストール環境の最終確認

## 目的
SQL Serverのインストール構成、エディション、インスタンス、機能、サービス情報を確認し、移行先のインストール構成を決定する。

## 確認SQL

```sql
-- 1. SQL Serverの基本情報
SELECT
    SERVERPROPERTY('ServerName') AS ServerName,
    SERVERPROPERTY('MachineName') AS MachineName,
    SERVERPROPERTY('InstanceName') AS InstanceName,
    SERVERPROPERTY('Edition') AS Edition,
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('ProductUpdateLevel') AS ProductUpdateLevel,
    SERVERPROPERTY('Collation') AS ServerCollation,
    SERVERPROPERTY('IsClustered') AS IsClustered,
    SERVERPROPERTY('IsHadrEnabled') AS IsHadrEnabled;

-- 2. SQL Serverサービス情報
SELECT
    servicename AS ServiceName,
    startup_type_desc AS StartupType,
    status_desc AS Status,
    service_account AS ServiceAccount,
    filename AS ExecutablePath
FROM sys.dm_server_services
ORDER BY servicename;

-- 3. SQL Serverの言語設定
SELECT
    @@LANGUAGE AS CurrentLanguage,
    @@DATEFIRST AS DateFirst,
    @@OPTIONS AS SessionOptions;

-- 4. サーバーの既定言語
SELECT
    name AS LoginName,
    default_language_name AS DefaultLanguage
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY name;
```

## 確認対象
- SQL Serverのエディション
- 製品バージョン・更新レベル
- インスタンス名
- サーバー照合順序
- クラスター構成
- Always On設定
- SQL Serverサービス
- サービスアカウント
- SQL Serverの言語設定
- ログインごとの既定言語

## 移行方法
SQL Server 2022 Standardのインストール時に、必要なインスタンス構成や照合順序を設定する。

サービスアカウント、起動方式、言語設定などは移行先の環境に合わせて再設定する。

## 注意事項
- SQL Server 2014 Expressから2022 Standardでは利用可能な機能が異なる。
- 旧PCと新PCでインスタンス名が変わる場合、接続文字列の変更が必要。
- サーバー照合順序はインストール後の変更が容易ではないため、事前に決定する。
- サービスアカウントは旧PCのアカウントをそのまま流用できるとは限らない。
- SQL Serverの更新プログラムは移行先で別途適用する。
- `@@LANGUAGE` は現在の接続セッションの言語であり、インスタンス全体の既定言語とは異なる。ログインごとの既定言語も確認する。

# ㉞ 移行漏れ・依存関係の最終確認

## 目的
データベース本体、SQL Serverインスタンス設定、外部接続、Windows環境を含めて移行対象の漏れを確認する。

## 確認SQL

```sql
-- 1. ユーザーデータベース一覧
SELECT
    name AS DatabaseName,
    state_desc AS State,
    compatibility_level AS CompatibilityLevel,
    collation_name AS CollationName,
    recovery_model_desc AS RecoveryModel,
    SUSER_SNAME(owner_sid) AS OwnerName
FROM sys.databases
WHERE database_id > 4
ORDER BY name;

-- 2. ログイン一覧
SELECT
    name AS LoginName,
    type_desc AS LoginType,
    is_disabled AS IsDisabled,
    default_database_name AS DefaultDatabase
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY name;

-- 3. リンクサーバー一覧
SELECT
    name AS LinkedServerName,
    provider AS Provider,
    data_source AS DataSource,
    is_data_access_enabled AS IsDataAccessEnabled,
    is_rpc_out_enabled AS IsRpcOutEnabled
FROM sys.servers
WHERE server_id <> 0
ORDER BY name;

-- 4. サーバー資格情報一覧
SELECT
    name AS CredentialName,
    credential_identity AS CredentialIdentity
FROM sys.credentials
ORDER BY name;

-- 5. サーバートリガー一覧
SELECT
    name AS TriggerName,
    is_disabled AS IsDisabled,
    create_date AS CreateDate,
    modify_date AS ModifyDate
FROM sys.server_triggers
ORDER BY name;

-- 6. エンドポイント一覧
SELECT
    name AS EndpointName,
    type_desc AS EndpointType,
    state_desc AS EndpointState
FROM sys.endpoints
ORDER BY name;

-- 7. SQL Server構成設定
SELECT
    name AS SettingName,
    value AS ConfiguredValue,
    value_in_use AS RunningValue
FROM sys.configurations
ORDER BY name;
```

## 確認対象
- ユーザーデータベース
- SQL Serverログイン
- サーバーロール・権限
- リンクサーバー
- 資格情報
- サーバートリガー
- エンドポイント
- SQL Server構成設定
- Database Mail
- バックアップ設定
- Windowsタスクスケジューラ
- 外部接続・プロバイダー
- サービス・ネットワーク設定

## 移行方法
これまでの確認項目をもとに、移行対象を以下に分類する。

1. データベースバックアップ・リストアで移行するもの
2. SQLスクリプトで再作成するもの
3. SQL Server構成マネージャーで設定するもの
4. Windows側で再設定するもの
5. 新環境では不要なもの

## 注意事項
- データベースバックアップで移行できる情報を重複して再作成しない。
- ログインとデータベースユーザーのSID対応を維持する。
- SQL Server 2014 Expressで利用できない機能は、移行元に存在すると仮定しない。
- SQL Server 2022 Standardで利用可能な機能を確認する。
- 移行元と移行先の設定差分を確認する。
- 移行完了後も旧PCは一定期間保持し、切り戻し可能な状態にする。
