# 第3章：カメラの利用

> 執筆者：ペンチョジュニア
> 最終更新：2026-05-21

## この章で学ぶこと

この章では、PhotosPickerを使ってフォトライブラリから写真を選択し、UIImagePickerControllerを使ってカメラで撮影した写真をアプリ内に表示する方法を学ぶ。SwiftUIだけでは直接カメラにアクセスできないため、`UIViewControllerRepresentable`でUIKitのコントローラーをSwiftUIに橋渡しする方法と、デリゲートパターンを管理する`Coordinator`の仕組みが主なテーマ。

## 模範コードの全体像

```swift
import SwiftUI
import PhotosUI

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                imageDisplayArea

                HStack(spacing: 20) {
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }
        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}
```

**このアプリは何をするものか：**

画面中央の大きなエリアに写真を表示する。「ライブラリ」ボタンを押すとフォトライブラリから写真を選択でき、「カメラ」ボタンを押すとカメラ撮影画面が全画面で開く。撮影またはライブラリ選択後、その写真が中央に表示される。カメラはシミュレータでは使えないため、カメラボタンはシミュレータで自動的に無効になる。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
@State private var selectedItem: PhotosPickerItem?

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
```

**何をしているか：**
`PhotosPicker`はiOS 16以降で使えるSwiftUI標準の写真選択コンポーネント。ユーザーがフォトライブラリから写真を選ぶと、その写真への参照が`selectedItem`（型：`PhotosPickerItem?`）に入る。`matching: .images`で動画や音声を除外し、静止画のみ選択できるようにしている。

**なぜこう書くのか：**
`PhotosPicker`はiOSシステムが用意した標準UIを呼び出すため、こちらでUIを作らなくても整ったピッカーが表示される。`PhotosPickerItem`はデータ本体ではなく参照（ハンドル）なので、選択直後に大きな画像データをメモリに読み込まず、必要なときだけ`loadTransferable`でデータを取得できる。

**もしこう書かなかったら：**
`matching: .images`を省略すると動画なども選択肢に出てきて、動画を選んだときに`UIImage`への変換が失敗する。実際に`matching:`を削除して動画を選んでみたら、`loadImage`関数内の`UIImage(data:)`変換が`nil`を返して画像が何も表示されなかった。

---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}

func loadImage(from item: PhotosPickerItem?) async {
    guard let item = item else { return }
    do {
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            selectedImage = Image(uiImage: uiImage)
        }
    } catch {
        print("画像の読み込みに失敗: \(error.localizedDescription)")
    }
}
```

**何をしているか：**
`onChange(of: selectedItem)`は`selectedItem`が変化したタイミングで処理を実行するモディファイア。変化を検知したら`Task`で非同期コンテキストを作り、`loadImage`関数を呼ぶ。`loadTransferable(type: Data.self)`が実際に写真データをメモリに読み込む非同期処理で、完了したら`UIImage`に変換し、さらにSwiftUIの`Image`型に変換して`selectedImage`に代入する。

**なぜこう書くのか：**
写真データの読み込みはディスクアクセスを伴うため、メインスレッドで直接実行するとUIがフリーズする。`async/await`で非同期に処理することでメインスレッドをブロックしない。`onChange`の`{ _, newItem in }`という書き方では第1引数が変化前の値、第2引数が変化後の値なので`newItem`だけ使えばいい。

**もしこう書かなかったら：**
`Task { }`で包まないと同期コンテキストから非同期関数を呼べずコンパイルエラーになる。`UIImage(data:)`がオプショナルを返すため`if let`なしでは強制アンラップが必要になりクラッシュリスクが生まれる。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}
}
```

**何をしているか：**
`UIViewControllerRepresentable`はUIKitの`UIViewController`をSwiftUIのビューとして使えるようにするプロトコル。`makeUIViewController`でカメラピッカーを作成・設定し、SwiftUIに渡す。`updateUIViewController`はSwiftUIの状態が変化したときに呼ばれるが、このアプリでは更新不要なので空実装にしている。

**なぜこう書くのか：**
SwiftUIにはカメラに直接アクセスするAPIがない。カメラ機能はUIKitの`UIImagePickerController`を使わなければならないため、このブリッジが必要。`@Binding`で`capturedImage`を受け取ることで、撮影完了後に親ビューの`capturedUIImage`に値を書き戻せる。

**もしこう書かなかったら：**
`updateUIViewController`を省略するとプロトコルの要件を満たせずコンパイルエラーになる。`picker.delegate = context.coordinator`を忘れると撮影完了・キャンセルのコールバックが届かず、カメラ画面が閉じない。

---

### Coordinatorパターン

```swift
func makeCoordinator() -> Coordinator {
    Coordinator(self)
}

