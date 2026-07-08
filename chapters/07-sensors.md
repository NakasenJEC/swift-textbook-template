# 第7章（応用）：歩数計・移動距離トラッカー

> 執筆者：ペンチョジュニア
> 最終更新：2026-07-08

> **注意：** センサーはシミュレータでは正しく動作しない（歩数・GPS移動ともに実データが得られない）。この章のコードは実機での動作を前提にしている。

## この章で学ぶこと

この章では、CoreMotion（歩数計）とCoreLocation（移動距離・速度）を組み合わせて、今日の活動を記録する「活動トラッカー」アプリを学ぶ。第2章で学んだ`CLLocationManagerDelegate`パターンを再利用しつつ、新たに`CMPedometer`でステップ数を取得し、2種類のセンサーを1つの`@Observable`クラスにまとめる方法を学ぶ。

## 模範コードの全体像

```swift
// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        stepCount = 0
        distance = 0
        locations = []

        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }
                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    var elapsedTime: TimeInterval {
        guard let start = startTime else { return 0 }
        return Date().timeIntervalSince(start)
    }

    var distanceInKm: Double { distance / 1000 }

    var speedInKmh: Double { currentSpeed * 3.6 }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()
    @State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    timerSection
                    statsGrid
                    controlButton
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
            .onReceive(timer) { _ in
                // @Observableなので自動で更新される（タイマーは再描画のトリガー役）
            }
        }
    }

    private var timerSection: some View {
        VStack(spacing: 4) {
            Text(formatTime(tracker.elapsedTime))
                .font(.system(size: 48, weight: .thin, design: .monospaced))
            if tracker.isTracking {
                Text("計測中").font(.caption).foregroundStyle(.green)
            }
        }
        .padding()
    }

    private var statsGrid: some View {
        LazyVGrid(columns: [GridItem(.flexible()), GridItem(.flexible())], spacing: 16) {
            StatCard(icon: "figure.walk", value: "\(tracker.stepCount)", unit: "歩", color: .blue)
            StatCard(icon: "map", value: String(format: "%.2f", tracker.distanceInKm), unit: "km", color: .green)
            StatCard(icon: "flame", value: String(format: "%.0f", tracker.caloriesBurned), unit: "kcal", color: .orange)
            StatCard(icon: "speedometer", value: String(format: "%.1f", tracker.speedInKmh), unit: "km/h", color: .purple)
        }
    }

    private var controlButton: some View {
        Button {
            if tracker.isTracking { tracker.stopTracking() } else { tracker.startTracking() }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon).font(.title2).foregroundStyle(color)
            Text(value).font(.title).bold()
            Text(unit).font(.caption).foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(RoundedRectangle(cornerRadius: 12).fill(color.opacity(0.08)))
    }
}

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度").font(.caption).foregroundStyle(.secondary)
            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))
                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)
                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h").font(.caption).foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}
```

**このアプリは何をするものか：**

「スタート」ボタンを押すと歩数計とGPSの計測が同時に始まり、経過時間・歩数・移動距離・消費カロリー・現在の速度がリアルタイムで画面に表示される。速度はゲージ状のメーターでも視覚的に確認できる。「ストップ」を押すと計測が止まる。実機のセンサーがないと歩数・速度がどちらも取得できないため、シミュレータでは数値が動かない。

## コードの詳細解説

### 2種類のセンサーを1つのクラスにまとめる設計

```swift
@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    private let pedometer = CMPedometer()
    private let locationManager = CLLocationManager()
    ...
}
```

**何をしているか：**
歩数を扱う`CMPedometer`（CoreMotionフレームワーク）と、位置・速度を扱う`CLLocationManager`（CoreLocationフレームワーク）という、由来の異なる2つのセンサーAPIを`ActivityTracker`という1つの`@Observable`クラスの中にまとめて持たせている。

**なぜこう書くのか：**
歩数と移動距離・速度はどちらも「今の活動」を構成する情報であり、`ContentView`側から見れば「1つの活動データソース」として扱いたい。もし`CMPedometer`と`CLLocationManager`を別々の`@Observable`クラスに分けると、`ContentView`は2つの状態オブジェクトを個別に監視する必要が出て、`isTracking`のようなアプリ全体の状態をどちらが管理するのかも曖昧になる。1つのクラスにまとめることで「計測を始める/止める」という操作を`startTracking()`/`stopTracking()`という単一の窓口に集約できる。

