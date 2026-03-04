---
title: "ホームラボをAIで強化する：Prometheus監視 × LM Studio × Discordで自動解析通知を構築した"
emoji: "🤖"
type: "tech"
topics: ["homelab", "prometheus", "grafana", "docker", "discord"]
published: true
---

自宅サーバーを運用していると「アラートが来たけど何が原因？」と調べるのが面倒になりがち。
今回はその課題を解決するため、**Prometheusのアラートをローカルで動くLLM（LM Studio）が自動解析し、Discordに日本語で通知する仕組み**を構築した。
あわせてGiteaとMinIOのメトリクス可視化も整備したので、まとめて紹介する。

## 環境

| 項目 | 内容 |
|---|---|
| OS | Ubuntu on WSL2（Windows 11） |
| コンテナ | Docker + Docker Compose |
| ネットワーク | `ops_net`（共有Dockerブリッジ） |
| 監視基盤 | Prometheus + Grafana + Alertmanager |
| LLM | LM Studio（192.168.1.202）/ qwen2.5-14b-instruct |
| 通知先 | Discord Webhook |

```
Internet → Cloudflare → cloudflared → Nginx Proxy Manager → ops_net services
LAN      →                            Nginx Proxy Manager → ops_net services
```

---

## Part 1: GiteaとMinIOのメトリクス監視をPrometheusに追加

### Gitea側の設定

Giteaは `/metrics` エンドポイントを標準で持っているが、デフォルトでは無効になっている。
`app.ini` に以下を追加して有効化する。

```ini
[metrics]
ENABLED = true
```

Gitea側の設定が終わったら、Prometheus の `prometheus.yml` にジョブを追加する。

```yaml
scrape_configs:
  - job_name: 'gitea'
    static_configs:
      - targets: ['gitea:3000']
    metrics_path: /metrics
```

`gitea` はコンテナ名で、`ops_net` 内で名前解決される。

### MinIO側の設定

MinIOはPrometheus用のスクレイプ設定をCLIで生成できる。

```bash
mc admin prometheus generate myminio
```

出力されたトークンを使って `prometheus.yml` にジョブを追加する。

```yaml
  - job_name: 'minio'
    bearer_token: '<生成されたトークン>'
    metrics_path: /minio/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ['minio:9000']
```

### 動作確認

```bash
# Prometheusのターゲット一覧で "UP" になっているか確認
curl -s http://localhost:19090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

---

## Part 2: GrafanaにGiteaとMinIOのダッシュボードを構築

### MinIOダッシュボード

MinIOはGrafana公式のダッシュボードIDが提供されている。
Grafana → Dashboards → Import → ID `13502` を入力するだけで即使える。

主要パネル：
- Cluster Capacity Used / Available
- Objects Count
- Network I/O（送受信バイト数）
- S3 API Request Rate（GET/PUT/DELETE別）

### Giteaダッシュボード

Giteaのダッシュボードは自作した。主に以下のメトリクスを可視化している。

```promql
# HTTPリクエスト数（ステータス別）
sum(rate(gitea_access_total[5m])) by (code)

# アクティブユーザー数
gitea_users

# リポジトリ数
gitea_repos

# Goランタイムのメモリ使用量
go_memstats_alloc_bytes{job="gitea"}
```

パネル構成：

| パネル | クエリ |
|---|---|
| HTTPリクエスト率 | `rate(gitea_access_total[5m])` |
| レスポンスタイム（p99） | `histogram_quantile(0.99, gitea_response_time_seconds_bucket)` |
| リポジトリ/ユーザー数 | `gitea_repos`, `gitea_users` |
| Goメモリ | `go_memstats_alloc_bytes` |

---

## Part 3: LM Studio × Discord Bot でAIアシスタントを構築

### 構成

```
Alertmanager
    ↓ POST /webhook
ai-alert (Flask)
    ↓ POST /v1/chat/completions
LM Studio (192.168.1.202:1234)
    ↓ 解析結果
