# 第2章：地図アプリの基本

> 執筆者：ペンチョジュニア
> 最終更新：2026-05-20

## この章で学ぶこと

この章では、MapKitを使ってアプリ内に地図を表示し、特定の位置にマーカーを配置する方法を学ぶ。東京の観光スポットをカテゴリ別マーカーで表示し、フィルタリングできるアプリを題材にする。地理座標の扱い方、MapKitのビューコンポーネント、そして`Set`を使ったフィルタリングが主なテーマ。

## 模範コードの全体像

```swift
import SwiftUI
import MapKit

struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            CategoryFilter(selectedCategories: $selectedCategories)
                .padding()
        }
    }
}
```

**このアプリは何をするものか：**

東京の観光スポット（浅草寺・東京タワー・スカイツリーなど6箇所）を地図上にカテゴリ別のカラーマーカーで表示する。画面下部のフィルターボタンで「寺社」「タワー」「公園」をON/OFFでき、表示するマーカーを絞り込める。地図は3D立体表示に対応している。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String { ... }
        var color: Color { ... }
    }
}
```

**何をしているか：**
地図上に表示する観光スポットのデータ構造を定義している。`CLLocationCoordinate2D`で緯度・経度を保持し、`Category`列挙型でランドマークの種類（寺社・タワー・公園）を表す。各カテゴリは固有のSF Symbolsアイコン名と色を持つ。

**なぜこう書くのか：**
カテゴリを文字列ではなく列挙型で定義することで、タイポを防ぎコンパイラが値を検査してくれる。`CaseIterable`を採用することで`Category.allCases`という配列が自動生成され、フィルターUIをループで生成できる。アイコンと色をカテゴリ自身のプロパティにすることで、カテゴリを知ればスタイルも自動的に決まる設計になっている。

**もしこう書かなかったら：**
カテゴリを文字列で管理すると「temple」と「templ」のようなタイポが実行時まで発見できない。`CaseIterable`がないと、フィルターボタンを作るたびに全カテゴリを手動でリストアップしないといけない。

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)

Map(position: $cameraPosition) { ... }
.mapStyle(.standard(elevation: .realistic))
```

**何をしているか：**
`MapCameraPosition`で地図の初期表示位置（東京の中心）と表示範囲を設定している。`$cameraPosition`でバインディングを渡すことでユーザーの地図操作（ドラッグ・ピンチ）がそのまま反映される。

**なぜこう書くのか：**
`MKCoordinateSpan`の`latitudeDelta`と`longitudeDelta`は表示する緯度・経度の「幅」を表す。値が小さいほど拡大された状態で表示される。`0.08`は東京全域がちょうど収まる程度の値。`$`（バインディング）で渡すことでユーザーが地図を動かした後の位置も保持できる。

**もしこう書かなかったら：**
`$`（バインディング）でなく値を直接渡すと、ビューが再描画されるたびに地図が初期位置に戻ってしまう。

---

### マーカーの表示

```swift
ForEach(filteredLandmarks) { landmark in
    Marker(
        landmark.name,
        systemImage: landmark.category.iconName,
        coordinate: landmark.coordinate
    )
    .tint(landmark.category.color)
}
```

**何をしているか：**
`filteredLandmarks`配列の各要素に対してMapKit標準の`Marker`を生成している。カテゴリごとに異なるSF Symbolsアイコンと色が適用される。

**なぜこう書くのか：**
`Marker`はMapKitが提供する標準のピンUIで、SF Symbolsのアイコンと色を簡単に組み合わせられる。`ForEach`でデータ駆動的に生成することで、スポットを追加・削除してもビューのコードを変えなくていい。

**もしこう書かなかったら：**
各ランドマークをハードコードで書くと、スポットを追加するたびにビューコードを修正しなければならない。

---

### フィルター機能

```swift
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}

// ボタン処理
if selectedCategories.contains(category) {
    selectedCategories.remove(category)
} else {
    selectedCategories.insert(category)
}
```

