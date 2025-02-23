---
class: content
---

<div class="doc-header">
  <h1>プログラムの計測と最適化</h1>
  <div class="doc-author">江本光晴</div>
</div>

# プログラムの計測と最適化

プログラムのパフォーマンスは、アプリケーションの動作速度やユーザー体験に大きく影響を与えます。しかし、単に「速い」コードを書くことだけが重要ではないです。適切な計測をして、ボトルネックを特定し、効率的な最適化を行うことが求められます。

本書では、プログラムのパフォーマンス計測手法を説明して、計測データの分析から最適化までの流れを示します。なお、コード例には Swift および Python を利用します。

計測環境は、MacBook Pro 14 インチ 2021 / Apple M1 Pro / メモリ 32 GB / macOS Sequoia 15.3.1 です。ソフトウェアバージョンは Python 3.12.3、Swift 6.0.2 です。

## パフォーマンス計測の基礎

プログラムのパフォーマンスを向上させるためには、まず状態を計測することが必要です。計測なしに最適化を行うと、改善の方向性が分からず、逆にパフォーマンスが低下することもあります。そのため、計測を適切に実施し、改善点を明確にすることが重要です。

### 計測の基本指標

プログラムのパフォーマンスを測定する際に利用する主な指標は次のとおりです。

| 指標 | 説明 | 単位 |
| :-- | :-- | :-- |
| 実行時間 | コードが完了するまでの時間　| 秒またはミリ秒 |
| CPU使用率 | 処理完了にどの程度のCPUリソースを使用したか | % |
| メモリ使用量 | プログラムが消費するメモリのサイズ | KB や MB　|

これらの指標を適切に測定し、分析することで、最適化の方向性を定めることができます。

### CPU 使用率について

CPU 使用率は、プログラムがどの程度の CPU 時間を使用しているかを示す指標です。利用する計測ツールでは取得できない場合もありますが、その場合は計算時間から計算します。

$$
{\rm CPU}使用率 = \frac{ユーザー時間 + システム時間}{経過時間}
$$

| 指標 | 説明 |
| :-- | :-- |
| 経過時間 | 実際にかかった時間 |
| ユーザー時間 | CPUがユーザー空間でコードを実行した時間 |
| システム時間 | CPUがカーネル空間でコードを実行した時間　|

ユーザー時間とシステム時間は、CPU 時間とも呼ばれます。経過時間は CPU 時間以外のデータ入出力などの処理時間が含まれます。システムやアプリケーションの特性によって異なりますが、一般的には 70 ～ 80％ 程度がよいとされています。

たとえば、プログラムが CPU を 4 秒間利用して、総実行時間が 10 秒の場合、CPU 使用率は 40％ になります。この 40％ は CPU 時間の他が多いので、データ入出力の処理改善を試みる、またはハードウェア自体を性能高いものに入れ替える、などと評価されるでしょう。

## パフォーマンス計測

プログラムのパフォーマンスを測定する方法はいくつかありますが、Swift、Python、および GNU time コマンドを利用して計測する方法を紹介します。

計測ツールのインタフェースは次のようにします。計測ツールに、計測したい実行ファイルのパスを与えることで、その実行ファイルの実行時間、CPU使用率、およびメモリ使用量を計測します。

```shell
% ./measure -e {計測対象の実行ファイルのパス}
```

なお、本記事では省略していますが、付録に添付するサンプルでは試行回数や CSV 出力のオプションも対応しています。

### Swift による計測

Swift で計測ツールを作成します。紙面とコード量の関係で、分割して説明します。実際のコードは付録のサンプルを参考してください。まず、コマンドラインの引数を取得します。なお、対象は iOS ではなく、macOS です。

```swift
import Foundation

// 計測対象の実行ファイルのパス
var execPath: String? = nil

// 計測コマンド（今回は measure）を除いた引数
var args = CommandLine.arguments.dropFirst()

while let arg = args.first {
  switch arg {
    case "-e":
      args = args.dropFirst()
      if let value = args.first {
        executablePath = value
        args = args.dropFirst()
      } else {
        print("Error: -e オプションには実行ファイルのパスを指定してください。")
        exit(1)
      }
    default:
      print("不明な引数: \(arg)")
      args = args.dropFirst()
  }
}
```

計測対象の実行ファイルは、次のように実行します。

```swift
let process = Process()
process.executableURL = URL(fileURLWithPath: execPath)

do {
    try process.run()
} catch {
    print("プロセスの起動に失敗しました: \(error)")
    exit(1)
}
```

