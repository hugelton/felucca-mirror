<!-- SPDX-License-Identifier: GPL-3.0-only -->
# Felucca-mirror 実装計画

作成 2026-10-05。対象ファームウェア: `fm-1-research-lab` の `felucca/`（コミット `f2468a8` 時点、v0.9beta 系）。
この文書は計画だけです。ファームウェアにはまだ何も手を入れていません。

---

## 0. 要約（結論）

| 論点 | 決定（推奨） |
| --- | --- |
| 描画 | UI の C コードを **clang `--target=wasm32` のフリースタンディング**でそのままコンパイル（emscripten は予備）。ホストの `ui_test.c` / `ui_render.c` と同じ組み立て方 |
| 送るもの | **(c) ハイブリッド、ただし主体は (b) 状態レプリケーション**。入力イベントのロックステップ (a) は採らない |
| 状態の送り方 | 「領域テーブル」に並べた RAM 範囲を **16 バイトブロック × 16 bit ハッシュ**で差分検出し、変わったブロックだけ送る。加えて毎フレームの小さな「ライブ欄」（時刻、フレーム番号、メータ、再生位置、LED…）と、必要なときだけ「スコープ軌跡」 |
| 描画キャッシュ | `ui.col[]`・`*_sig`・`roll[]` などの描画済みキャッシュは**送らない／上書きしない**（受信側は自分のキャッシュを保持）。抜けを検出したら受信側で強制全描画 |
| 転送 | **SysEx over USB-MIDI**（新コマンド MIRROR_*）。CDC + Web Serial は同じフレーム形式で予備に |
| 帯域 | 平常 ≈ 1–12 KB/s、上限を 14 KB/s（リンクの約 30 %）に制御。1 フレームを ~45 B の SysEx に分割して MIDI OUT のノートを挟めるように |
| ファーム費用 | フラッシュ +4〜5 KB、RAM +1.0〜1.3 KB、メインループで 1 フレームあたり ~0.1 ms。オーディオ ISR は一切触らない |
| 版の照合 | build.py が UI ハッシュ（ソース・生成ヘッダ・ビルドフラグ）を作り、ファームと WASM の両方に埋め込む。加えて実行時の「レイアウトハッシュ」（領域テーブルの offset/size）。どちらか不一致なら描画しない |
| 検証 | ① ネイティブ `ui_render` の PPM と WASM の出力のバイト一致 ② ホスト上のエンコーダ→デコーダ・ループバックでフレーム単位一致 ③ 実機デバッグビルドの「描画ハッシュ」照合 |

---

## 0.5 リリース目標: Felucca v1.5 と同時に公開

Felucca-mirror は **Felucca v1.5 と同時に公開**する（2026-10-05 決定）。v1.0 は現在リリース準備中なので、
ファームウェア側の変更はすべて v1.0 の後、v1.5 の開発中に入れる。

- **対応ファームウェアは v1.5 以降**。v1.0 には MIRROR がないので、ページは `INFO` の版を見て
  「Felucca v1.5 以降に更新してください」と表示する。
- **v1.5 のリリースビルドでは `FELUCCA_MIRROR` を既定で ON にする**（§9 Q1 はこれで決定）。リリース後に
  ファームの機能を後から足すことはできないので、v1.5 のファームウェアには、Web 側が後から使うものも
  含めておく:
  - `MIRROR_SUB` / `MIRROR_FRAME` / `MIRROR_KEY`、ライブ欄、LED・入力の欄、スコープ軌跡（M2）
  - `MIRROR_INPUT` の受け口（M5）。Web 側の操作 UI が v1.5 に間に合わなくても、ファームウェアには入れておく
  - プロトコル版 `proto = 1` を v1.5 で凍結する。以後の拡張は追加だけにする（未知のフィールドや part は無視）
- **v1.5 と同時に出すもの**: M0〜M4（画面、LED、鍵盤）と M7（リリース工程）。M5 の操作 UI と M6 の動画機能は
  Web 側だけで完結するので、v1.5 の後でも追加できる（間に合えば同梱）。
- **リリース工程**: v1.5 の `build.py` が `.fwsc` と同じビルドから `felucca_ui-<ui_hash>.wasm` を出し、manifest に
  `ui_hash` を記録する。v1.5.x のパッチリリースでも毎回 WASM を作る（UI ハッシュはソースが変われば変わる）。
- **予算の再計測**: 依頼時の空き（フラッシュ ~106 KB、RAM ~9 KB）は v1.0 準備前の値。v1.0 確定後と、
  v1.5 のほかの機能（USB オーディオ録音など）を入れた後に計り直し、MIRROR の +5 KB / +1.2 KB が収まるか確認する。

v1.5 に向けた順序:

| 時期 | 作業 |
| --- | --- |
| v1.0 リリースまで | このリポジトリだけで M0（WASM の描画一致）。ファームウェアには触れない |
| v1.5 開発の前半 | Felucca 側で M1（ホストのループバック）、M2（実機組み込み、負荷計測）。プロトコルをここで固める |
| v1.5 開発の後半 | M3・M4（Web MVP、LED・鍵盤）、M7（`build.py --mirror-wasm`、取り込みツール）、v1.5 の RC で T4・T5・T6 |
| v1.5 リリース | ファームウェアと対応 WASM、ミラーページを同時に公開 |
| v1.5 の後 | M5 の操作 UI、M6 の動画機能（ファームウェアの更新は不要） |

---

## 1. 前提として読んだこと（調査メモ）