**もしこう書かなかったら：**
別クラスに分けた場合、`ContentView`側で「歩数計測は始まったが位置情報はまだ止まっている」といった中途半端な状態が起こりやすくなり、開始・停止のタイミングを呼び出し側が毎回2箇所で管理しなければならなくなる。

---

### [weak self]でメモリリークを防ぐ

```swift
pedometer.startUpdates(from: Date()) { [weak self] data, error in
    guard let self = self, let data = data else { return }
    DispatchQueue.main.async {
        self.stepCount = data.numberOfSteps.intValue
        ...
    }
}
```

**何をしているか：**
`CMPedometer`にステップ更新のコールバッククロージャを渡す際、`[weak self]`をつけて`self`（`ActivityTracker`インスタンス自身）への参照を「弱参照」にしている。

**なぜこう書くのか：**
このクロージャは`pedometer`に保持され続け、歩数が更新されるたびに繰り返し呼ばれる。もし`self`を強参照のまま`self.stepCount = ...`のように書くと、「`pedometer`がクロージャを持つ」→「クロージャが`self`を持つ」→「`self`（ActivityTracker）が`pedometer`を持つ」という循環参照（retain cycle）が生まれ、`ActivityTracker`がどこからも使われなくなってもメモリから解放されなくなる。`[weak self]`にすることでこの環を断ち切り、`ActivityTracker`が不要になったタイミングで正しく解放できる。

**もしこう書かなかったら：**
`[weak self]`を外して`self`を強参照にすると、画面を閉じて`ContentView`が破棄されても`ActivityTracker`とそのクロージャが互いを参照し続けたままメモリに残り続ける「メモリリーク」が発生する。今回のアプリは1画面しかないので気づきにくいが、複数の画面を行き来するアプリでは徐々にメモリを圧迫していく。

---

### CLLocationManagerDelegateとの併用

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }
    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**
第2章の応用編で学んだ`CLLocationManagerDelegate`のメソッドをそのまま利用し、位置が更新されるたびに`currentSpeed`（速度）と`locations`（軌跡の配列）を更新している。`max(0, location.speed)`としているのは、GPSの精度上、静止時に`speed`が微小な負の値になることがあるためで、負の速度を0に丸めている。

**なぜこう書くのか：**
`CLLocationManager`は「位置が変わった」というイベントをdelegateパターンで通知する設計になっており、`didUpdateLocations`はその通知を受け取るための決まったメソッド名。第2章ですでに学んだ仕組みをそのまま再利用できることが、このパターンを最初に理解しておく価値を裏付けている。

**もしこう書かなかったら：**
`max(0, ...)`をせずに`location.speed`をそのまま使うと、静止しているのに速度メーターがわずかにマイナス方向に振れたり、`String(format: "%.1f", speed)`の表示が`-0.0`のような不自然な値になることがある。

---

### 計算プロパティによる単位変換

```swift
var distanceInKm: Double { distance / 1000 }

var speedInKmh: Double { currentSpeed * 3.6 }

var caloriesBurned: Double {
    Double(stepCount) * 0.04
}
```

**何をしているか：**
`CLLocationManager`や`CMPedometer`から得られる生データはメートル（`distance`）・秒速メートル（`currentSpeed`）という「SI単位系」だが、それを人間が読みやすいキロメートル・時速・カロリーに変換する計算プロパティを用意している。

**なぜこう書くのか：**
`currentSpeed * 3.6`は秒速（m/s）を時速（km/h）に変換する定番の係数（1時間=3600秒、1km=1000mなので `3600/1000 = 3.6`）。センサーの生データをそのままStateとして持たず、表示直前に計算プロパティで変換することで、`stepCount`や`distance`といった「元データ」は常にセンサーが返す生の値のまま保持できる。表示用の単位はいつでも計算し直せるので、元データを二重に持つ必要がない。

