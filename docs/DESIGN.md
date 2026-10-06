# WinVST Bridge 設計書（v0.1 草案）

Apple Silicon Mac の Logic Pro で、Windows 用 32bit VST2 プラグイン（.dll）を
Wine 経由で使えるようにする AU ブリッジの設計。

- 対象：Windows 用 32bit VST2（例：JackDark / Shattershot Lite）
- 環境：Apple Silicon Mac、Logic Pro（ネイティブ起動、Rosetta 不要）
- 前提：Wine（WoW64 モード）が動くこと。Wine 側は Rosetta 2 上で動くため、
  Rosetta 2 が全面的に使える macOS（27 まで）を前提とする

---

## 1. 全体構成

```
┌────────────── Logic Pro（arm64 ネイティブ）─────────────────┐
│                                                              │
│  [WinVSTBridge.component]  AUv2 / arm64                      │
│   ├─ 音声・MIDI・パラメータを共有メモリに書く               │
│   ├─ 「処理して」を合図 → 結果を待つ（同じブロック内）      │
│   └─ Cocoa ビュー（「エディタを開く」ボタン程度）          │
│                                                              │
└───────────────┬──────────────────────────────────────────────┘
                │ 共有メモリ（データ）＋ソケット（合図）
┌───────────────┴──── Wine（WoW64, Rosetta 2）────────────────┐
│  [winvsthost.exe]  32bit Windows 実行ファイル               │
│   ├─ Plugin.dll を LoadLibrary → VSTPluginMain/main         │
│   ├─ processReplacing / dispatcher を呼ぶ                   │
│   └─ エディタ画面（Wine の別ウィンドウ）                   │
└──────────────────────────────────────────────────────────────┘

[winvst-register]（macOS コマンドラインツール）
   DLL フォルダを走査 → 各 DLL を winvsthost.exe --probe で調べる
   → 設定 JSON を書き出し、AU の Info.plist に登録 → 再署名
```

| 部品 | 言語・ビルド | 役割 |
|---|---|---|
| `WinVSTBridge.component` | C++ / Objective-C++、Apple の AudioUnitSDK、arm64 | Logic に見えるプラグイン本体 |
| `winvsthost.exe` | C / C++、`i686-w64-mingw32`（32bit Windows） | Wine 上で DLL を読み込み、実際に処理する |
| `winvst-register` | C++ または Swift、macOS | DLL の登録・登録解除・一覧 |

### 決めたこと（これまでの相談の結果）

- **方式B（DLL ごとに Logic に登録）**：Logic のリストに「Shattershot Lite (Bridge)」の
  ように個別に出す。パラメータ名・エフェクトか音源かを DLL ごとに正しく持たせるため
- **1 インスタンスにつき 1 ホストプロセス**：落ちても他に影響しない。DLL 内部の
  グローバル状態が複数のインスタンス間で混ざらない
- **同期処理（追加レイテンシ 0 サンプル）**：Logic から受け取ったブロックを、
  そのブロックのうちに処理して返す
- 将来の移行先（Rosetta なしの環境）はこの設計の範囲外

---

## 2. Logic への登録（方式B の実現方法）

### 2.1 1 つのバンドルに複数のコンポーネントを登録する

DLL ごとに `.component` を複製すると、同じバイナリが何度も読み込まれ、
Objective-C のクラス名が衝突する。そこで、**バンドルは 1 つ**にして、
`Info.plist` の `AudioComponents` 配列に DLL の数だけ項目を並べる。

```xml
<key>AudioComponents</key>
<array>
  <dict>
    <key>name</key>        <string>JackDark: Shattershot Lite (Bridge)</string>
    <key>type</key>        <string>aufx</string>   <!-- 音源なら aumu -->
    <key>subtype</key>     <string>Wb01</string>   <!-- 登録ごとに一意 -->
    <key>manufacturer</key><string>WvBr</string>
    <key>factoryFunction</key><string>WinVSTBridgeEffectFactory</string>
    <key>version</key>     <integer>65536</integer>
  </dict>
  ...
</array>
```

- `type`：エフェクトは `aufx`、音源は `aumu`。MIDI を受けるエフェクトは `aumf`
- `factoryFunction`：エフェクト用と音源用の 2 つを 1 つのバイナリに用意する
- `subtype`：登録ツールが割り当てる。**一度割り当てたら変えない**
  （変えると Logic のプロジェクトがプラグインを見失う）。対応表は
  `registry.json` に保存する

