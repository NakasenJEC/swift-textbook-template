# 第6章（応用）：Tinder風スワイプカードUI

> 執筆者：ペンチョジュニア
> 最終更新：2026-07-08

## この章で学ぶこと

この章では、ドラッグジェスチャーとアニメーションを組み合わせて、カードを左右にスワイプして「好き/嫌い」を仕分けるUIを作る。動物カードを題材に、ドラッグ量に応じてカードを動かす・回転させる・透明度を変えるといった「指の動きに連動する見た目」をどう実装するかを学ぶ。

## 模範コードの全体像

```swift
// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？").font(.title2).bold()

            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！").font(.largeTitle)
                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(Array(animals.reversed())) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.last { removeCard(animal: top, direction: .left) }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }
                    Button {
                        if let top = animals.last { removeCard(animal: top, direction: .right) }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }
        switch direction {
        case .left: dislikedAnimals.append(animal)
        case .right: likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            VStack(spacing: 16) {
                Text(animal.emoji).font(.system(size: 80))
                Text(animal.name).font(.title).bold()
                Text(animal.description).font(.body).foregroundStyle(.secondary)
            }

            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .padding(.leading, 16)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
                    .padding(.trailing, 16)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}
```

**このアプリは何をするものか：**

画面中央に動物カードが1枚表示され、指でドラッグすると指の動きに追従してカードが動き・傾く。右に十分な距離スワイプすると「LIKE」として好きリストに、左にスワイプすると「NOPE」として苦手リストに加わり、次のカードが自動的に前面に出てくる。カードを最後まで判定すると「完了！」画面になり、もう一度シャッフルしてやり直せる。

## コードの詳細解説

### offsetとrotationの2つの@State

```swift
@State private var offset: CGSize = .zero
@State private var rotation: Double = 0
```

**何をしているか：**
`offset`はカードが画面上でどれだけズレているか（ドラッグ量そのもの）を表し、`rotation`はカードの傾き角度を表す。どちらもドラッグ中はリアルタイムに更新され、`.offset(offset)`と`.rotationEffect(.degrees(rotation))`でビューに反映される。

**なぜこう書くのか：**
第5章までの`@State`は「値が変わったらビューが再描画される」という使い方が中心だったが、ここでは`DragGesture`の`onChanged`が指の動きのたびに呼ばれるたびに`offset`を書き換えることで、指の動きにそのまま追従するアニメーションを実現している。`rotation`を`offset`から独立した`@State`にしているのは、`.onEnded`で「元に戻す」ときに`offset`と`rotation`を両方まとめて`0`に戻す必要があるためで、2つの値を別々に管理した方がコードの意図が明確になる。

**もしこう書かなかったら：**
`rotation`を`@State`にせずそのつど`offset`から計算するだけにすると、`.onEnded`で「バネのように戻す」アニメーションを両方の値に同時にかけたいときに`withAnimation`でまとめて操作しづらくなる。

---

### onChangedとonEndedで処理を分ける理由

```swift
.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
            rotation = Double(value.translation.width / 20)
        }
        .onEnded { value in
            if value.translation.width > swipeThreshold {
                // 右スワイプ確定
            } else if value.translation.width < -swipeThreshold {
                // 左スワイプ確定
            } else {
                // 元に戻す
            }
        }
)
```

**何をしているか：**
`onChanged`はドラッグ中（指が画面に触れて動いている間）に何度も呼ばれ、`onEnded`は指を離した瞬間に一度だけ呼ばれる。ドラッグ「途中」はカードを指に追従させるだけにとどめ、ドラッグが「終了」した時点で初めて「スワイプ成立か、キャンセルか」を判定している。

**なぜこう書くのか：**
判定を`onChanged`の中でやってしまうと、指がまだ画面上にあるのに毎回「スワイプ成立」と判定されかねず、ユーザーが「ちょっと動かしただけ」のつもりでもカードが飛んでいってしまう。`onEnded`でまとめて判定することで、「指を離した瞬間の最終的な移動距離」を基準に一度だけ確定的な判断ができる。

