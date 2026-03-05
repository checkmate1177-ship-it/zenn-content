---
title: "完全無料で毎朝AIニュースをDiscordに自動配信する「AI朝刊Bot」をGoogle Apps ScriptとGroq APIで作った"
emoji: "📰"
type: "tech"
topics: ["googleappsscript", "discord", "groq", "llm", "rss"]
published: true
---

毎朝、最新のAIニュースが自動でDiscordに届いたら便利じゃないか——そんな動機からはじまった。

**完全無料・PC電源不要**で「AI朝刊Bot」を構築したので、構成・コード・つまずきポイントを紹介する。

---

## 作ったもの

毎朝7時に以下の情報をDiscordに自動配信するBot：

- 📰 AI全般・LLM最新ニュース（日本語要約）
- 🆕 新モデル・新AIサービス登場情報
- 🔧 LLM新機能・アップデート情報
- 📌 今日のおすすめ記事（URL付き）
- 🏆 LLMランキング＆性能比較表
- 💡 毎日変わるAI豆知識

---

## 先に結論：コストは完全無料

| ツール | 用途 | 料金 |
|--------|------|------|
| Google Apps Script | スケジューラー・実行環境 | 無料 |
| Groq API | ニュース要約（LLaMA 3.3） | 無料（1日14,400リクエスト） |
| RSS | ニュース取得 | 無料 |
| Discord Webhook | 配信先 | 無料 |

課金ゼロで動き続けるのが最大のポイントだ。

---

## そもそもなぜGroq API？「Google AI Pro契約済みなのにAPIが使えなかった」話

正直に言うと、最初からGroq APIを使う予定ではなかった。

### Gemini APIで詰まった

はじめは **Gemini API** を使おうとした。自分はすでに**Google AI Pro（¥2,900/月）**を契約していたので「これで使えるだろう」と思っていたのだが、GASからAPIを叩くと即 `limit: 0` エラーが返ってきた。

```
Error: limit: 0
```

色々調べてわかった原因：

> **Google AI ProはGeminiチャット用のプランであり、Gemini APIとは完全に別物。**

APIをプログラムから呼び出すには、別途Google Cloudプロジェクトへの紐付けと、そちらでの無料枠またはAPI有効化が必要になる。毎月お金を払っているにも関わらずAPIは使えない、というのは罠だと思った。

完全無料にこだわる場合、Gemini APIは選択肢から外したほうが無難だ（Google Cloud経由で設定すれば使えるが、手順が増える）。

### v1〜v2の失敗遍歴

実はバージョン変遷はこんな感じだった：

| バージョン | 試したこと | 結果 |
|-----------|-----------|------|
| v1 | Claude API + GAS | Claudeは有料のため断念 |
| v2 | Gemini API | `limit: 0`エラー、Google AI Pro契約無効 |
| v3〜 | Groq API + RSS | ✅ 動作確認、ここから本番 |

### Groq APIに切り替えたら一発で動いた

