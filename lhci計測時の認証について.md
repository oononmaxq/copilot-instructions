# LHCI計測時の認証について

## 1. このメモの目的

Lighthouse CI（LHCI）で、認証がある画面をどう測るかを整理する。

---

## 2. 要点

- `puppeteerScript`はLHCIの正式な設定項目で、認証や事前操作に使用できる。
- 各URLの計測前に`puppeteerScript`が実行され、その状態を使ってLighthouseが走る。
- Cookieベースの認証は比較的扱いやすい。localStorage/sessionStorage前提なら、必要に応じて`disableStorageReset`も検討する。
- MSWで自アプリのログインAPIをモックすれば、認証自体をスキップできる。認証後のAPIも全てモックする前提であれば、最もシンプルな構成になる。

---

## 3. `puppeteerScript`とは何か

`puppeteerScript`はLHCIの`collect`で使える設定。
Puppeteerでブラウザを起動し、そのスクリプトを実行した後でLighthouseを実行する流れになっている。
URLごとに`puppeteerScript`が実行される。

### 対応可能な用途

- ログイン
- Cookieの投入
- localStorage/sessionStorageの事前設定
- モーダルを閉じる
- 計測前に必要な画面操作

### headlessモードでの動作

`puppeteerScript`はheadlessモードでも動作する。Puppeteer/LHCIはデフォルトでheadlessモードで動作するため、CI環境では基本的にheadlessで使用することになる。

Docker/CI環境で使う場合の設定例:

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      puppeteerScript: "./login.js",
      settings: {
        chromeFlags: ["--no-sandbox", "--headless=new"],
      },
    },
  },
};
```

**参考:**

- [Lighthouse CI - puppeteerScript](https://github.com/GoogleChrome/lighthouse-ci/blob/main/docs/configuration.md#puppeteerscript)
- [Lighthouse - Testing on Authenticated Pages](https://github.com/GoogleChrome/lighthouse/blob/main/docs/recipes/auth/README.md)
- [Chrome Developers - New Headless](https://developer.chrome.com/docs/chromium/new-headless)

---

## 4. 外部認証がある場合

### 4-1. 対応しやすいケース

**ブラウザ操作で完結する認証**は`puppeteerScript`で対応しやすい。

- 通常のログイン画面
- OIDCなどで外部に遷移するが、最終的にはフォーム入力とリダイレクトで戻るもの
- テスト用アカウントでログインできるもの

LHCIの認証ページ向けレシピは、Puppeteerを使った認証操作を前提としている。

### 4-2. 不安定になりやすいケース

- MFAが毎回必要 → ワンタイムコードの取得を自動化できない
- CAPTCHAが出る → 人間による操作が前提のため自動化できない
- SAML/ADFSなどで企業向けSSOの流れが重い → リダイレクトが多段階で不安定になりやすい
- 認証後のトークン受け渡しが複雑 → puppeteerScript内でのトークン共有が難しい

GitHub上でもSAML-SSOやトークン受け渡しに関する質問・課題報告があるが、現時点で確立された解決策は示されていない。

**関連する議論:**

- [SAML-SSO/ADFS認証の扱いについての質問（未解決）](https://github.com/GoogleChrome/lighthouse/discussions/15808)
- [puppeteerScriptでのトークン受け渡しが機能しない問題（未解決）](https://github.com/GoogleChrome/lighthouse-ci/issues/1076)

### 4-3. 認証フローが複雑な場合の選択肢

認証フローが複雑な場合、毎回ログイン操作を行うのではなく、以下の選択肢がある。

1. 計測用の認証バイパス経路を用意する（環境変数で切り替え等）
2. 事前発行したセッションやCookieを注入する（セッションが短命な場合は使えない）
3. 上記が難しい場合、`puppeteerScript`でログインする

LHCIは`puppeteerScript`を「柔軟な認証手段」として案内している。

### 4-4. MSWで認証をスキップする

MSW（Mock Service Worker）はブラウザのService Workerを使って通信を横取りする仕組み。

自アプリのログインAPI（`/api/auth/me`等）をモックして認証済みユーザー情報を返すようにすれば、認証をスキップする構成が可能。外部認証（SAML、ADFS、外部IdP）を使うアプリでも、自アプリ側でスキップすれば外部IdPへの接続は不要になる。

これはローカル開発でも一般的なアプローチで、LHCIでの計測でも適用できる。

#### MSWでカバーできないケース

- **認証後の本物のAPIを叩きたい場合** → MSWでログインをスキップしても、その後のAPIが本物の認証トークンを要求する場合は本番APIにアクセスできない。認証後のAPIも全てモックする必要がある
- **認証処理自体のパフォーマンスを測りたい場合** → ログインページやリダイレクトのパフォーマンスを測定したいなら、スキップできない

**参考:**

- [MSW - Browser Integration](https://mswjs.io/docs/integrations/browser/)