### 2.2 インスタンス生成時に、どの DLL かを判別する

ファクトリ関数の中で `AudioComponentGetDescription` を呼んで自分の `subtype` を取得し、
`~/Library/Application Support/WinVSTBridge/plugins/<subtype>.json` を読む。

```jsonc
// plugins/Wb01.json（登録ツールが --probe の結果から生成）
{
  "subtype": "Wb01",
  "dll": "/Users/<user>/WinVST/Shattershot Lite/Shattershot_Lite.dll",
  "dllSha256": "…",              // DLL が差し替えられたことを検出する
  "uniqueID": "…",               // VST の uniqueID
  "name": "Shattershot Lite",
  "vendor": "JackDark",
  "isSynth": false,
  "numInputs": 2, "numOutputs": 2,
  "flags": { "canReplacing": true, "programChunks": false, "hasEditor": true },
  "initialDelay": 0,
  "params": [ { "index": 0, "name": "FB ALPHA", "label": "" }, … ],
  "programs": [ "…", … ]
}
```

### 2.3 登録ツール `winvst-register`

```
winvst-register scan ~/WinVST        # フォルダを走査して新しい DLL を登録
winvst-register list                 # 登録済み一覧（subtype と DLL）
winvst-register remove Wb01          # 登録解除
winvst-register rescan Wb01          # パラメータ情報を取り直す（subtype は変えない）
```

処理の流れ：
1. `*.dll` を探し、PE ヘッダーで 32bit かどうかと、`VSTPluginMain` / `main` の
   エクスポートがあるかを確認する
2. `wine winvsthost.exe --probe <dll>` で情報を JSON に出力する
   （プロセスが落ちたりタイムアウトしたりした DLL は「非対応」と表示して飛ばす）
3. `Info.plist` を書き換える → `codesign --force -s -`（アドホック署名）で再署名
4. AU のキャッシュを更新する（`killall -9 AudioComponentRegistrar`）。
   Logic は次の起動時に再スキャンする

---

## 3. 通信（ブリッジ AU ⇔ winvsthost.exe）

### 3.1 データ：共有メモリ

- インスタンスごとに 1 つのファイルを用意する
  （例：`$TMPDIR/winvstbridge/<pid>-<n>.shm`）
- macOS 側は `mmap(MAP_SHARED)`、Wine 側は `Z:\…` のパスに対して
  `CreateFileMapping` + `MapViewOfFile` で同じファイルを開く
- **要検証（Phase 0）**：Wine のファイルマッピングが、macOS 側と同じページを共有するか。
  共有されない場合の代替手段は、データもソケットで送る方法
  （ステレオ 512 フレームでも約 4KB なので現実的）

共有メモリの配置（概略）：

```c
struct ShmHeader {
    uint32_t magic, version;
    uint32_t maxFrames, numIn, numOut;
    double   sampleRate;
    // 1 ブロックごとの要求
    uint32_t seq;            // AU がブロックごとに +1
    uint32_t numFrames;
    VstTimeInfoLite time;    // テンポ・小節位置・再生中かどうか
    // 応答
    uint32_t doneSeq;        // ホストが処理を終えたら seq を書き戻す
    uint32_t status;
};
// 続けて：入力バッファ[numIn][maxFrames], 出力バッファ[numOut][maxFrames]
// リングバッファ ×3：
//   AU→ホスト：パラメータ変更・MIDI（フレーム位置つき）
//   ホスト→AU：GUI でのパラメータ変更（audioMasterAutomate）・各種通知
```

### 3.2 合図：ソケット＋短いスピン

Wine 上の Windows 実行ファイルからは、macOS の Mach セマフォを直接使えない。
そこで次の組み合わせにする：

1. AU は `seq` を進めたあと、ソケットに 1 バイト書く
2. ホストは `recv` で待っている。起きたら処理して、`doneSeq` を書き戻す
3. AU は `doneSeq` を **数十マイクロ秒だけスピンして待ち**、それで終わらなければ
   ソケットの応答を `poll` で待つ

- ソケットは AF_UNIX を第一候補にする（Wine が対応していることを Phase 0 で確認する）。
  使えなければ `127.0.0.1` の TCP（`TCP_NODELAY`）を使う
- 片道の合図にかかる時間の目安は 10〜30µs。**Phase 0 で実測して方式を確定する**

