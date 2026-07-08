# 第8章：ウィジェット

> 執筆者：ペンチョジュニア
> 最終更新：2026-07-08

## この章で学ぶこと

この章では、WidgetKitを使ってホーム画面に表示できるウィジェットを実装する方法を学ぶ。日替わりで名言を表示する「QuoteWidget」を題材に、`TimelineProvider`によるウィジェットの更新タイミングの管理、`@Environment(\.widgetFamily)`によるサイズ別レイアウトの切り替え、そしてメインアプリとウィジェットでデータ（`QuoteStore`）をどう共有するかを学ぶ。

## 模範コードの全体像

```swift
// ============================================
// ■ メインアプリ側のコード（ContentView.swift）
// ============================================

import SwiftUI

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                VStack(spacing: 16) {
                    Text("今日の名言").font(.caption).foregroundStyle(.secondary)
                    Text("「\(todaysQuote.text)」")
                        .font(.title2).bold()
                        .multilineTextAlignment(.center)
                    Text("— \(todaysQuote.author)")
                        .font(.subheadline).foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(RoundedRectangle(cornerRadius: 16).fill(.blue.opacity(0.08)))
                .padding(.horizontal)

                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text).font(.body)
                        Text("— \(quote.author)").font(.caption).foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

// ============================================
// ■ 共有コード（QuoteStore.swift）
// メインアプリとウィジェットの両方から使う
// ============================================

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// ============================================
// ■ ウィジェット側のコード（QuoteWidget.swift）
// Widget Extension ターゲット内に記述する
// ============================================

import WidgetKit

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(date: Date(), quote: Quote(id: 0, text: "読み込み中...", author: ""))
    }

    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(date: Date(), quote: QuoteStore.todaysQuote())
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall: smallWidget
        case .systemMedium: mediumWidget
        default: mediumWidget
        }
    }

    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening").font(.caption).foregroundStyle(.blue)
            Text(entry.quote.text).font(.caption).bold()
                .multilineTextAlignment(.center).lineLimit(3)
            Text(entry.quote.author).font(.caption2).foregroundStyle(.secondary)
        }
        .padding(12)
    }

    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening").font(.title).foregroundStyle(.blue)
            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言").font(.caption2).foregroundStyle(.secondary)
                Text(entry.quote.text).font(.subheadline).bold()
                Text("— \(entry.quote.author)").font(.caption).foregroundStyle(.secondary)
            }
            Spacer()
        }
        .padding()
    }
}

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// ============================================
// ■ ウィジェットのエントリーポイント（QuoteWidgetBundle.swift）
// ============================================

@main
struct QuoteWidgetBundle: WidgetBundle {
    var body: some Widget {
        QuoteWidget()
    }
}
```

**このアプリは何をするものか：**

メインアプリを開くと「今日の名言」がハイライト表示され、その下に全7件の名言リストが並ぶ。ホーム画面にウィジェットを追加すると、アプリを開かなくても同じ「今日の名言」がSmall/Mediumのサイズで常時表示され、日付が変わると自動的に次の名言に切り替わる。

## コードの詳細解説

### TimelineProviderの3つのメソッド

```swift
struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry { ... }
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) { ... }
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) { ... }
}
```

**何をしているか：**
`placeholder`はウィジェットがまだデータを読み込めていない一瞬だけ表示される仮の見た目（今回は「読み込み中...」というダミーの名言）を返す。`getSnapshot`はウィジェットギャラリー（ウィジェットを選ぶ画面）でのプレビュー用に、実際のデータに近い1件のスナップショットを返す。`getTimeline`は実際にホーム画面へ配置された後、いつ・どんな内容を表示するかのスケジュール（`Timeline`）を返す。

**なぜこう書くのか：**
ウィジェットはメインアプリとは別のプロセスで動き、常に画面に張り付いているわけではなく、システムが決めたタイミングでしか更新できない。この3つのメソッドに役割を分けることで、「読み込み中の一瞬」「選択画面でのプレビュー」「実際に表示されている間の内容」という異なる文脈それぞれに適したデータをOSが要求できるようになっている。

**もしこう書かなかったら：**
`placeholder`がないとウィジェットを配置した瞬間に空白または不正な状態が一瞬表示される可能性がある。`getSnapshot`がないと、ウィジェットを追加する前のギャラリー画面でどんな見た目になるかユーザーが確認できない。

---

