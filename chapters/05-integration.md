# 第5章：機能統合の実践

> 執筆者：ペンチョジュニア
> 最終更新：2026-07-08

## この章で学ぶこと

この章は新しいAPIを学ぶ回ではなく、第2章（地図）・第3章（カメラ）・第4章（データ永続化）で学んだ技術を「組み合わせる」回である。写真を撮影し、撮影場所を地図上に記録する「フォトマップ」アプリを題材に、複数の技術をどう組み合わせるか、データの流れをどう設計するかを学ぶ。

## 模範コードの全体像

```swift
// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem { Label("マップ", systemImage: "map") }

            ListTab()
                .tabItem { Label("一覧", systemImage: "list.bullet") }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls { MapUserLocationButton() }

                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }
                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title).font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets { modelContext.delete(records[index]) }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }
                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }
                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...").foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") { saveRecord() }
                        .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }
        let record = PhotoRecord(
            title: title, memo: memo,
            latitude: location.latitude, longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }
                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title).font(.title2).bold()
                    if !record.memo.isEmpty {
                        Text(record.memo).foregroundStyle(.secondary)
                    }
                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}
```

**このアプリは何をするものか：**

写真を撮影（またはライブラリから選択）し、その場でのGPS座標と一緒に記録する「フォトマップ」アプリ。マップタブでは撮影地点にサムネイルつきのピンが立ち、タップすると詳細画面に飛べる。一覧タブでは記録をリストで見られる。すべての記録はSwiftDataで端末に永続化されるため、アプリを閉じても消えない。

## コードの詳細解説

### データモデルの設計（PhotoRecord）

```swift
@Model
class PhotoRecord {
    var latitude: Double
    var longitude: Double
    var imageData: Data?

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**
位置情報を`latitude`・`longitude`という2つの`Double`として個別に保存し、地図表示に必要な`CLLocationCoordinate2D`は`coordinate`という計算プロパティでその場で組み立てている。画像も`imageData: Data?`として生バイト列で保存し、表示するときだけ`uiImage`で`UIImage`に変換している。

**なぜこう書くのか：**
`CLLocationCoordinate2D`はSwiftDataの`@Model`が直接保存できる型ではない（`Codable`ではあるが、`@Model`のプロパティとしてサポートされているのは`Double`のようなプリミティブ型が基本）。そこで緯度・経度を単純な`Double`として保存し、必要なときだけ`CLLocationCoordinate2D`に組み立て直す。画像も同様で、`UIImage`型そのものではなく`Data`として保存することで、SwiftDataが扱える単純な型に落とし込んでいる。

**もしこう書かなかったら：**
`coordinate`を保存用プロパティにしてしまうと、SwiftDataのモデル定義でエラーになるか、余計な変換コードが増える。`imageData`を`Data?`ではなく`UIImage?`のまま保存しようとすると、SwiftDataがそのままでは永続化できず、結局どこかで`Data`への変換が必要になる。だったら最初から保存形式を`Data`にしておいた方が一貫している。

---

### 位置情報マネージャーを別クラスに切り出す

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}
```

**何をしているか：**
GPSの取得ロジックを`ContentView`とは独立した`LocationManager`という`@Observable`クラスにまとめ、`MapTab`が`@State private var locationManager = LocationManager()`として保持し、`AddRecordView`にはそれを`let locationManager: LocationManager`として渡している。

**なぜこう書くのか：**
第2章の応用編で学んだ`CLLocationManagerDelegate`パターンをそのまま使っているが、ここではビューの中に直接書かず独立したクラスに分離している。理由は2つある。1つは、マップタブと記録追加画面の両方が「現在地」という同じ情報を必要とするため、1つのインスタンスを共有した方が二重にGPSを起動せずに済む。もう1つは、位置情報の取得ロジックとUIの描画ロジックを分離することで、それぞれを独立してテスト・変更しやすくなる（関心の分離）。

**もしこう書かなかったら：**
`MapTab`と`AddRecordView`がそれぞれ独自に`CLLocationManager`を持つと、GPSセンサーへのアクセスが重複し、バッテリー消費が増えるうえ、2つの現在地が微妙にズレる可能性もある。ビューの中に直接ロジックを書くと、`ContentView`が肥大化し、位置情報部分だけを再利用することも難しくなる。

