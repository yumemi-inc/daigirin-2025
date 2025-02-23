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
{\rm CPU}使用率 = \frac{（ユーザー時間+システム時間）}{経過時間}
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

Swift でコマンドラインの引数を取得するには、次のようにします。なお、iOS ではなく、macOS が対象です。

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

計測対象の実行ファイルを実行する。

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

実行時間

```swift
// 試行開始時刻
let startTime = Date()

// 実行ファイルを実行する

// 実行時間（秒）
let execTime = Date().timeIntervalSince(startTime)

print("実行時間: \(String(format: "%.3f", execTime)) 秒")
```

Swift で CPU 使用率とメモリ使用量を取得するのは困難だったので、プロセスの情報を表示する ps コマンドを


```swift
// MARK: - プロセスの状態を取得する関数
/// 指定した PID のプロセスについて、ps コマンドで %CPU と RSS を取得する。
/// - Parameter pid: 監視対象のプロセスID
/// - Returns: (cpu: CPU使用率[%], rss: Resident Set Size[KB]) のタプル。取得できなければ nil を返す。
func getProcessStats(pid: pid_t) -> (cpu: Double, rss: Double)? {
let psPath = "/bin/ps"
    let psProcess = Process()
    psProcess.executableURL = URL(fileURLWithPath: psPath)
    // ps コマンドで %CPU と RSS（常駐セットサイズ）を取得
    psProcess.arguments = ["-p", "\(pid)", "-o", "%cpu=,rss="]

    let pipe = Pipe()
    psProcess.standardOutput = pipe

    do {
        try psProcess.run()
        psProcess.waitUntilExit()
    } catch {
        return nil
    }

    let data = pipe.fileHandleForReading.readDataToEndOfFile()
    guard let output = String(data: data, encoding: .utf8) else {
        return nil
    }
    // 出力例: " 0.0  1234"
    let components = output.trimmingCharacters(in: .whitespacesAndNewlines)
        .split(separator: " ", omittingEmptySubsequences: true)
    if components.count >= 2,
        let cpu = Double(components[0]),
        let rss = Double(components[1])
    {
        return (cpu, rss)
    }
    return nil
}
```

```swift
    let process = Process()
    process.executableURL = URL(fileURLWithPath: execPath)

    do {
        try process.run()
    } catch {
        print("プロセスの起動に失敗しました: \(error)")
        exit(1)
    }

    // プロセスIDを取得
    let pid = process.processIdentifier

    // 試行中の最大値を記録する変数
    var maxCpuObserved = 0.0
    var maxMemoryObserved = 0.0

    // プロセスが終了するまで、0.1秒間隔で ps コマンドにより CPU とメモリを監視
    while process.isRunning {
        if let stats = getProcessStats(pid: pid) {
            if stats.cpu > maxCpuObserved { maxCpuObserved = stats.cpu }
            if stats.rss > maxMemoryObserved { maxMemoryObserved = stats.rss }
        }
        usleep(100_000)  // 100ミリ秒待機
    }
    
    // プロセスの終了を待機（念のため）
    process.waitUntilExit()
    
    print("\n--- 平均結果 (\(results.count) 試行) ---")
print("最大 CPU 使用率: \(String(format: "%.2f", maxCpuObserved))%")
print("最大メモリ使用量: \(String(format: "%.2f", maxMemoryObserved / 1_000.0)) MB")
}

```

なお、iOS アプリが対象なら、Xcode のデバッガーや Instruments を利用しましょう。


### Python による計測

Python でも計測プログラムを作りました。実装は Swift とほぼ同等です。Python でも計測のため psutil を利用して

```shell
pip install psutil
```

計測関数