**もしこう書かなかったら：**
`distanceInKm`のような値も普通の保存型プロパティにしてしまうと、`distance`が更新されるたびに`distanceInKm`も手動で更新するコードが必要になり、更新し忘れるとどちらかの値が古いままになるというバグの温床になる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `CMPedometer` | 歩数・歩行距離を計測するCoreMotionのAPI（実機専用） | `pedometer.startUpdates(from: Date()) { data, error in ... }` |
| `CMPedometer.isStepCountingAvailable()` | 端末が歩数計測に対応しているかを事前確認する型メソッド | `isPedometerAvailable = CMPedometer.isStepCountingAvailable()` |
| `[weak self]` | クロージャ内で`self`を弱参照にし、循環参照を防ぐキャプチャリスト | `pedometer.startUpdates(...) { [weak self] data, error in ... }` |
| `Timer.publish(every:on:in:).autoconnect()` | 一定間隔でイベントを発行するCombineのタイマー | `Timer.publish(every: 1, on: .main, in: .common).autoconnect()` |
| `.onReceive(_:)` | Combineのpublisherから発行された値を受け取ってビューを更新する | `.onReceive(timer) { _ in ... }` |
| `location.speed` | `CLLocation`が持つ、その瞬間の移動速度（m/s） | `currentSpeed = max(0, location.speed)` |
| `Circle().trim(from:to:)` | 円の一部分だけを描画する（ゲージ・メーターの表現に使う） | `Circle().trim(from: 0, to: 0.75)` |

## 自分の実験メモ

**実験1：**
- やったこと：`[weak self]`を外して`self`を強参照にしたビルドを試した（Instrumentsは未使用、コードレベルの確認のみ）
- 結果：ビルド自体は通ったが、AIに聞いたところ複数画面をまたぐ設計にした場合はメモリリークの原因になると説明された
- わかったこと：クロージャに`self`を渡すときは、そのクロージャがどれくらい長く保持されるか（一度きりか、繰り返し呼ばれ続けるか）を意識して`[weak self]`の要否を判断する必要がある

**実験2：**
- やったこと：`caloriesBurned`の計算式`Double(stepCount) * 0.04`を`* 0.05`に変えてみた
- 結果：同じ歩数でも表示されるカロリーが増えた
- わかったこと：この計算はあくまで簡易的な目安であり、実際の消費カロリーは体重・歩幅・年齢などによって変わる。模範コードのコメントにも「簡易計算」と明記されていた理由が実感できた

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `Timer.publish(every: 1, on: .main, in: .common).autoconnect()`をわざわざ使っているのに、`.onReceive`のクロージャの中身が空（コメントだけ）なのはなぜですか？
   **得られた理解：** `@Observable`なプロパティ（`stepCount`など）はセンサーのコールバック側で更新されるたびに自動でビューを再描画するが、`elapsedTime`（経過時間）は`Date().timeIntervalSince(start)`のように「参照した瞬間の時刻」から計算される値で、勝手には変化を通知してくれない。タイマーは「1秒ごとにビューを再描画させるきっかけ」だけを作るために存在しており、`elapsedTime`の表示を更新する役目を担っている。

2. **質問：** `CMPedometer.isStepCountingAvailable()`をなぜ`init`の中で1回だけ呼んでいるのですか？計測を始めるたびに確認しなくていいのですか？
   **得られた理解：** 歩数計測に対応しているかどうかは端末のハードウェア性能によって決まる固定の情報で、アプリの実行中に変化するものではない。毎回確認する必要がないため、インスタンス生成時に1度だけ確認して`isPedometerAvailable`に結果を保持しておけば十分、という設計だと理解した。

3. **質問：** `location.speed`は`max(0, ...)`で0未満を切り捨てていますが、GPSの速度がマイナスになることが本当にあるのですか？
   **得られた理解：** GPSは複数の衛星からの信号を基に位置を推定する仕組み上、静止していても数センチ〜数メートルの誤差でわずかに位置がブレる。そのブレを基に速度を計算すると、理論上ありえないはずの負の値が瞬間的に出ることがある。`max(0, ...)`は、こうしたセンサーの物理的な誤差を吸収するための現実的なガードだとわかった。

## この章のまとめ

センサーを扱うコードは、UIKit/SwiftUIのAPIというより「ハードウェアの癖」とどう付き合うかが中心になる回だった。歩数・位置情報という由来の異なる2つのセンサーを1つの`@Observable`クラスに集約して単一の開始・停止操作にまとめる設計、クロージャに`self`を渡すときの`[weak self]`によるメモリ管理、GPSの微小な誤差を`max(0, ...)`で吸収する現実的な処理など、どれも「センサーは理想値を返すとは限らない」という前提に立って書かれていることが、実機でしか検証できない理由も含めて理解できた。
