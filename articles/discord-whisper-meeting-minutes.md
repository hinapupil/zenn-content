---
title: "Discord会議の録音をWhisper + Claude Codeで全自動議事録化する"
emoji: "🎙️"
type: "tech"
topics: ["whisper", "discord", "claudecode", "python", "自動化"]
published: false
---

## はじめに

小規模チームで週次の Discord ミーティングをやっていると、毎回の議事録作成が地味にだるい。

従来のフロー:

1. Craig（録音Bot）で通話を録音
2. zip をダウンロードして .aac を取り出す
3. NotebookLM にアップロード
4. コンテキスト（メンバー情報など）を手動で貼り付け
5. プロンプトを手動で投入
6. 出力を整形して Markdown に保存
7. テキスト参加者のチャット履歴を手動でコピペ

毎週 15〜20 分。これをコマンド 1 発にしたい。しかもローカル完結で、音声データを外部サービスに送りたくない。

## 完成イメージ

```bash
python minutes/transcribe.py "path/to/craig-xxx.aac.zip"
```

```
議事録の日付: 2026-03-15
展開完了: 3 件の .aac ファイル
Whisper モデル 'large' を読み込み中...
VAD モデルを読み込み中...
  文字起こし中: 1-alice.aac (alice)...
  文字起こし中: 2-bob.aac (bob)...
  文字起こし中: 3-charlie.aac (charlie)...
トランスクリプト保存: minutes/raw-recordings/transcript.txt (193 セグメント)
Discord チャット: 12 件のメッセージを取得
Claude Code で議事録を生成中...
議事録を保存しました: minutes/20260315.md
```

これだけ。Whisper の処理待ちを含めて 3〜5 分で完了する。

## アーキテクチャ

```
Craig zip (.aac × 話者数 + info.txt)
  ↓ zip 展開・話者識別（ファイル名から）
Whisper large (ローカル GPU)
  ↓ 話者別タイムスタンプ付き文字起こし + VAD フィルタ
Discord Bot API
  ↓ 録音時間帯のチャット履歴を自動取得
Claude Code CLI (claude -p)
  ↓ コンテキスト + トランスクリプト + チャットログ → 議事録生成
YYYYMMDD.md に保存
```

ポイントは **Craig が話者ごとに別ファイルで録音してくれる**こと。これのおかげで、話者分離（diarization）のための追加ライブラリが不要になる。

## 準備 1: Craig（録音Bot）の導入