### 3.3 タイムアウトと異常時の動作

- 待ち時間の上限：**バッファ 1 つ分の時間の約 70%**。超えたらそのブロックは無音を出力し、
  「遅延」カウンタを増やす（Logic 全体を止めない）
- オフラインのバウンス中（`kAudioUnitProperty_OfflineRender`）は締め切りがないので、
  上限を長く（数秒）とる
- ホストプロセスが落ちた場合：無音を出力し、AU の画面に「停止中・再起動」を表示する。
  再起動すると、直前に保存した状態から復元する

---

## 4. ホストプロセス `winvsthost.exe`

- 既存の `AU_PORT/research/work/tools/vsthost/vsthost.exe` を土台にする
- 起動時の引数：`--dll <path> --shm <path> --sock <path> --sr <rate> --max <frames>`
- **FPU 制御ワードを 0x027F に設定する**（本物の Windows の初期値。Wine の初期値は
  0x037F で、計算結果が変わる。RESUME.md より）
- 処理スレッドは `THREAD_PRIORITY_TIME_CRITICAL` に設定する。Wine の macOS 版で
  実際にどこまで反映されるかは Phase 0 で確認する
- DLL を読み込んだあと、**無音で数ブロック空回りさせてから「準備完了」を返す**
  （UPX 圧縮の DLL は実行時に展開されるため、最初の実行時に Rosetta の翻訳が走り、
  処理が一瞬止まる。それを本番前に済ませておく）
- `--probe` モード：DLL を開いて情報を JSON で標準出力に書き、終了する

### 4.1 VST2 の呼び出し

- VST2 SDK は Steinberg が配布を終えている。SDK は同梱せず、`AEffect` 構造体と
  必要なオペコードだけを自前のヘッダーで定義する
- 初期化の順番：
  `effOpen → effSetSampleRate → effSetBlockSize(maxFrames) → effMainsChanged(1)`
  - `maxFrames` は Logic の `MaximumFramesPerSlice`。実際に届くブロックはこれより
    小さいことがあるが、`effSetBlockSize` は変えない（SynthEdit のエンジンは内部の区切りを
    これで決めるため。RESUME.md の `SeVst::setBlockSize` を参照）
- 処理：`processReplacing` があればそれを使い、なければ出力バッファを 0 にしてから `process` を呼ぶ
- MIDI：`effProcessEvents` に `VstMidiEvent`（`deltaFrames` つき）で渡す
- ホストコールバック（`audioMasterCallback`）：
  `audioMasterVersion`（2400 を返す）、`GetTime`（共有メモリ上の時間情報を返す）、
  `Automate`（AU に通知）、`IdleBridge`、`SizeWindow`、`GetSampleRate`、`GetBlockSize`、
  `GetVendorString` などに対応する

### 4.2 エディタ画面

- `effEditOpen` にホスト側で作ったトップレベルの HWND を渡す。Wine の macOS ドライバが、
  それを通常の Mac ウィンドウとして表示する
- 表示・非表示は AU の Cocoa ビューと連動させる：
  Logic でプラグインのウィンドウを開いたらエディタを表示、閉じたら非表示
- ウィンドウのタイトルには「トラック名（取得できれば）＋プラグイン名」を表示する
- GUI での操作（`audioMasterAutomate`）は、リングバッファで AU に届ける。AU は
  `AUParameterListenerNotify` で Logic に知らせる（オートメーションの書き込みに対応）

---

## 5. ブリッジ AU `WinVSTBridge.component`

