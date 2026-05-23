---
title: "AI体験記 vol.7 — WSL2を捨てて、Mac miniに家を建てた"
emoji: 🏠
type: idea
topics:
  - macmini
  - Docker
  - Openclaw
  - Discord
  - LLM
published: false
---

:::message
この記事はWSL2で動かしていたDiscordボット群をMac mini M4に移行した体験記です。
技術的な正確さより、体験の正直さを優先して書いています。
:::

vol.6では、SOUL.mdとModelfileでボットに人格を持たせた話を書いた。

今回はその後の話。**家が狭くなってきたので、引っ越した。**

---

## なぜ引っ越しが必要だったか

WSL2でボットを動かしていると、あるストレスが積み重なってくる。

Windowsを再起動するたびにWSL2も止まる。ボットも止まる。その度に `docker compose up -d` をしなければならない。夜中に何かあっても気づけない。ComfyUI（画像生成）を動かすとVRAMを喰い合ってOllamaが詰まる。

要するに、**24時間動き続けるサーバーが欲しかった。**

最初はラズパイ5を考えていた。でもGPUがない。Ollamaを走らせるには非力すぎる。

そこでMac mini M4を選んだ。16GB統合メモリ。Apple Siliconでollama run gemma4:e4bが軽快に動く。コンパクトで静かで消費電力も少ない。理由はそれだけで十分だった。

---

## 移行の方針

やることは単純に見えた。WSL2で動いていたものをMac miniで動かす。それだけだ。

でも実際には、移行をきっかけに方針を変えたものもある。

ボット別に整理するとこうなる。

**Kage**（Grok-4.3・X情報収集）→ Mac miniで再構築、Docker起動。問題なし。

**Tedare**（アイデア出しエージェント）→ Mac miniで再構築。モデルをgpt-5-miniに変更。SOUL.mdとAGENTS.mdを設置し直した。

**Mob**（Ollamaと連携する分析ボット）→ **方針を変えた。** OpenClawで動かすのをやめ、Pythonスクリプトに切り替えた。詳しくは後述する。

**Shikou**（Claude Sonnetで動く思考エージェント）→ まだ移行していない。CCCBotへの移行を検討中。

---

## 移行中に詰まったこと3つ

### 1. SOUL.mdを置く場所が違った

WSL2時代、SOUL.mdをどこに置くか、正直あまり気にしていなかった。

Mac miniで新しく設定を組んでいたら、ボットがSOUL.mdを認識しない。あれ、人格がない。

調べてわかった。**SOUL.mdとMEMORY.mdはconfig/ではなくworkspace/に置く。**

`config/` はOpenclawの設定ファイルの場所。`workspace/` がボットの「作業スペース」でSOUL.mdを読みに行くのはそちらだ。WSL2時代は偶然うまくいっていたのか、たまたま無視されていたのか。移行して初めて気づいた。

### 2. 同じDiscordトークンを二箇所で使った

WSL2とMac miniで同時に同じボットを動かしてしまった期間があった。

移行テスト中、「Mac miniで動いた！」と確認したつもりで、WSL2側を止め忘れていた。

するとボットが断続的にオフラインになる。返事が返ってきたり来なかったり。

原因は**heartbeat ACK timeout**。同じDiscordトークンが二箇所から接続していると、Discordのサーバーがどちらに応答していいか迷う。結果、どちらも不安定になる。

移行するときは、**先にWSL2側を確実に止めてから**Mac miniを起動する。当たり前に聞こえるが、テスト中は「両方動かして比較」したくなる。その誘惑に負けてはいけない。

### 3. `docker compose restart` で固まる

Mac mini上でコンテナの設定を変えて、`docker compose restart` を実行した。

**固まった。**

数分待っても反応がない。Ctrl+Cも効かない。

解決策は `docker compose down && docker compose up -d` だ。`restart` はコンテナを再起動するが、状態がおかしくなっているときは素直に落として上げ直す方が確実だ。

この件は後で調べたら、ネットワーク設定が絡んでいるときに起きやすいらしい。OpenClawのDiscordプラグインがネットワーク接続を張ったまま `restart` しようとして詰まっていた可能性がある。

