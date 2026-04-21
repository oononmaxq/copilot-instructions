# lhci計測Template共通化 - 追加観点

> 本資料は「[lhci計測Template共通化時のアプリ差分の扱い.md](./lhci計測Template共通化時のアプリ差分の扱い.md)」の補足資料です。
> LHCI公式ドキュメント・GitLab CI運用観点・実運用課題の3軸で調査した結果をまとめています。

---

## 1. LHCI設定オプション（追加観点）

元資料「1) 差分の棚卸し」に追加すべき項目。

### 1-A. 実行環境

| 差分項目 | `.gitlab-yml` | `lighthouserc` | 必須/任意 | デフォルト案 | 備考 |
|---|---|---|---|---|---|
| chromeFlags | - | `collect.settings` | **必須** | `--no-sandbox` | Docker/CI環境で必須 |
| headlessモード | - | `collect.settings` | 任意 | `--headless=new` | 新旧で計測結果が異なる |
| CPUスロットリング | - | `collect.settings` | 任意 | `4` | CI環境のCPU性能に応じて調整 |

**補足:**
- `--no-sandbox` はDocker環境での実行に必須。セキュリティ上、CI専用環境でのみ使用を推奨
- `--headless=new` を明示的に指定することで、新しいHeadless Chrome実装を使用（安定性向上）

### 1-B. アプリ構成

| 差分項目 | `.gitlab-yml` | `lighthouserc` | 必須/任意 | デフォルト案 | 備考 |
|---|---|---|---|---|---|
| startServerReadyPattern | - | `collect` | 任意 | `listen\|ready` | サーバー起動検出の正規表現パターン |
| startServerReadyTimeout | - | `collect` | 任意 | `30000` | 起動待機タイムアウト(ms) |

**補足:**
- `startServerReadyPattern` はサーバーのstdout出力から起動完了を判定するパターン
- Next.js: `ready on`、Nuxt: `Listening on`、Express: `listening` など、フレームワークにより異なる

### 1-C. 計測条件

| 差分項目 | `.gitlab-yml` | `lighthouserc` | 必須/任意 | デフォルト案 | 備考 |
|---|---|---|---|---|---|
| aggregationMethod | - | `assert` | 任意 | `median` | 複数実行時の集約方法 |
| throttlingMethod | - | `collect.settings` | 任意 | `simulate` | devtools vs simulate |
| onlyCategories | - | `collect.settings` | 任意 | 全カテゴリ | 計測範囲の絞り込み |
| skipAudits | - | `collect.settings` | 任意 | なし | スキップする監査項目 |
| budgetPath | - | `collect.settings` | 任意 | - | パフォーマンス予算ファイル |

**補足:**
- `aggregationMethod`: `median`（中央値）、`optimistic`（最良値）、`pessimistic`（最悪値）から選択
- `throttlingMethod`: `simulate`（シミュレーション、高速）vs `devtools`（実測、正確）
- `onlyCategories`: `['performance']` のように絞ると計測時間短縮

### 1-D. 認証・ブラウザ制御（新カテゴリ）

| 差分項目 | `.gitlab-yml` | `lighthouserc` | 必須/任意 | デフォルト案 | 備考 |
|---|---|---|---|---|---|
| puppeteerScript | - | `collect` | 任意 | - | ログイン処理等のスクリプトパス |
| extraHeaders | - | `collect` | 任意 | - | カスタムHTTPヘッダー（JSON形式） |
| basicAuth | - | `collect` | 任意 | - | HTTP Basic認証（username, password） |
| disableStorageReset | - | `collect.settings` | 任意 | `false` | 認証状態保持時に`true` |

**補足:**
- `puppeteerScript`: ログイン後のページを計測する場合に必須。スクリプト内でログイン処理を実行
- `disableStorageReset`: `true`にするとlocalStorage/sessionStorageがクリアされない（認証状態保持）