実行時間は、実行前後の時間を取得して、その差分とします。

```swift
// 試行開始時刻
let startTime = Date()

// 実行ファイルを実行する（略）

// 実行時間（秒）
let execTime = Date().timeIntervalSince(startTime)
print("実行時間: \(String(format: "%.3f", execTime)) 秒")
```

Swift で CPU 使用率とメモリ使用量を取得したかったのですが、困難でした。そこで、プロセスの情報を表示する ps コマンドを Swift から利用します。

```swift
func getProcessStats(pid: pid_t) -> (cpu: Double, rss: Double)? {
  let process = Process()
  process.executableURL = URL(fileURLWithPath: "/bin/ps")
  process.arguments = ["-p", "\(pid)", "-o", "%cpu=,rss="]

  let pipe = Pipe()
  process.standardOutput = pipe

  do {
    try process.run()
    process.waitUntilExit()
  } catch {
    return nil
  }

  let data = pipe.fileHandleForReading.readDataToEndOfFile()
  guard let output = String(data: data, encoding: .utf8) else {
    return nil
  }
  let values = output.trimmingCharacters(in: .whitespacesAndNewlines)
      .split(separator: " ", omittingEmptySubsequences: true)
  guard let values.count >= 2 else { 
    return nil
  }
  if let cpu = Double(values[0]), let rss = Double(values[1]){
    return (cpu, rss)
  }
  return nil
}
```

この getProcessStats(pid:) を利用して、計測対象の CPU 使用率とメモリ使用量を取得します。計測対象のプロセス ID を取得して、計測します。

```swift
let process = Process()
process.executableURL = URL(fileURLWithPath: execPath)

do {
  try process.run()
} catch {
  print("プロセスの起動に失敗しました: \(error)")
  exit(1)
}

// ps で計測するため、計測対象のプロセス ID を取得する
let pid = process.processIdentifier

// 試行中の最大値を記録する変数
var cpuRate = 0.0
var memoryValue = 0.0

// プロセスが終了するまで、0.1秒間隔で ps コマンドにより CPU とメモリを監視
while process.isRunning {
  if let stats = getProcessStats(pid: pid) {
    cpuRate = stats.cpu > cpuRate ? stats.cpu : cpuRate
    memoryValue = stats.rss > memoryValue ? stats.rss : memoryValue
  }
  usleep(100_000)  // 100ミリ秒待機
}
    
// プロセスの終了を待機（念のため）
process.waitUntilExit()
    
print("CPU 使用率: \(String(format: "%.2f", cpuRate))%")
print("メモリ使用量: \(String(format: "%.2f", memoryValue / 1_000.0)) MB")
```

改めて、今回の対象は iOS ではなく、macOS です。iOS アプリが対象なら、Xcode のデバッガーや Instruments を利用しましょう。

### Python による計測

Python でも計測プログラムを作りました。実装は Swift とほぼ同等です。Python でも計測のため ps を利用しました。まず pip を利用して psutil をインストールします。

```shell
pip install psutil
```

計測関数は次のとおりです。

```python
import psutil
import subprocess
import time

def monitor_process(executable_path):
  # 測定対象のプログラムを開始
  process = subprocess.Popen([executable_path], stdout=subprocess.PIPE)

  # 初期値を取得
  initial_cpu = psutil.cpu_percent(interval=0.1)
  initial_memory = psutil.virtual_memory().used

  # 最大値を記録するための変数
  max_cpu_usage = 0
  max_memory_usage = 0

  # 試行開始時刻
  start_time = time.time()

  # プロセスが終了するまでモニタリング
  while process.poll() is None:
    # 現在のCPUとメモリの使用量を取得
    cpu_usage = psutil.cpu_percent(interval=0.1)
    memory_info = psutil.virtual_memory().used

    # 最大値を更新
    max_cpu_usage = max(max_cpu_usage, cpu_usage)
    max_memory_usage = max(max_memory_usage, memory_info)

  end_time = time.time()

  # 実行時間を計算
  execution_time = end_time - start_time

  # メモリをMB単位にする
  mb_memory = (max_memory_usage - initial_memory) / (1024 ** 2)

  # 結果を表示
  print(f"実行時間: {execution_time:.2f} seconds")
  print(f"最大 CPU 使用率: {max_cpu_usage:.2f}%")
  print(f"最大メモリ使用量: {mb_memory:.2f} MB")
```

作成した monitor_process() を使って、計測対象を計測します。

