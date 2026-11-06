# 読解ノート：自分のアプリのコード3箇所

> 髙橋　樹
> **AI利用レベル：** 1（質問のみ。説明は自分の言葉で書く）
> **最終更新：** 2026-11-6

生成AIに書いてもらったコードでも、自分で書いたコードでもかまいません。自分のアプリの中から3箇所を選んで、次の3つの問いに答えてください。分からないところは、生成AIに「この行は何をしていますか」「この行を消すとどうなりますか」と聞いてかまいません。ただし、書くのは自分の言葉です。

> 💡 担任の先生の指示で、項目が増えることがあります。

---
箇所1：通信方式を差し替え可能にするプロトコル（RoomTransport）
protocol RoomTransport {
    var onMessage: ((ServerMessage) -> Void)? { get set }
    func connect(initialMessage: ServerMessage)
    func send(_ message: ServerMessage) async throws
    func disconnect()
}

final class WebSocketClient: RoomTransport { … }    // 遠距離＝サーバー経由
final class MultipeerTransport: RoomTransport { … }  // 対面＝端末同士が直結

// RoomSession 側は「どちらか」を受け取るだけ
self.client = transport ?? WebSocketClient(url: url)

何をしているか：
オンライン対戦（サーバー経由）と対面対戦（サーバーなしの直結）という中身がまったく違う通信を、RoomTransport という共通の窓口でまとめている。ゲーム本体は「RoomTransport に送る／受け取る」としか知らず、実際にどちらの通信を使うかは外から差し込む。

なぜこう書くのか：
通信方式ごとにゲームのコードを分けて書くと、同じ処理を2か所に書くことになり、片方だけ直し忘れてバグる。プロトコルで「窓口の形」だけ決めておけば、ゲームのメッセージ定義も進行ロジックも1つで済み、入れ替わるのは通信層だけになる。新しい通信方式を足すときもゲーム本体に触らなくてよい。

もしこう書かなかったら：
RoomSession が WebSocketClient を直接持ってしまい、対面対戦を足すときにゲーム進行のコードを丸ごともう1セット書くことになる。実際この抽象化のおかげで、対面対戦の追加はゲームロジックを1行も変えずに済んだ。

箇所2：サーバーのロジックを端末内に持ち込む（LocalRelay）
@MainActor
final class LocalRelay {
    var hostOut: ((ServerMessage) -> Void)?         // ホスト自身へ返す
    var participantOut: ((ServerMessage) -> Void)?  // 相手へ送る

    func handle(from endpoint: Endpoint, _ message: ServerMessage) {
        switch message.type {
        case "createRoom":  … 
        case "joinRoom":    …
        case "startGame":   …
        case "gameAction":  …
        // server/src/relay.ts と同じ分岐をそのまま再現
        }
    }
}

何をしているか：
もともとサーバー側（relay.ts）にあった「部屋を作る・参加させる・対戦を開始する・アクションを処理する」という中継処理を、ホスト端末の中で動く LocalRelay として Swift に移植している。対面対戦では、ホスト端末が擬似的にサーバー役を兼ねる。

なぜこう書くのか：
対面対戦を「サーバーを立てずに」実現したいが、かといって対面専用の独自ロジックを書くと、オンラインと挙動がズレる。サーバーと同じ分岐をそのまま端末内に再現すれば、オンラインでも対面でもゲームの意味が完全に一致する。

もしこう書かなかったら：
対面用に簡易な中継を別途書くことになり、「オンラインでは正しいのに対面だけ挙動が違う」バグが出やすい。サーバー代をかけずに、かつ挙動の一貫性を保つには、既存ロジックの移植が一番安全だった。

箇所3：別スレッドの通知を安全にUIへ渡す＋接続順序の作り込み
// MultipeerConnectivity のデリゲートは別スレッドから呼ばれる
nonisolated func session(_ session: MCSession, peer peerID: MCPeerID,
                         didChange state: MCSessionState) {
    Task { @MainActor [weak self] in       // 必ずメインスレッドへ退避
        switch state {
        case .connected:    self?.peerConnected(peerID)
        case .notConnected: self?.peerDisconnected(peerID)
        default: break
        }
    }
}

// 参加ボタン：招待の「前」に joinRoom をバッファしておく
session.joinNearby()     // ← ここで先に送信予約
transport.invite(peer)   // ← 接続が完了した瞬間に確実に届く

何をしているか：
通信ライブラリからの通知は別スレッドで届くので、nonisolated で受けてから Task { @MainActor in … } でメインスレッドに移し、そこで初めてUIの状態を更新している。また参加処理では、相手を招待する前に joinRoom メッセージを予約しておき、接続が確立した瞬間に確実に送られるようにしている。

なぜこう書くのか：
UIの状態（@Published な変数など）を別スレッドから直接いじると、表示が壊れたりクラッシュする恐れがある。@MainActor への退避でそれを防ぐ。接続の順序も、「つながってから送る」では送信タイミングを逃すことがあるため、先に予約しておく方が取りこぼさない。

もしこう書かなかったら：
実際に joinNearby() が接続を切る処理を含んでいた初期版では、せっかくつながった対面接続を自分で壊してしまい「参加中…」で固まった。退避と順序を直したことで、通信が安定して動くようになった（ユーザー確認済み）。



「動いているけれど何をしているか分からない」

// ① 進行ループが「選んでください」で“止まる”側
func ask(_ title: String, _ options: [String], tappable: Bool = false, player: Player? = nil) async -> Int {
    if aborted { return max(options.count - 1, 0) }
    promptSeat = player.map { $0 === human ? 0 : 1 } ?? current
    promptId = UUID().uuidString            // この問いかけ1回限りのID
    prompt = Prompt(title: title, options: options, tappable: tappable)
    return await withCheckedContinuation { c in
        continuation = c                    // 再開ハンドルを保管して“ここで止まる”
    }
}

// ② あとで“再開”させる側（ローカルUIの選択）
func answer(_ index: Int) {
    promptId = nil
    prompt = nil
    let c = continuation
    continuation = nil                      // 先に消してから
    c?.resume(returning: index)             // 止まっていた ask を同じ地点から再開
}

// ③ 通信相手の選択を、正本の盤面で検証してから answer に流す
func answerOnline(seat: Int, promptId id: String, index: Int? = nil, cardId: String? = nil) -> Bool {
    guard online, !aborted, winner == nil, (0...1).contains(seat) else { return false }
    …
    guard self.promptId == id, promptSeat == seat, let prompt, continuation != nil else { return false }
    …
    answer(index)   // 検証を通ったものだけ、ローカルと同じ再開経路へ
    return true
}
