# 第4章：データの永続化

> 執筆者：ペンチョジュニア
> 最終更新：2026-05-21

## この章で学ぶこと

この章では、アプリを閉じてもデータが消えない「永続化」を2つの方法で学ぶ。`AppStorage`はユーザー設定のような小さなデータをUserDefaultsに保存し、`SwiftData`はメモのような構造化データをデータベースに保存する。`@Model`でデータモデルを定義し、`modelContext`で追加・削除を行い、`@Query`でデータを取得・監視するメモアプリを題材にする。

## 模範コードの全体像

```swift
import SwiftUI
import SwiftData

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// @main のAppファイルに記述：
// .modelContainer(for: Memo.self)

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button { isShowingSettings = true } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button { isShowingAddSheet = true } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) { MemoAddView() }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

struct MemoRow: View {
    let memo: Memo
    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title).font(.headline)
                Text(memo.content).font(.caption).foregroundStyle(.secondary).lineLimit(2)
                Text(memo.createdAt, style: .date).font(.caption2).foregroundStyle(.tertiary)
            }
            Spacer()
            if memo.isFavorite {
                Image(systemName: "star.fill").foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") { TextField("メモのタイトル", text: $title) }
                Section("内容") { TextEditor(text: $content).frame(minHeight: 200) }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") { TextField("タイトル", text: $memo.title) }
            Section("内容") { TextEditor(text: $memo.content).frame(minHeight: 200) }
            Section { Toggle("お気に入り", isOn: $memo.isFavorite) }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") { TextField("あなたの名前", text: $userName) }
                Section("表示設定") { Toggle("お気に入りを上に表示", isOn: $sortByFavorite) }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption).foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}
```

**このアプリは何をするものか：**

タイトルと内容を持つメモを作成・編集・削除できるアプリ。メモはSwiftDataによってアプリを閉じても端末に保存される。お気に入りマーク・名前・並び替え設定はAppStorageで保存されるため、こちらもアプリ再起動後も維持される。左上の歯車ボタンから設定画面を開き、名前やお気に入り優先表示をON/OFFできる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**
`@Model`マクロを付けることで、このクラスがSwiftDataの管理対象（データベースのテーブルに対応するエンティティ）になる。プロパティが列、各インスタンスが1行のレコードに対応する。`createdAt: Date = .now`はデフォルト引数で、作成日時を省略した場合は自動的に現在時刻が入る。

**なぜこう書くのか：**
`@Model`マクロがコンパイル時にSwiftDataに必要なコード（永続化・変更追跡・識別子生成など）を自動生成してくれる。構造体ではなくクラスを使うのは、SwiftDataが参照型を要求するため。`isFavorite`のデフォルト値を`false`にしておくことで、新規作成時にお気に入り未設定の状態をわざわざ指定しなくて済む。

**もしこう書かなかったら：**
`@Model`を省略するとSwiftDataがこのクラスを認識せず、`@Query`でデータを取得しようとしたときにコンパイルエラーになる。`struct`で書くとSwiftDataが受け付けない。

---

### データの追加・削除（modelContext）

```swift
@Environment(\.modelContext) private var modelContext

// 追加
let memo = Memo(title: title, content: content)
modelContext.insert(memo)

// 削除
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}
```

**何をしているか：**
`modelContext`はSwiftDataのデータベースへの「窓口」となるオブジェクト。`insert`でメモをデータベースに追加し、`delete`でデータベースから削除する。`@Environment(\.modelContext)`で環境から取得するため、ビューのどこでも同じコンテキストを使える。`onDelete`モディファイアがスワイプ削除のジェスチャーを検知して`deleteMemos`を呼ぶ。

**なぜこう書くのか：**
`modelContext.insert(memo)`を呼ぶだけで、SwiftDataが自動的にデータをSQLiteデータベースに書き込む。手動でファイルへの書き込みコードを書く必要がない。環境から取得する設計なので、親ビューで`.modelContainer`を設定しておけばすべての子ビューが同じコンテキストを共有できる。