```python
import argparse

if __name__ == "__main__":
  parser = argparse.ArgumentParser(description="measure exe file")
    
  # `-e` or `--exe` のオプション (実行ファイルのパス)
  parser.add_argument("-e", "--exe", required=True, help="file path")
    
  # 引数をパース
  args = parser.parse_args()

  # 計測する
  monitor_process(args.exe)
```

### GNU time を使用した計測

前節まで Swift や Python を使って計測ツールを作りました。しかしながら、内部で ps コマンドを利用しました。他の専用計測コマンドを利用するのがよいのではと、考えました。

macOS には計測コマンド time があります。ただし、このコマンドは時間計測できまずが、メモリ使用量が取得できません。そこで、メモリ使用量も計測できる GNU time を利用します。これは Homebrew でインストールできます。

```shell
brew install gnu-time
```

GNU time コマンドを使用すると、スクリプト全体の実行時間やメモリ使用量を簡単に測定できます。

```shell
% gtime node -v
v22.11.0
0.03user 0.01system 0:00.12elapsed 40%CPU (0avgtext+0avgdata 19360maxresident)k
0inputs+0outputs (12major+2646minor)pagefaults 0swaps
```

ただし、上記の実行結果を見てもらって分かるように、二次利用も見据えると結果表示が不安です。gtime には出力フォーマットのオプションがあるので、それを設定します。また、計測対象の指定など、簡単なシェルスクリプトを準備します。

```shell
# デフォルト値
EXECUTABLE=""

while getopts "e:" opt; do
  case $opt in
    e) EXECUTABLE="$OPTARG" ;;
    *) usage ;;
  esac
done

OUTPUT=$(gtime -f "%e %P %M" "$EXECUTABLE" 2>&1 >/dev/null)

REAL_TIME=$(echo "$OUTPUT" | awk '{print $1}')
CPU_USAGE_RAW=$(echo "$OUTPUT" | awk '{print $2}')
MEMORY_RAW=$(echo "$OUTPUT" | awk '{print $3}')

echo "経過時間 (REAL_TIME): $REAL_TIME 秒"
echo "CPU利用率 (CPU_USAGE): $CPU_USAGE_RAW"
echo "メモリ使用量 (MEMORY): $MEMORY_RAW MB"
```

３種類の計測ツールを紹介しました。紹介してなんですが、やはり専用の time を利用するのが、間違いないでしょう。たとえば、Swift のコードを Swift プロジェクト内で計測したい場合は Swift 一択ですが、そうでない場合は time を使うのがお勧めです。

## 実装差によるパフォーマンスの違い

プログラムの実装方法によって、パフォーマンスに大きな違いが生じます。特に、データ構造の選択やアルゴリズムの違いは、実行速度やリソース使用量に顕著な影響を及ぼします。具体的な例を挙げながら、実装差によるパフォーマンスの違いを分析します。

### データ構造の選択による影響

異なるデータ構造を選択することで、処理速度が大きく変わることもあります。たとえば、Python では list と set を使用した場合、検索の速度が異なります。

```python
import time

data = list(range(1000000))
start_time = time.time()
999999 in data  # リストの検索
print("リスト検索時間:", time.time() - start_time)

set_data = set(data)
start_time = time.time()
999999 in set_data  # セットの検索
print("セット検索時間:", time.time() - start_time)
```

||list|set|
|:-:|:-:|:-:|
| 検索時間（ミリ秒） | 6.49 | 0.04 |

この例では、list は線形検索をするため時間がかかりますが、set はハッシュテーブルを用いるため高速に検索できます。

### アルゴリズムの選択による影響

アルゴリズムの選択によっても、パフォーマンスに大きな違いが生じます。たとえば、ソートアルゴリズムを比較すると、バブルソートとクイックソートの処理時間が異なります。

```python
import random
import time

def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(0, n - i - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]

def quick_sort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[len(arr) // 2]
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return quick_sort(left) + middle + quick_sort(right)
```

バブルソートは ${\rm O}(n^2)$ の時間計算量をもつため、大きなデータセットでは極めて遅くなります。一方、クイックソートは ${\rm O}(n \log n)$ で動作するので、大幅に高速化されます。

```python
arr = [random.randint(0, 100000) for _ in range(10000)]

start_time = time.time()
bubble_sort(arr.copy())
print("バブルソート時間:", time.time() - start_time)

start_time = time.time()
quick_sort(arr.copy())
print("クイックソート時間:", time.time() - start_time)
```

なお、通常の開発では、ソート機能をアルゴリズムから実装することは、ほぼないです。標準フレームワークなどで提供されるソート関数を利用しましょう。

