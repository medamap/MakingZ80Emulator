# Z80エミュレータ実装（C# / C++ / GPU 対応）

## 1. 設計方針
- **ユニティ (C#) で試作し、最終的に C++ に移植**
- **CPUコアはプレーンな C# で実装**
- **入力・出力（I/O）や描画は抽象化し、プラットフォームごとに実装**
- **C# の便利機能（List, LINQ, ガベージコレクションなど）を避ける**
- **クロックとウェイトを正確にエミュレート**
- **ユニティ (C#) で試作し、最終的に C++ に移植**
- **GPU 版（CUDA / OpenCL / WebGPU / GLSL / HLSL）に対応**
- **Unity / UE5 でゲームエンジン対応**
- **CMake + Docker で環境を統一し、ビルドを自動化**

---

## 2. 命令デコード方式

C# の場合、デリゲートを使った関数ポインタ配列を利用する。  
例：「OpcodeHandler」というデリゲート型を定義し、256個の関数ポインタを持つ配列を作成する。  
命令のオペコードをインデックスとして、対応する関数を呼び出す。  

C++ では、関数ポインタ配列を使用し、 `this->*opcodeTable[opcode]();` のようにメンバ関数を実行する。  
C# の `delegate` の代わりに、 `void (Z80::*)()` 型の関数ポインタを使う。  

この方式を使うことで、スイッチ文よりもコンパクトに処理でき、命令デコードが高速化できる。  
また、新しい命令を追加する際も配列に関数を登録するだけで済む。  

---

### 2.1 GPU における命令デコード方式（検討中）

GPU 版では CPU とは異なるアプローチが必要になる。  
スレッド間の同期やメモリアクセスの特性を考慮し、いくつかの選択肢がある。

#### **2.1.1 命令テーブル方式**
- **CPU 版と同様に命令テーブル（関数ポインタ配列）を使用**
- スレッドごとに異なる命令を処理し、適切なカーネル関数を呼び出す
- メリット:
  - CPU 版のアーキテクチャをそのまま GPU に持ち込める
  - 各スレッドが独立して動作可能
- デメリット:
  - 命令のスレッド間での同期が難しい
  - メモリアクセス競合が発生する可能性がある

#### **2.1.2 分岐（if-else / switch）によるデコード**
- `switch(opcode)` を使用し、各スレッドで命令をデコード
- CUDA / OpenCL / WebGPU などのカーネル内で `if (opcode == XX) {}` を展開
- メリット:
  - 命令ごとにカーネルを変更せずに済む
  - 1 つのカーネルで全命令を処理できる
- デメリット:
  - スレッド間でのワープダイバージェンス（Warp Divergence）が発生する
  - スイッチ分岐が多くなると、カーネルの最適化が難しくなる

#### **2.1.3 命令デコードを事前に行い、プリデコード済み命令を実行**
- CPU 側で事前に命令デコードを行い、GPU 側では実行のみを行う
- メリット:
  - GPU のオーバーヘッドを減らせる
  - 命令フェッチとデコードのコストを CPU にオフロードできる
- デメリット:
  - CPU-GPU 間のデータ転送が必要になる
  - 事前デコードのため、柔軟な命令処理が難しくなる

#### **2.1.4 ワークグループごとに命令のフェッチ / デコード / 実行を分離**
- **M1（フェッチ）、M2（デコード）、M3（実行）を異なるスレッドで処理**
- 各ワークグループが、異なるフェーズを並列実行する
- メリット:
  - 並列処理の恩恵を最大限活かせる
  - 命令の同期を取りやすくなる
- デメリット:
  - メモリアクセスの最適化が必要（レジスタ / VRAM 共有の考慮）

---

### 2.2 どの方式を採用するか？
- 現段階では **命令テーブル方式（2.1.1）** または **事前デコード方式（2.1.3）** が有力
- CUDA / OpenCL / WebGPU など、バックエンドごとに最適な方式を選択する
- **最初は 2.1.2（switch 分岐方式）で実装し、最適化の過程で別の方式へ移行するのが現実的**

---

## 3. クロックとウェイト処理
Z80 の命令実行は **M1 (フェッチ) → M2 (オペランド取得) → M3 (メモリアクセス or 実行)** の流れで進む。  
各サイクルで **Tサイクル数 × 供給クロックの周期** を計算し、適切なウェイトを入れる。  

**Z80 の 1T サイクル時間**  
供給クロックを 3.579545MHz とすると、1T サイクルの時間は **約279ns**。  
各命令の合計サイクル数をこの値と掛けることで、適切なウェイト時間を求める。  

C# では `Stopwatch` を使用し、C++ では `std::chrono::high_resolution_clock` を使って処理を同期させる。  
スレッドスリープには、C# では `Thread.Sleep()`、C++ では `std::this_thread::sleep_for()` を使用。

---

## 4.1 ユニティでのフレーム同期処理
C# のユニティ版では `Time.deltaTime` を利用し、フレーム時間ごとに実行可能なサイクル数を計算する。  
C++ では `std::chrono` を使って、 `Time.deltaTime` と同様にフレーム単位の時間管理を行う。  
フレームごとに処理することで、シミュレーションのタイミングを調整しやすくする。

## 4.2 GPU でのフレーム同期処理（検討中）

GPU 版では、CPU 版とは異なる方法でフレーム同期を行う必要がある。  
CPU 版では `Time.deltaTime`（Unity）や `std::chrono`（C++）を使用してクロック同期を行うが、  
GPU ではスレッド単位での並列処理が主となるため、従来のウェイト処理とは異なるアプローチが求められる。

---

### 4.2.1 GPU でのクロック管理方法（検討中）

いくつかの方式が考えられる。

#### **A. GPU 側でウェイト（Atomic 操作 / バリア）を使用**
- **方法:**  
  - GPU 内で時間を計測し、一定のクロックごとに処理を進める
  - `atomicAdd()` や `barrier()` を利用して、スレッド間の同期を取る
- **メリット:**  
  - 完全に GPU 内で処理が完結し、CPU とのデータ転送を減らせる
  - CPU 側でのスケジューリングが不要になる
- **デメリット:**  
  - GPU における高精度タイマーの精度が不明確
  - スレッド間の同期コストが発生し、並列性が損なわれる可能性がある

#### **B. CPU 側でクロック制御を行い、GPU に実行タイミングを通知**
- **方法:**  
  - CPU 側で Z80 のクロックを管理し、一定の時間ごとに GPU に命令を送信
  - CUDA の `cudaEvent`、OpenCL の `clEnqueueBarrier` などを活用し、GPU の処理タイミングを制御
- **メリット:**  
  - CPU 版の実装と比較的統一しやすい
  - フレームごとの処理タイミングを厳密に管理可能
- **デメリット:**  
  - CPU から GPU への頻繁なデータ転送が発生し、オーバーヘッドになる
  - スレッドごとの並列処理を活かしにくい

#### **C. GPU のシェーダー内部でサイクルごとに分割実行**
- **方法:**  
  - 1 スレッドで 1 サイクルを処理し、スレッド数を増やすことで並列処理を最大化
  - 命令ごとにスレッドを管理し、スケジュールされた命令を逐次実行
- **メリット:**  
  - GPU の並列処理を最大限活かせる
  - CPU との通信が最小限で済む
- **デメリット:**  
  - メモリアクセスの競合が発生する可能性がある
  - 命令ごとにスレッドを起動するため、オーバーヘッドが発生する可能性がある

---

### 4.2.2 どの方式を採用するか？
- 現時点では **A（GPU 内でクロック管理）** または **B（CPU で制御）** が有力候補
- **最初は B（CPU でクロック管理）で実装し、後に A（GPU 完結型）を検討するのが現実的**
- **C（完全並列型）は実装が複雑なため、オーバーヘッドを測定してから導入を検討する**

---

## 5. Pub-Sub を活用したメモリアクセス

### 5.1 C# の `Publish<T>` によるメモリアクセス

C# では `Dictionary<Type, List<Delegate>>` を利用し、  
**ジェネリクス (`Publish<T>`) による型安全な Pub-Sub** を実装する。

- `Z80` が `MemoryAccess16Bit` のリクエストを発行 (`Publish<T>`)
- `Memory` がリクエストを受け取り、結果 (`MemoryReadResponse`) を `Publish<T>` で返す
- `Z80` は `MemoryReadResponse` を Subscribe して結果を取得

---

### 5.2 C++ の `Publish<T>` によるメモリアクセス

C++ では `std::unordered_map<std::type_index, std::vector<std::function<void(T)>>>` を使い、  
**テンプレート (`template`) を利用して型安全な Pub-Sub** を実装する。

- `Z80` が `Publish<MemoryAccess16Bit>(...)` を使ってリクエストを発行
- `Memory` がリクエストを Subscribe し、結果 (`MemoryReadResponse`) を `Publish<T>` で返す
- `Z80` が `MemoryReadResponse` を Subscribe して結果を取得

C++ では `typeid(T)` をキーにすることで **型ごとのイベントを管理** できる。

### 5.3 GPU の `Publish<T>` によるメモリアクセス（検討中）

GPU 版では、CPU 版の `Publish<T>`（Pub-Sub モデル）をそのまま適用することは難しい。  
スレッド間の同期、メモリアクセスの最適化、VRAM へのマッピングなどを考慮する必要がある。  

---

### 5.3.1 GPU での Pub-Sub の課題

GPU では、CPU のように `std::unordered_map`（C++）や `Dictionary<Type, List<Delegate>>`（C#）を  
そのまま利用することができないため、異なる実装アプローチが必要となる。  

考慮すべきポイント：
- **スレッド間でイベントをどのように通知するか**
- **メモリアクセスの競合をどのように回避するか**
- **GPU 内で最適なデータ共有方法を選択する**

---

### 5.3.2 GPU で考えられる `Publish<T>` の実装方式

#### **A. 共有メモリ（Shared Memory）を利用した通知システム**
- **方法:**
  - グローバルメモリではなく、共有メモリ（Shared Memory）を利用し、スレッド間でデータを交換
  - `atomicAdd()` や `atomicExch()` を利用し、Pub-Sub の通知を行う
- **メリット:**
  - メモリアクセスのオーバーヘッドを削減できる
  - スレッド間のデータ交換が高速に行える
- **デメリット:**
  - 共有メモリのサイズ制限がある（CUDA では 48KB など）
  - スレッドブロック間の同期は難しい

#### **B. VRAM にメッセージキューを実装**
- **方法:**
  - VRAM 上にキューを作成し、スレッド間でメッセージを積む
  - `atomicAdd()` を使ってキューのエントリを管理
- **メリット:**
  - スレッド間でのデータ交換を GPU 内で完結できる
  - CUDA / OpenCL / WebGPU すべてで同様の実装が可能
- **デメリット:**
  - メモリ競合が発生しやすく、適切なロック機構が必要
  - キューのオーバーヘッドが発生する

#### **C. CPU 側で `Publish<T>` を管理し、GPU はポーリング**
- **方法:**
  - CPU 側で `Publish<T>` を実装し、GPU 側は `__constant__` メモリや `clEnqueueReadBuffer` を使ってポーリング
  - 必要なデータのみを GPU 側に渡し、GPU は実行するだけ
- **メリット:**
  - GPU 側のオーバーヘッドを最小限に抑えられる
  - CPU 版の設計と統一しやすい
- **デメリット:**
  - CPU-GPU 間の通信が頻繁に発生する
  - レイテンシが大きくなり、リアルタイム性が低下する

#### **D. ワークグループ内でイベント駆動型の処理を行う**
- **方法:**
  - 各スレッドが `if (hasEvent)` のような条件分岐を持ち、該当するスレッドのみ処理を実行
  - `barrier()` を使用してスレッドグループ内で同期
- **メリット:**
  - シンプルな実装で済む
  - 各スレッドが独立して動作可能
- **デメリット:**
  - `warp divergence`（異なるスレッドが異なる処理を実行することによるパフォーマンス低下）が発生する
  - すべてのスレッドがポーリングするため、無駄な処理が発生する

---

### 5.3.3 どの方式を採用するか？
- 現段階では **B（VRAM にメッセージキューを実装）** または **C（CPU 側で管理）** が有力候補  
- 最初は **C（CPU 側で制御）** を実装し、その後 GPU 内で完結する方法（B もしくは A）へ移行するのが現実的  
- **D（ワークグループ内イベント駆動）はパフォーマンス面で問題があれば導入を検討**

---

## 6. まとめ

- 6.1 **命令デコードはテーブルルックアップ方式（C# は `delegate[]`、C++ は `関数ポインタ配列`）**
- 6.2 **クロック同期は `Stopwatch`（C#） → `std::chrono`（C++）に移植**
- 6.3 **ウェイト処理は `Thread.Sleep()`（C#） → `std::this_thread::sleep_for()`（C++）に移植**
- 6.4 **ユニティでは `Time.deltaTime` を活用し、フレームごとの実行に対応**
- 6.5 **C# の `Publish<T>` を C++ の `template` で実装し、型安全な Pub-Sub を実現**
- 6.6 **C# の MessagePipe を使わなくても、C++ で同等の機能を実装可能**
- 6.7 **GPU の実装については、既存のテクノロジーを活用しつつ、最適な方式を選択**

---

### 6.7 GPU の実装方針（整理）
GPU 版の実装は CPU 版と異なり、並列処理に最適化する必要がある。  
現在の検討内容を踏まえ、以下の点を整理する。

#### **6.7.1 命令デコード**
- **命令テーブル方式（関数ポインタ配列に相当）**
  - CUDA / OpenCL では `switch-case` によるデコードが一般的
  - WebGPU / GLSL では LUT（ルックアップテーブル）を活用する可能性あり
  - CPU 版の関数ポインタ配列と動作を統一できるように工夫する

#### **6.7.2 フレーム同期**
- **GPU 側で時間を管理（Atomic / Barrier）**
  - CUDA: `atomicAdd()` / `cudaEvent`
  - OpenCL: `barrier(CLK_GLOBAL_MEM_FENCE)`
  - WebGPU: `workgroupBarrier()`
- **CPU でクロック制御を行い、GPU にタイミングを通知**
  - `cudaMemcpyAsync()` でデータ転送を最小限に抑える

#### **6.7.3 メモリアクセス**
- **VRAM にメモリマップを配置**
  - CPU 側で Z80 のメモリマップを VRAM に直接転送
  - メモリアクセスは `__shared__` や `local memory` を活用
- **メモリアクセスの競合を回避する手法**
  - `atomicCAS()` や `atomicExch()` を使用し、書き込み競合を回避
  - 可能であれば、レジスタ内にキャッシュし、不要な VRAM アクセスを削減

#### **6.7.4 `Publish<T>` の GPU 版**
- **方式 1:** 共有メモリを使い、スレッド間でイベントキューを管理
- **方式 2:** CPU 側で Pub-Sub を管理し、GPU はポーリング
- **方式 3:** VRAM にメッセージキューを構築し、スレッドごとに処理を実行
- 最初は **方式 2（CPU 側管理）** を実装し、GPU 内で完結する方式を検討

#### **6.7.5 GPU 版のビルド管理**
- CMake の Variant を活用し、以下のバックエンドで切り替え可能にする
  - `-DZ80_BACKEND=CUDA`
  - `-DZ80_BACKEND=OPENCL`
  - `-DZ80_BACKEND=WEBGPU`
  - `-DZ80_BACKEND=GLES`
  - `-DZ80_BACKEND=HLSL`
- Docker の `docker-compose.yml` にて、各 GPU バックエンド用のコンテナを作成

---

### 6.8 今後の課題
- **GPU 版のプロトタイプを作成し、最適な並列化手法を検討**
- **CPU 版のコードとの共通化を進め、保守性を向上**
- **最適なメモリアクセス方法を見つけ、パフォーマンスを最大化**

---

## 7. 具体的な実装

### 7.1 実行予定タスク

#### **1. GitHubレポジトリ作成（Z80EmulatorModule）**
- **フォルダ/ファイル構成**
  - csharp/ # ネイティブ C# 版
    - core/ # Z80 の共通ロジック
    - cli/ # CLI 版エミュレータ
    - unity/ # Unity 対応版（ゲームエンジン向け）
  - cpp/ # ネイティブ C++ 版
    - core/ # Z80 の共通ロジック
    - cli/ # CLI 版エミュレータ
    - ue5/ # UE5 対応版（ゲームエンジン向け）
  - gpu/ # GPU 版エミュレータ
    - core/ # GPU 版の共通ロジック
    - cuda/ # CUDA 版
    - opencl/ # OpenCL 版
    - webgpu/ # WebGPU 版
    - glsl/ # GLSL (OpenGL ES) 版
    - hlsl/ # HLSL (UE5 / Unity) 版
  - game_engines/ # ゲームエンジン向けプロジェクト
    - unity/ # Unity プロジェクト
    - ue5/ # UE5 プロジェクト
  - docker/ # Docker 設定
  - tools/ # デバッグツール、スクリプト、ユーティリティ
  - docs/ # ドキュメント（設計資料、APIリファレンスなど）

- **.gitignore 設定**
  - `dotnet`, `VS 2022 Community`, `VSCode`, `CMake`, `Docker`, `Unity`, `UE5` に対応

- **CMake 対応**
  - Variant で CPU（C# / C++）、GPU（CUDA / OpenCL / WebGPU / GLSL / HLSL）のビルドに対応
  - Unity / UE5 はそれぞれのプラットフォームでビルド（Godot もいけたら面白い）

- **開発言語**
  - C#（.NET Standard / .NET Core）
  - C++（C++17 以降、プラットフォーム非依存）
  - CUDA / OpenCL / WebGPU / GLSL / HLSL

---

#### **2. 開発環境の構築**

- **Docker を用いた開発環境**
  - `docker-compose.yml` にて環境をセットアップ
  - **使用する OS:** `Ubuntu 22.04 LTS`
  - **ボリュームマウント**
    - `../csharp` → `/languages/csharp`
    - `../cpp` → `/languages/cpp`
    - `../gpu` → `/languages/gpu`
    - `../game_engines/unity` → `/languages/unity`
    - `../game_engines/ue5` → `/languages/ue5`

---

### **2.1 必要な開発ツールのインストール**

#### **基本ツール**
- `git`, `wget`, `curl`
- `build-essential`, `cmake`, `ninja-build`
- `clang`, `gcc`, `gdb`, `lldb`
- `python3`, `pip`, `python3-venv`

#### **C# 開発環境**
- `.NET SDK（dotnet-sdk）`
- `Mono（mono-complete）`
- `MSBuild（msbuild）`

#### **C++ 開発環境**
- `clang++`, `g++`
- `CMake`, `Ninja`
- `LLVM`, `libstdc++`
- `valgrind`（メモリリーク検出）
- `Google Test`（ユニットテストフレームワーク）

#### **GPU 開発環境**
- **CUDA（NVIDIA 向け）**
  - `cuda-toolkit-12.x`
  - `nvidia-container-toolkit`
  - `nvcc`
- **OpenCL（クロスプラットフォーム GPU 計算）**
  - `ocl-icd-opencl-dev`
  - `clinfo`（OpenCL の環境確認）
- **WebGPU**
  - `dawn`（Google 製 WebGPU 実装）
  - `emscripten`（WebGPU を WebAssembly で利用）
- **OpenGL ES / GLSL**
  - `mesa-utils`
  - `libgles2-mesa-dev`
  - `glslang-tools`（GLSL コンパイラ）
- **HLSL**
  - `DXC（DirectX Shader Compiler）`
  - `Wine`（Windows 版 HLSL ツールの実行用）

#### **ゲームエンジン開発環境**
- **Unity**
  - `Unity Hub`
  - `Unity Editor`
- **UE5**
  - `Unreal Engine 5`
  - `Epic Games Launcher`
  - `vulkan-tools`（UE5 で Vulkan を使用するため）

#### **開発支援ツール**
- `VSCode`
- `Chrome`（X Server 経由で使用可能に）
- `Docker`（コンテナ環境管理）
- `docker-compose`

---

### **2.2 Docker コンテナのセットアップ**
- `Dockerfile` にて、必要な開発ツールをインストール
- `docker-compose.yml` で以下の環境をセットアップ：
  - `z80-cpu`（C#/C++ 版）
  - `z80-gpu`（CUDA/OpenCL/WebGPU/GLSL/HLSL 版）
  - `z80-unity`（Unity 開発用）
  - `z80-ue5`（UE5 開発用）

---

### **2.3. GPU を Docker で使用可能にする**
- **NVIDIA（CUDA）**
  - `nvidia-container-toolkit` をインストール
  - `docker run --gpus all nvidia/cuda:12.x-base` で GPU 動作確認
- **OpenCL**
  - `docker run --device=/dev/dri --privileged opencl-container`
- **WebGPU**
  - `docker run --device=/dev/dri webgpu-container`

---

### **2.4. Docker の起動コマンド**
```sh
# 全環境をビルド & 起動
docker-compose up --build -d

# 個別に起動（例: GPU 版）
docker-compose up --build z80-gpu

---

#### **3. 開発の進め方**

- **初期の Z80 モジュールは CLI で実行可能とする**
  - **デバッグ用のステップ実行モードを実装**
  - **ユニットテストを C# / C++ 両方で実装**
- **C# は .NET Standard / .NET Core の両方でビルド可能**
- **C++ はプラットフォーム非依存で設計**
  - **初期は Linux（Ubuntu 22.04 LTS）環境で開発**
  - **環境依存コードはプリプロセッサで切り分け**
  - **ある程度安定したら Windows でのビルドも検討（特に Unity / UE5）**

---

### **3.1. ビルド環境の統一**
- **CMake で C++ / GPU 版を管理**
  - `cmake -DZ80_BACKEND=CUDA -B build_cuda`
  - `cmake -DZ80_BACKEND=CPU -B build_cpu`
- **.NET CLI で C# 版を管理**
  - `dotnet build csharp/cli/Z80Cli.csproj`
- **Docker で環境を統一**
  - `docker-compose up --build`

---

### **3.2. 開発環境の自動セットアップ**
- **スクリプト化により、セットアップ手順を簡略化**
  - `setup.sh`（Linux 用セットアップスクリプト）
  - `Makefile`（C++ / GPU 版のビルド支援）
  - `setup_windows.ps1`（Windows 用セットアップスクリプト ※将来対応）

---

### **3.3. ゲームエンジン向け開発の進め方**
- **Unity / UE5 への対応は初期開発が落ち着いてから実施**
- **初期は CLI 版を優先し、エミュレーションの完成度を上げる**
- **ゲームエンジン版は Ubuntu / Windows の両方で動作確認を行う**
- **Windows 版のビルド検証は WSL2 + Docker で対応可能かも**

---

### **3.4. GPU 版の開発方針**
- **まず CUDA 版を開発し、基本動作を確認**
- **次に OpenCL / WebGPU / GLSL / HLSL へ移植**
- **GPU 版の最適化はプロファイラ（Nsight, VTune）を活用**
- **Windows での GPU 版動作は DirectML なども検討可能**

---

### **3.5. GPU 版の動作確認方法**

GPU 版は CPU 版とは異なり、リアルタイムでのデバッグが難しいため、  
以下の手順で **動作確認の仕組みを設計** し、正しく動作するかを確認する。

---

#### **3.5.1. 1 命令ごとの実行とデバッグ**
- **1 命令を実行する前にレジスタダンプを行う**
- **命令を GPU で実行後、結果を CPU に転送**
- **実行ごとに CPU に戻り、キー入力で次の命令を進める**
- **ステップ実行モードを用意し、デバッグしやすくする**
- **最初は CUDA 版で実装し、OpenCL / WebGPU / GLSL / HLSL でも同様に動作確認**

---

#### **3.5.2. レジスタとメモリの可視化**
- **GPU 版でも CPU 版と同じようにレジスタダンプを出力**
- **GPU メモリ（VRAM）内のデータを定期的に CPU にコピー**
- **デバッグ時は CUDA / OpenCL の `cudaMemcpy` や `clEnqueueReadBuffer` を利用**
- **命令の結果が CPU 版と一致するか比較テストを実施**

---

#### **3.5.3. GPU 版のデバッグ環境**
- **CUDA**
  - `cuda-gdb` や `Nsight Systems` を使用
- **OpenCL**
  - `clinfo` で OpenCL の情報を確認
  - `Intel VTune` で OpenCL カーネルをプロファイル
- **WebGPU**
  - `Dawn` の `dawn_native` を使用してログ出力
- **GLSL / HLSL**
  - `glslangValidator` でシェーダーを事前検証
  - `RenderDoc` で GLSL / HLSL の実行結果をキャプチャ

---

#### **3.5.4. GPU 版の動作確認ステップ**
- **STEP 1:** 1 命令実行ごとに CPU に戻し、ステップ実行デバッグを可能にする  
- **STEP 2:** レジスタのダンプを GPU から取得し、CPU 版と比較  
- **STEP 3:** メモリアクセスの結果が CPU 版と一致するか検証  
- **STEP 4:** CUDA / OpenCL / WebGPU / GLSL / HLSL すべてで動作確認  
- **STEP 5:** 一定の命令が正しく動作したら、ステップ実行なしの連続実行へ移行  

---

### **3.6. 今後の課題**
- **GPU 版のパフォーマンス最適化**
- **GPU 版のバッチ処理によるスループット向上**
- **ステップ実行デバッグを不要にするためのモニタリングツール開発**
- **動作確認の自動化（スクリプト or テストケース）**

---

### **7.2 開発の進捗管理**
- **GitHub Issues でタスク管理**
- **Docker の環境構築を最初に完了**
- **CLI で基本動作を確認してから GUI 実装に進む**
- **C# 版を先行して開発し、C++ へ移植**
- **最終的に他のプラットフォーム（Windows, macOS）にも対応**

## 8. 作業の進め方

- 現在どこまで進んでおり、どこからのタスクが止まっているかを確認する  
- 開始地点のタスクを特定し、1つのタスクを **キリの良いポイントで複数のステップに分ける**  
- ステップ毎に **作業指示を ChatGPT から行い、それを実行する**  
  - 必要なら ChatGPT にコード作成を依頼し、動作確認を行う  
- ステップ完了時、「次のステップへお願いします」と依頼し、ChatGPT によってタスク内の次のステップへ進行する  
- タスク内のステップが全て完了すれば、次のタスクへと進行する  

---

## 9. 作業タスク（CPU/GPU 版対応）

### **9.1 環境構築**
1. **GitHub レポジトリ作成**
2. **.gitignore を設定**
3. **GitHub レポジトリを作業フォルダへチェックアウト**
4. **レポジトリ内に必要フォルダを作成**
5. **Docker**
   - **5.1 概要**
     - Docker 環境は **状況に応じて再編成・再構築** される  
     - .NET Core, C++17, Linux 環境を統合する  
     - GPU 版（CUDA / OpenCL / WebGPU / GLSL / HLSL）にも対応  
   - **5.2 Docker 環境構築**
     - `Dockerfile` を修正し、GPU 版を含めたマルチプラットフォーム対応  
     - `docker-compose.yml` を改良し、CPU 版 / GPU 版を簡単に切り替え可能にする  
     - `docker-compose up --build` で全バージョンの Z80 エミュレータを一括ビルド  
     - `chrome` と `vscode` をインストールし `docker` 内で開発が行えるようにする
     - docker 内に開発アカウント develop を設定し、開発中はそのアカウントを使用する
     - `chrome` と `vscode` はアカウント develop で使用し、プロファイルも永続化する
   - **5.3 `.env` の設定例**
     - `.env` は CMake で Variant を設定できるなら不要になる可能性あり
     - GPU_BACKEND=CUDA # または OPENCL / WEBGPU / GLES / HLSL
     - ENGINE=UE5 # または UNITY / NONE
   - **5.4 GPU 用 Docker のセットアップ**
     - `nvidia-container-toolkit` をインストールし、CUDA を有効化
     - `docker run --gpus all nvidia/cuda:12.x-base` で GPU の動作確認
     - OpenCL, WebGPU, GLSL, HLSL についても同様に対応
   - **5.5 マルチプラットフォームビルドの準備**
     - Windows でのビルドも想定し、`setup_windows.ps1` を用意
     - `.env` で CPU / GPU のバックエンドを切り替え可能にする

---

### **9.2 C# Z80 実装**
7. **`csharp/` に `Z80Shared` C# プロジェクトを作成**
   - Z80 **共通モジュール** 作成
     - **Pub-Sub の仕組みを作成**
  
9. **`csharp/` に `Z80Cpu` C# プロジェクトを作成**
   - **Z80 CPU モジュール作成**
     - **Pub-Sub 組み込み**
     - **レジスタの定義**
     - **メインループ処理**
       - レジスタダンプ機能
       - 1命令ごとに **キーボード入力を受け付ける**
       - **命令フェッチ処理 (M1 サイクル)**
       - **命令デコード処理 (M2 サイクル)**
       - **命令実行処理 (M3 サイクル)**
     - **8ビットロード命令を順次実装**
  
10. **`csharp/` に `Z80Memory` C# プロジェクトを作成**
   - **Z80 メモリモジュール作成**
     - **Pub-Sub 組み込み**
     - **初期は 64KByte のメモリを実装**
       - **将来の拡張**（メモリマップドI/O、バンク切り替え）を考慮  
     - **メモリ読み書きリクエストを実装**

---

### **9.3 C++ Z80 実装**
11. **Z80 CPU の C# 実装がある程度完了したら、C++ 版を並行して実装**
12. **C# / C++ を交互に実装し、同等の機能を実装する**
    - `cpp/` に `Z80Shared` / `Z80Cpu` / `Z80Memory` を作成
    - `CMake` または `Makefile` を用意し、クロスプラットフォーム対応  
13. **C++ 版がある程度完成したら、GPU 版の開発を開始**

---

### **9.4 GPU Z80 実装**
14. **CUDA 版の Z80 CPU コアを開発（ベースライン）**
    - `gpu/gpu_cuda/` に `Z80CudaCore.cu` を作成し、基本実装を進める
    - **最初は CPU 版の命令実行フローを GPU 版に移植**
    - **1命令ずつステップ実行可能なモードを作成**
    - **レジスタダンプを CPU に送信し、デバッグ可能な形にする**
15. **GPU メモリアクセスの実装**
    - **VRAM に Z80 のメモリマップを構築**
    - **CUDA の `__shared__` メモリを活用し、高速なメモリアクセスを実装**
    - **OpenCL / WebGPU / GLSL / HLSL 版でも再利用できるように設計**
16. **GPU 版の命令デコード方式を実装**
    - **最初は `switch-case` 方式で実装**
    - **最適化の段階で `LUT（ルックアップテーブル）` や `プリデコード方式` を導入**
17. **ステップ実行モードを搭載し、GPU 版のデバッグを可能にする**
    - **1命令ごとに CPU に結果を返し、キー入力待機**
    - **CUDA の `cudaMemcpy` でレジスタ情報を取得**
    - **Nsight / VTune でカーネルの実行状態を解析**
18. **OpenCL 版の移植**
    - **CUDA 版が動作したら、OpenCL に移植**
    - **カーネルの書き換えを行い、共通部分を `gpu/core/` に分離**
19. **WebGPU / GLSL / HLSL 版の開発**
    - **WebGPU 版を作成し、ブラウザ上で実行可能にする**
    - **GLSL / HLSL 版をゲームエンジン向けに最適化**
    - **UE5 / Unity で Z80 エミュレータを動作可能にする**
20. **GPU 版の最適化**
    - **メモリアクセスの高速化**
    - **並列処理の最適化**
    - **パフォーマンス測定と最適化（Nsight, VTune, RenderDoc）**

---

### **9.5 動作確認 & デバッグ**
- **CPU 版 / C++ 版 / GPU 版をそれぞれステップ実行できるようにする**  
- **レジスタダンプを全バージョンで取得し、正しく処理されているか確認**  
- **GPU 版と CPU 版で結果を比較し、正しく動作するかを検証**  
- **Docker 内で GPU 版が正しく動作するかテスト（CUDA / OpenCL / WebGPU）**  
- **ゲームエンジン対応版の動作確認（Unity / UE5）**  

---

### **9.6 まとめ**
- **Dockerfile / docker-compose.yml を GPU 版にも対応させる**
- **C# / C++ 版の実装を進めつつ、GPU 版を開発**
- **最初に CUDA 版を作成し、OpenCL / WebGPU などへ移植**
- **GPU 版のデバッグをしやすくするため、ステップ実行モードを作成**
- **最終的に GPU 版のパフォーマンスを最適化し、Unity / UE5 にも対応させる**

