# fizz-youtube-chat-source

**Fizz** コメント入力系部品 — YouTube Live Chat を polling して正規化コメント
([fizz-protocol](https://github.com/Aid-On/fizz-protocol) の `Comment`) を
NDJSON で stdout に流す。

責務は一行: **YouTube Data API v3 → comment ストリーム**。
dedupe は行わない ([fizz-comment-dedupe](https://github.com/Aid-On/fizz-comment-dedupe) の責務)。
書き込み (チャット送信) もしない — 読み取り専用、API key のみで OAuth 不要。

## Usage

```bash
almide build src/main.almd -o fizz-youtube-chat-source

YOUTUBE_API_KEY=... YOUTUBE_VIDEO_ID=... ./fizz-youtube-chat-source \
  | fizz-comment-dedupe \
  | ...
```

stdout: 1 行 1 コメント (`{"tag":"Comment","value":{...}}` 形式ではなく
`Comment.encode` の Codec JSON)。ログ・診断はすべて stderr。

## 挙動

- `videos.list` で `activeLiveChatId` を取得 (配信前は 60s 間隔で再試行 —
  配信開始を待てる)
- `liveChat/messages` を pageToken 引き継ぎで polling。サーバの
  `pollingIntervalMillis` を尊重 (下限 5s で clamp、quota 保護)
- 初回ページは履歴なので**流さない** (priming)
- `textMessageEvent` 以外 (superChat / member / 制裁系) はスキップ
- quotaExceeded → 5 分 backoff / chat ended → bootstrap に戻る /
  その他 transient → 15s retry

## Quota の現実

Data API v3 default 10,000 units/日、list = 5 units/回。
5 秒間隔 polling ≈ 3,600 units/時 ≈ **2.7 時間/日**。長配信は別 key を
用意するか間隔を上げること (混雑時はサーバ側 interval が 10-30s に伸びる
ため実測ではもう少し保つ)。

## Env

| var | 意味 |
|---|---|
| `YOUTUBE_API_KEY` | Data API v3 の API key (読み取り専用で十分) |
| `YOUTUBE_VIDEO_ID` | 対象配信の video ID |

## Tests

```bash
almide test
```

純粋ロジック (レスポンス解析 / エラー分類 / clamp / priming 対象の判定) を
HTTP なしでテストする。
