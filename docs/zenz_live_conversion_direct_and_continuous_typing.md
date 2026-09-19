# Zenz ライブ変換の直接表示と追加入力合成表示の仕様・設計書

本書は、mozkey におけるローカル AI（Zenz）ライブ変換の改善要求、その技術的背景、設計方針、および実装詳細について説明するドキュメントです。

---

## 1. 概要と背景

mozkey は Google Mozc をベースとした日本語入力システムであり、通常の辞書・統計変換に加えてローカル AI（Zenz）による高精度な文脈変換、およびタイピング中のデバウンス時間後に自動で変換を行う「ライブ変換」機能を備えています。

従来の Zenz ライブ変換では、以下の2つのユーザー体験上の課題が存在していました：

1. **二段階変換による画面のチラつき（Mozc通常変換の一瞬表示）**:
   ユーザーが文字を入力しデバウンス時間を経過すると、まず従来の Mozc 通常変換（第1候補漢字）が画面に表示され、その後の追加ディレイ（1000ms等）経過後に Zenz の推論結果で上書きされる挙動になっていたため、画面の漢字が一瞬切り替わるチラつきが発生していました。
2. **追加入力時の AI 変換結果の破棄**:
   AI による変換結果（例：「今日」）が表示された後、続けて次の文字を入力（例：'h', 'a'）すると、直前の AI 変換結果が画面から消去され、入力途中のローマ字やひらがなに戻ってしまい、タイピングの流れが途切れていました。

---

## 2. 要求事項

| No | 要求内容 | 目的・期待される効果 |
|:---|:---|:---|
| **要求 1** | **最初から AI（Zenz）結果を直接表示**<br>ライブ変換時に従来の Mozc 通常変換（第1候補漢字）を画面に表示させず、最初から AI による変換結果を直接表示する。 | 通常変換のチラつきを根絶し、スムーズで安定した AI ライブ変換体験を実現する。 |
| **要求 2** | **追加入力時の直前 AI 結果保持と合成表示**<br>AI による変換結果（例: 「今日」）が表示された後、続けてキー（'h', 'a' など）を入力した際、直前の AI 変換結果の表示を保ったまま追加入力を「今日h」「今日は」と追加で表示し続ける。タイピング停止後（デバウンス時間経過後）に AI 予測が行われた時点で再度変換を更新する。 | 流れるような連続タイピングを可能にし、文節や単語を意識せず思考の速度で入力できるようにする。 |
| **要求 3** | **CI（GitHub Actions）の Windows 限定化**<br>Push 時に Windows 以外のビルド（Linux, macOS, Android, Docker Lint）を実行せず、Windows 関連のビルド・チェックのみを実行するようにする。 | 不要なプラットフォームのビルド時間を削減し、迅速な Windows ビルド確認を可能にする。 |

---

## 3. 技術的分析と原因

### 3.1 通常変換がチラついていた原因
- 従来の `Session::MaybeStartLiveConversion` では、Mozc の推論結果を `Output(command)` でそのまま画面（クライアント UI）に出力していました。
- その後 `MaybeScheduleZenzLiveCorrection` を呼び、1000ms の追加ディレイを待ってから非同期で Zenz プロセスへリクエストを送信していました。
- そのため、「デバウンス時間経過 → Mozc 通常変換が画面に出る → 1000ms 待機 → Zenz 推論 → 画面が Zenz 結果で上書き」という二段階遷移が必ず発生していました。

### 3.2 追加入力時に AI 結果が消えていた原因
- 追加入力時（`Session::InsertCharacter` 等）に `CancelLiveConversionForEditing()` が呼び出されますが、内部で `ClearZenzLiveCorrectionState()` を呼び、直前の Zenz 表示状態（`zenz_live_key_`, `zenz_live_value_` など）をすべてゼロクリアしていました。
- また、Zenz 結果を表示した時点（`OutputZenzLiveCorrection`）で、Mozc のライブ変換状態変数（`live_conversion_key_`, `live_conversion_value_` など）が Mozc 通常変換の値のままであり、Zenz の変換値で同期されていませんでした。
- Mozc の `OutputPendingLiveConversion` は「すでに確定・表示されているプレフィックス（`live_conversion_value_`）＋ 新たに入力されたサフィックス（未変換文字列）」を画面に合成表示する能力を持っていましたが、プレフィックスが Zenz の値になっておらず、かつ編集時に消去されていたため機能していませんでした。

---

## 4. 設計および実装の詳細