- Apple の [AudioUnitSDK](https://github.com/apple/AudioUnitSDK)（Apache 2.0）を使う。
  `AUEffectBase` / `MusicDeviceBase` を継承する
  - JUCE を使わない理由：パラメータの数や名前、エフェクトか音源かを
    **インスタンス生成時に決めたい**ため
  - AUv3 にしない理由：AUv3 はサンドボックス化された別プロセスで動くので、
    Wine のプロセスを起動するのが難しい
- パラメータ：VST の 0〜1 の値をそのまま AU のパラメータ（0〜1）にする。
  名前は JSON から取る。数値の表示用には `effGetParamDisplay` の結果を使う（Phase 2）
- プリセット：VST のプログラムを AU のファクトリープリセットとして見せる
- 状態の保存（`ClassInfo`）：
  - `dllSha256`、VST の `uniqueID`
  - `effFlagsProgramChunks` があれば `effGetChunk` の結果を、なければ
    全パラメータの値と現在のプログラム番号を保存する
- レイテンシ：`initialDelay` を `kAudioUnitProperty_Latency` で Logic に伝える
- `Initialize` の中ではホストの起動を待たない（Logic のプロジェクト読み込みを止めないため）。
  準備ができるまでは無音を出力する。バウンス中なら準備完了まで待つ

---

## 6. Wine 環境

- あちらのセッションで `vsthost.exe` が動いている Wine をそのまま使う
  （種類とバージョンは Phase 0 で記録する）
- ブリッジ専用の `WINEPREFIX`：`~/Library/Application Support/WinVSTBridge/prefix`
- Wine のパスは `~/Library/Application Support/WinVSTBridge/config.json` に書く
- `wineserver -p`（常駐モード）で起動しておき、2 つ目以降のインスタンスを速く起動する

---

## 7. 開発の順番

### Phase 0：実現性の検証（測定だけ。製品のコードは書かない）

| # | 調べること | 合格の目安 |
|---|---|---|
| 0-1 | Wine のファイルマッピングが macOS 側と同じメモリを共有するか | 双方から書いた値が相手側で見える |
| 0-2 | 合図の往復時間（スピン／AF_UNIX／TCP） | 往復の中央値 < 50µs、99.9 パーセンタイル < 300µs |
| 0-3 | Shattershot の処理時間（Wine＋Rosetta、128 フレーム） | バッファ時間（2.9ms）の 20% 未満 |
| 0-4 | Wine 側スレッドの優先度設定が反映されるか、E コアに回されないか | 負荷をかけても 0-2 の値が悪化しない |

小さなテストプログラム 2 つ（macOS 側・Wine 側）で測る。

### Phase 1：最小構成のエフェクト

- `winvsthost.exe`（`--probe`、処理）、ブリッジ AU（エフェクトのみ、Cocoa ビューなし）、
  手作業での登録
- **合格条件**
  - `auval -v aufx Wb01 WvBr` に合格する
  - **音のビット一致**：同じ入力を与えたとき、ブリッジの出力が既存の `vsthost.exe` の
    出力と完全に一致する（あちらのセッションの比較ツール `cmp.sh` の手法を流用する）
  - Logic のバッファ 128 で、ノイズなしで再生できる

### Phase 2：実用レベル

- エディタ画面（Cocoa ビューとの連動）、GUI からのパラメータ変更、状態の保存と復元、
  プリセット、`winvst-register` コマンド

### Phase 3：音源と仕上げ

- 音源（`aumu`）・MIDI、テンポなどの時間情報、ホストの異常終了からの復帰、
  サンプルレートの変更、オフラインのバウンス

### Phase 4：検証

- 手持ちの JackDark プラグインをまとめて登録して、動作を一覧にする
- Logic のバッファ 32 / 64 / 128 / 256 と、トラック数を変えての負荷テスト

---

## 8. リポジトリ構成（予定）

```
AU_use_vst/
├─ docs/
│   └─ DESIGN.md            この文書
├─ au/                      WinVSTBridge.component（CMake, arm64）
├─ host/                    winvsthost.exe（mingw-w64 i686）
├─ register/                winvst-register
├─ shared/                  共有メモリの配置・プロトコル定義（両側から include）
├─ poc/                     Phase 0 の測定プログラム
└─ scripts/                 ビルド・インストール・auval
```

`shared/` のヘッダーは arm64（macOS）と i686（Windows）の両方でコンパイルする。
構造体の配置がずれないように、固定幅の型だけを使い、`static_assert` でサイズを確認する。

---

## 9. 制約と既知のリスク

| 項目 | 内容 |
|---|---|
| Rosetta 2 への依存 | Wine 側は Rosetta 2 が必要。macOS 27 までを前提にする |
| 小さいバッファ | 32〜64 フレームでは、ノイズが出る可能性がある（Phase 0 と Phase 4 で確認） |
| エディタ画面 | Logic のウィンドウの中には埋め込めず、Wine の別ウィンドウになる |
| 起動時間 | インスタンスごとに Wine のプロセスを起動するので、プロジェクトを開くときに数秒かかる |
| 非対応のもの | VST3、コピープロテクト（iLok など）、特殊なドライバを使うプラグイン |
| 64bit の Windows 用 VST | 同じ設計の x86_64 版ホストで対応できる見込み（今回の範囲外） |