- `felucca.c` が全ソースを 1 翻訳単位にまとめる。UI 部は `panel.c ui.c icons.c ui_graph.c ui_draw.c ui_menu.c ui_input.c ui_layer.c`（＋ `ui_name.c`, `ui_slice.c`, `favorites.c` が中から include）。
- LCD への出力は **`lcd_fill()` と `lcd_blit()` の 2 つだけ**（`gfx.c` はストリップ単位のキャンバス `cv_*` に描いて blit、フルフレームバッファなし）。ホストでは `ui_test.c` がこの 2 つを `host_screen[240*240]` に向けて差し替えている。→ WASM でも同じ差し替えで済む。
- 描画は**遅延描画**: `ui_draw()` は各要素の「前回描いたもの」（`ui.col[4][32]`, `ui.graph_sig/head_sig/foot_sig`, `ui.layer_sig`, `ui.menu_sig`, `nm.sig`, `ui.roll[]`, `proll`, `ts.meter[]` など）と比べて変わった所だけ描く。`ui.force` で全描画。
- メインループ（`main.c`）: `ui_input()` → `settings_poll()` → `ui_leds()` → `ui_draw()`、その後 15 ms まで `ui_input()` と `ed_service()` を回す（最大 ~60 fps）。
- 描画が読む**決定的でない**入力:
  - `fm1_ms`（点滅 250 ms、NAME のタップ、スロット一覧の 500 ms 更新、`draw` 中に値が進むこともある）
  - `ui.frame`（ロール数字のアニメーション、ピアノロールの追従、スコープの隔フレーム更新）
  - オーディオ ISR 由来: `trk[].seq_idx`（再生位置）、`trk[].peak`（ミキサーのメータ。**描画側が 0 に戻す**）、`scope_buf`/`scope_w`（HOME のオシロ、512 サンプル）、`song.cpu_q8`、`song.tick`、チェーン `chain.running/row`、モーション再生でのパラメータ変化
  - ADC: `song.batt_raw`（電池）、`song.master_q12`
  - USB 状態: `usb.config/suspended/up/setups/sof_seen`
  - フラッシュ: ユーザープリセット（`up_bank`、RAM ミラーあり）、プロジェクト 4 枠（`proj_slot`、SLOTS 画面は 500 ms ごとに解析）、ユーザーサンプル USR1–3（各 80 KiB、サンプル波形表示が IMA をデコード）、設定（パレット）
  - MIDI IN / エディタ由来の通知（`ui_notices` → `ui.msg`）
- ポインタを含む状態: `page_over`（`ui.layer == LAYER_SCL` のとき `&LY_SCL`）。キャッシュ内の `smp_zone_t *` 等。
- 乱数は UI 側には見当たらない（要確認: §9）。
- USB-MIDI: EP1 IN は TIMER5（2 kHz）からポーリング、ホストは 1 ms フレームに 1 パケット（64 B = 16 イベント = SysEx 48 B）。SysEx 送信キュー `sx_out_q` は 64 パケット（192 B）。`ota_wire_send()` はキューが空くまで**メインループを止めて待つ**（最大 200 ms）。SysEx 送信中は MIDI OUT のノートが出ない（「SysEx を先に、ノートで分割しない」）。
- 既存エディタプロトコル: コマンド 1–38（`ED_UI_STATE` 等まで）、64–71（MOTION, BACKUP, FM6）。`WATCH` + `PING` 1 s ごと、3 s 無通信で解除の仕組みがある。
- 予算（依頼文より）: フラッシュ空き ~106 KB、RAM 空き ~9 KB。

---

## 2. アーキテクチャの決定

### 2.1 描画: WASM に同じ C をコンパイル

- `core/mirror_view.c`（このリポジトリ）が `ui_test.c` と同じ方式で HAL をスタブし、`lcd_fill/lcd_blit` を 240×240 の RGB565 配列に向け、Felucca のソースを `#include` する（`hostsim.c` 経由で音源表も含む。表示に要るプリセット名・パラメータ記述・内蔵サンプルが入っているため）。
- ツールチェーン: **clang `--target=wasm32-unknown-unknown -nostdlib -O2` + `wasm-ld --gc-sections --export=…`**。Felucca は自前の `libc.c` を持ち malloc も FPU も使わないので、`memset/memcpy/memcmp` を数行足せば足りる。JS ランタイム不要で小さい（目安 300–700 KB、フォントと内蔵サンプル次第）。標準 C ライブラリが要る箇所が出たら emscripten（`/opt/homebrew/bin/emcc` あり）に切り替える。
- **ホスト (LP64) より WASM (ILP32) のほうが実機に近い**: `long` 32 bit、ポインタ 32 bit。残る差は `char` の符号（§8 リスク R2）。
- JS 側は WASM がエクスポートする画面バッファ（バイトスワップ済み RGB565）を RGBA に変換して `<canvas>` に `putImageData`、CSS `image-rendering: pixelated` で整数倍拡大。

### 2.2 何を送るか: 3 案の比較

| | (a) 入力イベントのロックステップ | (b) 状態スナップショット／差分 | (c) ハイブリッド（推奨: b が主） |
| --- | --- | --- | --- |
| 送るもの | 鍵盤・ボタン・ノブのエッジ＋初期状態 | UI が読む状態そのもの | (b) ＋毎フレームのライブ欄＋スコープ軌跡＋（表示用に）入力イベント |
| 帯域 | 最小（数十 B/s） | 中（差分なら 1–12 KB/s） | 中 |
| 正しさ | **成り立たない**。UI はオーディオ ISR（シーケンサ位置、アルペジオ、モーション再生、録音で書かれるステップ）、MIDI IN、エディタの SysEx、ADC、フラッシュ内容、ボタン長押し時間（`fm1_ms`）に依存。WASM 側で DSP まで含めサンプル単位で同じ時刻に同じことを起こす必要があり、1 回ずれると**永久にずれる** | 受けた状態から描くので、ずれは次の差分／キーフレームで**自己修復** | (b) と同じ |
| ファーム改修 | 入力フックは小さいが、決定性を保証するための改修が巨大 | 領域テーブル＋エンコーダ | 同左＋スコープ軌跡の取り出し |
| 結論 | 不採用（ただし入力イベントは「表示用」として (c) で再利用） | 採用の土台 | **採用** |

