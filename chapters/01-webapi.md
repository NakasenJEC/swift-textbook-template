# 第1章：WebAPIの基本

> 執筆者：ペンチョジュニア
> 最終更新：2026-05-20

## この章で学ぶこと

この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。iTunes Search APIを使って音楽を検索し、結果をリスト表示するアプリを題材にする。キーワードは「Codable」「async/await」「URLSession」の3つ。

## 模範コードの全体像

```swift
import SwiftUI

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?
    var id: Int { trackId }
}

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)
                    Button("検索") {
                        Task { await searchMusic() }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                if isLoading {
                    ProgressView("検索中...").padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView("曲を検索してみよう", systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください"))
                } else {
                    List(songs) { song in SongRow(song: song) }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed) else { return }
        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
        guard let url = URL(string: urlString) else { return }
        isLoading = true
        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            songs = []
        }
        isLoading = false
    }
}

struct SongRow: View {
    let song: Song
    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))
            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName).font(.headline).lineLimit(1)
                Text(song.artistName).font(.subheadline).foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}
```

**このアプリは何をするものか：**

テキストフィールドにアーティスト名を入力して「検索」ボタンを押すと、iTunes Search APIにリクエストを送り、マッチした曲のリスト（アルバムアート・曲名・アーティスト名）を画面に表示する。APIキー不要で誰でもすぐに動かせる。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?
    var id: Int { trackId }
}
```

**何をしているか：**
iTunesのAPIが返すJSONの「形」をSwiftの構造体として定義している。`SearchResponse`がJSON全体に対応し、その中の`results`配列の各要素が`Song`に対応する。

**なぜこう書くのか：**
`Codable`プロトコルを採用することで、`JSONDecoder`が自動的にJSONのキー名とSwiftのプロパティ名を照合してくれる。手動でJSONをパースするコードを書かなくて済む。`Identifiable`を採用し`id`を`trackId`から計算することで、SwiftUIの`List`が各要素を区別できるようになる。

**もしこう書かなかったら：**
`Codable`がなければJSONを`[String: Any]`として手動で解析し、全フィールドをキャストするコードが必要になる。`Identifiable`がなければ`List(songs, id: \.trackId)`のように書く必要が出る。

---

### API通信の処理

```swift
func searchMusic() async {
    guard let encodedText = searchText.addingPercentEncoding(
        withAllowedCharacters: .urlQueryAllowed) else { return }
    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
    guard let url = URL(string: urlString) else { return }
    isLoading = true
    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)
        songs = response.results
    } catch {
        songs = []
    }
    isLoading = false
}
```

**何をしているか：**
検索文字列をURL安全な形にエンコードし、iTunes APIへのURLを組み立てる。`URLSession.shared.data(from:)`でデータを取得し、`JSONDecoder`でSwiftの型にデコードして`songs`配列に代入する。

**なぜこう書くのか：**
`async/await`を使うことで、本来コールバック地獄になるところを上から下に読めるシンプルなコードで書ける。`await`は「ここで一時停止してネットワーク応答を待つ」という意味で、メインスレッドをブロックしない。`addingPercentEncoding`は日本語やスペースなどURLに使えない文字を`%XX`形式に変換するため必要。

**もしこう書かなかったら：**
`addingPercentEncoding`をしないと日本語アーティスト名の検索が常に失敗する。`isLoading`フラグを使わないと、検索中に画面が固まっているように見えてユーザーが不安になる。

---

### ビューの構成

```swift
if isLoading {
    ProgressView("検索中...").padding()
    Spacer()
} else if songs.isEmpty {
    ContentUnavailableView("曲を検索してみよう", systemImage: "music.note",
        description: Text("アーティスト名を入力して検索ボタンを押してください"))
} else {
    List(songs) { song in SongRow(song: song) }
}
```

**何をしているか：**
3つの状態（読み込み中・結果なし・結果あり）を`if-else if-else`で切り替えて表示している。

**なぜこう書くのか：**
`@State`変数（`isLoading`, `songs`）が変化するたびにSwiftUIが自動的にビューを再描画するため、状態をフラグで管理するだけでUIが自動更新される。`ContentUnavailableView`はiOS 17以降の標準コンポーネントで、空状態を適切に表現できる。

**もしこう書かなかったら：**
空状態の表示がないと、検索前がただの真っ白な画面になり、何をすればいいか分からなくなる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `Codable` | JSONとSwift構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| `async/await` | 非同期処理を同期的に書ける構文（iOS 15+） | `let (data, _) = try await URLSession.shared.data(from: url)` |
| `URLSession.shared.data(from:)` | URLからデータをダウンロードするAsync API | `try await URLSession.shared.data(from: url)` |
| `JSONDecoder().decode(_:from:)` | JSONデータをCodable型にデコード | `try JSONDecoder().decode(SearchResponse.self, from: data)` |
| `AsyncImage` | URLから画像を非同期でダウンロードするSwiftUIビュー | `AsyncImage(url: URL(string: urlString)) { image in ... }` |
| `ContentUnavailableView` | 空状態を表示する標準コンポーネント（iOS 17+） | `ContentUnavailableView("タイトル", systemImage: "icon")` |
| `addingPercentEncoding` | 文字列をURL安全な形にエンコード | `text.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)` |

## 自分の実験メモ

**実験1：**
- やったこと：`limit=25`を`limit=50`に変えた
- 結果：取得件数が50件になった
- わかったこと：APIのクエリパラメータを変えるだけで挙動を変えられる

**実験2：**
- やったこと：`country=jp`を`country=us`に変えた
- 結果：米国のiTunesで検索されるため検索結果が変わった
- わかったこと：APIのパラメータがレスポンスデータに直接影響する

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `Codable`はどうやってJSONのキーとSwiftのプロパティを対応させているのか？
   **得られた理解：** デフォルトではプロパティ名とJSONキーが完全一致する必要がある。キー名が違う場合は`CodingKeys`列挙型でマッピングを定義できる。

2. **質問：** `async`と`await`は何が違うのか、なぜ両方必要なのか？
   **得られた理解：** `async`は「この関数は非同期で動く」という宣言。`await`は「ここで非同期処理を待つ」という呼び出し側のマーク。非同期関数を呼ぶ側は必ず`await`を付けないといけない。

3. **質問：** `guard let url = URL(string: urlString) else { return }` はなぜifではなくguardを使うのか？
   **得られた理解：** `guard`は「条件を満たさない場合は早期リターン」という意図を明示する。コードの主流れを深いネストにせず読みやすくする。失敗ケースを先に排除するSwiftの慣習。

## この章のまとめ

WebAPIとの通信は「URLを作る→データを取得する→デコードする→表示する」という4ステップで完結する。`Codable`がデコードの手間を省き、`async/await`が非同期処理をシンプルに書かせてくれる。この2つのしくみを理解すれば、どのAPIでも同じパターンで対応できる。
