# SPC 曲頭ノイズ修正 — 経緯・原因・修正箇所まとめ

---

## 問題の症状

`gme_start_track()` → `gme_mute_voice()` × n → `gme_clear_blip_buffer()` の順で呼んだとき、  
NSF・GBS は曲頭ノイズが出ないのに **SPC だけ** 「ブツッ」というノイズや「曲頭が削られている」感覚が出た。

---

## NSF・GBS と SPC の構造的な違い

### NSF・GBS（Classic_Emu + Blip_Buffer）

```
clear_buf_impl_() → buf->clear()
```

- Blip_Buffer は「波形の変化点」を蓄積するバッファ
- `clear()` を呼んでも **DSP は一切進まない**
- 次の `play_()` はエミュレータの現在状態から即座に生成を再開
- → クリア後も正確な位置から音が出る

### SPC（Snes_Spc DSP + Fir_Resampler<24>）

- DSP はストリーミング型（連続して native 32kHz サンプルを生成し続ける）
- `Fir_Resampler<24>` が DSP の出力を **3200 native サンプル単位でバッファする**
- `resampler.clear()` でバッファをゼロクリアしても **DSP の位置は変わらない**

---

## 問題の根本原因

### start_track() のサイレンス検出が終わった時点の状態

```
DSP 位置:  native 3200 サンプル目
buf[]:     出力サンプル 0〜2047（native 0〜1486 に対応）→ buf_remain = 2048
resampler: native 1487〜3199 のサンプルが pending（約 54ms 分）
```

### clear_blip_buffer() が呼ばれると

```
buf_remain = 0          → buf[] の 2048 サンプルを破棄
clear_buf_impl_()
  └─ resampler.clear() → pending の 1714 サンプルを破棄（DSP 位置は 3200 のまま）
```

### 次の play_() で起きること

```
resampler.read() → 0（バッファ空）
play_and_filter(3200, ...) → apu.play(3200, ...) → DSP が 3200〜6399 に進む
                                                    ↑ ここから出力が始まる
```

**DSP は 3200 から再スタートする = native 0〜3199（≒100ms）が永遠に失われる。**

- native 0〜1486 は `buf[]` に入っていたが `buf_remain = 0` で破棄
- native 1487〜3199 は resampler の pending にあったが `resampler.clear()` で破棄
- 次の出力は native 3200 から → ユーザーには「頭 100ms 削られている」と聞こえる

---

## 試みた修正と失敗の理由

| 修正内容 | 結果 |
|:---|:---|
| `resampler.clear()` のみ | native 3200 からスタート → 100ms カット（根本原因） |
| `+ filter.clear()` | IIR フィルタのコールドスタート過渡応答が追加されて悪化 |
| `+ play_(64, warmup)` | resampler を 64 出力サンプル分だけウォームアップ。FIR コールドスタートは消えるが **DSP がさらに 3200 サンプル先に進む**（カットが 100ms + α に）。エコーも 100ms 分プリビルドされリバーブのような音に |
| `apu.clear_echo()` を追加 | エコーテールは消えるが 100ms カットとリバーブは残る |
| ウォームアップなしに戻す | 100ms カットが残る。FIR コールドスタート（0.5ms）も体感できる |

**共通の限界: `resampler.clear()` を呼ぶ以上、DSP 位置と resampler のズレは解消できない。**

---

## 最終的な解決策

**SPC を最初からリロードし、ミュートを適用した状態でサイレンス検出をやり直す。**

```
clear_buf_impl_()
  ├─ apu.load_spc()              → DSP を native 位置 0 に戻す
  ├─ filter.set_gain()           → フィルタゲインを再設定
  ├─ apu.clear_echo()            → エコーバッファをクリア
  ├─ apu.set_tempo()             → テンポを再適用
  ├─ apu.mute_voices(mute_mask) → ミュートを適用（← ここが肝心）
  ├─ resampler.clear()           → FIR バッファをリセット
  ├─ filter.clear()              → IIR フィルタをリセット
  └─ redo_silence_detection_()   → サイレンス検出をやり直す
       └─ fill_buf() ループ（start_track() と同じロジック）
            └─ ミュートした状態で non-silence が見つかるまで生成
                 → buf[] に正しい位置からのミュート済み音声が入る
```

この方法であれば：

- DSP は位置 0 から再スタートするためカットが発生しない
- ミュートがサイレンス検出の段階から適用されているため、unmuted な音声はバッファに入らない
- エコーバッファはリロード直後にクリアされているため残響なし

---

## 修正ファイルと変更箇所

### `gme/Music_Emu.h`

```cpp
// protected に追加
int mute_mask() const { return mute_mask_; }
void redo_silence_detection_();
```

`mute_mask_` は private だったため、派生クラス（Spc_Emu）からアクセスできるよう getter を追加。

---

### `gme/Music_Emu.cpp`

`redo_silence_detection_()` を新規追加。  
`start_track()` のサイレンス検出ループと同じロジック（エミュレータ本体の再初期化は呼び出し元に任せる）。

```cpp
void Music_Emu::redo_silence_detection_()
{
    buf_remain = 0; emu_time = 0; out_time = 0; out_time_scaled = 0;
    emu_track_ended_ = false; track_ended_ = false;
    silence_time = 0; silence_count = 0;

    if ( !ignore_silence_ )
    {
        for ( long end = max_initial_silence * out_channels() * sample_rate(); emu_time < end; )
        {
            fill_buf();
            if ( buf_remain | (int) emu_track_ended_ ) break;
        }
        emu_time = buf_remain;
        out_time = 0; out_time_scaled = 0; silence_time = 0; silence_count = 0;
    }
}
```

---

### `gme/Spc_Emu.cpp` — `clear_buf_impl_()`

**変更前:**
```cpp
void Spc_Emu::clear_buf_impl_()
{
    resampler.clear();
    filter.clear();
    // 64 サンプルのウォームアップ（FIR コールドスタート対策として試みたが DSP が進むため NG）
    const int resampler_latency = 64;
    sample_t warmup [resampler_latency];
    (void) play_( resampler_latency, warmup );
}
```

**変更後:**
```cpp
void Spc_Emu::clear_buf_impl_()
{
    if ( apu.load_spc( file_data, file_size ) ) return;
    filter.set_gain( (int) (gain() * SPC_Filter::gain_unit) );
    apu.clear_echo();
    apu.set_tempo( (int) (tempo() * apu.tempo_unit) );
    apu.mute_voices( mute_mask() );
    resampler.clear();
    filter.clear();
    redo_silence_detection_();
}
```

---

## NSF・GBS への影響

`clear_buf_impl_()` の override は `Classic_Emu`（NSF・GBS）と `Spc_Emu`（SPC）で別々に実装されている。  
今回の変更は `Spc_Emu::clear_buf_impl_()` のみ。NSF・GBS は従来通り `buf->clear()` で動作し、影響なし。

---

## コミット

```
774dfa7  Fix SPC track-start cut by reloading DSP with muting before silence detection
```