**もしこう書かなかったら：**
判定を`onChanged`だけで済ませようとすると、ドラッグの途中で誤発火したり、逆に指を離しても状態がリセットされないままになるといった不具合が起きやすい。

---

### swipeThresholdとswipeProgress

```swift
private let swipeThreshold: CGFloat = 100

private var swipeProgress: CGFloat {
    min(abs(offset.width) / swipeThreshold, 1.0)
}
```

**何をしているか：**
`swipeThreshold`は「これだけドラッグしたらスワイプ成立とみなす」距離（100ポイント）。`swipeProgress`は現在のドラッグ量がその閾値に対して何%進んでいるかを`0.0〜1.0`で表し、`.opacity(swipeProgress)`でLIKE/NOPEの文字の透明度に連動させている。

**なぜこう書くのか：**
`min(..., 1.0)`で上限を1.0にクリップしているのは、閾値を超えて大きくドラッグしても不透明度が100%を超えないようにするため。この値を変えるとUXが変わる。値を小さくする（例：50）と少し動かしただけでスワイプが成立してしまい暴発しやすくなる。大きくする（例：200）と、確実な意思表示をしないとスワイプが成立せず誤操作は減るが操作が重く感じられる。

**もしこう書かなかったら：**
閾値を設けずドラッグした分だけ即座に確定してしまうと、「ちょっと動かして確認したい」という操作ができなくなり、UIとして扱いにくくなる。

---

### DispatchQueue.main.asyncAfterで遅延実行する理由

```swift
withAnimation(.easeOut(duration: 0.3)) {
    offset = CGSize(width: 500, height: 0)
}
DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
    onSwipe(.right)
}
```

**何をしているか：**
カードを画面外（`width: 500`）まで飛ばすアニメーションを`0.3`秒かけて実行し、そのアニメーションが終わるタイミングに合わせて`0.3`秒後に`onSwipe`（実際に配列からカードを取り除く処理）を呼んでいる。

**なぜこう書くのか：**
`withAnimation`はアニメーションの「開始」を指示するだけで、完了を待ってくれるわけではない。もし`onSwipe`をアニメーションと同時に呼んでしまうと、カードが画面外に飛んでいくアニメーションの途中で配列からデータが消え、ビューが唐突に消える（アニメーションが完了する前にカードの実体がなくなる）という見た目になる。アニメーション時間と同じ`0.3`秒だけ遅らせることで、「飛んでいくのを見届けてから消す」という自然な体験になる。

**もしこう書かなかったら：**
遅延を入れずに即座に`onSwipe`を呼ぶと、カードが画面外へ飛ぶアニメーションが再生されないまま配列から消え、パッと瞬間移動したような不自然な見た目になる。

---

### animals.reversed()でForEachしている理由

```swift
ForEach(Array(animals.reversed())) { animal in
    SwipeCardView(animal: animal) { direction in
        removeCard(animal: animal, direction: direction)
    }
}
```

**何をしているか：**
`animals`配列を逆順にしてから`ForEach`でカードを並べている。

**なぜこう書くのか：**
`ZStack`は配列の要素を「後に書いたものほど手前（上）」に重ねて描画する。`animals`配列の並び順そのままだと、配列の最後の要素が一番手前に来てしまい、ユーザーが最初に触るべきカード（配列の先頭）が一番奥に埋もれてしまう。`reversed()`しておくことで、配列の先頭要素が`ZStack`の最後に描画され、結果として一番手前・一番上に表示される。

