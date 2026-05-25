# AI質問ログ：第3章 カメラの利用

## 使用した生成AIツール

Claude（claude.ai 無料版）

## 質問と回答の記録

### Q1

**質問：**
`UIViewControllerRepresentable`はなぜ必要なのですか？SwiftUIにカメラを使うAPIはないのですか？

**AIの回答の要点：**
SwiftUIはiOS 13で導入された比較的新しいフレームワークで、カメラに直接アクセスするネイティブAPIはまだ存在しない。カメラ機能はUIKitの`UIImagePickerController`（またはAVFoundation）を使う必要がある。`UIViewControllerRepresentable`はUIKitのViewControllerをSwiftUIのビューとして扱えるようにするプロトコルで、カメラに限らずSwiftUIに相当品がないUIKitコンポーネント全般に使える橋渡し手段。

**自分の理解：**
SwiftUIとUIKitは別々のフレームワークで、SwiftUIが全部カバーしているわけではない。「SwiftUIで足りないときはUIKitを借りる」ための入り口が`UIViewControllerRepresentable`だと分かった。使い方は毎回同じパターン（makeUIViewController / updateUIViewController / makeCoordinator）なので、一度覚えれば他のUIKitコンポーネントにも応用できる。

---

### Q2

**質問：**
`Coordinator`クラスはなぜ`NSObject`を継承しないといけないのですか？Swiftのクラスではダメなのですか？

**AIの回答の要点：**
`UIImagePickerControllerDelegate`などUIKitのデリゲートプロトコルはObjective-Cのランタイムをベースにしている。Objective-Cのランタイムは「NSObjectを継承したオブジェクト」しかデリゲートとして認識できない。純粋なSwiftクラス（NSObjectを継承しないもの）をデリゲートに設定しようとすると実行時クラッシュまたはコンパイルエラーになる。

**自分の理解：**
iOSの土台にはObjective-Cがあり、SwiftはObjective-Cと共存している。デリゲートを使うUIKitコンポーネントを扱うときは、必ずこの「Objective-Cとの互換性」問題に直面する。`NSObject`継承は「このクラスはObjective-Cランタイムとも話せます」という宣言だと理解した。

---

### Q3

**質問：**
`@Binding var capturedImage: UIImage?`で値を書き戻す仕組みを教えてください。なぜコピーではなく元の変数が変わるのですか？

**AIの回答の要点：**
`@Binding`はデータそのものではなく「その変数へのアクセス経路（参照）」を渡す。`ContentView`が`$capturedUIImage`という形でアドレスを`CameraView`に渡すと、`CameraView`が`parent.capturedImage = image`と書いたとき、そのアドレスにある`ContentView`の`capturedUIImage`変数が直接書き換えられる。Swiftの値型（構造体）は普通コピー渡しだが、`@Binding`を使うことで参照渡しと同等の動作を実現している。

**自分の理解：**
`@Binding`は「子から親への書き戻し」を実現する仕組み。第1章で`TextField(text: $searchText)`と書いたとき、テキストフィールドが入力内容を`searchText`に書き戻していたのも同じ原理だったと気づいた。SwiftUIで「子ビューが親の変数を変更したい」ときは`@Binding`を使う、という使いどころが分かった。

---

### Q4

**質問：**
`@ViewBuilder`をプロパティに付けると何が変わるのですか？付けないとどうなりますか？

**AIの回答の要点：**
通常のSwiftプロパティは「単一の値を返す」ことしかできず、`if-else`などの条件分岐を直接書けない。`@ViewBuilder`を付けると、SwiftUIのビュー構築DSL（ドメイン固有言語）の文法が使えるようになり、`if-else`やその他の制御フローを書きながら複数種類のビューを返せるようになる。

**自分の理解：**
`var body: some View { ... }`が`if-else`を書けるのは`@ViewBuilder`が暗黙的に付いているから。`imageDisplayArea`という別プロパティに切り出すときに明示的に`@ViewBuilder`を付けることで、`body`と同じように条件分岐を書ける。コードを分割して読みやすくするための重要な道具だと分かった。

---

### Q5

**質問：**
`.fullScreenCover`と`.sheet`の違いは何ですか？カメラにはなぜ`.fullScreenCover`を使うのですか？

**AIの回答の要点：**
`.sheet`はモーダルシートとして画面の一部を覆う（下からスライドして途中で止まり、上部に前の画面が少し見える）。`.fullScreenCover`は文字通り全画面を覆い、前の画面が完全に隠れる。`UIImagePickerController`はデフォルトで全画面表示されるUIKitコンポーネントのため、`.sheet`で表示すると見た目が不自然になる（ファインダーが画面の一部に収まって操作しにくい）。カメラUIは全画面を使うことを前提に設計されているので`.fullScreenCover`が適切。

**自分の理解：**
`.sheet`で試したら確かに違和感があった。カメラのファインダーが小さく表示され、上部に「写真アプリ」の画面がちらっと見えた。`.fullScreenCover`に戻すとカメラが全画面を正しく占有した。モーダルの見た目の違いを実際に確認できてよかった。

---

## 今日の質問を振り返って

Q2（`NSObject`の必要性）が一番発見が多かった。「なぜこうしなければいけないのか」という背景にObjective-Cランタイムとの互換性があると分かり、iOSの歴史的な積み重ねを初めて意識できた。Q5はコードを実際に変えて動かすことで違いが体験として残った。次は応用編のCoreImageフィルターにも挑戦してみたい。