### 2.3 状態レプリケーションの方式

**領域テーブル** `mirror_regions.h`（X マクロ、ファームと WASM の両方が include）:

```c
/* MR(id, base, size)   レプリケートする RAM 範囲
 * ML(id, field)        その範囲内で「受信側の値を保つ」フィールド（描画キャッシュ） */
MR(UI,     &ui, sizeof ui)
  ML(UI,   col) ML(UI, graph_sig) ML(UI, head_sig) ML(UI, foot_sig) ML(UI, frame)
  ML(UI,   roll) ML(UI, roll_bpm) ML(UI, layer_sig) ML(UI, menu_sig)
MR(SONG,   &song, sizeof song)          /* cpu_q8 / batt_raw / master_q12 はライブ欄で上書き */
MR(TP0,    trk[0].p, sizeof trk[0].p)   /* 各トラック: p[], step[], engine..user, held/nheld/latched, … */
MR(TS0,    trk[0].step, sizeof trk[0].step)
...
MR(NAME,   &nm, sizeof nm)  ML(NAME, sig)
MR(CHAIN,  &chain_config, …) MR(CHAINR, &chain, …)
MR(MOTION, motion stores …) MR(FAV, &favorites, …) MR(LAYER, &lys, …) MR(SLICEUI, …)
MR(PROJ,   proj_name / proj_cur / pat_last …)
```

- `track_t` は UI 状態とオーディオ専用状態（`v[NVOICE]`、フィルタ、LFO 位相など ~2 KB/トラック）が混在するので、**UI が読むメンバだけ**を範囲として列挙する。ボイス状態は送らない。
- 合計は概算 **5.5〜6.5 KB**（`ui` ~0.45 KB、`p[]` 182 B×4、`step[]` 768 B×4、モーション 260 B×4、その他）。実測は M1 で。
- **差分検出**: 範囲を 16 B ブロックに切り、各ブロックの 16 bit ハッシュ（FNV-1a を畳む）を `mr_hash[]` に保持（~400 ブロック × 2 B ≈ **0.8 KB RAM**）。シャドウコピーは持たない（RAM が足りない）。ハッシュが変わったブロックを送り、送れたら `mr_hash` を更新。送れなかったものは次フレームへ繰り越し。
- **巡回リフレッシュ**: 毎フレーム 2 ブロックを変化に関係なく送る（~400 ブロックを ~3 s で一巡）。ハッシュ衝突、SysEx 取りこぼし、ISR とのティアリング（録音中のステップ書き込み等）を自己修復する。
- **キーフレーム**: 購読開始時と、受信側からの要求（抜け検出）で全ブロックを順に送る（~7 KB、~0.6 s）。受信側は完了まで「同期中」表示。
- **描画キャッシュ（ML）は受信側の値を保持**。理由: 送信側のキャッシュは「送信側が描いたもの」を表す。受信側がフレームを取りこぼしていた場合にそれを上書きすると、描いていない要素を「描いた」と思い込んで古いピクセルが残る。キャッシュを各自で持てば、同じフレーム列を描く限り両者のキャッシュは一致する。
- **取り込み時点**: メインループの `ui_draw()` の**直前**（`mirror_capture()`）。`ui_draw()` が `msg_t--`、`hot_t--`、`force = 0`、`t->peak = 0` などを書き換えるので、「描画前の状態」を送れば受信側の `ui_draw()` が同じ書き換えを再現する。
- **ポインタ**: レプリケート範囲に入れない。`page_over` は受信側で `ui.layer` から再計算（`mirror_fixup()`）。ほかに見つかれば同様に index へ変換。

**ライブ欄**（毎フレーム、範囲外の値・ISR 由来の値）: `frame`（`ui.frame` の下位 14 bit）、`fm1_ms`（下位 21 bit）、`song.playing/tick`、`trk[i].seq_idx`・`arp_note`、`trk[i].peak`（4×u16）、`cpu_q8`、`batt_raw`、`master_q12`、USB 状態ビット、`page_over` の種類、`fm1_led[]`（11 B、フェーズ 2）、`fm1_in.notes/buttons`（フェーズ 2、表示用）。約 25〜45 B。

**スコープ軌跡**: 512 サンプルの `scope_buf` を送るのは重い（1 KB/フレーム）。`graph_scope()` が実際に使うのは「トリガ後の 240 点を `46/peak` で縮めた y 値（0..122、7 bit に収まる）」だけなので、`ui_graph.c` を「軌跡を計算する関数」と「軌跡から描く関数」に分け、**y 値 240 個（差分符号化で ~120 B）**を送る。HOME のスコープ表示中かつスコープが再描画されるフレーム（元から隔フレーム）だけ。WASM では計算関数をフックで置き換えて受信値を使う。

**時刻の扱い**: `fm1_ms` はライブ欄の値で固定してから `ui_draw()` を呼ぶ。実機では描画中に `fm1_ms` が進み、点滅の境界でまれに差が出る（R4）。フェーズ 1b で `ui_draw()` 内の `fm1_ms` 参照をフレーム頭で確定した `ui_now` に置き換えることを提案（実機の見た目は変わらない）。

**フラッシュ由来のデータ**:
- ユーザープリセット・プロジェクト・設定・お気に入り: 購読開始時に既存の **`BACKUP_LIST` / `BACKUP_GET`（65–66）** で取得し、WASM の RAM 版ストア（ホストビルドの `FELUCCA_FLASH=0` 経路、`up_bank` / `proj_slot`）に流し込む。変化は `up_gen` やセーブ時に MIRROR フレームの「ストア世代」欄で通知 → ページが該当オブジェクトだけ再取得。
- ユーザーサンプル（240 KiB）: SAMPLE/SLICE の波形表示を開いたときだけ遅延取得（~6 s）。取得中は LCD の外にバッジを出し、その間はピクセル一致を保証しない（R6）。