```python
start_time = time.time()
arr.copy().sort()
print("組み込みソート時間:", time.time() - start_time)
```

| | バブルソート | クイックソート | 組み込みソート |
| :-: | :-: | :-: | :-: |
| 計算時間（ミリ秒） | 3103.04 | 9.52 | 1.02 |

### 並列処理と逐次処理の違い

CPU のコアを活用することで、パフォーマンスを向上させることができます。たとえば、次の Python の関数を複数実行するシーンを考えます。

```python
# n秒待つだけの関数
def task(n):
  time.sleep(n)
  print(f"Task {n} done")
```

Python の multiprocessing モジュールを使用すると、複数のプロセスを並列に実行できます。

```python
import multiprocessing
import time

if __name__ == '__main__':

  # 直列処理
  start_time = time.time()
  task(2); task(2); task(2); task(2);
  print("直列処理時間:", time.time() - start_time)

  # 並列処理
  start_time = time.time()

  # 4つのプロセスを作成
  processes = [
    multiprocessing.Process(target=task, args=(2,)) for _ in range(4)
  ]

  # 各プロセスを開始
  for p in processes:
    p.start()

  # すべてのプロセスが終了するのを待つ
  for p in processes:
    p.join()

  print("並列処理時間:", time.time() - start_time)
```

直列実行すると 8 秒かかる処理も、並列化することで 2 秒で完了します。なお、マルチコアといえど、コア数は限られます。検証環境の MacBook Pro のコア数は 10（パフォーマンス: 8、効率性: 2）個です。同時並列数が 10 を超えると、パフォーマンスに影響してきます。

| 並列数 | 4 | 8 | 16 | 32 | 64 | 128 | 256 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| 時間（秒）| 2.06 | 2.06 |2.09 | 2.15 | 2.29 | 2.56 | 3.15 |

iPhone 16 シリーズに採用されている Apple A18 のコア数は 6（パフォーマンス: 2、効率性: 4）個です。OS 自体が制御に利用する分も除くと、実際に利用できるコアは少ないです。アプリの複数処理を安易に並列処理に回すと、すぐにコア全部を使い切ります。私も以前に、多数のダウンロード処理のパフォーマンスを改善するため、並列ダウンロードを実装しました。しかし、当初は並列数を適当に決めてたので、逆にパフォーマンスは下がりました。並列数は、利用端末がもつコア数から動的に設定しましょう。

## 効率的な最適化 V.S. 可読性や保守性

プログラムのパフォーマンス最適化を追求することは重要ですが、極端に最適化を重視するとコードの可読性や保守性が犠牲になることがあります。最適化のしすぎが問題となる例を挙げつつ、適切なバランスの取り方を考えます。

### 過度な最適化の例

- 可読性の低い極端な最適化

次の Python のコードは、リスト内包表記を使ってリストの要素の合計を求めるものですが、可読性は必ずしも高いとはいえないです。

```python
from functools import reduce

data = [1, 2, 3, 4, 5]
result = reduce(lambda x, y: x + y, data)
```

このコードは reduce() を利用してリストの合計を求めることができますが、合計を求めるならば sum() を利用した場合の方が可読性は高くなります。

```python
data = [1, 2, 3, 4, 5]
result = sum(data)
```

- 極端なループの最適化

Swift でリストの合計を求める際に、ループを最適化しすぎた例です。このコードでは &+ を使用してオーバーフローを回避しているので、Int.max を超えてもクラッシュしません。

```swift
let data = [1, 2, 3, 4, 5]
var result = 0
for i in 0..<data.count {
    result &+= data[i] // &+ はオーバーフローを防ぐための演算子
}
```

オーバーフローを考慮しなくても十分な場合もあります。また、オーバーフローが起こるようなレンジを取り扱うのであれば、別のアプローチを考えましょう。過度な最適化でしょう。

```swift
let data = [1, 2, 3, 4, 5]
let result = data.reduce(0, +)
```

Swift の標準フレームワークには sum がないので reduce を利用して、書き直しました。

### 可読性と保守性を優先すべきケース

- チーム開発時の可読性

コードは自分だけが読むわけではなく、他の開発者も改修する必要があります。最適化を優先するあまり、コードが複雑になりすぎると、あとから修正するのが難しくなります。また、自身が改修する場合でも、数週や数ヶ月も経てば、仕様や当時の実装意図を覚えてないものです。

- 将来的な拡張性

過度に最適化されたコードは、変更が難しくなります。たとえば、次のようなケースです。

```python
def process_data(data):
    return [x * 2 for x in data if x % 2 == 0]
```