### Timeline(entries:, policy:)による更新スケジュール

```swift
let tomorrow = Calendar.current.startOfDay(
    for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
)
let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
```

**何をしているか：**
`entries: [entry]`で「今表示する内容」を1件だけ登録し、`policy: .after(tomorrow)`で「次にこのウィジェットの内容を再計算してよいのは明日の0時以降」とOSに伝えている。

**なぜこう書くのか：**
ウィジェットは自分の意思で「今すぐ更新して」とはできず、OSに更新タイミングを申告する仕組みになっている。名言は1日1回しか変わらない仕様なので、次の更新予定を「明日の0時」と正確に伝えることで、無駄な再計算・再描画のリクエストをOSに送らせずに済む。これはバッテリーとシステムリソースを節約するための設計。

**もしこう書かなかったら：**
`policy`を`.never`にすると一度表示した名言がずっと固定されたままになり、日付が変わっても名言が変わらなくなる。逆に`.atEnd`（配列の最後のエントリの直後に再計算）のような短い周期にすると、実際には必要ないタイミングでも更新リクエストが発生し、システムから「更新しすぎ」と判断されてウィジェット全体の更新頻度を制限されるおそれがある。

---

### @Environment(\.widgetFamily)によるサイズ別レイアウト

```swift
struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall: smallWidget
        case .systemMedium: mediumWidget
        default: mediumWidget
        }
    }
}
```

**何をしているか：**
`@Environment(\.widgetFamily)`で、このウィジェットが今どのサイズ（Small/Medium/Largeなど）でホーム画面に配置されているかを取得し、サイズごとに異なるレイアウト（`smallWidget`・`mediumWidget`）を出し分けている。

**なぜこう書くのか：**
ユーザーはホーム画面でウィジェットのサイズを自由に選べる。Smallサイズは面積が小さく引用文を3行までしか表示できないが、Mediumサイズは横に長く「今日の名言」というラベルや著者名を余裕を持って配置できる。1つの`TimelineEntry`（データ）から、サイズに応じて異なる見た目を組み立てる責務をこの`switch`文が担っている。

**もしこう書かなかったら：**
サイズ分岐をしないと、Small配置時にMedium用の横長レイアウトが押し込まれて文字が見切れたり、逆にSmall用の小さいレイアウトがMediumの余白の大部分を無駄にするといった、サイズに合わない見た目になる。

---

### メインアプリとウィジェットでのコード共有（QuoteStore）

```swift
struct QuoteStore {
    static let quotes: [Quote] = [ ... ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}
```

**何をしているか：**
名言データそのものと「今日の名言を選ぶロジック」を`QuoteStore`という1つの型にまとめ、メインアプリの`ContentView`とウィジェットの`QuoteProvider`の両方から`QuoteStore.todaysQuote()`という同じ呼び出しで参照している。`dayOfYear % quotes.count`は「1年の何日目か」を名言の件数で割った余りを使うことで、日付が変わるたびに規則的に（同じ日なら誰が呼んでも同じ）名言が決まる仕組みになっている。

**なぜこう書くのか：**
メインアプリとウィジェットは実行時には別プロセスとして動くが、「今日の名言は同じでなければならない」という要件がある。ロジックを2箇所に別々に書いてしまうと、片方だけ修正し忘れたときにアプリとウィジェットで表示される名言がズレてしまう。`QuoteStore`を両方のターゲットに追加する（共有ファイルにする）ことで、ロジックの二重管理を防いでいる。`dayOfYear`を使った決定的な計算にしているのも、乱数を使うと「アプリで見た名言」と「ウィジェットで見た名言」が一致しなくなるためで、同じ日付から必ず同じ結果を導く純粋な計算にする必要がある。