---

### タブ構成にした設計判断

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab().tabItem { Label("マップ", systemImage: "map") }
            ListTab().tabItem { Label("一覧", systemImage: "list.bullet") }
        }
    }
}
```

**何をしているか：**
アプリのトップレベルを`TabView`で構成し、「マップ」と「一覧」という2つの独立した画面をタブで切り替えられるようにしている。

**なぜこう書くのか：**
このアプリには「地図で場所を確認したい」「一覧で記録を見返したい」という2つの異なる利用シーンがあり、両方とも頻繁に行き来する操作ではなく、それぞれが完結した画面である。`TabView`はこのような「並列に存在する複数のトップレベル画面」を表現するのに適している。`NavigationSplitView`は主にiPadのようなワイド画面で「一覧→詳細」という親子関係を表現するのに向いており、今回のようにマップと一覧が対等な関係にある場合は`TabView`の方が自然だと考えた。

**もしこう書かなかったら：**
`NavigationStack`だけで実現しようとすると、マップ画面から一覧画面に移動するために毎回「戻る」操作が必要になり、頻繁に行き来したいという利用シーンに合わない。`NavigationSplitView`にすると、iPhoneの縦画面では逆に使いにくいレイアウトになる可能性がある。

---

### 写真選択と位置情報の連携（保存の流れ）

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        if let data = try? await newItem?.loadTransferable(type: Data.self) {
            selectedImageData = data
            if let uiImage = UIImage(data: data) {
                previewImage = Image(uiImage: uiImage)
            }
        }
    }
}

func saveRecord() {
    guard let location = locationManager.currentLocation else { return }
    let record = PhotoRecord(
        title: title, memo: memo,
        latitude: location.latitude, longitude: location.longitude,
        imageData: selectedImageData
    )
    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**
第3章で学んだ`PhotosPicker`で写真を選ぶと`onChange`が発火し、`Data`として読み込んでプレビュー表示用の`Image`に変換する。保存ボタンが押されると、その時点の`locationManager.currentLocation`（第2章の技術）と選択した画像データ（第3章の技術）をまとめて`PhotoRecord`（第4章の技術）としてSwiftDataに保存する。

**なぜこう書くのか：**
3つの章の技術がここで初めて1つの関数の中で合流する。`guard let location = ... else { return }`で、位置情報がまだ取得できていない状態では保存自体をブロックしている。これは「不完全なデータを保存してしまう」ことを防ぐための設計で、保存ボタン自体も`.disabled(title.isEmpty || locationManager.currentLocation == nil)`で無効化されている。

**もしこう書かなかったら：**
位置情報のチェックをしないと、GPSがまだ取得できていないタイミングで保存された場合に`latitude`・`longitude`にダミー値（0, 0など）を入れることになり、地図上で明後日の場所（ギニア湾付近）にピンが立ってしまう不具合が起きる。

---

### 地図上のカスタムAnnotation

```swift
Annotation(record.title, coordinate: record.coordinate) {
    Button {
        selectedRecord = record
    } label: {
        if let uiImage = record.uiImage {
            Image(uiImage: uiImage)
                .resizable()
                .clipShape(Circle())
                .overlay(Circle().stroke(.white, lineWidth: 2))
        } else {
            Image(systemName: "photo.circle.fill")
        }
    }
}
```

**何をしているか：**
標準の`Marker`（涙滴形のピン）ではなく、`Annotation`を使って撮影した写真そのものを丸くトリミングしてピンの代わりに表示している。タップすると`selectedRecord`に値が入り、`.sheet(item:)`で詳細画面が開く。

**なぜこう書くのか：**
`Marker`は色とSF Symbolsのアイコンしか指定できず、任意のSwiftUIビュー（今回は写真のサムネイル）を表示できない。`Annotation`は`content`クロージャに任意のビューを渡せるため、「その場所で撮った写真をそのままピンにする」という視覚的にわかりやすいUIを実現できる。一方で`RecordDetailView`内のミニマップでは単純に位置を示せればよいため、あえてシンプルな`Marker`を使い分けている。

**もしこう書かなかったら：**
すべて`Marker`にすると、どのピンがどの写真の記録なのか、タップして開くまでわからなくなる。逆に一覧性が重要でない詳細画面のミニマップまで`Annotation`で凝ったビューにすると、無駄に処理コストがかかる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `TabView` | 複数の独立した画面をタブで切り替えるコンポーネント | `TabView { MapTab().tabItem { ... } }` |
| `Annotation` | 地図上に任意のSwiftUIビューを配置できるマーカー（`Marker`より自由度が高い） | `Annotation(title, coordinate: coord) { CustomView() }` |
| `.sheet(item:)` | Optionalな値がセットされたときにシートを表示する（`isPresented:`と違い対象データを渡せる） | `.sheet(item: $selectedRecord) { record in ... }` |
| `MapCameraPosition` | 地図の表示範囲・中心を`@State`で管理する型 | `@State private var cameraPosition: MapCameraPosition = .automatic` |
| `UserAnnotation` | 地図上に現在地マーカーを表示する標準ビュー | `Map { UserAnnotation() }` |
| `loadTransferable(type:)` | `PhotosPickerItem`から実際のデータを非同期に取得する | `try? await item.loadTransferable(type: Data.self)` |
| 複数`@Query`の併用 | 同じモデルに対し画面ごとに異なるソート条件で`@Query`を使い分けられる | `MapTab`は無ソート、`ListTab`は`sort: \.createdAt, order: .reverse` |

## 自分の実験メモ

**実験1：**
- やったこと：`AddRecordView`の`saveRecord()`から`guard let location = ...`のチェックを一時的に外し、位置情報取得前に保存ボタンを押せるようにしてみた
- 結果：`latitude`・`longitude`がどちらも`0`のまま`PhotoRecord`が保存され、マップを見るとアフリカ沖の海上にピンが立った
- わかったこと：位置情報のnilチェックは「エラーを防ぐため」ではなく「意味のあるデータだけを保存するため」に必要なガードだと実感できた

**実験2：**
- やったこと：`Annotation`のサムネイルを`Marker`に差し替えて挙動を比べた
- 結果：ピンは表示されるが、どの記録がどのピンなのか写真を開くまでわからなくなった
- わかったこと：`Marker`と`Annotation`は「軽量だが情報量が少ない」か「重いが情報量が多い」かのトレードオフになっている

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `CLLocationCoordinate2D`を直接SwiftDataの`@Model`のプロパティにできないのはなぜですか？
   **得られた理解：** SwiftDataがサポートする型は基本的にプリミティブ型（Int, Double, String, Bool, Data, Date）とそれらのOptional・配列が中心。`CLLocationCoordinate2D`は構造体だがSwiftDataの永続化対象として直接サポートされていないため、`Double`2つに分解して保存し、計算プロパティで組み立て直すのが定石だとわかった。

2. **質問：** `LocationManager`を`MapTab`が保持して`AddRecordView`に渡しているのはなぜ、`AddRecordView`側で新しく作らないのですか？
   **得られた理解：** 新しく作ると、GPSの初期化・認可リクエストが画面を開くたびに走り直し、位置情報の取得に毎回時間がかかる。既存のインスタンスを渡す（依存性注入に近い考え方）ことで、マップタブがすでに取得している現在地をそのまま再利用でき、体験が速くなる。

3. **質問：** `.sheet(item:)`と`.sheet(isPresented:)`はどちらもシートを表示しますが、使い分けの基準は何ですか？
   **得られた理解：** `isPresented:`は「表示する/しない」という真偽値だけを扱う場合に向く。`item:`は「表示するとともに、どのデータを表示するか」を同時に扱いたい場合に向く。今回の詳細画面のように「タップされた記録を表示する」というケースでは、`selectedRecord`という1つの変数がフラグとデータを兼ねられる`item:`の方が状態管理がシンプルになる。

## この章のまとめ

統合アプリを作って実感したのは、新しい概念よりも「複数の技術を1つのデータフロー（写真選択→位置取得→SwiftDataへの保存→地図・一覧への反映）にまとめる設計力」が問われるということだった。位置情報マネージャーを独立したクラスに切り出す、保存前にデータの整合性をguardでチェックする、画面の関係性に応じてTabViewとNavigationStackを使い分けるといった判断は、どれも第2〜4章で学んだ個々の技術そのものではなく、それらを組み合わせるときに必要になる一段上の設計判断だった。
