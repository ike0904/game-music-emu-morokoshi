# SPC だけ曲頭ノイズが残る理由

## 症状

`gme_start_track()` → `gme_mute_voice()` → `gme_clear_blip_buffer()` の後、  
NSF・GBS は曲頭ノイズが消えたが、SPC だけ消えない。

---

## NSF/GBS が直った理由（比較）

NSF・GBS は **Classic_Emu + Blip_Buffer** を使う。

- Blip_Buffer は波形の「変化点（デルタ）」を蓄積し、再生時に PCM を合成する
- `clear_buf_impl_()` → `buf->clear()` で再生バッファが完全リセットされる
- 過去の音声を「残響として持ち続ける」機構が存在しない
- クリア後の最初の生成は常にクリーン

---

## SPC の構造的な違い

SPC は **Snes_Spc DSP + Fir_Resampler** を使う。DSP は以下を持つ：

### 1. エコーバッファ（最大の問題）

- SNES RAM 上に格納された独立したエコー FIR 機構
- DSP はエコーバッファに音声を書き込みつつ、過去のバッファ内容を読み出してミックスする
- `gme_mute_voice()` でボイスの**直接出力**はゼロになるが、  
  **エコーバッファ内の既存コンテンツは消えない**
- エコーバッファが減衰するまで、全チャンネルの残響が DSP 出力に混合され続ける

### 2. FIR リサンプラー（32kHz → 出力レート）

- `Fir_Resampler<24>` が約 3200 サンプルのバッファを持つ
- `resampler.clear()` で FIR 履歴がゼロになる（コールドスタート過渡応答が発生する）
- `play_(64, warmup)` で 64 サンプル捨てることで FIR ゼロ履歴はフラッシュできる

### 3. Spc_Filter（IIR ハイパス/ローパス）

- `filter.clear()` でゼロリセットするとコールドスタート過渡応答が発生する
- `start_track()` のサイレンス検出で既に安定状態になっているため、クリアしない方がよい

---

## ノイズが発生する流れ

```
start_track()
  ├─ apu.load_spc()
  ├─ resampler.clear()
  ├─ filter.clear()
  ├─ apu.clear_echo()     ← エコーバッファをリセット
  └─ サイレンス検出ループ（全チャンネル有効で何千サンプルも生成）
       │
       └─ この間、エコーバッファに全チャンネルの音声が蓄積される

gme_mute_voice() × n    ← ボイス直接出力はゼロに
                            ただしエコーバッファの内容は変化なし

gme_clear_blip_buffer()
  └─ clear_buf_impl_()
       ├─ resampler.clear()
       └─ play_(64, warmup)
            └─ play_and_filter(3200, ...)
                 └─ apu.play(3200, ...)
                      │
                      └─ DSP 出力 = ミュートしたボイス出力
                                  + エコーバッファ内の全チャンネル残響  ← ★ノイズ源

↓ 3200 サンプルのうち 64 出力サンプル分だけ捨てる
↓ 残り ~3180 ネイティブサンプルがリサンプラーバッファに滞留

実再生開始
  └─ resampler.read() がバッファに残ったエコー混入サンプルを読み出す
       └─ ユーザーに「曲頭ノイズ」として聞こえる
```

---

## 試みた修正と結果

| 修正内容 | FIR cold-start | IIR cold-start | Echo tail | 曲頭カット |
|:---|:---:|:---:|:---:|:---:|
| `resampler.clear()` のみ | ❌ | — | ❌ | なし |
| `+filter.clear()` `+warmup(64)` | ✅ | ❌ | ❌ | あり（100ms） |
| `+warmup(64)` フィルタクリアなし | ✅ | ✅ | ❌ | あり（100ms） |
| `+apu.clear_echo()` `+warmup(64)` | ✅ | ✅ | ✅ | あり（100ms）❌ |
| **`+apu.clear_echo()` のみ（現在）** | △ | ✅ | ✅ | なし |

### ウォームアップの副作用（重要）

`play_(64, warmup)` を呼ぶと、リサンプラーバッファを埋めるために  
`apu.play(3200, ...)` が実行される → SPC DSP が **3200ネイティブサンプル（≒100ms）先に進む**。

この結果：
- 曲頭約 100ms が欠ける（ユーザーが「削られている」と体感）
- エコーが 100ms 分プリビルドされ、再生開始時に「リバーブの余韻」として聞こえる

→ **ウォームアップによる修正は NG**。ノイズが乗る方がマシ。

---

## 現在の修正（案B）

```cpp
void Spc_Emu::clear_buf_impl_()
{
    apu.clear_echo();   // エコーバッファをリセット（最大の問題を解決）
    resampler.clear();  // FIR 履歴をリセット（24タップ≒0.5ms の緩やかなアタックが残る）
    // filter は意図的にクリアしない（サイレンス検出で既に安定状態）
    // ウォームアップは NG（DSP が先に進んで曲頭が欠ける）
}
```

`apu.clear_echo()` は SPC RAM 上のエコーバッファを 0xFF で上書きする  
（`start_track_()` でも同じ処理を実施）。

FIR コールドスタート（24タップ、約 0.5ms の緩やかなアタック）は残るが、  
エコーテールや曲頭カットよりはるかに小さい問題のため、許容範囲と判断する。

---

## それでも直らない場合の検討事項

- `SPC_ISOLATED_ECHO_BUFFER` マクロが定義されている場合、`clear_echo()` は何もしない → ビルド設定を確認
- エコーバッファ以外の DSP 内部状態（ガウス補間履歴、ADSR エンベロープ等）が原因の可能性
- `apu.load_spc()` を再呼び出しして DSP を完全リセット後にミュートを適用する（ただし再生位置がリセットされる問題あり）

---

## 参照コード

- `gme/Spc_Emu.cpp`: `clear_buf_impl_()`, `start_track_()`
- `gme/Snes_Spc.cpp:249`: `Snes_Spc::clear_echo()`
- `gme/Fir_Resampler.h`: `Fir_Resampler<24>`, `clear()`, `read()`
- `gme/Spc_Filter.cpp`: `SPC_Filter::run()`, `clear()`