**もしこう書かなかったら：**
`modelContext.insert`を呼ばずにインスタンスを作るだけでは、そのメモはメモリ上にしか存在せずアプリを閉じると消える。`deleteMemos`の中で`modelContext.delete`の代わりに配列から取り除くだけでは、画面上からは消えてもデータベースには残り続けてしまう。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

**何をしているか：**
`@Query`はSwiftDataのデータベースからデータを取得し、変更があるたびに自動でビューを更新するプロパティラッパー。`sort: \Memo.createdAt, order: .reverse`で作成日時の新しい順に並べている。`memos`はデータベースの中身と常に同期された配列として機能する。

**なぜこう書くのか：**
`@Query`を使うと、データベースへの変更（追加・削除・編集）が起きたとき自動的に`memos`が再取得され、SwiftUIがビューを再描画する。手動でデータを再読み込みするコードを書く必要がない。`\Memo.createdAt`はKeyPathという書き方で「MemoのcreatedAtプロパティを使ってソートする」という意味。

**もしこう書かなかったら：**
`@Query`を使わず普通の配列で管理すると、メモを追加してもリストが更新されず、手動でリロード処理を書く必要が出る。`sort`を指定しないと追加順序が不定になり、毎回違う順番で表示される可能性がある。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
```

**何をしているか：**
`@AppStorage`はUserDefaults（iOSがアプリごとに用意する小さなキーバリューストア）に値を読み書きするプロパティラッパー。`"sortByFavorite"`がUserDefaultsのキー名で、値を変更するとすぐにUserDefaultsに保存され、アプリを閉じて再起動しても値が維持される。`@State`と同様にビューを自動的に再描画する。

**なぜこう書くのか：**
ユーザーの名前や表示設定のような「小さくてシンプルな設定値」はSwiftDataのような重いデータベースを使う必要がなく、UserDefaultsで十分。`@AppStorage`はUserDefaultsを`@State`と同じ感覚で扱えるようにした薄いラッパーで、保存・読み込みのコードを意識せずに使える。デフォルト値（`false`, `""`）はまだ一度も保存されていない初回起動時に使われる。

**もしこう書かなかったら：**
`@State`だけで管理するとアプリを閉じるたびに設定がリセットされる。UserDefaultsを直接使うと`UserDefaults.standard.set(...)` / `UserDefaults.standard.bool(forKey:)`という煩雑なコードが必要になり、値変更時の画面更新も自分で実装しなければならない。

---

### @Bindableによる直接編集

```swift
struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            TextField("タイトル", text: $memo.title)
            TextEditor(text: $memo.content)
            Toggle("お気に入り", isOn: $memo.isFavorite)
        }
    }
}
```

**何をしているか：**
`@Bindable`はSwiftDataの`@Model`クラスのプロパティに`$`でバインディングを作れるようにするプロパティラッパー。`$memo.title`と書くと`TextField`がテキストを直接`memo.title`に書き込めるようになる。変更はSwiftDataが自動的に検知してデータベースに保存する。

**なぜこう書くのか：**
SwiftDataの`@Model`クラスは参照型なので変更を自動追跡できる。`@Bindable`を付けることで、変更のたびに手動で`modelContext.save()`を呼ばなくても変更が自動的にデータベースに反映される。

**もしこう書かなかったら：**
`@Bindable`なしで`$memo.title`と書こうとするとコンパイルエラーになる。コピーを`@State`に取って編集し、完了時に書き戻すという手間が必要になる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `@Model` | SwiftDataでクラスをデータベースエンティティにするマクロ（iOS 17+） | `@Model class Memo { var title: String ... }` |
| `@Query` | SwiftDataからデータを取得し変更を自動監視するプロパティラッパー | `@Query(sort: \Memo.createdAt, order: .reverse) var memos: [Memo]` |
| `modelContext` | データの追加・削除を行うSwiftDataの操作窓口 | `modelContext.insert(memo)` / `modelContext.delete(memo)` |
| `@Environment(\.modelContext)` | 環境からmodelContextを取得する | `@Environment(\.modelContext) private var modelContext` |
| `.modelContainer(for:)` | SwiftDataのデータベースをアプリ全体に提供するSceneモディファイア | `.modelContainer(for: Memo.self)` |
| `@AppStorage` | UserDefaultsにバインドされた永続的な設定値（`@State`と同じ使い勝手） | `@AppStorage("userName") private var userName: String = ""` |
| `@Bindable` | `@Model`クラスのプロパティへのバインディングを可能にするラッパー | `@Bindable var memo: Memo` → `$memo.title` が使える |
| `.onDelete(perform:)` | Listのスワイプ削除ジェスチャーに処理を紐付けるモディファイア | `.onDelete(perform: deleteMemos)` |
| `KeyPath` | プロパティへの参照を表す構文。`\型名.プロパティ名`と書く | `\Memo.createdAt`（`@Query`のsortに使用） |
| `Date = .now` | 現在時刻を表す`Date`のデフォルト引数 | `init(createdAt: Date = .now)` |

## 自分の実験メモ

**実験1：**
- やったこと：`@Query`の`order: .reverse`を`order: .forward`に変えた
- 結果：メモの表示順が逆になり、古いメモが上に、新しいメモが下に表示された
- わかったこと：`@Query`のソート順は1文字変えるだけで切り替えられる。データ取得の条件変更がビューのコードに直結している

**実験2：**
- やったこと：`@AppStorage("userName")`の値を設定画面で入力した後、アプリをシミュレータで完全に終了して再起動した
- 結果：ナビゲーションタイトルが「（名前）のメモ帳」として再起動後も維持されていた
- わかったこと：`@AppStorage`は本当にアプリを閉じても値が消えない。`@State`と見た目のコードが同じなのに、裏でUserDefaultsに保存されているという違いが体験として確認できた

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `@Query`と`@State`の違いは何ですか？どちらも変化したらビューを更新しますよね？
   **得られた理解：** `@State`はメモリ上のデータをビューに繋ぐ。`@Query`はSwiftDataのデータベースをビューに繋ぐ。`@State`はアプリを閉じると消えるが、`@Query`で取得したデータはデータベースに保存されているため再起動後も存在する。「どこにデータが保存されているか」が違う。

2. **質問：** `modelContext.insert(memo)`を呼ぶだけで本当にデータベースに保存されるのですか？`save()`みたいなメソッドは呼ばなくていいのですか？
   **得られた理解：** SwiftDataはデフォルトで自動保存（autosave）が有効になっている。`insert`や`delete`を呼ぶと変更がコンテキストに積まれ、一定のタイミング（ビューの更新時など）で自動的に保存される。明示的に`try modelContext.save()`を呼ぶこともできるが、通常は不要。これはCore Dataのときに毎回`save()`を書く必要があった点が改善されたSwiftDataの特徴。

3. **質問：** `@AppStorage`はどこにデータを保存しているのですか？SwiftDataとは何が違うのですか？
   **得られた理解：** `@AppStorage`はUserDefaultsという仕組みを使い、アプリのサンドボックス内のplistファイルに保存される。文字列・数値・Bool・URLなどシンプルな値向けで、構造体や配列（単純なもの以外）は直接保存できない。SwiftDataはSQLiteデータベースを使い、複雑な構造のオブジェクトや大量のデータを効率よく保存できる。使い分けは「設定値→AppStorage、構造化データ→SwiftData」。

## この章のまとめ

データの永続化には「何を保存するか」によって道具が変わる。シンプルな設定値（ON/OFF・名前など）は`@AppStorage`でUserDefaultsに保存し、構造化されたデータ（メモ・タスク・レコードなど）はSwiftDataを使う。SwiftDataは`@Model`でモデルを定義し、`modelContext`で操作し、`@Query`で取得・監視するという3ステップで動く。どちらも「データが変わるとビューが自動更新される」という点は`@State`と同じ設計思想で、SwiftUI全体を貫くパターンになっている。