**抜けとピクセル一致の約束**:
- 取りこぼしなし: **全フレーム一致**（アニメーション含む）。
- 取りこぼし（`frame` の飛び）: 受信側は強制全描画（`ui.force = 1` ＋ローカルキャッシュ初期化）。実機の全描画と同じ静止画になる。ロール数字やメータ減衰などの過渡だけが最大 `ROLL_FRAMES` 程度ずれうる。

### 2.4 転送

| | SysEx over USB-MIDI（推奨） | CDC + Web Serial |
| --- | --- | --- |
| 生帯域 | 48 B/ms（1 ms に 1 パケット）≈ 48 KB/s、7 bit 化で ~42 KB/s 実効 | 64 B/ms ≈ 64 KB/s、8 bit そのまま |
| ブラウザ | Chrome / Edge / Firefox（SysEx 許可）。Safari 不可 | Chrome / Edge のみ。毎回ポート選択 |
| 共有 | Web エディタと同じポート・同じ許可。エディタ画面に統合も可能 | コンソールと共用 → モード切替が要る |
| 副作用 | SysEx 送信中は MIDI OUT のノートが待つ（R3） | MIDI に影響しない |

- **MIRROR_* コマンド**（既存 `F0 7D 46 4C` の空き番号。72 以降を仮置き、実装時に `editor.c` の enum と照合）:
  - `72 MIRROR_SUB` 要求: `on`(0/1), `proto`, `flags`（bit0 LED/入力, bit1 スコープ, bit2 操作許可, bit3 描画ハッシュ）, `max_rate`（fps） / 応答: `proto`, `FELUCCA_VERSION`, `FELUCCA_ID`, `ui_hash`（u64 を 10×7 bit）, `layout_hash`（u32）, 領域ブロック数, パレット番号, ビルドフラグ（SLICE, FM4, ICONS）
  - `73 MIRROR_FRAME`（プッシュ）: `seq`(7 bit), `part`, `last`、最初の part にライブ欄、続いて `[block idx (2B)][pack7 16 B]…`、任意でスコープ軌跡、任意で描画ハッシュ（u32）
  - `74 MIRROR_KEY` 要求: キーフレーム再送（全体、または範囲）
  - `75 MIRROR_INPUT`（フェーズ 3）: Web → 機器の入力イベント
  - 購読は `WATCH` と同じく `PING` 1 s ごと、3 s 無通信で自動停止。
- **1 フレームを ~45 B の SysEx（1 USB パケット）単位に分割**。SysEx の切れ目でノート（`midi_out_q`）が出られるので、MIDI OUT の遅延は最大 ~1 ms 程度に抑えられる見込み。
- **送信は非ブロッキング**: MIRROR 専用の小さな出力リング（~320 B）に組み立て、メインループの待ち（`while (fm1_ms - m < 15u)`）の中で `sx_out_q` に空きがある分だけ流す。`ota_wire_send()` を使うエディタ応答と**同じ SysEx の途中で混ざらない**よう、`ota_wire_send()` の先頭で「送信途中の MIRROR メッセージを完結させる」（最大 1 メッセージ ≈ 1 ms）。
- **レート制御**: 1 フレームの上限 ~380 B、毎秒の上限 ~14 KB。超える分は繰り越し（変化したブロックはハッシュ未更新のまま残るので自然に後で送られる）。`max_rate` で 30 fps 等に落とせる。
- CDC 版: 同じペイロードを COBS でフレーミングし 8 bit のまま送る（pack7 不要）。M2 で SysEx の帯域が不足と判明した場合のみ実装。

**帯域の見積もり**（M1 で実測して更新する）:

| 場面 | 1 フレームあたり | 60 fps 換算 |
| --- | --- | --- |
| 何もしていない | ライブ欄 ~30 B（変化なしなら 10 Hz の心拍だけ） | ~0.3–2 KB/s |
| ノブを回している | ライブ欄 + 4–6 ブロック×21 B ≈ 150 B | ~9 KB/s |
| 再生中 SEQ/STEP 画面 | ライブ欄（再生位置）＋時々ブロック ≈ 40–80 B | ~3–5 KB/s |
| HOME スコープ | +~120 B を隔フレーム | +~3.6 KB/s |
| ページ切替・プリセット読込 | 数十ブロック、上限で数フレームに分散 | 一時的に上限 14 KB/s |

### 2.5 ファームウェア側のコスト

| 項目 | 見積もり |
| --- | --- |
| CPU | ハッシュ ~6 KB/フレーム ≈ 2 万サイクル未満 + pack7 と組み立て → **~0.1 ms/フレーム**（メインループのみ）。購読していないときはほぼゼロ（フラグ確認だけ） |
| オーディオ ISR | **変更なし**。ISR が書く値は読むだけ（16/32 bit の整列読み、ティアリングは巡回リフレッシュで修復）。TIMER5 の `usb_poll` も変更なし |
| RAM | ハッシュ ~0.8 KB + 出力リング ~0.32 KB + 状態 ~0.05 KB ≈ **1.2 KB**（9 KB の空き内）。16 bit ハッシュを 8 bit にすれば −0.4 KB（衝突は巡回で修復） |
| フラッシュ | **+4〜5 KB**（§3 の内訳） |
| MIDI OUT | 購読中、ノートが最大 ~1 ms 遅れうる（R3）。M2 で計測 |

---

## 3. ファームウェアの変更（ファイル別、Felucca 側で別途作業）

> Felucca 側の変更は**このリポジトリでは行わない**。下記は Felucca 側のセッションに渡すための仕様。すべて `FELUCCA_MIRROR` で囲む。開発中の既定は 0、M2 の実機検証を通ったら 1 にして、v1.5 のリリースには ON で入れる（§0.5）。