このコードは、フィルタリングとマッピングの処理が一緒になっています。コードゴルフやワンライナーを目指すのであれば十分ですが、一般開発においては機能変更は難しくなります。

```python
def filter_even_numbers(data):
    return [x for x in data if x % 2 == 0]

def double_numbers(data):
    return [x * 2 for x in data]

def process_data(data):
    return double_numbers(filter_even_numbers(data))
```

このように関数を分けることで、機能変更を柔軟に対応できます。

### 最適化と可読性のバランスの取り方

プログラムを最適化する際には、次のような基準を考慮するとバランスが取れたコードを書けます。

1. ボトルネックの特定
    - 最適化を行う前に計測する
    - 本当に問題となっている箇所を特定する
2. パフォーマンス向上の効果
    - 最適化によるパフォーマンス向上が、コードの可読性や保守性を損なうはないか
    - または多少損なっても、劇的にパフォーマンスが改善されるなど、メリットの方が大きいか
3. 分かりやすい書き方
    - コードを他の開発者が見ても理解しやすいかどうかを意識する
4. コメントの活用
    - 最適化したコードには適切なコメントを追加し、意図が伝わるようにする

## まとめ

本記事では、Swift と Python を用いて、パフォーマンス計測を説明し、計測結果の分析方法や最適化の具体例を示しました。また、最適化の重要性と、可読性・保守性とのバランスの取り方について解説しました。最適化を過度に追求することでコードが読みにくくなる場合もあり、状況によってはシンプルな書き方を優先することが重要です。

### コードへの意識

私がプログラムのパフォーマンスを意識したキッカケはいくつかあります。その度に、私のコードの書き方や意識はアップデートされます。

大学院で研究している時、当時の私は MATLAB を利用してシミュレーション実験をしていました。実験は一晩中かかり、理論やパラメータを修正するのも大変でした。MATLAB はインタプリタ言語であり、並列計算に特化した言語です。MATLAB はその特性上、実装がパフォーマンスに直結します。C 言語のように実装していたコードが低パフォーマンスの原因でした。そこで、リファクタリングを行いました。ループを極力減らし、並列計算で一括計算するように修正しました。その結果、一晩かかっていた実験は、一時間で済むようになりました。プログミング言語の特性を知るのが大切だと認識しました。

そして、時は進み、私は iPhone 3GS でアプリ開発を始めました。当時のハードウェアは品素で、メモリが数 10 MB を超えると、アプリが不安定になり、40 ~ 50 MB でクラッシュするのは当たり前でした。ハードウェアの特性を理解して、実装するのは大切です。

また、アプリ開発において、共通する画面などが多く、共通化に走ったときもありました。しかし、共通化や汎用化された View は次第に肥大となり負債になりました。今は過剰な共通化や汎用化はせずに、コンポーネントレベルで切り分けています。コードの書き方は日々アップデートされます。

### よいコードとは

ソフトウェア開発は常に進化しています。ハードウェアの進歩に伴い、かつては不可欠だった最適化手法が不要になることもあれば、新たなパフォーマンスの課題が生まれることもあります。重要なのは、状況に応じた適切な選択をし続けることです。

また、チーム開発においては、パフォーマンスだけでなく、コードの可読性や保守性、拡張性を意識することが求められます。技術的負債を生まないために、過度な最適化を避け、適切な設計を心がけることが重要です。

この先、AI やビッグデータ、リアルタイム処理などの分野において、さらなるパフォーマンスの追求が必要になる場面も増えるでしょう。そのためにも、基本的なパフォーマンス計測と最適化の知識を持ち、状況に応じた適切な判断を下せるようになることが、開発者にとっての大きな強みになります。

最後に、プログラムのパフォーマンスは決して単一の指標で測るものではありません。パフォーマンス、可読性、保守性のバランスを考えながら、最適な設計を選択し続けることが、さらによい開発へとつながるでしょう。

## 付録

本記事で挙げたサンプルコードは GitHub のリポジトリを公開しています。興味ある方は見てください。

```url
https://github.com/mitsuharu/MeasureScripts
```

<hr class="page-break"/>

## Qiita 記事の案内

本記事は Qiita でも読むことができます。後述の URL または QR コードからアクセスしてください。加筆や修正などがある場合は Qiita 記事で対応しますのでご確認ください。また、ご質問等があれば、お気軽にコメントしてください。

```url
https://qiita.com/mitsuharu_e/items/xxxx
```

![記事のQRコード](./images_emoto/qr-code.jpg "記事のQRコード")