```
import psutil
import subprocess
import time
import argparse

def monitor_process(executable_path):
    # 測定対象のプログラムを開始
    process = subprocess.Popen([executable_path], stdout=subprocess.PIPE)

    # 初期値を取得
    initial_cpu = psutil.cpu_percent(interval=0.1)  # 実行前のCPU使用率
    initial_memory = psutil.virtual_memory().used  # 実行前のメモリ使用量 (バイト)

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

    # 結果を表示
    print("\n--- Monitoring Results ---")
    print(f"実行時間: {execution_time:.2f} seconds")
    print(f"最大 CPU 使用率: {max_cpu_usage:.2f}%")
    print(f"最大メモリ使用量: {(max_memory_usage - initial_memory) / (1024 ** 2):.2f} MB")
```

```
import argparse

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Monitor CPU and memory usage of an executable.")
    
    # `-e` or `--exe` のオプション (実行ファイルのパス)
    parser.add_argument("-e", "--exe", required=True, help="Path to the executable file")
    
    # `-c` or `--count` のオプション (数値)
    parser.add_argument("-c", "--count", type=int, required=False, help="Number of iterations")

    # 引数をパース
    args = parser.parse_args()

    # 受け取った値の表示
    print(f"Executable Path: {args.exe}")
    print(f"Count: {args.count}")

    monitor_process(args.exe)
```    



### GNU time を使用した計測


macOS には計測コマンド time があります。ただし、このメモリ使用量がしゅとくできません。そこで、GNU time を利用します。Homebrew でインストールできます。

```
brew install gnu-time
```


GNU `time` コマンドを使用すると、スクリプト全体の実行時間やメモリ使用量を簡単に測定できます。

```sh
gtime -f "%e %P %M" ls
# 実行時間、CPU利用率、メモリ利用量、
```

シェルスクリプトを

```sh
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
echo "CPU利用率 (CPU_USAGE): $CPU_USAGE_RAW %"
echo "メモリ使用量 (MEMORY): $MEMORY_RAW MB"
```


## 実装差によるパフォーマンスの違い

プログラムの実装方法によって、パフォーマンスに大きな違いが生じることがあります。特に、データ構造の選択やアルゴリズムの違いは、実行速度やリソース使用量に顕著な影響を及ぼします。本節では、具体的な例を挙げながら、実装差によるパフォーマンスの違いを分析します。

### 5.1 データ構造の選択による影響

異なるデータ構造を選択することで、処理速度が大きく変わることがあります。例えば、Pythonでは `list` と `set` を使用した場合、検索の速度が異なります。

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

この結果、`list` は線形検索を行うため時間がかかりますが、`set` はハッシュテーブルを用いるため高速に検索できます。

### 5.2 アルゴリズムの選択による影響

アルゴリズムの選択によっても、パフォーマンスに大きな違いが生じます。例えば、ソートアルゴリズムを比較すると、バブルソートとクイックソートの処理時間が異なります。

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

arr = [random.randint(0, 100000) for _ in range(10000)]

start_time = time.time()
bubble_sort(arr.copy())
print("バブルソート時間:", time.time() - start_time)

start_time = time.time()
quick_sort(arr.copy())
print("クイックソート時間:", time.time() - start_time)
```

この結果、バブルソートは O(n^2) の時間計算量を持つため大きなデータセットでは極めて遅くなりますが、クイックソートは O(n log n) で動作し、大幅に高速化されます。

なお、通常の開発では、ソート機能をアルゴリズムから実装することは、ほぼないです。標準フレームワークなどで提供されるソート関数を利用することで十分です。


```python
start_time = time.time()
arr.copy().sort()
print("組み込みソート時間:", time.time() - start_time)
```

| | バブルソート | クイックソート | 組み込みソート |
| :-: | :-: | :-: | :-: |
| 計算時間（ミリ秒） | 3103.04 | 9.52 | 1.02 |


### 5.3 並列処理と逐次処理の違い

CPUのコアを活用することで、パフォーマンスを向上させることができます。例えば、Python の `multiprocessing` モジュールを使用すると、複数のプロセスを並列に実行できます。

```python
import multiprocessing
import time

# n秒待つだけの関数
def task(n):
    time.sleep(n)
    print(f"Task {n} done")