| ファイル | 変更 | フラッシュ | RAM |
| --- | --- | --- | --- |
| `src/mirror_regions.h`（新規） | 領域テーブル（MR/ML の X マクロ）。ファームと WASM が共有。`layout_hash` はここから実行時に計算（各 MR の size、各 ML の offsetof/sizeof） | ~0.4 KB（テーブル） | 0 |
| `src/mirror.c`（新規） | 購読状態、`mirror_capture()`（ハッシュ比較、ライブ欄、フレーム組み立て）、`mirror_pump()`（非ブロッキング送出）、キーフレーム、巡回リフレッシュ、レート制御、`MIRROR_*` の処理。`#ifdef MIRROR_DECODER` でデコーダ（`mirror_apply()`, `mirror_fixup()`）も同じファイルに置き、WASM とホストテストが使う | ~2.5–3 KB | ~1.2 KB |
| `src/felucca.c` | `editor.c` の後に `#include "mirror.c"`、`FELUCCA_MIRROR` の既定 | 0 | 0 |
| `src/editor.c` | enum に MIRROR 番号、ディスパッチで `mirror_handle()`、`PING` で購読延長、`INFO` のタグブロックに MIRROR 能力（`4D` 系と衝突しないタグ） | ~0.1 KB | 0 |
| `src/usb.c` | `sx_room()`（`SXQ - (so_w - so_r)`）を追加。`ota_wire_send()` 冒頭で `mirror_finish_msg()` | ~0.1 KB | 0 |
| `src/main.c` | `ui_draw()` の直前に `mirror_capture()`、15 ms 待ちループに `mirror_pump()` | ~0.05 KB | 0 |
| `src/ui_graph.c` | `graph_scope()` を `scope_trace(int8_t y[240])` と描画に分割。`SCOPE_TRACE_HOOK` で WASM が置き換え。`mirror_capture()` はスコープ表示中のみ `scope_trace()` を呼ぶ（描画と二重計算になるので、軌跡を static に保持して描画側で再利用） | ~0.1 KB | +240 B（軌跡保持。描画の `snap[512]` 1 KB を 240 B に減らせればむしろ −0.8 KB） |
| `src/ui_draw.c` ほか（フェーズ 1b、任意） | `ui_draw()` 中の `fm1_ms` 参照をフレーム頭で確定した `ui_now` に | ~0 | 4 B |
| `src/lcd.c` | デバッグビルド（`FELUCCA_MIRROR_HASH=1`）のみ `lcd_fill/lcd_blit` に描画ハッシュのフック。リリースでは 0 B | 0（リリース） | 0 |
| `src/console.c` | `mirror` コマンド: 購読状態、送信 B/s、繰り越しブロック数、最大フレーム長、取りこぼし | ~0.2 KB | 0 |
| `build.py` | `FELUCCA_MIRROR` オプション。**UI ハッシュ**を計算して `-DFELUCCA_UI_HASH=0x…` を渡し、manifest に `ui_hash` を記録。`--mirror-wasm` で同じソース・同じ生成物・同じフラグから `felucca_ui-<hash>.wasm` も出力 | — | — |
| `web/EDITOR_PROTOCOL.md` | MIRROR 節を追加（正本はファーム側に置く） | — | — |
| `host/mirror_test.c`（新規） | §7 のループバック試験を `run_tests.sh` に組み込む | — | — |
| `host/target_budget.*` | `mirror_capture` のコストを予算表に追加 | — | — |

合計: **フラッシュ +4〜5 KB、RAM +1.0〜1.3 KB**。

---

## 4. Web 側

### 4.1 このリポジトリの構成（予定）

受信と描画の中身はプラットフォームに依存しない C の「コア」にまとめ、Web・スマホ・M5Stack は
その外側の薄い層にする（§10）。

```
Felucca-mirror/
  core/
    mirror_view.c      HAL スタブ、lcd_fill/lcd_blit → screen[]、Felucca ソースの include、
                       mv_* の API（apply / render / screen / leds / resync）。OS も libc も使わない
    mirror_view.h      各プラットフォームが使う API
    shim_libc.c        memset/memcpy/memcmp（freestanding 用）
  wasm/
    build.sh           FELUCCA_SRC=…/felucca  GEN=…/build/felucca/gen を受け取り clang で core を wasm に
  web/
    index.html         ミラーページ（単体）
    mirror.js          WebMIDI 接続、MIRROR_SUB/PING、SysEx 再組立て、BACKUP 取得
    renderer.js        WASM 読み込み、ハッシュ照合、apply → render → canvas
    panel.js           フェーズ 2: LED・鍵盤・ボタン・ノブの描画（SVG）
    record.js          フェーズ 4: 録画、.fmr 記録・再生
  test/
    parity_render.mjs  ui_render の全シーンを WASM で描き PPM と比較
    loopback.c         エンコーダ → バイト列 → デコーダ（別プロセス）→ 画面比較
  platforms/           （v1.5 の後、§10）
    m5stack/           ESP-IDF プロジェクト: USB ホスト MIDI → core → LCD
    ios/               ネイティブアプリ: CoreMIDI → core → 画面
    android/           （必要になったら。まずは Chrome の WebMIDI で Web 版を使う）
  tools/
    import_release.py  build.py が出した wasm と manifest を site/wasm/ に取り込む
  site/wasm/index.json { "ui_hash": { "version": "...", "file": "..." } }
```

Felucca のソースはこのリポジトリにコピーしない。ビルド時にパス（`FELUCCA_SRC`）で参照する。リリース用 WASM は Felucca の `build.py` が作ったものを取り込む（ファームと**同じビルドから**出ることを保証するため）。