### 4.1 ライブ変換発動時の Mozc 通常変換抑止と即時推論リクエスト
- **該当箇所**: [`src/session/session.cc`](file:///C:/workfile/mozkey/src/session/session.cc) (`MaybeStartLiveConversion`)
- **変更内容**:
  1. `use_zenz_live_correction()` が有効な場合、`Output(command)`（画面への Mozc 通常変換出力）をスキップします。
  2. 代わりに `OutputPendingLiveConversion(command)` を呼び出し、画面には未確定状態（未変換 preedit）を維持します。
  3. 二段階ディレイを挟まず、直ちに `ScheduleZenzLiveCorrectionInternal(command, /*start_immediately=*/true)` を呼び出して Zenz 推論リクエストを発行し、ポーリング待機状態（24ms）に入ります。
  4. これにより、Mozc 通常変換はバックグラウンドで安全保護用（フォールバック用）に参照されるのみとなり、画面には一切表示されず、Zenz の結果が最初から直接画面に現れます。

### 4.2 直前の AI 変換結果の同期と追加入力中の合成表示
- **該当箇所**: [`src/session/session.cc`](file:///C:/workfile/mozkey/src/session/session.cc) (`OutputZenzLiveCorrection`, `CancelLiveConversionForEditing`)
- **変更内容**:
  1. **AI 変換結果の同期 (`OutputZenzLiveCorrection`)**:
     Zenz から結果（例: 「今日」）が返って画面に出力する際、次回の追加入力のために `live_conversion_key_`, `live_conversion_preedit_`, `live_conversion_value_`, `live_conversion_preedit_output_` を Zenz の値で上書き同期します。同時に、Space キーでの通常変換復元用に元の Mozc 出力を `zenz_live_mozc_preedit_output_` に退避します。
  2. **追加入力時の状態保護 (`CancelLiveConversionForEditing`)**:
     文字入力時に呼ばれる関数において、直前の AI 状態を消去していた `ClearZenzLiveCorrectionState()` をやめ、未完了リクエストのみをキャンセルする `CancelPendingZenzLiveCorrection()` に変更しました。
  3. **合成表示の維持 (`OutputPendingLiveConversion`)**:
     ユーザーが 'h', 'a' と続けて入力した際、`live_conversion_value_`（「今日」）をベースとし、新しく入力された差分（"h" や "は"）をサフィックスとして合成し、画面には「今日h」「今日は」と下線付きで途切れず表示され続けます。
  4. **タイピング停止後の再予測**:
     ユーザーが手を止めるとデバウンス時間が経過し、再度 `MaybeStartLiveConversion()` が発動して画面の「今日は」を保持したまま次の Zenz 予測（例：「今日は晴れ」）がリクエストされ、完了時にシームレスに更新されます。

### 4.3 Space キーによる Mozc 通常変換への復帰
- **該当箇所**: [`src/session/session.cc`](file:///C:/workfile/mozkey/src/session/session.cc) (`RevertZenzLiveCorrectionToNormalConversion`)
- **変更内容**:
  AI 変換結果が表示されている状態でユーザーが Space キーを押した場合、退避されていた `zenz_live_mozc_preedit_output_` および `zenz_live_mozc_value_` を用いて、従来の Mozc 通常変換候補に正しく切り替わるように復元処理を更新しました。

### 4.4 CI（GitHub Actions）の Windows 限定化
- **該当箇所**: [`.github/workflows/`](file:///C:/workfile/mozkey/.github/workflows/)
- **変更内容**:
  - `windows.yaml`: `on: push` を維持し、手動実行用の `workflow_dispatch` を追加。
  - `secure-offline.yaml`: Windows 2025 で実行されるセキュリティ・オフラインチェックのため維持。
  - `android.yaml`, `linux.yaml`, `macos.yaml`, `macos_zenz_formal_package_dual_native.yml`, `lint.yaml`:
    `on: push` および `pull_request` トリガーを削除し、手動実行 `on: workflow_dispatch` のみに変更。

---

## 5. テスト・検証

[`src/session/session_test.cc`](file:///C:/workfile/mozkey/src/session/session_test.cc) において以下の単体テストを整備・追加しました：

1. **`ZenzLiveCorrectionPositiveDelayStartsImmediatelyWithoutShowingNormalConversion`**
   - ライブ変換開始時に Mozc 第1候補漢字が出力されず、未確定 preedit が画面に表示され、直ちに Zenz 推論ポーリングが開始されることを検証。
2. **`ZenzLiveCorrectionZeroDelayStartsImmediatelyAndKeepsLiveOutput`**
   - ゼロディレイ時にも Mozc 通常変換が出力されず pending preedit が維持されることを検証。
3. **`ZenzLiveConversionDisplaysZenzDirectlyWithoutMozcNormalConversion`** (要求1のテスト)
   - 「きょう」と入力した際、Mozc 通常変換（例：「凶」）が画面に一切表示されず、Zenz から結果（「今日」）が届いた時点で直接表示されることを検証。
4. **`SubsequentKeystrokesPreservePreviousZenzResultAndAppendInputs`** (要求2のテスト)
   - 「今日」が表示された後、'h' を入力すると「今日h」、続けて 'a' を入力すると「今日は」と表示が保たれることを検証。
   - デバウンス経過後も「今日は」が保持され、新たな AI 推論結果（「今日は晴れ」）が届いた時点で更新される一連のフローを検証。

---

## 6. 変更ファイル一覧

- [`src/session/session.h`](file:///C:/workfile/mozkey/src/session/session.h): メンバ変数 `zenz_live_mozc_preedit_output_` および内部メソッドのシグネチャ追加
- [`src/session/session.cc`](file:///C:/workfile/mozkey/src/session/session.cc): ライブ変換制御、AI 結果同期、追加入力合成表示ロジックの実装
- [`src/session/session_test.cc`](file:///C:/workfile/mozkey/src/session/session_test.cc): `SessionTestPeer` の拡張、既存テスト更新、新規テスト2件の追加
- [`.github/workflows/*.yaml`](file:///C:/workfile/mozkey/.github/workflows/): Windows 以外の CI ワークフローの push トリガー無効化
