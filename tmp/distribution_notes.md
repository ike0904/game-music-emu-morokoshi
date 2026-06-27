# GitHub 公開・DLL 配布 注意点まとめ

## このフォークのライセンス

libgme は **LGPL v2.1** のライブラリ。このフォークも同様に LGPL v2.1 が適用される。  
ライセンス全文: `license.txt`（リポジトリ内に同梱済み）

---

## GitHub 公開の注意点

### リポジトリは public にすること（LGPL 要件）

LGPL では、ライブラリのソースコードをエンドユーザーが「入手可能な状態」にする義務がある。  
**private リポジトリのままでは LGPL に抵触する可能性がある。**

- **解決策①（推奨）**: このリポジトリを GitHub で public にする
- 解決策②: アプリ配布物にソースコード一式を同梱する（手間がかかる）

### README に明記すること（現状 OK ✓）

- `gme_clear_blip_buffer()` など追加した API の説明
- オリジナルリポジトリへのリンク: https://github.com/libgme/game-music-emu
- LGPL ライセンスである旨

### フォーク元のクレジット

- フォーク元: game-music-emu (libgme) — Shay Green 他, LGPL v2.1
- `copyright.txt` または README に記載があれば十分

---

## DLL 配布の注意点（Morokoshi Time に同梱する場合）

### 1. DLL は独立ファイルとして配布すること ✓

- `dist/dll/libgme.dll` として独立配布 → OK
- EXE に静的リンクした場合は「結合著作物」となり、アプリ全体を LGPL に従う必要が出る可能性がある
- **DLL を分離している現状の構成を維持すること**

### 2. LGPL ライセンステキストを同梱すること ⚠️

アプリ配布物（インストーラーやフォルダ）に以下のいずれかを含める必要がある。

| 方法 | 手間 |
|:---|:---|
| `license.txt`（LGPL全文）をフォルダに同梱 | 小 |
| アプリの「ライセンス表示」画面に LGPL テキストを掲載 | 中 |
| README 等に LGPL である旨＋ GitHub URL を記載 | 小 |

**最低限**: 「このアプリは libgme (LGPL v2.1) を使用しています。ソース: <GitHub URL>」程度の記載

### 3. ユーザーが DLL を差し替えられる状態にすること

LGPL の「リンク自由の保証」要件。`dist/dll/libgme.dll` が外部ファイルであれば、ユーザーが自分でビルドした DLL と入れ替え可能なので OK。

### 4. Morokoshi Time アプリ本体は非公開で問題ない

LGPL の義務は **ライブラリ（DLL）側** のみ。  
DLL を分離配布し、ソースを公開（または同梱）していれば、アプリ本体のソースは非公開・独自ライセンスのままで OK。

---

## チェックリスト

- [ ] `game-music-emu-morokoshi` リポジトリを GitHub で public にする
- [ ] アプリ配布物に LGPL ライセンステキスト（または URL）を含める
- [ ] `dist/dll/libgme.dll` が独立ファイルとして配布される構成を維持する
- [ ] README にオリジナルリポジトリへのリンクを記載する（現状 OK ✓）