### 4.2 コアの API（WASM ではそのままエクスポート）

`mv_init()`, `mv_ui_hash()`, `mv_layout_hash()`, `mv_apply(ptr, len)`（MIRROR_FRAME の part をデコード）, `mv_store(obj, ptr, len)`（BACKUP オブジェクト投入）, `mv_render()`（`fm1_ms`・`ui.frame` をセットして `ui_draw()`）, `mv_screen()`（240×240 RGB565 のポインタ）, `mv_leds()`, `mv_resync()`。
加えて、描画のたびに画面全体ではなく**変わった矩形**を知らせるコールバック（`lcd_fill/lcd_blit` の呼び出しそのもの）を任意で受けられるようにする。M5Stack では、これを LCD へそのまま流せばよい。

### 4.3 ページの流れ

1. 「接続」→ `navigator.requestMIDIAccess({sysex: true})` → Felucca のポートを選ぶ
2. `INFO`（エディタと同じ）→ `MIRROR_SUB on` → 応答の `ui_hash` で `site/wasm/index.json` を引き、WASM を取得
3. WASM の `ui_hash` / `layout_hash` と照合。不一致なら「このファームウェア (vX) 用のミラーがありません」と表示して**描画しない**。開発用に WASM ファイルのドラッグ＆ドロップ読込
4. `BACKUP_LIST/GET` でストアを取得 → キーフレーム受信 →「同期中」解除
5. 以後 MIRROR_FRAME ごとに apply → requestAnimationFrame で最新フレームを render（受信が描画より速いときも各フレームの apply と render は 1:1 で行い、canvas への転送だけ間引く）
6. 1 s ごとに `PING`。`seq` の飛びで `mv_resync()`、長く飛んだら `MIRROR_KEY`

### 4.4 Windows の注意

Windows の従来 MIDI ドライバではポートが排他になり、エディタとミラーを別タブで同時に開けない。→ ミラーをエディタ画面のパネルとしても組み込めるよう、`mirror.js` は既存エディタの SysEx 受信ループに差し込める作りにする（§9 Q2）。

---

## 5. フェーズとマイルストーン

| # | 内容 | 完了条件 |
| --- | --- | --- |
| **M0** スパイク（ファーム改修なし） | `ui_render.c` のシーン群を wasm32 でビルドし node で実行 | ネイティブ `ui_render` の PPM と**全シーン・全パレットでバイト一致**。WASM サイズ・描画時間の記録 |
| **M1** プロトコルとループバック（ホストのみ） | `mirror_regions.h` / `mirror.c`（エンコーダ＋デコーダ）を Felucca 側ブランチで作成、`host/mirror_test.c` | 台本どおりの UI 操作（全ページ、メニュー、確認、NAME、レイヤ、再生、録音）で受信側画面が**全フレーム一致**。取りこぼし注入時は強制全描画後に一致。帯域とブロック数の実測表 |
| **M2** 実機組み込み（`FELUCCA_MIRROR=1` の試験ビルド） | SysEx 経路、非ブロッキング送出、スコープ軌跡、コンソール統計 | オーディオ ISR の負荷（`cpu_q8`）が購読前後で変わらない、音切れなし、MIDI OUT 遅延 ≤ 1–2 ms、帯域 ≤ 14 KB/s、描画ハッシュ照合で静止画面の不一致 0 |
| **M3** Web MVP | 単体ページ: 接続、照合、同期、表示（整数倍、pixelated） | 実機と並べて目視・描画ハッシュで一致。30 分連続で切断・ずれなし |
| **M4** LED と鍵盤 | `fm1_led[]`・`fm1_in` を描く本体パネル（SVG、`panel.c` の配置から） | 押鍵・LED が遅れなく表示 |
| **M5** Web からの操作（任意） | `MIRROR_INPUT`: 鍵盤・ボタン・ノブのイベントを実機の入力経路に注入（`fm1_input_edges` / `fm1_enc_take` に加算）。購読フラグで明示的に許可したときのみ、実機の物理入力が常に優先 | 操作が実機の操作と同じ結果になる（状態は実機が持つのでずれない） |
| **M6** デモ動画 | クリーン表示（UI なし・単色/クロマ背景・枠あり/なし）、×1–×8 拡大、`canvas.captureStream(60)` + MediaRecorder で WebM、**.fmr 記録**（MIRROR フレーム列＋時刻をそのまま保存）→ オフラインで 1 フレームずつ再描画して PNG 連番（取りこぼしのない映像を後から作れる）、押鍵オーバーレイ | 60 fps の素材が作れる |
| **M7** リリース工程 | `build.py --mirror-wasm`、`tools/import_release.py`、`site/wasm/index.json`。公開先は別途相談 | リリースごとに対応 WASM が並ぶ |

M0 と M1 はファームウェアに触れずに進められ、方式の正否（特に R1, R2）をここで判定する。

---

## 6. 版と同期

- **UI ハッシュ**（build.py）: `felucca/src/**`、`build/felucca/gen/*.h`（フォント・アイコン・キーキャップ・パレット・表）、ビルドフラグ（`FELUCCA_SLICE/FM4/ICONS/…`）、コンパイラの `char` 符号設定を連結した SHA-256 の先頭 64 bit。粗いが安全（ソースのどこかが変われば別物扱い）。ファームに `FELUCCA_UI_HASH` として埋め込み、manifest にも書く。
- **レイアウトハッシュ**（実行時）: `mirror_regions.h` の各範囲の size、各 ML の offset/size、ブロック数、`sizeof(track_t)` 等からファームと WASM がそれぞれ計算。構造体レイアウトのコンパイラ差（R2）を検出する。
- **エンジン順・パレット順**は同じソース・同じ生成物から作られるので UI ハッシュで担保される。
- **配布**: `site/wasm/<ui_hash>/felucca_ui.wasm` と `index.json`（`ui_hash → version, file, sha256`）。ページは機器の応答で引く。古いリリース分も残す。
- **プロトコル版** `proto` は MIRROR_SUB の要求・応答の両方に入れ、ページは自分が扱える範囲でなければ停止。

