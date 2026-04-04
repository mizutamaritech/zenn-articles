---
title: "Openclaw体験記 vol.5 — OllamaをOpenclawに繋げるまでの道"
emoji: 🔗
type: idea
topics:
  - Ollama
  - Openclaw
  - Docker
  - LLM
  - AI
published: false
---

:::message
この記事は貧弱GPUをローカルLLMと繋げようとした体験記です。
技術的な正確さより、体験の正直さを優先して書いています。
:::

vol.4でOllamaが動いた。ターミナルで日本語が返ってきた。

でも、それはただのターミナルチャットだ。目標はDiscordのBOTとして動かすこと——Openclawと繋げることだ。

---

## まずOpenclawの設定を変える

Openclawは`~/.openclaw/openclaw.json`で動作を制御している。

ここにモデルの指定がある。デフォルトはAnthropicのAPIを使う設定になっているが、Ollamaを使うように書き換えればいい——そう思っていた。

実際には、OpenclawがローカルのOllamaに繋がるように設定する方法をAIに聞いて回った。

ここで気づく。OpenclawはもともとClaude専用に作られているので、Ollamaをサポートするにはちょっとした工夫がいる。

別のOpenclawインスタンスを用意する、という結論に辿り着いた。

既存のBOTはそのまま。新しくBOTをもう一台立てて、そちらをOllamaに繋げる。

---

## 新しいディレクトリを作る
```bash
mkdir ~/localbot
cd ~/localbot
```

既存のDockerファイル構成をベースに、OllamaのエンドポイントをURLで渡す形だ。

OLLAMA_BASE_URL=http://host.docker.internal:11434

`host.docker.internal`というのは、Dockerコンテナの中からホスト（Windows）のlocalhostにアクセスするための特別なホスト名だ。コンテナの中からは`localhost`が使えない。

---

## モデルをカスタマイズする

ここで初めてModelfileという概念が出てくる。

OllamaはModelfileを使って、ベースモデルに設定を上書きできる。システムプロンプト、パラメータ、プロンプトのテンプレートなど。

最初はこう書いた。

FROM qwen2.5:7b
SYSTEM """
あなたはAIアシスタントです。
"""

シンプルに見えるが、ここから沼が始まる。

---

## モデルを変えることになった

Qwen2.5でしばらく動かしていたが、SoulやModelfileで指定してもプロンプトの使い方が悪いせいもあると思うがかなり中国語に引っ張られる。

ツール呼び出し（function calling）がうまく動かないなど使っていてもあまりピンとこない。

小さいローカルモデルってこんなものなのか、全然おもしろくないとガッカリした。

Openclawはツールを使って外部の情報を取得したり、操作したりする。モデルがツール呼び出しに対応していないと、その機能が使えない。

調べると、GGUFモデルでもツール対応しているものとそうでないものがある。

最終的にQwenよりもう少し大きい`gemma3-tools:12b`というモデルが良さそうだったのでそれにした。

Ollama上での管理名をつけて、Modelfileから作成する。
```bash
ollama create localbot -f /path/to/Modelfile
```

モデルを変えたことで、ツール呼び出しが安定した。

---

## Modelfileの更新手順が固まった

最初はどうやって更新するのかわからなかった。Modelfileを書き換えてもコンテナには反映されない。

正しい手順はこうだ。

1. Modelfileを編集
2. コンテナにコピー
3. `ollama create`で再作成
4. BOTを再起動
```bash
# コンテナへコピー
docker cp ./Modelfile ollama:/root/Modelfile

# モデル再作成
docker exec ollama ollama create localbot -f /root/Modelfile

# BOT再起動
cd ~/localbot && docker compose restart
```

この手順を知るまでに、「なぜ変わらないんだ」と何度もなった。

---

## 詰まりポイントのまとめ

**1. `host.docker.internal`を使う**
コンテナからホストのOllamaに繋ぐには`localhost`ではなくこちらを使う。

**2. Modelfileの変更はコンテナに反映されない**
`docker cp` → `ollama create` → 再起動の手順を踏む必要がある。

**3. ツール対応モデルを選ぶ**
Openclawのツール機能を使うなら、function callingに対応したモデルを選ばないと動かない。

---

DiscordでBOTが返事をするようになった。

APIにお金を払わずに、自分のGPUでBOTが動いている。

だけど特におもしろさは今の所感じない。API経由の方が断然おもしろい。

そこを自分専用のAIをローカルでチューニング？育てる？しておもしろくなるかというところが醍醐味だ。

次のvol.6では、このBOTに人格を持たせる話と、Ollama編の締めくくりをする。

