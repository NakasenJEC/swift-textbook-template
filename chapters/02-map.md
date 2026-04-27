# 第2章：地図アプリの基本

> 執筆者：ウカンセン
> 最終更新：2026-04-27

## この章で学ぶこと

この章では、MapKitを使ってアプリ内に地図を表示し、複数のランドマークをマーカーとして配置する方法を学ぶ。また、SwiftUIの状態管理（@State）やIdentifiableプロトコルを利用して、カテゴリによるフィルター機能を実装する流れを理解する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

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

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、東京の観光スポットを地図上に表示し、カテゴリごとにフィルターできる地図アプリである。  
ユーザーは地図上のマーカーをタップすることで、その場所の名前や説明を見ることができる。また、画面下部のフィルターを操作することで、特定のカテゴリのみを表示することが可能である。


## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category
}
```

**何をしているか：**
地図上に表示する観光スポット（ランドマーク）の情報をまとめたデータ構造を定義している。

**なぜこう書くのか：**
Identifiableをつけることで、それぞれのデータに一意のIDが割り当てられ、ForEachやMapで正しく識別できるようになる。また、HashableをつけることでSetなどで扱えるようになる。

**もしこう書かなかったら：**
Identifiableがないと、ForEachでエラーが出たり、UIの更新時にどの要素が変わったか判断できなくなる。

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)

Map(position: $cameraPosition, selection: $selectedLandmark) {
}
```

**何をしているか：**
地図の初期表示位置（東京駅付近）と表示範囲を設定し、Mapビューを表示している。

**なぜこう書くのか：**
@Stateを使うことで、カメラ位置を動的に変更できるようになる。MapにBindingで渡すことで、地図の操作と状態が同期される。

**もしこう書かなかったら：**
@Stateを使わないと、地図の位置が固定され、ユーザー操作に応じた更新が反映されない可能性がある。

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
    .tag(landmark)
}
```

**何をしているか：**
フィルターされたランドマークのリストをもとに、地図上にマーカーを表示している。

**なぜこう書くのか：**
ForEachを使うことで、配列のデータをまとめてUIに反映できる。Markerを使うことで、簡単に地図上にピンを表示できる。

**もしこう書かなかったら：**
ForEachを使わない場合、マーカーを1つずつ手動で書く必要があり、データ数が増えると管理が難しくなる。

---

### フィルター機能

```swift
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}
```

**何をしているか：**
選択されたカテゴリに応じて、表示するランドマークを絞り込んでいる。

**なぜこう書くのか：**
Setを使うことで、複数カテゴリを効率よく管理できる。また、computed propertyとして書くことで、状態が変わるたびに自動的に再計算される。

**もしこう書かなかったら：**
@Stateを使わないと、ボタンを押してもUIが更新されない。また、filterを使わないとカテゴリごとの表示切り替えができない。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目             | 説明                 | 使用例                                    |
| -------------- | ------------------ | -------------------------------------- |
| 例：`Map`       | SwiftUIで地図を表示するビュー | `Map(position: $cameraPosition)`       |
| 例：`Marker`     | 地図上にマーカーを表示する      | `Marker("名前", coordinate: coordinate)` |
| `@State`       | 状態を管理し、UI更新を行う     | `@State private var selectedCategory`  |
| `Identifiable` | 一意のIDを持つデータとして扱う   | `struct Item: Identifiable`            |
| `filter`       | 条件に合う要素だけ取り出す      | `array.filter { 条件 }`                  |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：マーカーの座標を変更してみた
- 結果：地図上の位置が変わった
- わかったこと：coordinateの値によって表示位置が決まる

**実験2：**
-やったこと：selectedCategoriesの初期値を空にした
- 結果：マーカーが表示されなくなった
- わかったこと：フィルター条件に一致しないと表示されない

## AIに聞いて特に理解が深まった質問 TOP3

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：Mapの新しい書き方（iOS17）と従来のcoordinateRegionを使う方法の違いは何か？**  
   **得られた理解：**  
   iOS17以降では、MapCameraPositionを使って地図の表示位置を管理する新しい書き方が導入されている。従来のcoordinateRegionを直接渡す方法に比べて、カメラの移動やズームなどの制御が柔軟になり、状態管理とも連携しやすくなっている。

2. **質問：CaseIterableをつけると何ができるのか？ForEach(Category.allCases)はどのように動作しているのか？**  
   **得られた理解：**  
   CaseIterableをつけることで、enumのすべてのケースを配列として取得できるallCasesプロパティが自動生成される。これにより、ForEachで全カテゴリをループ処理し、ボタンやUIを動的に生成できるようになる。

3. **質問：MarkerとAnnotationの違いは何か？それぞれの使いどころは？**  
   **得られた理解：**  
   Markerは簡単に地図上にマーカーを表示するためのコンポーネントであり、標準的な見た目で素早く実装できる。一方、Annotationはクロージャ内で自由にViewを定義できるため、よりカスタマイズされた表示が可能である。用途に応じて使い分ける必要がある。

## この章のまとめ

この章では、MapKitを使った地図表示の基本と、SwiftUIにおける状態管理の仕組みを学んだ。
