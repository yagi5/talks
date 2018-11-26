# Microservices実装ガイド in Go at メルカリ
Hidetatsu Yaginuma @yagi5 株式会社メルカリ

Go Conference 2018 Autumn

<div style="text-align:right"><img src ="/Users/yagi5/Downloads/Gocon/mercari_icon3.png"/></div>

---

# 自己紹介

* Hidetatsu Yaginuma(柳沼 秀龍) @_yagi5
* 株式会社メルカリ Backend Engineer(2018/11/1~)
* Microservicesを作ってる
* GoとvimとMicroservicesが好き

---

<div style="text-align:center"><img src ="/Users/yagi5/Downloads/Gocon/mercari.png"/></div>

---

# 今日話すこと

* 実際どのようにMicroservicesを実装しているのか
* 作ってみてわかる、Goのいいところ

---

# メルカリとMicroservices

---

# メルカリとMicroservices

* メルカリはPHP製のモノリシックAPIだった
* 組織的課題に立ち向かうためにマイクロサービスに移行

---


# どのようにMicroservicesを実装しているのか

---

# Microservicesを作る流れ

* Protocol Buffers定義
↓
* サービス実装

---
# Microservicesを作る流れ
## Protocol Buffers定義

* Protocol Buffers(protobuf) とは？
  - スキーマ言語
* gRPCにおけるインタフェース定義に使用できる
* メルカリではサービス間通信にgRPCを採用している

---

# Microservicesを作る流れ
## Protocol Buffers定義

* 各サービスのインタフェースとしてのProtocol Buffersを開発者が定義
* 全サービスのprotoファイルを集約するリポジトリを作っている

---

# Microservicesを作る流れ

## Protocol Buffers定義

* 開発者がProtocol Buffersを作成、git push
* CIが `protoc` を実行し、言語ごとのクライアントコードを別リポジトリに自動生成
  - サーバーコードも生成され、開発者が実装に利用する

---

# Microservicesを作る流れ

## Protocol Buffers定義

* CIイメージ

<div style="text-align:center"><img src ="/Users/yagi5/Downloads/Gocon/proto-ci.png" /></div>

---

# Microservicesを作る流れ

## Protocol Buffers定義

* Dockerfileたち

<div style="text-align:center"><img src ="/Users/yagi5/Downloads/Gocon/dockerfiles.png" /></div>

---

# Microservicesを作る流れ

## Protocol Buffers定義

* すべてのメンバーはprotoファイルを読めばAPIのインタフェースを理解できる
  - 他チームのサービスの詳細を知らなくて良い

---

# Microservicesを作る流れ

## サービス実装の流れ

* テンプレートプロジェクトをcloneして名前をつける
* rm -rf .git
* git init
* サービスを頑張って作る :muscle:

---

# テンプレートプロジェクトについて

---

# テンプレートプロジェクトについて

* `echo` と呼ばれている (リクエストされたメッセージをそのままレスポンスする)
* Microservices開発に必要な `基本的な機能` が入っている

---

# テンプレートプロジェクトについて
## 基本的な機能

---

# テンプレートプロジェクトについて
## 基本的な機能

* ミドルウェア
  - `grpc-echosystem/go_grpc_middleware` の `Server Chain(Interceptors)` の仕組みを使用している
  - `WithUnaryServerChain()` には複数のintercepterを登録できる
  - `WithUnaryServerChain(A(), B(), C())` と実行することで、 `A -> B -> C -> endpoint -> C -> B -> A` と呼び出される
  - ここでDatadog、SentryなどのMiddlewareを登録している

---

```
opts := []grpc.ServerOption{
  grpc_middleware.WithUnaryServerChain(
    
    // Logger
    grpc_zap.UnaryServerInterceptor(
      config.Logger
    ),
    
    // Datadog
    grpc_dd.UnaryServerInterceptor(
      grpc_dd.WithServiceName(config.DDServiceName)
    ),
    
    // 認証、Sentry、Recoverなど
  ),
  grpc_middleware.WithStreamServerChain(
    grpc_zap.StreamServerInterceptor(config.Logger),
  ),
}
return newServer(echoService, config.Logger, opts...)
```

---

# テンプレートプロジェクトについて
## 基本的な機能

* ロギング
  - opencensus、zap(zapgrpc)を利用している
  - opencensusのexporterにはDatadogを利用している
  - main()でtracerをセットアップ
  - サービスの実装の中では、APM取得にDatadogのspanを使用
  
```go
span, ctx := ddtracer.StartSpanFromContext(
    ctx, 
    "someFunc",
    tracer.ResourceName(""),
    tracer.StartTime(time.Now()),
)
defer span.Finish()
// 実際の処理
```

---

<div style="text-align:center"><img src ="/Users/yagi5/Downloads/Gocon/dd-span.png"/></div>

---

# テンプレートプロジェクトについて
## 基本的な機能

* その他の機能
  - 認証認可
  - Sentryのエラー通知
  - HealthZ

---
  
# テンプレートプロジェクトについて
## 独立したチームを作ることを助ける

* Microservicesでは各チームは独立しており、別のチームが作ったAPIが正しく動くことを期待しない
  - RateLimit/CircuitBreaker
* チームは独立しているべき
* しかしながら、どこまでをチーム固有で持つか、どこまでをテンプレートとして共通的に使えるよう提供するかは非常に難しい

---

# 作ってみてわかる、Goのいいところ

---

# 作ってみてわかる、Goのいいところ

* ネットワーク/ミドルウェアレイヤとの相性がいい
  - メルカリではgo-kitやgo-microなどは使わず、Microservicesプラットフォームを内製している
  - ネットワークレイヤのコードが書きやすい
  - Gatewayなどのミドルウェアが書きやすい

---

# 作ってみてわかる、Goのいいところ

* 静的解析ツールで書き方を強制できる
  - メルカリのMicroservicesでは大量の開発者がたくさんのチームに分散している
  - Goの経験もまちまちな中で、ある程度書き方を統一できる
  - また、 `Goらしい書き方` を覚えることにも繋がる
  - `gofmt` `go vet` `golint` `misspell` `errcheck` `staticcheck` でチェックしている

---

# 作ってみてわかる、Goのいいところ

* シングルバイナリで動く
* High performance
* チューニングしやすい(並列実行、`go test -bench`、 `pprof`)
  - 多数のサービスがあっても早く動いてくれる(だろう)

---

# GoはMicroservicesを作るのに適しているか？

* ネットワークレイヤーのコードが書きやすく、ミドルウェア(API Gatewayなど)を作るのに適している
* 普通に書いてもそれなりに早く動いてくれるので、依存サービスがたくさんあったりしてもパフォーマンスの面であまり怖がらなくていい
* シングルバイナリで動くので、実行環境を作るのが楽
* 適していると思う、おすすめ

---

# メルカリでのMicroservicesのこれから

---

# メルカリでのMicroservicesのこれから

* 今はGoで作っているが、将来はチームごとに技術スタックを主体的に選択できるようにしたい
* まだまだやることは残っている
* 「Microservices難しい」って大体毎日言ってる
* でもすごく面白いので、一緒に作りましょう
* エンジニアを募集しています!!!

---

# ご清聴ありがとうございました