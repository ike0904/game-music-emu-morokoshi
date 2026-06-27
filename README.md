# game-music-emu (Morokoshi Time 改造版)

libgme 0.6.5 をベースに、チャンネルミュート時の曲頭ノイズ問題を修正したフォーク。

## 変更内容

- `gme/Music_Emu.h` / `gme/Music_Emu.cpp` : `clear_blip_buffer()` および仮想メソッド `clear_buf_impl_()` を追加
- `gme/Classic_Emu.h` / `gme/Classic_Emu.cpp` : `clear_buf_impl_()` の実装（Multi_Buffer クリア）
- `gme/Spc_Emu.h` / `gme/Spc_Emu.cpp` : `clear_buf_impl_()` の実装（Fir_Resampler + SPC_Filter クリア）
- `gme/gme.h` / `gme/gme.cpp` : 公開 API `gme_clear_blip_buffer()` を追加

## 使い方

`gme_start_track()` および `gme_mute_voice()` の後に `gme_clear_blip_buffer()` を呼ぶことで、
INIT ルーチン実行時にバッファされた全チャンネルのサンプルをクリアし、曲頭ノイズを除去する。

## ライセンス

GNU Lesser General Public License (LGPL v2.1)  
オリジナル: https://github.com/libgme/game-music-emu
