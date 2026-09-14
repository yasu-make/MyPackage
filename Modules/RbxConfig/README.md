# RbxConfig

Roblox `ConfigService` のシングルトンラッパーです。サーバーでクラウド設定を読み、クライアントへ同期します。

クラウド上の設定を使う場合は本モジュール、Studio 上の Configuration / Attribute / テーブルなどローカル設定を使う場合は [BetterConfig](../BetterConfig/README.md) を使ってください。

## インストール

Wally パッケージ名は `zac134/rbx-config` です。`sleitnick/signal` に依存します。

```toml
[dependencies]
RbxConfig = "zac134/rbx-config@0.0.2"
```

手動で入れる場合は `Modules/RbxConfig/` を ReplicatedStorage にコピーし、同じく `sleitnick/signal@^2.0` を用意してください。モジュールは `script.Parent.signal` を require します。

利用前に [Creator Dashboard](https://create.roblox.com/) の **Configure → Config Service** でキーを作成してください。キー名は `InitServer` / `InitClient` に渡すデフォルトのテーブルと揃えます。詳細は [ConfigService のドキュメント](https://create.roblox.com/docs/cloud/open-cloud/usage-configuration) を参照してください。

## クイックスタート

サーバーとクライアントで同じデフォルトのテーブルを渡します。共有モジュールに切り出すのが簡単です。

```lua
-- 共有デフォルト（例: ReplicatedStorage/shared/RbxConfigSetting）
return {
    max_players_per_team = 4,
}
```

```lua
-- サーバー
local RbxConfig = require(ReplicatedStorage.Modules.RbxConfig):InitServer(configSettings)
print(RbxConfig:GetValue("max_players_per_team"))

RbxConfig:GetValueChangedSignal("max_players_per_team"):Connect(function(newValue)
    print("changed:", newValue)
end)
```

```lua
-- クライアント
local RbxConfig = require(ReplicatedStorage.Modules.RbxConfig):InitClient(configSettings)
print(RbxConfig:GetValue("max_players_per_team"))
```

より長い実例は次を参照してください。

- [`/Examples/shared/RbxConfigSetting.luau`](../../Examples/shared/RbxConfigSetting.luau)
- [`/Examples/server/RbxConfig.server.luau`](../../Examples/server/RbxConfig.server.luau)
- [`/Examples/client/RbxConfig.client.luau`](../../Examples/client/RbxConfig.client.luau)

## 動作の要点

- シングルトンです。`InitServer` / `InitClient` はそれぞれ一度だけ呼びます。二度目は警告して無視します。
- サーバー初期化時はまずデフォルトを `_values` に入れ、続けて `ConfigService:GetConfigAsync()` でグローバルスナップショットを 1 回取得して同じキーを上書きします。取得失敗時はデフォルトのままです。
- クライアントは `InitClient` 時に `RemoteFunction`（`RbxConfigRemoteFunction`）でサーバーの `_values` をまとめて取得します。失敗時は渡したデフォルトを使います。
- その後の更新は `RemoteEvent`（`RbxConfigRemoteEvent`）でキー単位に配信されます。

### 値の優先順位

1. テスト上書き（`SetTestingValue`。サーバーセッション内のみ）
2. スナップショット由来の `_values`
3. `InitServer` / `InitClient` に渡したデフォルト

### 注意（現在の実装）

**遅延監視（lazy observation）**  
ConfigService のライブ更新を購読し、クライアントへ配信するのは `GetValueChangedSignal` を呼んだキーだけです。`GetValue` だけで読んでいるキーは、初期スナップショットのまま残ることがあります。クラウド側を変えても自動では追従しません。ライブ更新が必要なら、そのキーで `GetValueChangedSignal` を呼んでください。

**プレイヤー別スナップショット**  
`GetValueForPlayer` は初回で `ConfigService:GetConfigForPlayerAsync` の結果をキャッシュし、`PlayerRemoving` で捨てます。キャッシュ後は再取得も `Refresh` もしないので、入室後のクラウド変更は反映されません。

**テスト上書き**  
`SetTestingValue` / `ClearTestingValue` はサーバープロセス限りです。再起動で消えます。設定すると全クライアントへ即座に配信します（ライブ監視の有無は問いません）。本番ロジックには使わないでください。

## API

### `InitServer(configSettings) → ServerConfigClass`

サーバー用に初期化します。戻り値は `GetValue` / `GetValueChangedSignal` に加え、サーバー専用メソッドを持ちます。

### `InitClient(configSettings) → RbxConfig`

クライアント用に初期化します。戻り値は `GetValue` と `GetValueChangedSignal` のみです。`configSettings` はサーバーと同じキー・デフォルトにしてください。

### サーバー・クライアント共通

| メソッド | 説明 |
|---|---|
| `GetValue(key)` | 上記の優先順位で現在値を返します。 |
| `GetValueChangedSignal(key)` | 値が変わったときに発火する Signal（`Connect` / `Once` / `Wait`）。サーバーではこの呼び出しでそのキーの ConfigService 監視を開始します。クライアントではサーバーからの `RemoteEvent` を受けて発火します。 |

### サーバーのみ

| メソッド | 説明 |
|---|---|
| `GetValueForPlayer(key, player)` | プレイヤー向けスナップショットから値を返します。テスト上書きがあればそれを優先します。取得失敗時はグローバル値またはデフォルトに戻します。 |
| `SetTestingValue(key, value)` | テスト上書きを設定し、シグナル発火と全クライアントへの配信を行います。 |
| `ClearTestingValue(key)` | 上書きを外し、スナップショット（なければデフォルト）へ戻してクライアントへ配信します。 |

## BetterConfig との違い

| | RbxConfig | BetterConfig |
|---|---|---|
| 設定の置き場 | クラウド（ConfigService） | ローカル（Configuration / Attribute / テーブル） |
| サーバー | 必要 | 不要 |
| 再起動なしの遠隔更新 | 監視しているキーのみ（上記の制限あり） | しない（ローカルのみ） |
| プレイヤー別の値 | 初回キャッシュの `GetValueForPlayer` | なし |

クラウドの A/B や機能フラグには RbxConfig、Studio 上のローカル設定には BetterConfig が向きます。

## ライセンス

MIT License