---

## 7. 検証

1. **T1 描画の同一性（M0）**: `ui_render` のシーン（全ページ×全エンジン×全パレット、ロール数字のフィルムストリップ、FM6 チャート）を WASM でも描き、ネイティブの PPM と**バイト比較**。PNG 化は既存の `ui_render.py` を流用し、差分があれば差分画像を出す。
2. **T2 ループバック（M1）**: `hostsim` / `ui_test` の操作台本を「送信側」として実行し、各フレームで `mirror_capture()` の出力をファイルに書く。**別プロセス**（アドレスが異なるのでポインタ混入が露見する）の「受信側」が `mirror_apply()` → `ui_draw()` し、両者の `host_screen` を毎フレーム比較。取りこぼし・遅延・並べ替え・ハッシュ衝突（ハッシュを故意に弱くする）を注入して、収束までのフレーム数を測る。
3. **T3 WASM ループバック**: T2 の受信側を node 上の WASM に差し替え（ホスト LP64 と WASM ILP32 の差を検出）。
4. **T4 実機の描画ハッシュ（M2）**: デバッグビルドで `lcd_fill/lcd_blit` の（矩形＋画素）列の CRC32 を毎フレーム MIRROR_FRAME に載せ、WASM 側でも同じ CRC を計算。不一致を数えてページの開発パネルに出す。実機には画面バッファがないので、これが実機での唯一の機械的な一致確認になる。
5. **T5 負荷**: 購読あり／なしで `cpu_q8`、ui フレーム時間、MIDI OUT のノート遅延（ループバックケーブルで計測）、SysEx 帯域（コンソールの `mirror`）。
6. **T6 長時間**: 30 分の連続再生・操作で切断・ずれ・メモリ増加がないこと（ページ側）。

---

## 8. リスク

| # | リスク | 対策 |
| --- | --- | --- |
| R1 | レプリケート範囲の漏れ（UI が読むのに送っていない状態） | T2 のループバックで全画面・全操作を網羅して検出。`ui*.c` が参照するグローバルを grep した一覧をテーブルの横にコメントで保守 |
| R2 | コンパイラ差: `char` の符号、構造体の詰め方、符号付きシフト、未定義動作の最適化差 | 実機ツールチェーンの `char` 符号に WASM 側を合わせる（`-fsigned-char`/`-funsigned-char`）、レイアウトハッシュ、T1 のバイト一致 |
| R3 | MIRROR の SysEx が MIDI OUT のノートを遅らせる | 1 USB パケット単位に分割、帯域上限、購読中のみ。M2 で計測し、不足なら CDC 転送へ |
| R4 | 描画中に `fm1_ms` が進む・ISR が状態を書き換える（ティアリング） | `ui_now` 導入（フェーズ 1b）、巡回リフレッシュで自己修復 |
| R5 | 描画キャッシュの扱いを誤ると古いピクセルが残る | ML（受信側保持）＋抜け検出時の強制全描画。T2 で取りこぼし注入 |
| R6 | ユーザーサンプル波形は大きなデータに依存 | 遅延取得、取得中は一致保証外と明示 |
| R7 | RAM 9 KB 空きの中で +1.2 KB | ハッシュを 8 bit に、出力リングを縮小、スコープの `snap` 削減で相殺 |
| R8 | `ota_wire_send()` が MIRROR のメッセージ完結待ちで最大 ~1 ms 余分に待つ | 既存の 200 ms タイムアウト内。エディタの応答遅延として許容範囲か M2 で確認 |
| R9 | OTA 中・UBOOT 画面・クラッシュ画面・起動時の `panel_setup` は `ui_draw()` 外で描かれる | ミラーは対象外とし、ページに「更新中」等を表示（ライブ欄の状態コードで通知） |
| R10 | 将来の UI 変更でテーブル更新を忘れる | `run_tests.sh` に T2 を常設。レイアウトハッシュで不整合は必ず検出される |

---

## 9. 未決事項（相談したいこと）

- ~~**Q1** リリースビルドで `FELUCCA_MIRROR` を既定 ON にするか~~ → **v1.5 から既定 ON**（§0.5）。
- **Q2** ミラーを単体ページにするか、Web エディタに統合するか（Windows のポート排他を考えると統合が楽）。両対応を想定して設計する。
- **Q3** プロトコル文書の正本を Felucca 側（`web/EDITOR_PROTOCOL.md`）に置くことでよいか。
- **Q4** 公開先（hugelton のサイト / GitHub Pages など）と時期。WASM の配布は GPL の対応ソース提供（公開リポジトリ hugelton/Felucca 側のソースとの対応）と、フォントのライセンス表示（OFL / MIT）をページに載せる前提。
- **Q5** ページ上の表記: 機器のブランド名・ロゴは使わない方針でよいか（本体パネル図も一般的な形状で描く）。
- **Q6** ユーザーサンプル波形のミラーを MVP から外してよいか。
- **Q7** UI 側に乱数・時刻以外の非決定要素がないかの最終確認（M1 の T2 で検出する想定）。
- **Q8** 実機ツールチェーンの `char` 符号と、構造体レイアウトに影響するオプションの確認。

---

## 10. 今後: WASM・スマホ・M5Stack で動かす（計画のみ、v1.5 の後）

### 10.1 考え方

この方式では、機器から来るのは「状態」で、画面を作るのは受信側の C コードです。描画コードは整数演算だけで
OS にも依存しないので、**C が動くところならどこでも同じ画面が出せます**。そこで、受信と描画を `core/` に
まとめ、プラットフォームごとの違いは「どこからバイト列が来るか」と「どこへ画素を出すか」だけにします。

