# lhci計測Template共通化時のアプリ差分の扱い

## 目的

- LighthouseCIを複数フロントアプリへ横展開する際、組み込み方式とアプリ差分の受け持ち先を整理する。
- 共通pipeline-template化に向けて、意思決定に必要な論点と選択肢を提示できる状態にする。

### 1) 差分の棚卸し

| 差分カテゴリ | 差分項目 | `.gitlab-yml` | `lighthouserc` | 必須/任意 | デフォルト案 |
|---|---|---|---|---|---|
| 実行環境 | Nodeバージョン | `image` | - | 任意 | 20 |
| 実行環境 | パッケージマネージャ | `script` | - | 必須 | npm |
| 実行環境 | ESM/CJS | - | `collect` | 任意 | CJS |
| アプリ構成 | buildコマンド | `script` | - | 任意※1 | - |
| アプリ構成 | startコマンド | `script` | `collect` | 任意※2 | - |
| アプリ構成 | 配信方式 | `script` | `collect` | 必須 | server起動 |
| アプリ構成 | base path | - | `collect` | 任意 | `/` |
| 計測条件 | 対象URL | - | `collect` | 必須 | - |
| 計測条件 | mobile/desktop | - | `collect` | 任意 | desktop |
| 計測条件 | 実行回数 | - | `collect` | 任意 | 3 |
| 計測条件 | assertion閾値 | - | `assert` | 任意 | デフォルトプロファイル |
| 外部依存 | API依存 | `script` | `collect` | 任意 | なし |
| 外部依存 | モック要否 | `before_script` | - | 任意 | なし |
| 認証 | 認証要否 | `before_script` | `collect` | 任意 | none |

※1 静的配信（staticDistDir）の場合は必須、サーバ起動の場合は不要なケースもある
※2 静的配信（staticDistDir）の場合は不要

#### LHCIコマンド一覧（参考）

> 出典: [LHCI configuration.md](https://github.com/GoogleChrome/lighthouse-ci/blob/main/docs/configuration.md#file-structure)

| コマンド | 役割 | 対応する `lighthouserc` セクション |
|---|---|---|
| `lhci autorun` | collect → assert → upload を一括実行 | `collect` / `assert` / `upload` |
| `lhci collect` | Lighthouseでページを計測し `.lighthouseci/` に保存 | `collect` |
| `lhci assert` | 計測結果をしきい値と照合し合否判定 | `assert` |
| `lhci upload` | 計測結果を LHCI server / temporary-public-storage / filesystem に保存 | `upload` |
| `lhci healthcheck` | 設定の診断（セットアップ確認用） | - |
| `lhci open` | ローカルのレポートをブラウザで開く | - |
| `lhci wizard` | 対話形式でプロジェクト作成・トークンリセット | `wizard` |
| `lhci server` | LHCI結果管理サーバを起動 | `server` |

**CIパイプラインで使うのは主に `autorun`（または `collect` + `assert` + `upload` の個別実行）**

### 2) 差分の受け持ち先（案）

| 差分項目 | 受け持ち先（第一候補） | 受け持ち先（第二候補） | 共通template側の責務 | アプリ側の責務 |
|---|---|---|---|---|
| **▼実行環境** | | | | |
| Nodeバージョン | YAML（image） | variables | 共通固定 or 上書き可能にする | 変数で上書き |
| パッケージマネージャ | variables | template分割 | install処理の共通化 | lockfile/コマンド指定 |
| ESM/CJS | config | - | config読み込み方針の統一 | configファイル配置 |
| **▼アプリ構成（GitLabCI観点）** | | | | |
| buildコマンド | variables | script | 変数展開でコマンド実行 | 変数定義 or npm script |
| startコマンド | variables | config | 変数展開 or config読み込み | 変数/config定義 |
| 配信方式 | config | variables | 両パターン(static/server)のscript提供 | 配信方式の選択 |
| base path | config | variables | config読み込み | configで定義 |
| **▼計測条件（LighthouseCI観点）** | | | | |
| 対象URL | config | variables | config読み込み | configで定義 |
| mobile/desktop | config | - | デフォルトdesktop提供 | configで上書き |
| 実行回数 | config | - | デフォルト値(3)提供 | configで上書き |
| assertion閾値 | config | - | デフォルトプロファイル提供 | configでカスタム |
| **▼外部依存（モック/API観点）** | | | | |
| API依存 | config | script | startコマンドでのAPI接続考慮 | 環境変数/config定義 |
| モック要否 | variables | before_script | モック起動hook提供 | モック定義/変数指定 |
| **▼認証観点** | | | | |
| 認証要否 | config | puppeteerScript | 認証hookの共通化 | 認証情報/script定義 |

### 3) 共通変数（案）

| 変数名 | 必須/任意 | 例 | 備考 |
|---|---|---|---|
| **▼ アプリ構成（GitLabCI観点）** | | | |
| `APP_BUILD_CMD` | 必須 | `npm run build` | |
| `APP_START_CMD` | 任意 | `npm run start` | |
| `APP_HOST` | 任意 | `localhost` | |
| `APP_PORT` | 必須 | `3000` | |
| `APP_BASE_URL` | 任意 | `http://localhost:3000` | |
| **▼ 計測条件（Lighthouse CI観点）** | | | |
| `LHCI_CONFIG_PATH` | 任意 | `./lighthouserc.js` | 未指定時はプロジェクトルートを検索 |
| `LHCI_TARGET_URLS` | 任意 | `http://localhost:3000/,http://localhost:3000/about` | config未定義時のフォールバック |
| `LHCI_ASSERT_PRESET` | 任意 | `lighthouse:recommended` | デフォルトプロファイル使用 |
| `READINESS_URL` | 任意 | `http://localhost:3000/health` | サーバ起動確認用エンドポイント |
| `READINESS_TIMEOUT_SEC` | 任意 | `60` | 起動待ちタイムアウト秒数 |
| **▼ モック/API観点** | | | |
| `MOCK_START_CMD` | 任意 | `npm run mock` | モック起動コマンド |
| **▼ 認証観点** | | | |
| `AUTH_MODE` | 任意 | `none` / `basic` / `puppeteer` | 認証方式の指定 |


### 4) テンプレ分割(案)

| 類型 | 想定アプリ | 特徴 | 前提条件 | 共通化方針 |
|---|---|---|---|---|
| A: 静的（SSG） | SSG/静的ホスティング | `staticDistDir`中心 | distディレクトリが生成されること | ベーステンプレ（最小構成） |
| B: サーバ起動 | SPA/SSR | `startServerCommand`中心 | startコマンドでサーバ起動可能 | サーバ起動・待機処理をテンプレ化 |
| C: 認証あり | ログイン後導線あり | 認証hookが必須 | 認証情報が環境変数で渡せること | 認証hookをextends可能に |
| D: モック必須 | CIで外部API接続不可 | モック起動/切替が必須 | モックサーバが独立起動可能 | モック起動スクリプト提供 |