if __name__ == '__main__':

    # 直列処理
    start_time = time.time()
    task(2)
    task(2)
    task(2)
    task(2)
    print("直列処理時間:", time.time() - start_time)

    # 並列処理
    start_time = time.time()

    # 4つのプロセスを作成
    processes = [multiprocessing.Process(target=task, args=(2,)) for _ in range(4)]

    # 各プロセスを開始
    for p in processes:
        p.start()

    # すべてのプロセスが終了するのを待つ
    for p in processes:
        p.join()

    print("並列処理時間:", time.time() - start_time)
```

逐次実行すると 8 秒かかる処理も、並列化することで 2 秒で完了します。


## 効率的な最適化 V.S. 可読性や保守性

プログラムのパフォーマンス最適化を追求することは重要ですが、極端に最適化を重視するとコードの可読性や保守性が犠牲になることがあります。最適化のしすぎが問題となる例を挙げつつ、適切なバランスの取り方を考えます。

### 過度な最適化の例

#### 可読性の低い極端な最適化

次の Python のコードは、リスト内包表記を使ってリストの要素の合計を求めるものですが、可読性が非常に低いものになっています。

```python
from functools import reduce

data = [1, 2, 3, 4, 5]
result = reduce(lambda x, y: x + y, data)
print(result)
```

このコードは `reduce()` を使うことで一行でリストの合計を求めることができますが、`sum()` を使うよりも可読性が低く、初心者には理解しにくいものです。

```python
data = [1, 2, 3, 4, 5]
result = sum(data)
print(result)
```

こちらの方が直感的で理解しやすく、メンテナンスがしやすいです。

#### 極端なループの最適化

次に、Swift でリストの合計を求める際に、ループを最適化しすぎた例です。

```swift
let data = [1, 2, 3, 4, 5]
var result = 0
for i in 0..<data.count {
    result &+= data[i] // &+ はオーバーフローを防ぐための演算子
}
print(result)
```

このコードでは `&+` を使用してオーバーフローの警告を避けていますが、通常の `+` を使用しても十分な場合が多く、過度な最適化になっています。

```swift
let data = [1, 2, 3, 4, 5]
let result = data.reduce(0, +)
print(result)
```

これにより、コードがシンプルになり、他の開発者が読みやすくなります。

### 可読性と保守性を優先すべきケース

#### チーム開発時の可読性

コードは自分だけが読むわけではなく、他の開発者もメンテナンスする必要があります。最適化を優先するあまり、コードが複雑になりすぎると、後から修正するのが難しくなります。

#### 将来的な拡張性

過度に最適化されたコードは、変更が難しくなることがあります。例えば、次のようなケースです。

```python
def process_data(data):
    return [x * 2 for x in data if x % 2 == 0]
```

このコードは一行で書かれていますが、フィルタリング (`if x % 2 == 0`) とマッピング (`x * 2`) の処理が一緒になっており、後から機能を追加しにくいです。

```python
def filter_even_numbers(data):
    return [x for x in data if x % 2 == 0]

def double_numbers(data):
    return [x * 2 for x in data]

def process_data(data):
    return double_numbers(filter_even_numbers(data))
```

このように関数を分けることで、後から処理を追加しやすくなります。

### 最適化と可読性のバランスの取り方

プログラムを最適化する際には、次のような基準を考慮するとバランスが取れたコードを書けます。

1. **ボトルネックの特定:** 最適化を行う前に、計測を行い、本当に問題となっている箇所を特定する。
2. **パフォーマンス向上の効果:** 最適化による速度向上が、コードの可読性や保守性を損なうほどの価値があるかを検討する。
3. **分かりやすい書き方:** コードを他の開発者が見ても理解しやすいかどうかを意識する。
4. **コメントの活用:** 最適化したコードには適切なコメントを追加し、意図が伝わるようにする。


## まとめ

本書では、SwiftとPythonを用いたパフォーマンス計測手法を詳細に解説し、計測結果の分析方法や最適化の具体例を示しました。また、最適化の重要性と、可読性・保守性とのバランスの取り方について解説しました。

最適化を過度に追求することでコードが読みにくくなる場合があり、状況によってはシンプルな書き方を優先することが重要です。