[Groq](https://console.groq.com) はLLaMA・Mixtral等のオープンモデルをAPI経由で使えるサービスで、**無料枠が非常に太い**：

- **1日あたり14,400リクエスト**まで無料
- LLaMA 3.3 70B（高品質モデル）が使える
- 英語ニュースの日本語翻訳・要約の精度が実用十分

毎朝1回の自動配信なら、まず無料枠を超えることはない。

---

## 構成図

```
毎朝7時（自動実行）
↓
Google Apps Script（無料・クラウド）
  ├─ RSSフィード取得（TechCrunch・The Verge・VentureBeat等）
  └─ Groq API（LLaMA 3.3）で日本語要約・生成
↓
Discord Webhook
↓
Discordチャンネルに配信 🎉
```

GAS（Google Apps Script）はサーバーレスで、Googleのインフラ上で動く。PCを起動しておく必要がない。

---

## セットアップ手順

### 1. Groq APIキーを取得する

1. [https://console.groq.com](https://console.groq.com) にアクセス
2. Googleアカウントでサインアップ
3. 「API Keys」→「Create API Key」で新しいキーを発行してコピー

### 2. Discord Webhookを作成する

1. 通知を受け取りたいチャンネルを右クリック → 「チャンネルの編集」
2. 「連携サービス」→「ウェブフックを作成」
3. 名前を設定して「ウェブフックのURLをコピー」

### 3. Google Apps Scriptにコードを貼り付ける

1. [https://script.google.com](https://script.google.com) で新規プロジェクトを作成
2. 後述のコードを貼り付ける
3. コード冒頭の2行を自分の情報に書き換える

```javascript
const GROQ_API_KEY = "gsk_xxxxxxxxxxxxxxxx"; // Groq APIキー
const DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/..."; // Discord Webhook URL
```

### 4. テスト実行

関数ドロップダウンで `testSendNow` を選択して ▶ 実行。Discordに朝刊が届けば成功。

### 5. 自動化を設定する

関数ドロップダウンで `setDailyTrigger` を選択して ▶ 実行。毎朝7時の自動配信がスタートする。

:::message
`setDailyTrigger` の実行を忘れると自動配信されない。コードを貼り付けた後、**必ずこの関数を実行する**のを忘れずに。
:::

---

## コード全文（v8）

```javascript
// ========================================
// AI朝刊Bot - Google Apps Script
// Groq API（無料）+ RSS + Discord Webhook
// v8: 新モデル・新サービス・性能比較追加
// ========================================

// ★ここを設定してください★
const GROQ_API_KEY = "gsk_XXXXXXXXXXXXXXXXXXXXXXXX";
const DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/xxxxx/xxxxx";

// 一般AIニュース RSSフィード
const RSS_FEEDS = [
  { url: "https://techcrunch.com/category/artificial-intelligence/feed/", name: "TechCrunch AI" },
  { url: "https://www.theverge.com/rss/ai-artificial-intelligence/index.xml", name: "The Verge AI" },
  { url: "https://venturebeat.com/category/ai/feed/", name: "VentureBeat AI" },
  { url: "https://japan.zdnet.com/rss/japan/ai/", name: "ZDNet Japan AI" },
  { url: "https://aismiley.co.jp/feed/", name: "AISmiley" },
];

// LLMアップデート・新モデル専用 RSSフィード
const LLM_UPDATE_FEEDS = [
  { url: "https://openai.com/blog/rss.xml",        name: "OpenAI Blog",    org: "OpenAI",    emoji: "🟢" },
  { url: "https://www.anthropic.com/rss.xml",      name: "Anthropic Blog", org: "Anthropic", emoji: "🟠" },
  { url: "https://blog.google/technology/ai/rss/", name: "Google AI Blog", org: "Google",    emoji: "🔵" },
  { url: "https://ai.meta.com/blog/rss/",          name: "Meta AI Blog",   org: "Meta",      emoji: "🟣" },
  { url: "https://mistral.ai/feed.xml",            name: "Mistral Blog",   org: "Mistral",   emoji: "⚪" },
  { url: "https://techcrunch.com/category/artificial-intelligence/feed/", name: "TechCrunch AI", org: "Media", emoji: "📰" },
];

const UPDATE_KEYWORDS = [
  "update", "release", "launch", "new feature", "new model", "introducing", "announce",
  "アップデート", "リリース", "新機能", "新モデル", "発表", "公開", "追加", "登場"
];

const NEW_MODEL_KEYWORDS = [
  "new model", "introducing", "launch", "release", "gpt-", "claude ", "gemini ", "llama ",
  "mistral", "grok", "new ai", "ai tool", "ai service", "ai app", "ai assistant",
  "新モデル", "新サービス", "新しいAI", "登場", "リリース", "発表", "公開"
];

const LLM_RANKING = [
  { rank: 1, model: "Gemini 2.5 Pro",    org: "Google",    score: "1380", note: "総合1位" },
  { rank: 2, model: "GPT-4o",            org: "OpenAI",    score: "1340", note: "コーディング強" },
  { rank: 3, model: "Claude 3.7 Sonnet", org: "Anthropic", score: "1320", note: "論理・文章強" },
  { rank: 4, model: "Grok-3",            org: "xAI",       score: "1300", note: "リアルタイム情報" },
  { rank: 5, model: "Gemini 2.0 Flash",  org: "Google",    score: "1270", note: "高速・無料枠あり" },
];

function sendDailyAINews() {
  try {
    const { articles, articlesWithMeta } = fetchRSSArticles();
    const { llmUpdates, newModels } = fetchLLMAndNewModels();
    const { summary, pickup, tip } = generateContentWithGroq(articles, articlesWithMeta, newModels);
    const today = Utilities.formatDate(new Date(), "Asia/Tokyo", "yyyy年MM月dd日");

    sendToDiscord(buildHeader(today));
    Utilities.sleep(600);
    sendToDiscord(summary);
    Utilities.sleep(600);
    if (newModels.length > 0) { sendToDiscord(buildNewModelsMessage(newModels)); Utilities.sleep(600); }
    if (llmUpdates.length > 0) { sendToDiscord(buildLLMUpdateMessage(llmUpdates)); Utilities.sleep(600); }
    sendToDiscord(pickup);
    Utilities.sleep(600);
    sendToDiscord(buildRanking(today));
    Utilities.sleep(600);
    sendToDiscord(tip);
    Utilities.sleep(600);
    sendToDiscord(buildFooter());
  } catch (e) {
    sendToDiscord("❌ エラー：" + e.message);
  }
}

function fetchRSSArticles() {
  const articles = [];
  const articlesWithMeta = [];
  const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000);

  RSS_FEEDS.forEach(feed => {
    try {
      const res = UrlFetchApp.fetch(feed.url, { muteHttpExceptions: true });
      const xml = res.getContentText();
      const doc = XmlService.parse(xml);
      const root = doc.getRootElement();
      const ns = root.getNamespace();

      // RSS 2.0 形式
      const channel = root.getChild("channel", ns) || root.getChild("channel");
      if (channel) {
        const items = channel.getChildren("item");
        items.slice(0, 5).forEach(item => {
          const title = item.getChildText("title") || "";
          const desc  = item.getChildText("description") || "";
          const link  = item.getChildText("link") || "";
          const pubDate = item.getChildText("pubDate") || "";
          const date = pubDate ? new Date(pubDate) : null;
          if (!date || date >= yesterday) {
            articles.push(`${title}: ${desc.replace(/<[^>]+>/g, "").slice(0, 100)}`);
            articlesWithMeta.push({ title, desc: desc.replace(/<[^>]+>/g, "").slice(0, 150), link, source: feed.name, date: pubDate });
          }
        });
      }

      // Atom 形式
      const atomNs = XmlService.getNamespace("http://www.w3.org/2005/Atom");
      const entries = root.getChildren("entry", atomNs);
      entries.slice(0, 5).forEach(entry => {
        const title = entry.getChildText("title", atomNs) || "";
        const summary = entry.getChildText("summary", atomNs) || entry.getChildText("content", atomNs) || "";
        const linkEl = entry.getChild("link", atomNs);
        const link = linkEl ? (linkEl.getAttribute("href") || linkEl).toString() : "";
        const updated = entry.getChildText("updated", atomNs) || entry.getChildText("published", atomNs) || "";
        const date = updated ? new Date(updated) : null;
        if (!date || date >= yesterday) {
          articles.push(`${title}: ${summary.replace(/<[^>]+>/g, "").slice(0, 100)}`);
          articlesWithMeta.push({ title, desc: summary.replace(/<[^>]+>/g, "").slice(0, 150), link, source: feed.name, date: updated });
        }
      });
    } catch(e) {
      // フィード取得失敗は無視して続行
    }
  });

  return { articles: articles.slice(0, 15), articlesWithMeta: articlesWithMeta.slice(0, 15) };
}

function fetchLLMAndNewModels() {
  const llmUpdates = [];
  const newModels = [];
  const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000);

  LLM_UPDATE_FEEDS.forEach(feed => {
    try {
      const res = UrlFetchApp.fetch(feed.url, { muteHttpExceptions: true });
      const xml = res.getContentText();
      const doc = XmlService.parse(xml);
      const root = doc.getRootElement();
      const ns = root.getNamespace();

      const getItems = () => {
        const channel = root.getChild("channel", ns) || root.getChild("channel");
        if (channel) return channel.getChildren("item");
        const atomNs = XmlService.getNamespace("http://www.w3.org/2005/Atom");
        return root.getChildren("entry", atomNs);
      };

      getItems().slice(0, 5).forEach(item => {
        const atomNs = XmlService.getNamespace("http://www.w3.org/2005/Atom");
        const title = item.getChildText("title") || item.getChildText("title", atomNs) || "";
        const link  = item.getChildText("link")  || (item.getChild("link", atomNs) ? item.getChild("link", atomNs).getAttribute("href").getValue() : "") || "";
        const pubDate = item.getChildText("pubDate") || item.getChildText("updated", atomNs) || item.getChildText("published", atomNs) || "";
        const date = pubDate ? new Date(pubDate) : null;
        const titleLower = title.toLowerCase();

        if (!date || date >= yesterday) {
          if (UPDATE_KEYWORDS.some(kw => titleLower.includes(kw))) {
            llmUpdates.push({ title, link, source: feed.name, org: feed.org, emoji: feed.emoji });
          }
          if (NEW_MODEL_KEYWORDS.some(kw => titleLower.includes(kw.toLowerCase()))) {
            newModels.push({ title, link, source: feed.name, org: feed.org, emoji: feed.emoji });
          }
        }
      });
    } catch(e) {}
  });

  return { llmUpdates, newModels };
}

function generateContentWithGroq(articles, articlesWithMeta, newModels) {
  const prompt = `
あなたはAIニュースキュレーターです。以下の英語・日本語ニュース記事を元に、日本語でまとめてください。

## ニュース記事（最新24時間）
${articles.join("\n")}

## 出力形式（厳守）
### 🤖 AI全般
• [ニュース1のタイトル（日本語）]
  → [2〜3文の日本語要約]
• [ニュース2のタイトル（日本語）]
  → [2〜3文の日本語要約]
（3〜5件）

### 🧠 LLM・生成AI
• [LLM関連ニュース（日本語）]
  → [2〜3文の日本語要約]
（2〜4件）

---

### 📌 今日のおすすめ記事
**[記事タイトル（日本語）]**
> [3〜4文の詳しい解説]
🔗 [元のURL]

---

### 💡 今日のAI豆知識
**[豆知識タイトル]**
[3〜4文の豆知識本文。初心者にわかりやすく。]

今日の日付に関連したネタや、最近話題のAIトピックに絡めてください。
`;

  const response = callGroqAPI(prompt);
  const parts = response.split("---");

  return {
    summary: parts[0] ? parts[0].trim() : response,
    pickup:  parts[1] ? parts[1].trim() : "📌 おすすめ記事を取得できませんでした。",
    tip:     parts[2] ? parts[2].trim() : "💡 AI豆知識を取得できませんでした。",
  };
}

function callGroqAPI(prompt) {
  const payload = {
    model: "llama-3.3-70b-versatile",
    messages: [{ role: "user", content: prompt }],
    max_tokens: 2000,
    temperature: 0.7,
  };

  const options = {
    method: "post",
    contentType: "application/json",
    headers: { Authorization: "Bearer " + GROQ_API_KEY },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true,
  };

  const res = UrlFetchApp.fetch("https://api.groq.com/openai/v1/chat/completions", options);
  const json = JSON.parse(res.getContentText());

  if (json.error) throw new Error("Groq API Error: " + json.error.message);
  return json.choices[0].message.content;
}

function sendToDiscord(message) {
  if (!message || message.trim() === "") return;

  // 2000文字制限対策：1900文字で分割
  const chunks = [];
  let remaining = message;
  while (remaining.length > 0) {
    chunks.push(remaining.slice(0, 1900));
    remaining = remaining.slice(1900);
  }

  chunks.forEach(chunk => {
    const options = {
      method: "post",
      contentType: "application/json",
      payload: JSON.stringify({ content: chunk }),
      muteHttpExceptions: true,
    };
    UrlFetchApp.fetch(DISCORD_WEBHOOK_URL, options);
    Utilities.sleep(300);
  });
}

function buildHeader(today) {
  return `━━━━━━━━━━━━━━━━━━━━━━
📰 **AI朝刊** ${today}
━━━━━━━━━━━━━━━━━━━━━━
おはようございます！今日のAI・LLMニュースをお届けします 🌅`;
}

function buildNewModelsMessage(items) {
  if (items.length === 0) return "";
  let msg = "🆕 **新モデル・新サービス情報**\n";
  items.slice(0, 5).forEach(item => {
    msg += `${item.emoji} **[${item.org}]** ${item.title}\n`;
    if (item.link) msg += `   🔗 ${item.link}\n`;
  });
  return msg;
}

function buildLLMUpdateMessage(items) {
  if (items.length === 0) return "";
  let msg = "🔧 **LLM新機能・アップデート情報**\n";
  items.slice(0, 5).forEach(item => {
    msg += `${item.emoji} **[${item.org}]** ${item.title}\n`;
    if (item.link) msg += `   🔗 ${item.link}\n`;
  });
  return msg;
}

function buildRanking(today) {
  let msg = "🏆 **LLMランキング＆性能比較**（Chatbot Arena参考）\n";
  msg += "```\n";
  msg += "順位  モデル               組織         スコア  特徴\n";
  msg += "────  ───────────────────  ──────────   ──────  ──────────────\n";
  LLM_RANKING.forEach(r => {
    msg += `${String(r.rank).padEnd(6)}${r.model.padEnd(21)}${r.org.padEnd(13)}${r.score.padEnd(8)}${r.note}\n`;
  });
  msg += "```\n";
  msg += "📊 **用途別おすすめ**\n";
  msg += "💬 日常会話・文章生成 → Claude 3.7 Sonnet\n";
  msg += "💻 コーディング → GPT-4o / Claude 3.7 Sonnet\n";
  msg += "🔍 最新情報・検索 → Grok-3 / Gemini 2.0 Flash\n";
  msg += "⚡ 高速・無料利用 → Gemini 2.0 Flash / LLaMA 3.3（Groq）\n";
  msg += "🧠 高度な推論 → Gemini 2.5 Pro / Claude 3.7 Sonnet";
  return msg;
}

function buildFooter() {
  return `━━━━━━━━━━━━━━━━━━━━━━
今日も良い一日を！ 🤖✨
━━━━━━━━━━━━━━━━━━━━━━`;
}

// テスト用：今すぐ送信
function testSendNow() {
  sendDailyAINews();
}

// 毎朝7時の自動トリガーを設定
function setDailyTrigger() {
  // 既存トリガーを削除してから設定
  ScriptApp.getProjectTriggers().forEach(t => {
    if (t.getHandlerFunction() === "sendDailyAINews") {
      ScriptApp.deleteTrigger(t);
    }
  });
  ScriptApp.newTrigger("sendDailyAINews")
    .timeBased()
    .everyDays(1)
    .atHour(7)
    .inTimezone("Asia/Tokyo")
    .create();
  Logger.log("毎朝7時のトリガーを設定しました");
}
```

---

## 実際の配信イメージ

### AI全般・LLMニュース

```
━━━━━━━━━━━━━━━━━━━━━━
📰 AI朝刊 2026年03月05日
━━━━━━━━━━━━━━━━━━━━━━
おはようございます！今日のAI・LLMニュースをお届けします 🌅

🤖 AI全般
• Nvidia、OpenAI・Anthropicへの投資を縮小へ
  → NvidiaのCEO、Jensen Huangは、同社のOpenAIとAnthropicへの
    投資は今が最後になるだろうと述べた。AI市場での競争激化を示唆。

• Apple Music、AI生成音楽に透明性ラベルを導入
  → Apple Musicは、AI生成音楽と人間制作の楽曲を区別するラベルを
    導入する計画を発表。クリエイター保護の観点から注目される。

🧠 LLM・生成AI
• Google、NotebookLMのビデオ概要機能を強化
  → 研究ノートを「シネマティック」なビデオ概要に変換できる機能が
    追加された。教育・リサーチ用途での活用が期待される。
```

### LLMランキング

```
🏆 LLMランキング＆性能比較（Chatbot Arena参考）
順位  モデル               組織         スコア  特徴
────  ───────────────────  ──────────   ──────  ──────────────
1     Gemini 2.5 Pro       Google       1380    総合1位
2     GPT-4o               OpenAI       1340    コーディング強
3     Claude 3.7 Sonnet    Anthropic    1320    論理・文章強
4     Grok-3               xAI          1300    リアルタイム情報
5     Gemini 2.0 Flash     Google       1270    高速・無料枠あり
```

---

## つまずきポイントと対処法

### ① Gemini APIの罠（再掲・重要）

Google AI Pro（¥2,900/月）を契約していても、**Gemini APIのプログラム呼び出しはできない**。

APIを使うには：
1. Google Cloudプロジェクトを別途作成
2. Gemini APIを有効化
3. APIキーを発行

この手順が必要で、Google AI Proサブスクリプションとは無関係。完全無料にこだわるなら最初からGroq APIを使うほうが圧倒的に楽だ。

### ② Discordの2000文字制限

Discordは1メッセージあたり**2000文字**の上限がある。ニュースをまとめると軽く超えるため、コード内で**1900文字で自動分割**して対処している。

```javascript
function sendToDiscord(message) {
  const chunks = [];
  let remaining = message;
  while (remaining.length > 0) {
    chunks.push(remaining.slice(0, 1900));
    remaining = remaining.slice(1900);
  }
  chunks.forEach(chunk => { /* ... */ });
}
```

### ③ RSSの日付フォーマットが媒体ごとにバラバラ

各メディアのRSSはAtom形式とRSS 2.0形式が混在しており、日付フィールドも以下のようにバラバラ：

| フォーマット | 日付フィールド |
|---|---|
| RSS 2.0 | `pubDate` |
| Atom | `published` / `updated` |

コードではどちらの形式も処理できるよう、両方のパターンに対応している。

### ④ GASの実行時間上限（6分）

Google Apps Scriptの無料枠は**1回の実行時間が6分まで**。RSSフィード数を増やしすぎると超過する可能性がある。現在の構成（5〜6フィード）では問題ない。

### ⑤ トリガーの設定忘れ

コードを貼り付けただけでは自動配信されない。**必ず `setDailyTrigger` を実行すること**。GASのトリガー管理画面（左メニューの時計アイコン）でも確認できる。

---

## バージョン変遷

| バージョン | 主な変更内容 |
|-----------|-------------|
| v1 | Claude API + GAS の基本構成（有料のため断念） |
| v2 | Gemini API に変更（`limit: 0`エラーで失敗） |
| v3 | Groq API + RSS に切り替え。基本動作を確認 |
| v4 | LLMランキング・AI豆知識を追加 |
| v5 | おすすめ記事セクションを追加 |
| v6 | Discordレイアウト改善・英語タイトルの日本語翻訳を強化 |
| v7 | ソースURL・記事日付・24時間フィルターを追加 |
| **v8（最終）** | **新モデル・新サービス情報・LLM性能比較表を追加** |

---

## 今後の拡張アイデア

- 📅 週次まとめ（日曜日だけ1週間の振り返り）
- 🌍 日本AI企業ニュース（NTT・ソフトバンク・PFNなど）
- 📈 AI関連株価（NVIDIA・Microsoft・Googleなど）
- 🗓️ 今週のAIイベント・カンファレンス情報

---

## まとめ

| やりたいこと | 結論 |
|---|---|
| 完全無料でLLM APIを使いたい | **Groq API**（1日14,400リクエスト無料） |
| サーバー不要で定期実行したい | **Google Apps Script**（無料・クラウド） |
| Google AI ProでAPI呼び出したい | **できない**（別途Google Cloud設定が必要） |
| Discordに長文を送りたい | 2000文字制限に注意、分割送信が必要 |

完全無料・PC電源不要で、毎朝AIニュースが届く環境が15分で作れた。
Groq APIのLLaMA 3.3は英語→日本語の翻訳・要約の精度も十分実用的で、AIニュースのキャッチアップとしてかなり便利だ。ぜひ試してほしい。

---

*本記事の作成にはClaude（Anthropic）を活用しました。*