**puppeteerScript例:**
```javascript
module.exports = async (browser, context) => {
  const page = await browser.newPage();
  await page.goto('http://localhost:3000/login');
  await page.type('[name="email"]', process.env.TEST_USER);
  await page.type('[name="password"]', process.env.TEST_PASS);
  await page.click('[type="submit"]');
  await page.waitForNavigation();
  await page.close();
};
```

### 1-E. アップロード・レポート（新カテゴリ）

| 差分項目 | `.gitlab-yml` | `lighthouserc` | 必須/任意 | デフォルト案 | 備考 |
|---|---|---|---|---|---|
| uploadTarget | - | `upload` | 任意 | `temporary-public-storage` | アップロード先 |
| serverBaseUrl | - | `upload` | 任意 | - | LHCIサーバーURL |
| buildToken | `variables` | `upload` | 任意 | - | アップロード認証トークン |
| githubToken | `variables` | `upload` | 任意 | - | GitHub連携トークン |

**補足:**
- `uploadTarget` の選択肢:
  - `temporary-public-storage`: Google公開ストレージ（7日間保持、PoC向け）
  - `lhci`: 自前のLHCIサーバー（長期保存、トレンド分析向け）
  - `filesystem`: ローカルファイル出力

---

## 2. GitLab CI運用観点（新セクション）

元資料では `.gitlab-ci.yml` と `lighthouserc` の設定項目のみ整理されているが、CI/CD運用観点での考慮が不足。

### 2-A. キャッシュ・アーティファクト

| カテゴリ | 項目 | 共通template側 | アプリ側 |
|---|---|---|---|
| **キャッシュ** | node_modules | キャッシュキー設計（lockfileベース） | lockfile提供 |
| **キャッシュ** | ビルド成果物(.next等) | パス定義 | フレームワーク依存 |
| **アーティファクト** | レポート保存期間 | デフォルト90日 | 上書き可 |
| **アーティファクト** | 保存パス | `.lighthouseci/` | - |

**推奨設定例:**
```yaml
cache:
  - key:
      files:
        - package-lock.json
      prefix: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/

artifacts:
  paths:
    - .lighthouseci/
  expire_in: 90 days
```

### 2-B. 実行制御

| カテゴリ | 項目 | 共通template側 | アプリ側 |
|---|---|---|---|
| **並列実行** | matrix (URL/mobile-desktop) | matrix定義 | 変数で指定 |
| **リトライ** | 失敗時のリトライ回数 | `retry: 2` | 上書き可 |
| **リトライ** | リトライ条件 | `stuck_or_timeout_failure` | - |
| **タイムアウト** | ジョブタイムアウト | 変数化 | アプリ毎に設定 |
| **条件実行** | 実行ブランチ/タグ | rules定義 | 上書き可 |
| **条件実行** | 変更検知(monorepo) | - | `rules:changes` |

**推奨設定例:**
```yaml
# 並列実行（mobile/desktop）
parallel:
  matrix:
    - LHCI_MODE: [mobile, desktop]

# リトライ
retry:
  max: 2
  when:
    - stuck_or_timeout_failure
    - script_failure

# 条件実行
rules:
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    when: always
  - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    when: always
```

### 2-C. 共通変数（追加案）

元資料「3) 共通変数（案）」への追加候補。

| 変数名 | 必須/任意 | 例 | 備考 |
|---|---|---|---|
| `LHCI_CHROME_FLAGS` | 任意 | `--no-sandbox` | CI環境用Chromeフラグ |
| `LHCI_UPLOAD_TARGET` | 任意 | `lhci` | アップロード先 |
| `LHCI_SERVER_URL` | 任意 | `https://lhci.example.com` | LHCIサーバーURL |
| `LHCI_BUILD_TOKEN` | 任意 | (masked) | アップロードトークン |
| `ARTIFACT_EXPIRE_IN` | 任意 | `90 days` | レポート保存期間 |
| `CACHE_KEY_PREFIX` | 任意 | `${CI_PROJECT_NAME}` | キャッシュキー接頭辞 |
| `JOB_TIMEOUT` | 任意 | `15m` | ジョブタイムアウト |
| `RETRY_MAX` | 任意 | `2` | リトライ回数 |

