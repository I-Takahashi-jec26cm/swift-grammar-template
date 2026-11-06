# 日専祭アプリ：HackYou!

> **チーム：** 蒋　髙橋　吉田
> **AI利用レベル：** 2（生成コード可）
> **最終更新：** 2026/10/7

> 💡 制作の進め方は担任の先生から案内があります。このファイルの項目は、担任の先生の指示で増えることがあります。

---

## 何ができるアプリか（3行）
新感覚✖️超本格派トレーディングカードゲーム
これまで無かった勝利条件「手札が０になったら負け」を採用
簡単モードと本格派モードがあるのでTCG初心者も上級者も楽しめる！
CPU対戦だけでなくリアルタイム対戦も実装予定

## 画面

（スクリーンショットを貼る。GitHub の編集画面に画像をドラッグ＆ドロップすると貼れます。）![Uploading Simulator Screenshot - iPhone 17 - 2026-10-16 at 16.38.31.png…]()
<img width="1206" height="2622" alt="Simulator Screenshot - iPhone 17 - 2026-10-16 at 16 38 27" src="https://github.com/user-attachments/assets/4bf467b9-8eab-4b0a-a9de-850014464a56" />
<img width="1206" height="2622" alt="Simulator Screenshot - iPhone 17 - 2026-10-16 at 16 38 21" src="https://github.com/user-attachments/assets/23c31fcb-2924-4ed7-a777-201f1aa03ab1" />


## 使った文法・技術

【HackYou で使用した文法一覧】

■ Swift 言語機能
・struct / class / enum 定義（struct RoomSessionView: View、final class MultipeerTransport）
・final class（継承禁止）（final class WebSocketClient: RoomTransport）
・protocol（プロトコル定義）（protocol RoomTransport）
・プロトコル準拠（class MultipeerTransport: NSObject, RoomTransport, ObservableObject）
・extension（拡張）（extension MultipeerTransport: MCSessionDelegate）
・enum（列挙型）（enum Role { case host, participant }）
・switch 文での分岐（switch session.disconnection { case .host: … }）
・オプショナル(?)（var onMessage: ((ServerMessage) -> Void)?）
・オプショナルバインディング（if let battle = session.battle / guard let self）
・nil 合体演算子(??)（participantPeer ?? session.connectedPeers.first）
・オプショナルチェイン（self?.onMessage?($0)）
・guard 文（早期リターン）（guard role == .participant, browser == nil else { return }）
・クロージャ（{ [weak self] in self?.onMessage?($0) }）
・キャプチャリスト [weak self]（循環参照防止）
・末尾クロージャ（Button { … } label: { … }）
・省略引数 $0（relay.hostOut = { self?.onMessage?($0) }）
・ジェネリクス（JSONEncoder().encode(m) / decode(ServerMessage.self, …)）
・try / do-catch / try?（do { try session.send(…) } catch { … }）
・throws 関数（func send(_:) async throws）
・async / await（Task { await session.startGame() }）
・Task { }（非同期実行）（Task { @MainActor [weak self] in … }）
・属性 @MainActor（@MainActor final class MultipeerTransport）
・nonisolated（アクター隔離解除）（nonisolated func session(_:peer:didChange:)）
・static let / 型プロパティ（static let serviceType = "hackyou-vs"）
・private(set) アクセス制御（@Published private(set) var discoveredHosts）
・計算プロパティ（private var explanation: String { … }）
・引数ラベル・デフォルト引数（init(role:)、transport: RoomTransport? = nil）
・外部/内部引数名 _（func handle(from endpoint: Endpoint, _ msg: …)）
・#if canImport(UIKit) 条件コンパイル
・配列操作 append / removeAll(where:)（discoveredHosts.removeAll { $0 == peerID }）
・文字列処理（String(trimmed.prefix(60))）
・Codable（Encodable / Decodable）（ServerMessage の JSON 変換）

■ SwiftUI 機能
・View プロトコル + body（var body: some View）
・不透明型 some View
・プロパティラッパー（@ObservedObject / @StateObject / @State / @Published / @EnvironmentObject / @Environment）
・ObservableObject（class RoomSession: ObservableObject）
・レイアウト（VStack / HStack / ScrollView / Group / GeometryReader）
・ForEach（id 指定）（ForEach(transport.discoveredHosts, id: \.self)）
・条件付きビュー（if session.showsEndScreen { … }）
・修飾子チェーン（.font(…).foregroundStyle(…).padding(…)）
・Button / Label / Image(systemName:)（SF Symbols）
・状態変化ハンドラ（.onChange(of:) / .task { } / .onDisappear { }）
・モーダル表示（.sheet / .confirmationDialog）
・キーパス式（\.scenePhase、\.self）
・環境値注入（.environment(\.colorScheme, .dark)）

## 生成AIの使い方（どの場面で、どう使ったか）

アイデアを送り、プログラムは全部組んでもらいました

## 展示で工夫した点

チュートリアルを作り、わかりやすくした。