Discord Webhook
```

LM StudioはOpenAI互換のAPIを提供しているため、`/v1/chat/completions` エンドポイントをそのまま使える。

### ai-alertコンテナの実装

`webhook.py` の核心部分：

```python
def ask_lm(alert_text: str) -> str:
    prompt = (
        "以下のサーバー異常アラートを日本語で簡潔に解析して、"
        "原因と対処法を3行以内で説明して\n\n"
        f"{alert_text}"
    )
    payload = {
        "model": "qwen2.5-14b-instruct",
        "messages": [{"role": "user", "content": prompt}],
        "temperature": 0.3,
        "max_tokens": 300,
    }
    resp = requests.post(
        f"{LM_STUDIO_URL}/v1/chat/completions",
        json=payload,
        timeout=60,
    )
    return resp.json()["choices"][0]["message"]["content"].strip()
```

`temperature: 0.3` に抑えることで、AIが余計な創造性を発揮せず、事実ベースの解析をしやすくなる。

### Dockerfileとdocker-compose

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask requests
COPY webhook.py .
EXPOSE 5000
CMD ["python", "webhook.py"]
```

```yaml
services:
  ai-alert:
    build: .
    container_name: ai-alert
    restart: unless-stopped
    ports:
      - "0.0.0.0:5000:5000"
    env_file:
      - .env
    networks:
      - ops_net

networks:
  ops_net:
    external: true
```

`.env` に必要な変数：

```env
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...
LM_STUDIO_URL=http://192.168.1.202:1234
```

### Alertmanagerの設定

`alertmanager.yml` にWebhookレシーバーを追加する。

```yaml
route:
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: ai-alert

receivers:
  - name: 'default'
    discord_configs:
      - webhook_url: 'https://discord.com/api/webhooks/...'

  - name: 'ai-alert'
    webhook_configs:
      - url: 'http://ai-alert:5000/webhook'
        send_resolved: true
```

`ai-alert` コンテナは `ops_net` に参加しているため、`http://ai-alert:5000` でコンテナ名解決される。

---

## 動作確認

### Webhookに手動でアラートを送信してテスト

```bash
curl -X POST http://localhost:5000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "alerts": [{
      "status": "firing",
      "labels": {
        "alertname": "HighCPUUsage",
        "instance": "192.168.1.200:9182",
        "job": "windows_exporter"
      },
      "annotations": {
        "summary": "CPU usage is above 90%",
        "description": "CPU usage has been above 90% for more than 5 minutes"
      }
    }]
  }'
```

Discordに以下のような通知が届く：

```
[AI Alert Analysis]
[FIRING] HighCPUUsage | 192.168.1.200:9182
  Summary: CPU usage is above 90%
  Detail: CPU usage has been above 90% for more than 5 minutes

解析:
CPUが5分以上90%を超えています。原因はバックグラウンドプロセスの暴走または
高負荷なタスクの実行が考えられます。タスクマネージャーで高CPU使用プロセスを
特定し、不要なら終了させてください。
```

### ヘルスチェック

```bash
curl http://localhost:5000/health
# {"status": "ok"}
```

---

## まとめ

| 構築したもの | 効果 |
|---|---|
| Gitea/MinIOのPrometheus監視 | サービス固有のメトリクスを可視化 |
| Grafanaダッシュボード | 一画面でGitea/MinIOの状態を把握 |
| LM Studio連携Webhook | アラート内容をAIが日本語で解説 |
| Discord自動通知 | 夜中のアラートも翌朝に原因が分かる |

全コンポーネントがDockerコンテナで動いており、`ops_net` というDockerネットワークで繋がっているため、ホスト名でサービス間通信できるのがホームラボ構成の強みだ。

LM Studioをサブマシン（192.168.1.202）に分離しているため、LLMの推論負荷がメインサーバーに影響しない点も気に入っている。

次はAlertmanagerのルーティングを整備して、重要度別に通知先とAI解析の有無を切り替える仕組みを作る予定。

---

## 参考

- [Prometheus公式ドキュメント](https://prometheus.io/docs/)
- [Grafana Dashboards - MinIO (ID: 13502)](https://grafana.com/grafana/dashboards/13502)
- [LM Studio - OpenAI互換API](https://lmstudio.ai/docs/api)
- [Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli)