---

## 3. 運用・安定性観点（新セクション）

### 3-A. 計測安定性

| 観点 | 課題 | 対策 | 責務 |
|---|---|---|---|
| 結果のばらつき | 7つの変動要因が存在 | 複数回計測+中央値採用 | 共通template |
| CI環境特有の問題 | Docker内でのChrome実行 | chromeFlags設定 | 共通template |
| サードパーティスクリプト | 外部依存で計測不安定化 | puppeteerScriptでブロック | アプリ側 |

**7つの変動要因:**
1. ページの非決定性（A/Bテスト、動的広告）
2. ローカルネットワーク変動
3. Tier-1ネットワーク変動
4. ウェブサーバー変動
5. クライアントハードウェア変動
6. リソース競合（バックグラウンドプロセス）
7. ブラウザ非決定性

**対策:**
- 計測回数: 最低3回実行し、中央値を採用
- CI環境: 専用ランナー使用が理想的
- 外部リクエスト: puppeteerScriptでGA/広告等をブロック

### 3-B. 可視化・通知

| 観点 | 選択肢 | 責務 |
|---|---|---|
| MR/PRコメント | GitHub App / GitLab連携 | 共通template |
| Slack通知 | CI連携 or カスタムWebhook | 共通template |
| トレンド分析 | LHCI Server | インフラ |

**GitHub連携:**
- GitHub Appをインストール・認可
- `LHCI_GITHUB_APP_TOKEN` を環境変数に設定
- PRに自動コメント+レポートリンク付与

**LHCI Server:**
- 時系列でのスコア追跡
- ビルド間の変化差分表示
- パフォーマンス予算の進捗管理

### 3-C. セキュリティ

| 観点 | 考慮点 | 対策 |
|---|---|---|
| トークン管理 | Build Token vs Admin Token | CI Secretsに格納、Build Tokenのみ使用 |
| 認証情報 | puppeteerScriptでの扱い | 環境変数経由、ログ出力防止 |

**トークンの使い分け:**
- **Build Token**: 新規データアップロードのみ（CI用、公開可）
- **Admin Token**: 編集・削除すべて（管理用、厳格管理）

---

## 4. テンプレ分割（追加観点）

元資料「4) テンプレ分割（案）」への追加候補。

| 類型 | 追加考慮点 |
|---|---|
| A: 静的（SSG） | キャッシュ戦略が単純、アーティファクトのみ |
| B: サーバ起動 | `startServerReadyPattern`の設定必須 |
| C: 認証あり | `puppeteerScript` + `disableStorageReset` |
| D: モック必須 | before_scriptでのモック起動タイミング |
| **E: マルチページ** | URL数に応じたタイムアウト調整、並列化検討 |
| **F: LHCI Server連携** | uploadセクションの設定、トークン管理 |

### E: マルチページ計測

複数ページを計測する場合の考慮点:

- **URL数**: 3〜5個が目安（1 URL × 3 runs = 計15計測 = 約10〜15分）
- **計測時間**: URL数に応じてタイムアウト調整が必要
- **並列化**: URL毎の計測は順序実行（ネットワーク競合回避）

### F: LHCI Server連携

LHCI Serverを使用する場合の追加設定:

```yaml
variables:
  LHCI_UPLOAD_TARGET: "lhci"
  LHCI_SERVER_URL: "https://lhci.example.com"
  LHCI_BUILD_TOKEN: $LHCI_BUILD_TOKEN  # CI Secret
```

---

## 参考資料

- [Lighthouse CI Configuration](https://github.com/GoogleChrome/lighthouse-ci/blob/main/docs/configuration.md)
- [Lighthouse Variability](https://github.com/GoogleChrome/lighthouse/blob/main/docs/variability.md)
- [GitLab CI/CD Variables](https://docs.gitlab.com/ci/variables/)
- [GitLab CI Caching](https://docs.gitlab.com/ci/caching/)