[Craig](https://craig.chat/) は Discord の通話を話者別に録音してくれる Bot。

### サーバーへの追加

1. https://craig.chat/ にアクセス
2. 招待リンクからサーバーに追加
3. 必要な権限（ボイスチャンネルへの参加、メッセージ送信）を許可

### 録音の操作

| 操作 | コマンド | 備考 |
|------|---------|------|
| 録音開始 | `/join` | ボイスチャンネル内で実行。ニックネームが `[RECORDING]` に変わる |
| 録音停止 | `/stop` | Craig がチャンネルから退出する |
| 録音一覧 | `/recordings` | 直近 5 件の録音リンクを表示 |

録音開始時に Craig から DM でダウンロードリンクが届く。録音停止後、そこから zip をダウンロードする。

### zip の中身

```
craig-{session-id}.aac.zip
├── info.txt              # 録音開始時刻、チャンネル情報
├── 1-alice.aac           # 話者別の音声ファイル
├── 2-bob.aac
└── 3-charlie.aac
```

`{番号}-{ユーザー名}.aac` というファイル名から話者を識別できる。`info.txt` の `Start time` から録音開始時刻も取得できる。

## 準備 2: Discord Bot（チャット取得用）の作成

テキストのみで参加するメンバーの発言は音声に含まれない。Discord Bot API で録音時間帯のチャット履歴を自動取得する。

### アプリケーション作成

1. [Discord Developer Portal](https://discord.com/developers/applications) にログイン
2. **「New Application」** → 名前を入力（例: `minutes-bot`）→ **Create**

### Bot の追加とトークン取得

1. 左メニューの **「Bot」** をクリック
2. **「Add Bot」** → Bot を追加
3. **「Token」** セクションの **「Copy」** でトークンをコピー

トークンは初回のみ表示される。見逃した場合は **「Reset Token」** で再生成できるが、旧トークンは無効化される。

:::message alert
OAuth2 ページの「クライアントシークレット」は Bot Token とは別物。API 認証には Bot ページのトークンを使う。
:::

### Message Content Intent の有効化

1. Bot ページ下部の **「Privileged Gateway Intents」**
2. **「Message Content Intent」** を **ON** → **Save Changes**

これがないとメッセージ本文が空で返ってくる。

### サーバーに招待

1. 左メニューの **「OAuth2」** → **「OAuth2 URL Generator」**
2. **SCOPES**: `bot` にチェック
3. **BOT PERMISSIONS**: `Read Message History` と `View Channels` にチェック
4. 生成された URL をブラウザで開き、対象サーバーを選択して認証

### チャンネル ID の取得

1. Discord クライアントの設定 → **詳細設定** → **「開発者モード」** を ON
2. メッセージを取得したいチャンネルを右クリック → **「チャンネル ID をコピー」**

### `.env` の設定

プロジェクトルートに `.env` を作成:

```
DISCORD_BOT_TOKEN=ここにBotトークン
DISCORD_CHANNEL_ID=ここにチャンネルID
```

`.gitignore` に `.env` を追加しておくこと。

## 準備 3: Whisper 環境構築

### Python バージョンの罠

**PyTorch は Python 3.13 に未対応**（2026 年 3 月時点）。3.12 の仮想環境を作る必要がある。

```bash
uv venv --python 3.12 .venv-whisper
.venv-whisper/Scripts/activate  # Windows
# source .venv-whisper/bin/activate  # macOS/Linux
```

### インストール

```bash
# PyTorch (CUDA 12.1)
uv pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Whisper
uv pip install openai-whisper
```

CPU のみの場合は `--index-url` を省略する。

### GPU 別モデル選択

| モデル | VRAM 目安 | 日本語精度 |
|--------|----------|-----------|
| tiny | ~1GB | いまいち |
| base | ~1GB | そこそこ |
| medium | ~5GB | 実用的 |
| large | ~10GB | かなり良い |

RTX 3060（12GB）なら large が余裕で動く。初回実行時にモデル（約 2.9GB）をダウンロードする。

## 準備 4: Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
```

Claude のサブスクリプションプランが必要。API キーは不要。

## スクリプト解説

### zip 展開と話者識別

Craig のファイル名 `{番号}-{ユーザー名}.aac` を活用して、話者分離ライブラリなしで話者を識別する。

```python
def parse_speaker(filename: str) -> str:
    """'1-alice.aac' → 'alice'"""
    stem = Path(filename).stem
    match = re.match(r"\d+-(.+)", stem)
    return match.group(1) if match else stem
```

### Whisper 文字起こし + ハルシネーション対策

Whisper は無音区間で「ご視聴ありがとうございました」「チャンネル登録お願いします」などのハルシネーションを生成することがある。3 段階で対策する。

**1. Whisper のパラメータ調整**

```python
result = model.transcribe(
    str(aac),
    language="ja",
    condition_on_previous_text=False,  # 前文脈に引きずられるループを防止
    no_speech_threshold=0.6,           # 無音判定を厳しく
    logprob_threshold=-1.0,            # 低確信度セグメントを除外
)
```

`condition_on_previous_text=False` が最も効果的。デフォルト `True` だと前のセグメントの出力に引きずられてハルシネーションがループする。

**2. Silero VAD（Voice Activity Detection）で無音区間を事前検出**

Whisper の前に VAD で「人が話している区間」を検出し、Whisper の結果と突合する。VAD が「無音」と判定した区間の Whisper 出力は捨てる。

```python
def is_in_speech(start, end, speech_ranges):
    """セグメントが発話区間と重なるか判定"""
    for s_start, s_end in speech_ranges:
        if start < s_end and end > s_start:
            return True
        if s_start > end:
            break
    return False
```

**3. 既知パターンのフィルタ**

```python
HALLUCINATION_PATTERNS = re.compile(
    r"(ご視聴ありがとうございました|ご清聴ありがとうございました|"
    r"チャンネル登録|お便りお待ちして|字幕は自動生成|"
    r"Thanks for watching|Please subscribe)",
    re.IGNORECASE,
)
```

### Discord API でチャット取得

Craig の `info.txt` から録音開始時刻を取得し、Discord の snowflake ID に変換してメッセージを取得する。

```python
def datetime_to_snowflake(dt: datetime) -> int:
    """datetime を Discord snowflake ID に変換"""
    DISCORD_EPOCH = 1420070400000
    ms = int(dt.timestamp() * 1000)
    return (ms - DISCORD_EPOCH) << 22

url = f"https://discord.com/api/v10/channels/{channel_id}/messages?after={snowflake}&limit=100"
headers = {
    "Authorization": f"Bot {token}",
    "User-Agent": "DiscordBot (minutes-bot, 1.0)",
}
```

:::message alert
**User-Agent ヘッダは必須。** Python 標準の `urlopen` はデフォルトで User-Agent を送らないため、Cloudflare に弾かれて `error code: 1010` になる。`requests` ライブラリを使うか、明示的に `User-Agent` を設定すること。
:::

### Claude Code CLI で議事録生成

`claude -p`（パイプモード）にコンテキスト・テンプレート・トランスクリプト・チャットログをまとめて渡す。

```python
result = subprocess.run(
    [
        claude_cmd, "-p",
        "--output-format", "text",
        "--system-prompt", "あなたは議事録生成アシスタントです。...",
    ],
    input=full_prompt,
    capture_output=True,
    text=True,
    encoding="utf-8",
)
```

議事録のフォーマットはテンプレート（`generate-minutes.md`）で制御する。過去の議事録を例として渡すことで、フォーマットの一貫性を保てる。

## ハマりポイントまとめ

### Python 3.13 で PyTorch が動かない

```
ERROR: No matching distribution found for torch
```

PyTorch は 3.12 までの対応。`uv venv --python 3.12` で回避。

### Discord API が Cloudflare に弾かれる（error code: 1010）

Python の `urllib.request.urlopen` は User-Agent を送らない。`requests` ライブラリに切り替えるか、明示的にヘッダを設定する。

### Whisper のハルシネーション

無音区間で「ご視聴ありがとうございました」を生成する。`condition_on_previous_text=False` + VAD + パターンフィルタの 3 段構えで対策。

### Bot Token と OAuth2 Client Secret の混同

Developer Portal に 2 種類の秘密情報がある。Bot API には **Bot ページのトークン**を使う。OAuth2 ページのクライアントシークレットは別物。

## まとめ

- **手動 15〜20 分 → コマンド 1 発（待ち時間 3〜5 分）**
- **ローカル完結**: 音声データを外部サービスに送らない
- **Craig の話者分離活用**: diarization ライブラリ不要
- **テキスト参加者も自動取得**: Discord Bot API で漏れなし

NotebookLM などの SaaS でも文字起こし→要約はできるが、音声データの外部送信が気になる場合や、議事録フォーマットを細かくコントロールしたい場合はこのアプローチが向いている。

## 次にやりたいこと

現状はまだ「Craig の zip を手動でダウンロードしてコマンドを実行する」というステップが残っている。次は以下を自動化して、**会議が終わったら議事録の下書きが勝手にできている**状態を目指したい。

- **録音の開始・終了の自動化**: minutes-bot にスケジュール機能を持たせるか、Craig の `/join` `/stop` を Bot 経由で自動実行する
- **録音ファイルの自動取得**: Craig のダウンロードリンクを Bot が監視し、zip を自動ダウンロード
- **パイプラインの自動トリガー**: zip 取得をトリガーにして文字起こし→議事録生成を自動実行

生成された議事録はあくまで下書き。人間がレビュー・修正してから Discord チャンネルに共有する流れは変わらない。

続編として記事にする予定。

:::details transcribe.py 全文
```python
"""議事録自動生成スクリプト

Craig (Discord録音ボット) の zip → Whisper 文字起こし → Claude Code で議事録生成
"""

import argparse
import json
import os
import re
import shutil
import subprocess
import zipfile
from datetime import datetime, timezone
from pathlib import Path
import requests as _requests

MINUTES_DIR = Path(__file__).parent
PROJECT_ROOT = MINUTES_DIR.parent
RAW_DIR = MINUTES_DIR / "raw-recordings"
WEEKDAYS_JA = ["月", "火", "水", "木", "金", "土", "日"]


def load_dotenv():
    """プロジェクトルートの .env を読み込んで環境変数にセット"""
    env_path = PROJECT_ROOT / ".env"
    if not env_path.exists():
        return
    for line in env_path.read_text(encoding="utf-8").splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        if "=" in line:
            key, _, value = line.partition("=")
            os.environ.setdefault(key.strip(), value.strip())


def extract_zip(zip_path: Path) -> list[Path]:
    """Craig zip を展開し、.aac ファイルのリストを返す"""
    RAW_DIR.mkdir(exist_ok=True)
    aac_files = []
    with zipfile.ZipFile(zip_path) as zf:
        for name in zf.namelist():
            if name.endswith(".aac"):
                zf.extract(name, RAW_DIR)
                aac_files.append(RAW_DIR / name)
    print(f"展開完了: {len(aac_files)} 件の .aac ファイル")
    return aac_files


def parse_speaker(filename: str) -> str:
    """'1-alice.aac' → 'alice'"""
    stem = Path(filename).stem
    match = re.match(r"\d+-(.+)", stem)
    return match.group(1) if match else stem


def parse_start_time_from_zip(zip_path: Path) -> datetime | None:
    """zip 内の info.txt から録音開始時刻を取得"""
    try:
        with zipfile.ZipFile(zip_path) as zf:
            if "info.txt" in zf.namelist():
                info = zf.read("info.txt").decode("utf-8", errors="replace")
                match = re.search(
                    r"Start time:\s+(\d{4}-\d{2}-\d{2})T(\d{2}:\d{2}:\d{2})", info
                )
                if match:
                    return datetime.fromisoformat(
                        f"{match.group(1)}T{match.group(2)}+00:00"
                    )
    except Exception:
        pass
    return None


def datetime_to_snowflake(dt: datetime) -> int:
    """datetime を Discord snowflake ID に変換"""
    DISCORD_EPOCH = 1420070400000
    ms = int(dt.timestamp() * 1000)
    return (ms - DISCORD_EPOCH) << 22


def fetch_discord_chat(start_time: datetime, duration_hours: float = 3.0) -> str | None:
    """Discord API でミーティング中のチャット履歴を取得"""
    token = os.environ.get("DISCORD_BOT_TOKEN")
    channel_id = os.environ.get("DISCORD_CHANNEL_ID")
    if not token or not channel_id:
        return None

    after_snowflake = datetime_to_snowflake(start_time)
    end_time = start_time.timestamp() + duration_hours * 3600

    url = f"https://discord.com/api/v10/channels/{channel_id}/messages?after={after_snowflake}&limit=100"
    headers = {
        "Authorization": f"Bot {token}",
        "User-Agent": "DiscordBot (minutes-bot, 1.0)",
    }

    try:
        resp = _requests.get(url, headers=headers)
        resp.raise_for_status()
        messages = resp.json()
    except Exception as e:
        print(f"警告: Discord チャット取得に失敗しました: {e}")
        return None

    messages.reverse()

    lines = []
    for msg in messages:
        msg_time = datetime.fromisoformat(msg["timestamp"]).timestamp()
        if msg_time > end_time:
            break
        author = msg["author"]["username"]
        content = msg["content"]
        if content:
            lines.append(f"{author}: {content}")

    if not lines:
        print("Discord チャット: メッセージなし")
        return None

    print(f"Discord チャット: {len(lines)} 件のメッセージを取得")
    return "\n".join(lines)


def parse_date_from_zip(zip_path: Path) -> str | None:
    """zip 内の info.txt から録音日を取得"""
    start = parse_start_time_from_zip(zip_path)
    if start:
        from datetime import timedelta
        jst = start + timedelta(hours=9)
        return jst.strftime("%Y-%m-%d")
    return None


def format_timestamp(seconds: float) -> str:
    """秒数を HH:MM:SS 形式に変換"""
    h = int(seconds // 3600)
    m = int((seconds % 3600) // 60)
    s = int(seconds % 60)
    return f"{h:02d}:{m:02d}:{s:02d}"


HALLUCINATION_PATTERNS = re.compile(
    r"(ご視聴ありがとうございました|ご清聴ありがとうございました|"
    r"チャンネル登録|お便りお待ちして|字幕は自動生成|"
    r"Thanks for watching|Please subscribe)",
    re.IGNORECASE,
)


def load_vad_model():
    """Silero VAD モデルを読み込み"""
    import torch
    model, utils = torch.hub.load(
        "snakers4/silero-vad", "silero_vad", trust_repo=True,
    )
    get_speech_timestamps = utils[0]
    return model, get_speech_timestamps


def convert_to_wav16k(audio_path: Path) -> Path:
    """ffmpeg で 16kHz モノラル WAV に変換"""
    wav_path = audio_path.with_suffix(".wav")
    if wav_path.exists():
        return wav_path
    subprocess.run(
        ["ffmpeg", "-i", str(audio_path), "-ar", "16000", "-ac", "1", "-y", str(wav_path)],
        capture_output=True,
    )
    return wav_path


def get_speech_ranges(audio_path, vad_model, get_speech_timestamps):
    """VAD で発話区間（秒）のリストを返す"""
    import torch
    import wave
    import numpy as np

    wav_path = convert_to_wav16k(audio_path)
    with wave.open(str(wav_path), "rb") as wf:
        sr = wf.getframerate()
        frames = wf.readframes(wf.getnframes())
        audio = np.frombuffer(frames, dtype=np.int16).astype(np.float32) / 32768.0
    wav_tensor = torch.from_numpy(audio)

    timestamps = get_speech_timestamps(
        wav_tensor, vad_model, sampling_rate=sr,
        threshold=0.3,
        min_speech_duration_ms=100,
        min_silence_duration_ms=300,
    )
    return [(ts["start"] / sr, ts["end"] / sr) for ts in timestamps]


def is_in_speech(start, end, speech_ranges):
    """セグメントが発話区間と重なるか判定"""
    for s_start, s_end in speech_ranges:
        if start < s_end and end > s_start:
            return True
        if s_start > end:
            break
    return False


def transcribe(aac_files, model="large"):
    """Whisper で文字起こしし、時系列統合トランスクリプトを返す"""
    import whisper

    print(f"Whisper モデル '{model}' を読み込み中...")
    w = whisper.load_model(model)

    print("VAD モデルを読み込み中...")
    vad_model, get_speech_timestamps = load_vad_model()

    all_segments = []

    for aac in sorted(aac_files):
        speaker = parse_speaker(aac.name)
        print(f"  文字起こし中: {aac.name} ({speaker})...")

        speech_ranges = get_speech_ranges(aac, vad_model, get_speech_timestamps)
        speech_sec = sum(e - s for s, e in speech_ranges)
        print(f"    VAD: 発話区間 {speech_sec:.0f}秒 / {len(speech_ranges)} 区間")

        result = w.transcribe(
            str(aac),
            language="ja",
            condition_on_previous_text=False,
            no_speech_threshold=0.6,
            logprob_threshold=-1.0,
        )

        prev_text = ""
        for seg in result["segments"]:
            seg_text = seg["text"].strip()
            if not is_in_speech(seg["start"], seg["end"], speech_ranges):
                continue
            if seg.get("no_speech_prob", 0) > 0.5:
                continue
            if seg_text == prev_text:
                continue
            if HALLUCINATION_PATTERNS.search(seg_text) and len(seg_text) < 30:
                continue
            all_segments.append((seg["start"], speaker, seg_text))
            prev_text = seg_text

    all_segments.sort(key=lambda x: x[0])

    lines = []
    for start, speaker, text in all_segments:
        ts = format_timestamp(start)
        lines.append(f"[{ts}] {speaker}: {text}")

    transcript = "\n".join(lines)
    out_path = RAW_DIR / "transcript.txt"
    out_path.write_text(transcript, encoding="utf-8")
    print(f"トランスクリプト保存: {out_path} ({len(all_segments)} セグメント)")
    return transcript


def find_latest_minutes():
    """minutes/ 内の最新 YYYYMMDD.md を読んで返す"""
    md_files = sorted(MINUTES_DIR.glob("[0-9]" * 8 + ".md"), reverse=True)
    if md_files:
        return md_files[0].read_text(encoding="utf-8")
    return None


def generate_minutes(transcript, date_str, chat_log=None):
    """claude CLI で議事録を生成"""
    dt = datetime.strptime(date_str, "%Y-%m-%d")
    weekday = WEEKDAYS_JA[dt.weekday()]
    date_label = f"{dt.year}-{dt.month:02d}-{dt.day:02d}({weekday})"

    context = (MINUTES_DIR / "context.md").read_text(encoding="utf-8")
    template = (MINUTES_DIR / "generate-minutes.md").read_text(encoding="utf-8")
    prev = find_latest_minutes() or "(前回の議事録なし)"

    prompt_text = template.replace("YYYY-MM-DD(曜日)", date_label)
    prompt_text = prompt_text.replace("前回の議事録をコピペ", prev)

    chat_section = ""
    if chat_log:
        chat_section = f"\n# Discord チャット履歴\n{chat_log}\n"

    full_prompt = f"""以下のコンテキストとトランスクリプトを元に議事録を生成してください。
議事録の Markdown のみを出力してください。前置き、説明、確認メッセージは一切不要です。

# コンテキスト
{context}

# 指示
{prompt_text}

# トランスクリプト
{transcript}
{chat_section}"""

    claude_cmd = shutil.which("claude")
    if not claude_cmd:
        print("エラー: claude CLI が見つかりません。npm install -g @anthropic-ai/claude-code を実行してください")
        raise SystemExit(1)

    print("Claude Code で議事録を生成中...")
    result = subprocess.run(
        [
            claude_cmd, "-p",
            "--output-format", "text",
            "--setting-sources", "user",
            "--system-prompt",
            "あなたは議事録生成アシスタントです。指示に従い、議事録のMarkdownのみを出力してください。",
        ],
        input=full_prompt,
        capture_output=True,
        text=True,
        encoding="utf-8",
    )

    if result.returncode != 0:
        print(f"エラー: claude CLI が失敗しました\n{result.stderr}")
        raise SystemExit(1)

    return result.stdout.strip()


def main():
    parser = argparse.ArgumentParser(description="Craig zip から議事録を自動生成")
    parser.add_argument("zip_path", type=Path, help="Craig の zip ファイルパス")
    parser.add_argument("--date", type=str, help="議事録の日付 (YYYY-MM-DD)")
    parser.add_argument("--model", type=str, default="large", help="Whisper モデル")
    parser.add_argument("--skip-transcribe", action="store_true", help="文字起こしをスキップ")
    args = parser.parse_args()

    load_dotenv()

    if not args.zip_path.exists():
        print(f"エラー: ファイルが見つかりません: {args.zip_path}")
        raise SystemExit(1)

    date_str = args.date or parse_date_from_zip(args.zip_path) or datetime.now().strftime("%Y-%m-%d")
    print(f"議事録の日付: {date_str}")

    aac_files = extract_zip(args.zip_path)

    if args.skip_transcribe:
        transcript_path = RAW_DIR / "transcript.txt"
        if not transcript_path.exists():
            print("エラー: transcript.txt が見つかりません")
            raise SystemExit(1)
        transcript = transcript_path.read_text(encoding="utf-8")
        print("既存の transcript.txt を使用")
    else:
        transcript = transcribe(aac_files, model=args.model)

    chat_log = None
    start_time = parse_start_time_from_zip(args.zip_path)
    if start_time:
        chat_log = fetch_discord_chat(start_time)

    minutes = generate_minutes(transcript, date_str, chat_log=chat_log)

    dt = datetime.strptime(date_str, "%Y-%m-%d")
    out_name = dt.strftime("%Y%m%d") + ".md"
    out_path = MINUTES_DIR / out_name
    out_path.write_text(minutes, encoding="utf-8")
    print(f"議事録を保存しました: {out_path}")


if __name__ == "__main__":
    main()
```
:::
