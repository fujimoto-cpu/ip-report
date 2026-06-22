# KONNEKT IP News Weekly Report — 生成ルール

## 1. HTML生成ルール（最重要）

**毎回、直前のレポートをテンプレートとして取得すること。ゼロからHTMLを書き起こすことは禁止。**

### テンプレート取得手順

```bash
# GITHUB_TOKEN は Claude Code のセッション冒頭で毎回 export すること（値はユーザーが管理）
# export GITHUB_TOKEN="..."

# Step 1: index.html から REPORTS 配列の先頭日付（直前号）を取得
PREV_DATE=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/fujimoto-cpu/ip-report/contents/index.html" \
  | python3 -c "
import sys, json, base64, re
d = json.load(sys.stdin)
html = base64.b64decode(d['content']).decode()
m = re.search(r'const REPORTS = \[.*?\"(\d{4}-\d{2}-\d{2})\"', html, re.DOTALL)
print(m.group(1) if m else '')
")

# Step 2: 直前号の HTML を取得して /tmp/template.html に保存
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/fujimoto-cpu/ip-report/contents/${PREV_DATE}.html" \
  | python3 -c "
import sys, json, base64
d = json.load(sys.stdin)
open('/tmp/template.html', 'w').write(base64.b64decode(d['content']).decode())
"

echo "テンプレート取得完了: $PREV_DATE"
```

### テンプレートを使った HTML 更新手順

1. `/tmp/template.html` を読み込む
2. 以下の箇所だけを差し替える：
   - `<title>` タグ内の日付
   - `.date-label` の日付文字列（例: `2026年6月22日（月）`）
   - `footer-note` の日付
   - `summary-banner` の本文（今週のハイライト）
   - 全 `article-card` ブロック（記事タイトル・本文・画像・リンク・日付）
   - `insight-card` の提言本文
   - `backnumber` リスト（今号を先頭に追加し、最古号を削除）
3. それ以外のCSS・構造・クラス名・セクション順序は変更しない

---

## 2. 記事採用ルール

- **掲載期限**: 今日の日付から14日以内に公開された記事のみ
- **重複除外**: 過去28日分のレポート（GitHub上の直近4週分）に掲載済みの記事は除外
- **カテゴリ構成**:
  - 🇯🇵 日本の記事: 最大3本（キャラクターIP・ライセンス・アーティストグッズ・懐かしいキャラクター復刻）
  - 🌍 グローバルの記事: 最大3本（entertainment IP・celebrity D2C・K-pop/Korean）

---

## 3. 画像ルール

- **画像は必ず実在するURLのみ使用**。プレースホルダー（placehold.co 等）禁止
- **fabricated URL 禁止**（自分で推測したスラグ名のURLは使わない）
- 画像URLの確認方法: WebSearch で og:image または CDN URL が検索結果に現れたものを使用
- PR Times CDN パターン: `https://prcdn.freetls.fastly.net/release_image/{company_id}/{release_id}/{hash}-{dimensions}.jpg`
  - hash は MD5 であり推測不可。検索で発見したもののみ使用可
- 確認できない記事は `<img>` タグなしで掲載する（カードはテキスト表示になる）

---

## 4. 投稿先ルール（セキュリティ）

| 送信先 | チャンネルID | 可否 |
|--------|------------|------|
| #team_konnekt | C0B4C8Q48G6 | ✅ 唯一の許可送信先 |
| #general | — | ❌ 禁止 |
| #社員room | C010G8J82JF | ❌ 禁止 |
| #新規事業検討チーム | C036YM3BJJZ | ❌ 禁止 |
| 個人DM (藤本) | U034EFSD81G | ❌ 禁止 |

**Notionリンクは絶対にSlackに送らない。**

---

## 5. GitHub API 操作

```bash
export GITHUB_TOKEN="<トークンはユーザーが管理・Claude Codeセッション冒頭で設定>"
```

- **このトークン設定はすべてのGitHub API呼び出しの前に必ず実行すること**
- リポジトリ: `fujimoto-cpu/ip-report`
- GitHub Pages URL: `https://fujimoto-cpu.github.io/ip-report/`
- HTML アップロード: `PUT /repos/fujimoto-cpu/ip-report/contents/{date}.html`
- index.html 更新: REPORTS 配列の先頭に新しい日付を追加

---

## 6. 実行フロー（毎週月曜日）

1. 今日の日付を確認（`date +%Y-%m-%d`）
2. **テンプレート取得**（上記 §1 の手順を必ず実行）
3. GitHub から過去4週分の exclude_urls/exclude_titles を収集
4. WebSearch で今週の記事を収集（カテゴリごとに検索）
5. 画像URLを検索で確認（確認できないものは画像なしで掲載）
6. テンプレートをベースにHTMLを生成（§1 参照）
7. `/tmp/{YYYY-MM-DD}.html` に保存
8. GitHub API で `{date}.html` をアップロード
9. `index.html` の REPORTS 配列先頭に新日付を追加
10. `#team_konnekt (C0B4C8Q48G6)` にSlack投稿
