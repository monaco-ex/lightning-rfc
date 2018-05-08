# BOLT #0: 導入と索引

ようこそ、みなさん! これらライトニング技術の基礎 (Basis of Lightning Technology == BOLT) 文書群は、必要に応じてオンチェーン・トランザクションを矯正することによる、相互連携によるオフチェーンのビットコイン送金のためのレイヤー2プロトコルについて記述しています。

いくつかの要件は微妙です。私たち(訳注:仕様策定者たち)は、あなたがここで理解する結果の背後にある理由と動機に光を当てるよう努めています。ねばなりません。私たちには至らない部分があると思っていますので、もし、あなたが、混乱したり間違えている箇所を見つけた場合には、私たちに連絡を取り改善の手助けをしてください。

本文書はバージョン 0 です。

1. [BOLT #1](01-messaging.md): 基本プロトコル
2. [BOLT #2](02-peer-protocol.md): チャネル管理のためのピア・プロトコル
3. [BOLT #3](03-transactions.md): Bitcoin トランザクションと Script フォーマット
4. [BOLT #4](04-onion-routing.md): オニオン・ルーティング・プロトコル
5. [BOLT #5](05-onchain.md): オン・チェーンのトランザクション処理に関する推奨事項
7. [BOLT #7](07-routing-gossip.md): P2P ノードとチャネル発見
8. [BOLT #8](08-transport.md): 暗号化と認証がなされたトランスポート
9. [BOLT #9](09-features.md): 割り当てられた機能フラグ
10. [BOLT #10](10-dns-bootstrap.md): DNS ブートストラップと Assisted Node Location
11. [BOLT #11](11-payment-encoding.md): ライトニング支払いのための請求書プロトコル

## 用語と用語ガイド

* *ノード*:
   * ビットコインネットワークに接続されたコンピュータやその他デバイス。

* *ピア*:
   * *チャネル* を通じて他方と取引する *ノード* たち。

* *MSAT*:
   * ミリ satoshi (satoshi の 1000分の1)、しばしばフィールド名に使われる。

* *資金源トランザクション*:
   * *チャンネル* 上の両方の *ピア* に支払う不可逆的なオンチェーン取引
    それは相互同意によってのみ支払えます。

* *チャネル*:
   * 2つの *ピア* の間で相互交換の高速なオフチェーン手法。
    資金を取引するために、ピアは、更新された *コミットメント取引* を作るための書名を交換します。

* *コミットメント・トランザクション*:
   * *資金源トランザクション* を消費するためのトランザクション。
    それぞれが常に消費可能なコミットメントトランザクションを持つよう、各 *peer* はこのトランザクションのための署名を保持します。
    After a new commitment transaction is negotiated, the old one is *revoked*.

* *HTLC*: Hashed Time Locked Contract.
   * 2 つの *ピア* の間での条件付き支払い:
    受取人はその署名と*支払いプリイメージ*を提示することによって支払いを行うことができ、そうでなければ、支払人は所定の時間後にそれを消費することにより、契約を取り消すことができる。
    それらは *コミットメント・トランザクション* からの出力として実装される。

* *支払いハッシュ*
   * The *HTLC* は *支払いプリイメージ* のハッシュである支払いハッシュを含む。

* *支払いプリイメージ*
   * 秘密鍵を知っている唯一存在である最終受取人に行われた、支払いであることの証明。
    最終受取人は資金を開放するためのプリイメージをリリースする。
    支払いプリイメージは *HTLC* 内の *支払いハッシュ* としてハッシュ化されている。

* *コミットメント取り消し秘密鍵*:
   * それぞれの *コミットメント・トランザクション* は、他の *ピア* が全てのアウトプットの消費を直ちに許可する、固有の *コミットメント取り消し* 秘密鍵の値を持っています。
    revealing this key is how old commitment transactions are revoked.
    取り消しをサポートするために、それぞれのコミットメント・トランザクションの出力はコミットメント取り消し公開鍵を参照する。

* *コミットメント毎のシークレット*:
   * すべての *コミットメント・トランザクション*  は、以前のコミットメントごとの一連のコミットメントのシークレットをコンパクトに格納できるように生成される、コミットメント毎のシークレットから自身の鍵を導出します。

* *相互クローズ*:
   * *チャネル*の協調クローズ。(あるアウトプットが小さすぎて含まれない場合を除いて)、各 *ピア* への出力を伴う *資金源トランザクション* の無条件の消費をブロードキャストすることによって達成される。

* *片側クローズ*:
   * *コミットメント・トランザクション* をブロードキャストすることによって達成される、*チャネル*の非協力的な閉鎖。
    このトランザクションは *相互クローズ* トランザクションよりも大きく（すなわち、効率が悪い）、コミットメントがブロードキャストされているピアは、以前にネゴシエートされた期間、自身のアウトプットにアクセスできない。

* *トランザクション取り消しによるクローズ*:
   * 取り消された *コミットメント・トランザクション* のブロードキャストによって発生する、 *チャネル* の不正なクローズ。
    他の *ピア* は *コミットメント取り消し秘密鍵* を知るので、 *ペナルティ・トランザクション* を生成できる。

* *ペナルティ・トランザクション*
   * *コミットメント取り消し秘密鍵* を用いた、取り消された *commitment transaction* の全てのアウトプットを消費する、トランザクション。
    A *peer* uses this if the other peer tries to "cheat" by broadcasting a revoked *commitment transaction*.

* *コミットメント番号*:
   * それぞれの *コミットメント・トランザクション* のための、48ビットの加算カウンタ;
    各カウンタは *チャネル* の中の *ピア* 毎に独立で、0から始まる。

* *奇数ならOK*:
   * 機能のために強制またはオプショナルかを示す幾つかの数値フィールドに与えられるルール。
    偶数は両方のエンドポイントが問い合わせの機能をサポートしなければならないことを示し、奇数番号は機能が方々のエンドポイントによって無視されても良いことを示す。

* `chain_hash`:
   * BOLTドキュメントのいくつかで、ターゲットブロックチェーンのジェネシス・ハッシュを示すのに使用される。
    これは、*nodes* に、いくつかのブロックチェーン上の *channels* を作成して参照することを可能とする。
    ノードは、未知の `chain_hash` を参照するメッセージを無視しなければならない。
    `bitcoin-cli`とは違い、ハッシュは逆転されず直接使用される。

     メインチェーンのBitcoinブロックチェーンでは、 `chain_hash` 値は（16進数で) `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000` でなければならない.

## テーマソング

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## 著者

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