**もしこう書かなかったら：**
もしランダムに名言を選ぶ実装（`quotes.randomElement()`など）にすると、メインアプリを開くたびに違う名言が表示され、ホーム画面のウィジェットに表示されている名言と一致しなくなり、ユーザーが混乱する。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `TimelineProvider` | ウィジェットの内容と更新タイミングを定義するプロトコル | `struct QuoteProvider: TimelineProvider { ... }` |
| `TimelineEntry` | ウィジェットの1回分の表示内容と表示日時を表す型 | `struct QuoteEntry: TimelineEntry { let date: Date; let quote: Quote }` |
| `Timeline(entries:policy:)` | いつ・何を表示するかのスケジュールをOSに渡す型 | `Timeline(entries: [entry], policy: .after(tomorrow))` |
| `@Environment(\.widgetFamily)` | ウィジェットの現在のサイズを取得する環境値 | `@Environment(\.widgetFamily) var family` |
| `StaticConfiguration` | ユーザー設定を持たないシンプルなウィジェットの構成 | `StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in ... }` |
| `.containerBackground(_:for:)` | ウィジェットの背景を指定する（iOS 17以降必須） | `.containerBackground(.fill.tertiary, for: .widget)` |
| `WidgetBundle` | 複数のウィジェットを1つのExtensionにまとめるための入れ物 | `@main struct QuoteWidgetBundle: WidgetBundle { var body: some Widget { QuoteWidget() } }` |
| `Calendar.ordinality(of:in:for:)` | ある日付が特定の単位（年など）の中で何番目かを求める | `Calendar.current.ordinality(of: .day, in: .year, for: Date())` |

## 自分の実験メモ

**実験1：**
- やったこと：`getTimeline`内の`policy`を`.after(tomorrow)`から`.after(Date().addingTimeInterval(60))`（1分後）に変更した
- 結果：ウィジェットがより頻繁に再計算を要求するようになったが、実際に表示される名言自体は`todaysQuote()`のロジック上、日付が変わらない限り同じ結果だった
- わかったこと：`policy`は「いつ再計算してよいか」を決めるだけで、再計算した結果まで必ず変わるとは限らない。無駄に短い間隔を指定してもバッテリーを消費するだけで意味がないと実感した

**実験2：**
- やったこと：`QuoteStore.todaysQuote()`の計算式を`dayOfYear % quotes.count`から`Int.random(in: 0..<quotes.count)`に変えて、メインアプリとウィジェットを見比べた
- 結果：アプリを開くたびに違う名言が表示され、ホーム画面のウィジェットの名言と一致しなくなった
- わかったこと：同じロジックを共有していても、計算の中身が「決定的（同じ入力なら同じ出力）」でなければ、プロセスが違う場所から呼んだときに結果がズレてしまう

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** ウィジェットはなぜ自分の意思でいつでも再描画できないのですか？普通のSwiftUIビューと何が違うのですか？
   **得られた理解：** ウィジェットはホーム画面上で常時動いているアプリではなく、システム（springboard）が管理する軽量なプロセスとして表示されるだけ。バッテリーやリソースの消費を抑えるため、更新はOSがスケジュールした`Timeline`のタイミングでしか行われない。通常のSwiftUIビューが`@State`の変化で即座に再描画されるのとは根本的に仕組みが違うとわかった。

2. **質問：** `QuoteWidgetBundle`と`QuoteWidget`にどちらも`@main`がついていますが、これはエラーにならないのですか？
   **得られた理解：** `@main`はターゲット（実行ファイル単位）ごとに1つだけ許される。メインアプリのターゲットと、Widget Extensionのターゲットはビルド上は別の実行ファイルなので、それぞれに`@main`があっても問題ない。`QuoteWidgetBundle`がWidget Extension全体のエントリーポイントで、その中で`QuoteWidget`をリストアップする構造になっている。

3. **質問：** `QuoteStore`を「共有ファイルとして両方のターゲットに追加する」とコメントにありますが、具体的にどういう操作をすることですか？
   **得られた理解：** Xcodeでファイルを選択し、右側のFile Inspectorにある「Target Membership」で、メインアプリのターゲットとWidget Extensionのターゲットの両方にチェックを入れる操作を指す。1つの`.swift`ファイルの実体は1つのままで、コンパイル時にどちらのターゲットにも含めてビルドされる、という仕組みだと理解した。

## この章のまとめ

ウィジェットは「メインアプリの縮小版」ではなく、独立したライフサイクルで動く別プロセスという前提を理解することが最初の関門だった。`TimelineProvider`はその制約（自由に再描画できない）をうまく扱うための仕組みで、「いつ・何を表示するか」を前もってOSに申告するという発想がこれまでの章のリアルタイム更新（`@State`・`@Query`）とは大きく異なっていた。また、メインアプリとウィジェットという2つの独立したプロセスで表示内容を一致させるには、ロジックを共有ファイルにまとめ、かつそのロジック自体を「同じ入力なら必ず同じ出力になる」決定的な計算にしておく必要があるという、複数プロセス間の一貫性という新しい視点を得た。