**もしこう書かなかったら：**
`reversed()`を外すと、スワイプ操作でいちばん上に見えているカードと、実際に配列の先頭にあるカード（次に表示されるべきカード）がズレてしまい、ユーザーが触っているカードと画面上のカードの対応関係が崩れる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `DragGesture` | ドラッグ操作を検出するジェスチャーレコグナイザー | `.gesture(DragGesture().onChanged { value in ... })` |
| `.onChanged` / `.onEnded` | ジェスチャーの「途中」と「終了」で別の処理を紐付けるメソッド | `.onChanged { value in offset = value.translation }` |
| `value.translation` | ドラッグ開始位置から現在位置までのズレを`CGSize`で表す | `value.translation.width` |
| `.offset(_:)` | ビューを元の位置から`CGSize`分ずらして表示するモディファイア | `.offset(offset)` |
| `.rotationEffect(_:)` | ビューを指定した角度だけ回転させるモディファイア | `.rotationEffect(.degrees(rotation))` |
| `withAnimation(.spring)` / `.easeOut(duration:)` | アニメーションの種類・時間を指定して値の変化を滑らかにする | `withAnimation(.easeOut(duration: 0.3)) { offset = ... }` |
| `DispatchQueue.main.asyncAfter(deadline:)` | 指定時間後にメインスレッドで処理を実行する遅延実行API | `DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) { ... }` |
| `(SwipeDirection) -> Void` | クロージャ型のプロパティ。子ビューから親ビューへイベントを伝える | `let onSwipe: (SwipeDirection) -> Void` |

## 自分の実験メモ

**実験1：**
- やったこと：`swipeThreshold`を`100`から`30`に変更した
- 結果：ほんの少しカードを動かしただけでLIKE/NOPEが確定してしまい、意図せずスワイプしてしまうことが増えた
- わかったこと：閾値はユーザーの「確実な意思表示」と「誤操作の防止」のバランスを取る値で、小さすぎると暴発しやすいアプリになる

**実験2：**
- やったこと：`DispatchQueue.main.asyncAfter`の遅延時間を`0.3`から`0`に変更した
- 結果：カードが画面外に飛んでいくアニメーションの途中で急にカードが消え、次のカードが唐突に現れるようになった
- わかったこと：アニメーションの再生時間とデータを実際に変更するタイミングをそろえないと、見た目とデータの状態がズレて不自然になることを体験できた

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `rotation`を`offset.width / 20`から都度計算するのではなく`@State`として持っているのはなぜですか？
   **得られた理解：** `.onEnded`で「元の位置に戻す」ときに`rotation`も同時に`0`にリセットする必要があり、`withAnimation`でまとめて変化させたい値は両方とも`@State`として独立させておいた方が、アニメーションの対象として扱いやすいとわかった。

2. **質問：** `DispatchQueue.main.asyncAfter`を使わずに、アニメーションの完了を待つ他の方法はありますか？
   **得られた理解：** SwiftUIには`withAnimation(_:completion:)`（アニメーション完了時にクロージャを呼べるAPI）が存在し、そちらを使えば時間をハードコードせずに済む。ただし今回の模範コードではアニメーション時間が固定（0.3秒）でわかりやすいため、あえてシンプルな`asyncAfter`が選ばれていると理解した。

3. **質問：** `.simultaneously`という書き方をどこかで見たのですが、今回のコードのように`.gesture()`を1つしか使わない場合との違いは何ですか？
   **得られた理解：** `.simultaneously(with:)`は複数のジェスチャー（例：ドラッグと拡大縮小）を同時に認識させたいときに使う。今回はドラッグ1種類しか使っていないため`.gesture()`を1つ付けるだけで十分だが、例えば「ドラッグしながら拡大」のような複合ジェスチャーを実装する場合はこの仕組みが必要になる。

## この章のまとめ

ジェスチャーの実装で重要なのは「ドラッグの途中（onChanged）」と「ドラッグの終了（onEnded）」で担当する処理をはっきり分けることだった。途中は見た目を指に追従させるだけにとどめ、終了時にまとめて「成立か、キャンセルか」を判定することで、誤操作の少ない自然な操作感が生まれる。また、アニメーションは「開始を指示するもの」であって「完了を待ってくれるもの」ではないため、データを変更するタイミングをアニメーションの時間に合わせて意図的にずらす必要があるという、今までの章にはなかった「時間」を扱う設計の難しさを学んだ。