class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    let parent: CameraView

    init(_ parent: CameraView) {
        self.parent = parent
    }

    func imagePickerController(
        _ picker: UIImagePickerController,
        didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
    ) {
        if let image = info[.originalImage] as? UIImage {
            parent.capturedImage = image
        }
        parent.dismiss()
    }

    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        parent.dismiss()
    }
}
```

**何をしているか：**
`Coordinator`はUIKitのデリゲートコールバックを受け取る仲介役のクラス。`imagePickerController(_:didFinishPickingMediaWithInfo:)`は撮影完了時に呼ばれ、撮影画像を`parent.capturedImage`（= 親ビューの`@Binding`）に書き込む。`imagePickerControllerDidCancel`はキャンセル時に呼ばれ、カメラ画面を閉じる。

**なぜこう書くのか：**
UIKitのデリゲートパターンはObjective-Cの設計に基づいており、SwiftUIの構造体には直接実装できない（構造体は参照型でないためデリゲートに使えない）。そのため`NSObject`を継承したクラスである`Coordinator`が代わりにデリゲートを引き受ける。`parent`を保持することで、コールバック時に親のSwiftUIビューの`@Binding`変数に値を書き戻せる。

**もしこう書かなかったら：**
`UINavigationControllerDelegate`を採用しないと`UIImagePickerController`のデリゲート設定が正しく動かない。`parent.dismiss()`を呼ばないとカメラ画面が閉じず、ユーザーがアプリに戻れなくなる。`NSObject`を継承しないとObjective-Cランタイムのデリゲートとして認識されずクラッシュする。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `PhotosPicker` | フォトライブラリから写真を選択するSwiftUI標準コンポーネント（iOS 16+） | `PhotosPicker(selection: $selectedItem, matching: .images) { ... }` |
| `PhotosPickerItem` | 選択した写真への参照（ハンドル）。データ本体は`loadTransferable`で取得する | `@State private var selectedItem: PhotosPickerItem?` |
| `loadTransferable(type:)` | `PhotosPickerItem`から非同期でデータを読み込む | `try await item.loadTransferable(type: Data.self)` |
| `UIViewControllerRepresentable` | UIKitの`UIViewController`をSwiftUIビューとして使うためのプロトコル | `struct CameraView: UIViewControllerRepresentable { ... }` |
| `makeCoordinator()` | UIKitのデリゲートを受け取る`Coordinator`インスタンスを生成するメソッド | `func makeCoordinator() -> Coordinator { Coordinator(self) }` |
| `UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
| `isSourceTypeAvailable(.camera)` | カメラが利用可能か確認するクラスメソッド。シミュレータでは`false`を返す | `.disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))` |
| `.fullScreenCover(isPresented:)` | Bool値がtrueになると全画面でビューを表示するモディファイア | `.fullScreenCover(isPresented: $isShowingCamera) { CameraView(...) }` |
| `onChange(of:)` | 値が変化したタイミングで処理を実行するモディファイア | `.onChange(of: selectedItem) { _, newItem in ... }` |
| `@ViewBuilder` | 条件分岐を含む複雑なビューを返すプロパティや関数に付ける属性 | `@ViewBuilder private var imageDisplayArea: some View { ... }` |
| `@Environment(\.dismiss)` | シートやフルスクリーンカバーを閉じるための環境値 | `@Environment(\.dismiss) private var dismiss` |

## 自分の実験メモ

**実験1：**
- やったこと：`matching: .images`を削除した
- 結果：フォトライブラリのピッカーに動画も表示されるようになった。動画を選択すると`UIImage(data:)`の変換が失敗して画像が何も表示されなかった
- わかったこと：`matching`引数はフィルター。`.images`で静止画のみに絞らないと、他のメディアタイプが混入して変換処理が壊れる

**実験2：**
- やったこと：`updateUIViewController`の中に`print("update呼ばれた")`を追加した
- 結果：カメラ画面を表示している間、SwiftUIが再描画されるたびにコンソールにログが出力された
- わかったこと：`updateUIViewController`はSwiftUIの再描画のたびに呼ばれる。カメラピッカーは状態を維持するので空実装で問題ないが、設定変更が必要なUIKitコンポーネントではここで更新処理を書く必要がある

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `UIViewControllerRepresentable`はなぜ必要なのですか？SwiftUIだけでカメラを使う方法はないのですか？
   **得られた理解：** SwiftUIはiOS 13から導入された新しいUIフレームワークで、カメラAPIはまだSwiftUIネイティブのものが存在しない。カメラはUIKitの`UIImagePickerController`を使う必要があり、`UIViewControllerRepresentable`がそのブリッジ役になる。このパターンはカメラに限らず、SwiftUIに相当するビューがないUIKitコンポーネント全般に使える。

2. **質問：** `Coordinator`はなぜ`NSObject`を継承しなければいけないのですか？
   **得られた理解：** `UIImagePickerControllerDelegate`はObjective-Cのプロトコルで、Objective-CのランタイムはNSObjectを継承したクラスしかデリゲートとして認識できない。SwiftのプロトコルだけではObjective-Cランタイムに「このオブジェクトはデリゲートとして使える」と伝える仕組みがないため、`NSObject`継承が必須になる。

3. **質問：** `@Binding var capturedImage: UIImage?`で`CameraView`から`ContentView`に値を書き戻せる仕組みを教えてください。
   **得られた理解：** `@Binding`はデータの「参照」を渡す仕組み。`ContentView`が持つ`@State private var capturedUIImage`のアドレスを`CameraView`に渡し、`CameraView`側から`parent.capturedImage = image`と書くと`ContentView`の`capturedUIImage`が直接更新される。コピーではなく参照渡しなので、片方を変えるともう片方も変わる。第1章の`searchText`を`TextField`に渡すときも同じ仕組みだった。

## この章のまとめ

カメラ機能の実装には「SwiftUIとUIKitの橋渡し」という新しい技術が必要だった。`UIViewControllerRepresentable`でUIKitコントローラーをSwiftUIに組み込み、`Coordinator`でデリゲートコールバックを受け取り、`@Binding`で結果を親ビューに返すという3層構造が核心。フォトライブラリからの選択は`PhotosPicker`一つで完結するが、カメラは必ずこのUIKit連携パターンが必要になる。「SwiftUIで完結できないときは`UIViewControllerRepresentable`で橋渡しする」というパターンは、カメラ以外のUIKitコンポーネントにも同じように応用できる。