```
  [FM-1] --USB-MIDI--> 受け口 (WebMIDI / CoreMIDI / USB ホスト / 中継)
                         |
                         v
                     core (C) : mv_apply -> mv_render -> 240x240 RGB565
                         |
                         v
                     出力 (canvas / UIKit / ESP32 の LCD)
```

v1.5 で決めるプロトコルと `core/` の API はこの展開を前提にしておく（後から変えない）:
- `core/` は libc・OS・浮動小数点に依存しない。メモリ確保は静的のみ
- バイト順はリトルエンディアン前提（WASM、ARM、ESP32 はすべてそう）。画素は FM-1 の LCD と同じバイトスワップ済み RGB565
- 1 フレームの受信バッファは 512 B 未満、キーフレームも分割で受ける（小さなマイコンで受けられるように）

### 10.2 WASM（ブラウザ）

v1.5 の本命。デスクトップの Chrome / Edge / Firefox。加えて:
- **中継モード**: USB をつないだ PC のページが、受けた MIRROR フレームを WebSocket（または WebRTC）でそのまま
  転送する。見る側は同じページで、MIDI なしで描ける。スマホ・タブレット・別の PC・配信用 PC で見られる。
  状態だけを送るので、同じ LAN なら帯域は問題にならない
- **.fmr の再生**（M6）: 記録した状態ストリームをどこでも再生できる

### 10.3 スマホ

| | Android | iPhone / iPad |
| --- | --- | --- |
| 直接つなぐ | Chrome は WebMIDI に対応。USB-C（OTG）で FM-1 をつなげば **Web 版がそのまま動く見込み**（要実機確認: SysEx の許可、給電） | Safari に WebMIDI がない。**ネイティブアプリが必要**: CoreMIDI（USB クラスコンプライアント MIDI は iOS が標準で認識）で受けて、`core/` を C のままリンクして描く |
| 中継で見る | Web 版の中継モード | Web 版の中継モード（Safari で WASM は動く。MIDI は不要） |

- iOS アプリは Swift の薄い層 + `core/`（C）。画面は 240×240 のバッファを `CGImage`（または Metal のテクスチャ）で整数倍表示
- **ライセンス上の注意**: GPL と App Store の配布条件の相性は過去にも問題になっている。App Store に出すかは別途検討
  （TestFlight や自前ビルドにとどめる、中継モードで足りるならアプリを作らない、なども選択肢）
- 画面の向き・ノッチ・ダークモードはページ／アプリ側の枠だけの話で、240×240 の中身には影響しない

### 10.4 M5Stack 系

「小さい外部モニタ」として FM-1 の横に置く用途。

- **第一候補: CoreS3**（ESP32-S3、320×240 の LCD、USB-C が OTG ホストになれる）。240×240 は 320×240 の中央に等倍で
  そのまま入る。左右の 40 px に LED や再生状態を出せる
- 大きい画面の機種（ESP32-P4 系のタブレット型など）は ×2 / ×3 の整数倍で表示
- **受け口**: ESP-IDF の USB ホストで FM-1 の USB-MIDI（クラスコンプライアント）を開き、SysEx を組み立てて
  `mv_apply()` へ。BACKUP の取得や PING も同じ経路で送る
- **出力**: `core/` の `lcd_fill/lcd_blit` コールバックを、そのまま LCD の矩形転送（`esp_lcd` / M5GFX の pushImage）へ。
  画素形式は FM-1 と同じなので変換なし。実機と同じ遅延描画になるので SPI の帯域も小さくて済む
- **容量**: `core/` に入るのはフォント・アイコン・パレット・パラメータ表・プリセット名など。UI に要らない DSP の
  コードは `--gc-sections` で落とす。内蔵サンプル（波形表示用）を含めても ESP32-S3 のフラッシュ（8〜16 MB）に収まる見込み。
  RAM は Felucca の UI 状態＋キャンバス（240×124×2 B ≈ 60 KB）程度で、内部 SRAM か PSRAM に収まる
- **未確認**: CoreS3 が OTG ホストとして FM-1 に給電できるか（FM-1 は電池があるが、VBUS がないと USB を
  つながない可能性がある）、ESP-IDF の USB ホストで MIDI クラスを扱う手間、Wi-Fi で中継モードの受け手になる案
- 将来的には、M5Stack のボタン・ダイヤルで `MIRROR_INPUT` を送る「リモコン」にもできる

### 10.5 そのために Felucca 側で v1.5 までにやっておくこと

- UI に要るデータ（エンジンの記述・プリセット名・パラメータ表・内蔵サンプル）と DSP コードを、`#include` の
  単位で分けられるようにしておく（`core/` が音源のコードを丸ごと取り込まなくて済むように）。必須ではないが、
  M5Stack でのサイズとビルド時間が大きく変わる
- `mirror.c` のデコーダ部分を `core/` からそのまま使えるようにする（デコーダに OS 依存を入れない）

### 10.6 さらに先: Felucca そのものを動かす（参考）

ホストの `hostsim.c` は音源まで含めて全体をコンパイルできるので、同じ方法で **Felucca 全体を WASM や
スマホで動かす**（AudioWorklet で発音、画面の鍵盤で演奏）ことも技術的には可能です。ただしこれはミラーとは
別の製品になるので、この計画には含めません。やるなら別リポジトリで検討します。

---

## 11. 次の一歩

1. M0: `core/mirror_view.c` と `wasm/build.sh` を書き、`ui_render` のシーンを WASM で描いて PPM 比較（Felucca のソースは読むだけ）。
2. M0 の結果で WASM 方式の正否と R2 を確定してから、M1 の Felucca 側作業（`mirror_regions.h` / `mirror.c` / `host/mirror_test.c`）を依頼する。
