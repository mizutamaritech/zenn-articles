---
title: "Openclaw体験記 vol.4 — WSL2でOllamaを動かすまでの道"
emoji: "🦙"
type: "idea"
topics: ["Ollama", "Docker", "WSL2", "GPU", "AI"]
published: ture
---

:::message
この記事は貧弱GPUをローカルLLMを繋げようとした体験記です。
技術的な正確さより、体験の正直さを優先して書いています。
:::

vol.3でOpenclawと一緒に記事を公開するところまで来た。

結局Openclaw経由だとAPI消費激しいので普通にAPI経由じゃなくAIに記事書いてもらいそれを人間が検証している方が安くつくという結論に辿り着いた。

経験としては申し分ない経験ができたと思う。ここは経験は知識と言う言葉を信じよう。

そうなると自然に次は**コストが掛からないローカルで動くAIをPCで動かしたくなるというのが人の性。**

---

## 最初にやった失敗

公式サイトの手順通りにWSL2へOllamaをネイティブインストールした。
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama serve
```

**フリーズ。**

エラーメッセージが出ない。カーソルが点滅したまま、何も起きない。

CUDA初期化でハングしていた。0.17.7で試して、0.18.0に上げて試した。どちらも同じだった。

CUDAは、もともと並列計算に強いGPUの能力を、画像処理以外の一般計算にも直接使えるようにした開発環境。

NVidia、NVidiaと世間で騒がれているのはこういう需要からなのかという事が分かり理解できた。

話を戻すとエラーが出ないというのをそのままAIに丸投げ

AIに投げてもGPUドライバーの問題、WSL2の問題、Ollamaの問題、ほとんどはネイティブ版を前提に書かれていて、解決策が見つからなかった。

---

## Docker経由という迂回路

AIに調べてもらっていたらDockerでOllamaを動かす方法が出てきた。

なぜDockerで解決するのか。OllamaをDockerコンテナとして動かすことで、CUDA環境をホスト（Windows）側に任せられる。WSL2内でCUDAを初期化しなくていい。

必要なものは2つ。

- Docker Desktop（WSL2統合を有効にしておく）
- NVIDIA Container Toolkit

NVIDIA Container Toolkitのインストールは、NVIDIAの公式aptリポジトリ経由が確実だ。ここで詰まる人が多いらしいのでそのまま書いておく。
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

インストールできたら、GPUが見えているか確認する。
```bash
sudo docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

搭載しているGPUのスペックが表示された。

いつもこういうエラーが解決した時って何とも言えない感動が湧いてくる。まあ私が解決している訳ではないのだがそれでもその嬉しさはひとしおです。

---

## GGUFという言葉の壁

Ollamaでモデルをダウンロードしようとして、HuggingFaceを見ていたら混乱した。

Safetensors、GGUF、Q4_K_M、量子化……

初心者には難解だ。

整理すると単純だ。

| 形式 | 用途 | 特徴 |
|---|---|---|
| Safetensors | 学習・ファインチューニング | フル精度、重い |
| GGUF | 推論（Ollama/llama.cpp） | 量子化済み、軽い |

OllamaはGGUFを使う。HuggingFaceで見ているSafetensorsは別物だ。

量子化というのは、モデルを圧縮する技術だ。Q4_K_Mなら4bitに圧縮している。精度は少し落ちるが、VRAMに乗る。

色々調べるとこの貧弱GPUならQwen2.5/7Bが一番性能が良いように思えたのでQwenの7Bを選択。

`qwen2.5:7b` をpullすると約4〜5GBのGGUFファイルが来る。`ollama_data` というDockerボリュームに `sha256-...` という名前で保存される。ファイル名に意味はない。Ollamaが管理している。

---

## とりあえず動いた

Ollamaコンテナを起動して、モデルをpullして、ターミナルから話しかけた。
```bash
sudo docker exec -it ollama ollama run qwen2.5:7b
```

日本語で返ってきた。

コンテナの中でLLMが動いている。自分のGPUで。APIにお金を払わずに。

**なんか、自分専用のAIを飼ったような気分**

でも、これはただのターミナルチャットだ。それでも十分満足だがどうせならOpenclawで動かしたい。

---

## 詰まりポイントのまとめ

振り返ると、この工程で詰まりやすい箇所は3つだった。

**1. WSL2ネイティブ版は黙って止まる**
エラーが出ないので原因がわからない。早めにDockerへ切り替える判断が正解だった。

**2. `--gpus all` を忘れるとCPU動作になる**
動いているように見えて、ただ遅いだけ。気づきにくい。

**3. NVIDIA Container ToolkitはNVIDIAの公式aptから入れる**
Ubuntu標準のaptから入れると古いバージョンが来てうまく動かないことがある。

---

次のvol.5では、ここから先——モデルのカスタマイズとOpenclawとの連携で詰まった話を書くとしよう。

詰まりはまだ続く。