---

## Mobの方針転換：OpenClaw→Python CLI

Mobはもともと、OllamaのローカルLLMをOpenClaw経由でDiscordに繋いでいたボットだ。

でも使い続けるうちに感じていた。**OpenClawはDiscordの会話に応答するフレームワークで、バッチ処理には向いていない。**

Mobにやらせたいのは「Kageの投稿を30件溜めたら分析して要約する」という処理だ。会話ではなく、監視して、条件を満たしたら動く。

OpenClawでそれをやると、余計な仕組みを多く持ち込むことになる。

だから**Pythonスクリプトに切り替えた。** Discord.pyでKageの投稿を監視して、Ollamaに投げて、結果を返す。最小限の構成だ。

ディレクトリ名は `~/mobcli/` にした。

---

## Mob CLIの構成

Claude Codeを使って自動生成した。指示を出してファイルを作ってもらう、という作業を繰り返した。

構成はシンプルだ。

```
~/mobcli/
├── config.py      # トークン・チャンネルIDなどの設定
├── store.py       # 投稿を溜めておくストレージ
├── analyzer.py    # Ollamaに送って分析する処理
├── mob.py         # Discord Botの本体
└── tests/         # テストコード
```

動作はこうだ。

- 監視チャンネル（Kageが投稿するチャンネル）を常時監視
- Kageの投稿が30件溜まったら自動で分析を実行
- `@mob01 まとめて` と書いても手動でトリガーできる

出力のフォーマットはこう定めた。

```
[重要度:高][信頼性:一次][カテゴリ:経済/技術] 〇〇が...
```

タグをつけることで、後でフィルタリングしやすくなる。Tedareやその先の処理に渡しやすい形を意識した。

起動方法はこれだけだ。

```bash
cd ~/mobcli
source venv/bin/activate
python3 mob.py
```

---

## 現在の構成

Mac miniの中身をまとめるとこうなっている。

```
Mac mini M4
├── Docker
│   ├── Kage（Grok-4.3・X情報収集）
│   └── Tedare（gpt-5-mini・アイデア出し）
├── Ollama（ホスト直接インストール・gemma4:e4b）
└── ~/mobcli/（Python Discord Bot）
    └── mob01（Kageの投稿を監視・分析・要約）
```

KageとTedareはDockerで動かしている。OpenClawベースだ。

OllamaはDockerではなくホストに直接インストールした。Dockerごしにやるより、Mac mini本体のApple Siliconを素直に使った方が速い。

MobCliはDockerなし。Pythonの仮想環境だけで動く。

---

## 目指しているパイプライン

今動いているのはここまでだ。

```
Kage（収集）→ MobCLI（分析・圧縮）→ Tedare（発散・アイデア）
```

最終的にはこうしたい。

```
Kage（収集）→ MobCLI（分析・圧縮）→ Tedare（発散・アイデア）→ Shikou（戦略・統合）
```

Shikouはまだ移行していないので、パイプラインとしては最後の一段が欠けている。

Kageがニュースを拾い、MobCLIがそれを要約・分類し、Tedareがアイデアに展開し、ShikouがZennの記事にする——そういうことができたらと思っている。

---

## 引っ越して気づいたこと

Mac miniに移してから、ボットたちが**静かに動き続けている感覚**がある。

WSL2のときは、PCを使いながらいつも気にかけていた。Windowsを再起動するとき、ComfyUIを起動するとき、毎回ボットのことを頭の隅に置いておく必要があった。

今はそれがない。Mac miniは勝手に動いている。朝起きたらKageが昨夜のXのポストを溜め込んでいる。

**インフラとして機能し始めた、という感覚だ。**

システムを動かすことに気を取られなくなって初めて、次に何をするか考えられるようになった。

Shikouの移行、MobCLIのプロンプト改善、ファイルR/W権限の設計。やりたいことはまだ積み上がっている。

でも今は、まず30件溜まるのを待っている。

---

*次回はMobCLIの実際の出力と、Tedareへの引き継ぎを書く予定です。*