**何をしているか：**
選択中のカテゴリを`Set`（集合）で管理する。ボタンを押すと`Set`から追加/削除し、`filteredLandmarks`が`Set`に含まれるカテゴリのランドマークだけを返す computed property になっている。

**なぜこう書くのか：**
`Set`はArrayと違い重複を持たず、`contains`・`insert`・`remove`が高速（O(1)）。「どのカテゴリが選択中か」という2値的な状態管理に最適なデータ構造。`filter`クロージャと組み合わせることで、選択状態が変わるたびに`filteredLandmarks`が自動再計算される。

**もしこう書かなかったら：**
`Array`でフィルター状態を管理すると重複チェックや削除が複雑になる。またArrayの`contains`はO(n)なのでカテゴリ数が増えると遅くなる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `Map(position:)` | SwiftUIで地図を表示するビュー（MapKit） | `Map(position: $cameraPosition) { ... }` |
| `Marker` | 地図上にピンを表示するコンポーネント | `Marker("名前", systemImage: "icon", coordinate: coord)` |
| `CLLocationCoordinate2D` | 緯度・経度を表す構造体 | `CLLocationCoordinate2D(latitude: 35.68, longitude: 139.76)` |
| `MKCoordinateRegion` | 地図の表示範囲（中心座標＋スパン）を定義 | `MKCoordinateRegion(center: coord, span: span)` |
| `CaseIterable` | 列挙型の全ケースを配列として取得できるプロトコル | `Category.allCases` で全ケースを取得 |
| `Set<T>` | 重複なし・順序なしの集合型 | `Set<Category>` で`contains`・`insert`・`remove`が使える |
| `ZStack` | ビューを重ねて表示するレイアウトコンテナ | `ZStack(alignment: .bottom) { 地図 / フィルターUI }` |
| `.mapStyle` | 地図のスタイルを設定 | `.mapStyle(.standard(elevation: .realistic))` |

## 自分の実験メモ

**実験1：**
- やったこと：`span`の値を`0.01`に変えた
- 結果：地図が非常に拡大された状態で表示され、東京駅周辺しか見えなくなった
- わかったこと：`latitudeDelta`の値が小さいほど拡大表示される

**実験2：**
- やったこと：`.mapStyle(.standard(elevation: .realistic))`を`.mapStyle(.satellite)`に変えた
- 結果：地図が衛星写真表示に切り替わった
- わかったこと：`.standard``.satellite``.hybrid`などのスタイルを1行変えるだけで地図の見た目が変わる

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `Set`はなぜ`Array`より「カテゴリの選択状態を管理するのに向いている」のですか？
   **得られた理解：** `Set`は「順序なし・重複なし」の集合。`contains`が常にO(1)で動くため、複数カテゴリを頻繁にON/OFFするフィルター管理に最適。Arrayの`contains`はO(n)で要素数が増えるほど遅くなる。

2. **質問：** `var filteredLandmarks: [Landmark] { ... }`はいつ計算されますか？
   **得られた理解：** computed propertyはアクセスされるたびに計算される。`selectedCategories`が`@State`なので変更があったときだけビューが再描画され、`filteredLandmarks`も再計算される。

3. **質問：** `ZStack(alignment: .bottom)`の`alignment`は何を制御しているのですか？
   **得られた理解：** `ZStack`の中のビューをどの位置に揃えるかを指定する。`.bottom`にするとすべての子ビューが下端に揃うため、地図の上にフィルターUIを下に貼り付けるように重ねられる。

## この章のまとめ

MapKitを使う場合「データ（座標）をモデルで定義→`Map`に渡す→`Marker`で表示」というシンプルな流れで地図アプリが作れる。フィルタリングには`Set`が便利で、`CaseIterable`との組み合わせで全カテゴリを自動的に列挙できる。第1章の`@State`とビューの自動更新という概念が地図のフィルターにもそのまま活きている。
