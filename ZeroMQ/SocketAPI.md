Socket API
=========

Sockets are the de facto standard API for network programming. That’s why ZeroMQ presents a familiar socket-based API. One thing that make ZeroMQ especially tasty to developers is that it uses different socket types to implement any arbitrary messaging pattern. Furthermore ZeroMQ sockets provide a clean abstraction over the underlying network protocol which hides the complexity of those protocols and makes switching between them very easy.

ソケットは、ネットワークプログラミングの事実上の標準APIです。それが、ZeroMQが使い慣れたソケットベースのAPIを提供する理由です。 開発者にとってのZeroMQの魅力の1つは、様々なソケットタイプを使用して任意のメッセージングパターンを実装ができることです。さらに、ZeroMQソケットは、基盤となるネットワークプロトコル上でクリーンな抽象化を提供するため、これらのプロトコルの複雑さが隠され、プロトコル間の切り替えが非常に簡単になります。

<!-- TOC -->

- [Key differences to conventional sockets](#key-differences-to-conventional-sockets)
- [Socket lifetime](#socket-lifetime)
- [First example](#first-example)
- [Bind vs Connect](#bind-vs-connect)
    - [When should I use bind and when connect?](#when-should-i-use-bind-and-when-connect)
- [High-Water-Mark](#high-water-mark)
- [Messaging Patterns](#messaging-patterns)
    - [Request-reply pattern](#request-reply-pattern)
        - [REQ socket](#req-socket)
        - [REP socket](#rep-socket)
        - [DEALER socket](#dealer-socket)
        - [ROUTER socket](#router-socket)
    - [Publish-subscribe pattern](#publish-subscribe-pattern)
        - [Topics](#topics)
        - [PUB socket](#pub-socket)
        - [SUB socket](#sub-socket)
        - [XPUB socket](#xpub-socket)
        - [XSUB socket](#xsub-socket)
    - [Pipeline pattern](#pipeline-pattern)
        - [PUSH socket](#push-socket)
        - [PULL socket](#pull-socket)
    - [Exclusive pair pattern](#exclusive-pair-pattern)
        - [PAIR socket](#pair-socket)
    - [Client-server pattern](#client-server-pattern)
        - [CLIENT socket](#client-socket)
        - [SERVER socket](#server-socket)
    - [Radio-dish pattern](#radio-dish-pattern)
        - [RADIO socket](#radio-socket)
        - [DISH socket](#dish-socket)

<!-- /TOC -->

# Key differences to conventional sockets

Generally speaking, conventional sockets present a synchronous interface to either connection-oriented reliable byte streams (SOCK_STREAM), or connection-less unreliable datagrams (SOCK_DGRAM). In comparison, ZeroMQ sockets present an abstraction of an asynchronous message queue, with the exact queueing semantics depending on the socket type in use. Where conventional sockets transfer streams of bytes or discrete datagrams, ZeroMQ sockets transfer discrete messages.

一般的に、従来のソケットは、コネクション型で信頼性の高いバイトストリーム（SOCK_STREAM）、またはコネクションレスで信頼性の低いデータグラム（SOCK_DGRAM）に対する同期インターフェースを提供します。これに対して、ZeroMQソケットは非同期メッセージキューの抽象化を提供し、使用中のソケットタイプに応じた正確なキューイングセマンティクスを提供します。従来のソケットがバイトストリームまたは個別のデータグラムを転送するのに対し、ZeroMQソケットは個別のメッセージを転送します。

ZeroMQ sockets being asynchronous means that the timings of the physical connection setup and tear down, reconnect and effective delivery are transparent to the user and organized by ZeroMQ itself. Further, messages may be queued in the event that a peer is unavailable to receive them.

ZeroMQソケットが非同期であるということは、物理的な接続のセットアップと切断、再接続、および効果的な配信のタイミングがユーザーに対して透過的であり、ZeroMQ自体によって構成されることを意味します。さらに、ピアがメッセージを受信できない場合、メッセージはキューに入れられる場合があります。

Conventional sockets allow only strict one-to-one (two peers), many-to-one (many clients, one server), or in some cases one-to-many (multicast) relationships. With the exception of PAIR sockets, ZeroMQ sockets may be connected to multiple endpoints, while simultaneously accepting incoming connections from multiple endpoints bound to the socket, thus allowing many-to-many relationships.

従来のソケットでは、厳密な1対1（2つのピア）、多対1（多くのクライアント、1つのサーバー）、または場合によっては1対多（マルチキャスト）の関係のみが許可されます。 PAIRソケットを除き、ZeroMQソケットは複数のエンドポイントに接続でき、同時にソケットにバインドされた複数のエンドポイントからの着信接続を受け入れ、多対多の関係を可能にします。

# Socket lifetime

ZeroMQ sockets have a life in four parts, just like BSD sockets:

ZeroMQソケットは、BSDソケットと同様に4つの部分に分かれています。

* Creating and destroying sockets, which go together to form a karmic circle of socket life
* Configuring sockets by setting options on them and checking them if necessary
* Plugging sockets into the network topology by creating ZeroMQ connections to and from them.
* Using the sockets to carry data by writing and receiving messages on them.

* ソケットを作成および破棄します。これらは一緒になって、ソケットライフのカルマの輪を形成します。
* ソケットにオプションを設定しソケットを構成し、必要に応じてチェックする。
* ZeroMQコネクションを作成してソケットをネットワークトポロジに接続します。
* ソケットを使用して、メッセージを書き込んだり受け取ったりしてデータを運ぶ。

# First example

So let’s start with some code, the “Hello world” example (of course).

それではコードから始めましょう、（もちろん）「Hello world」の例で。

```C
//  Hello World sever
#include <zmq.h>
#include <string.h>
#include <stdio.h>
#include <unistd.h>

int main (void)
{
    //  Socket to talk to clients
    void *context = zmq_ctx_new ();
    void *responder = zmq_socket (context, ZMQ_REP);
    int rc = zmq_bind (responder, "tcp://*:5555");
    assert (rc == 0);

    while (1) {
        char buffer [10];
        zmq_recv (responder, buffer, 10, 0);
        printf ("Received Hello\n");
        sleep (1);          //  Do some 'work'
        zmq_send (responder, "World", 5, 0);
    }
    return 0;
}
```

The server creates a socket of type reply (you will read more about Request-reply pattern later), binds it to port 5555 and then waits for messages. You can also see that we have zero configuration, we are just sending strings.

サーバーは、replyタイプのソケットを作成し（要求と応答のパターンについては後で詳しく説明します）、ポート5555にバインドしてからメッセージを待ちます。 また、ゼロ設定で、単に文字列を送信しています。

```C
//  Hello World client
#include <zmq.h>
#include <string.h>
#include <stdio.h>
#include <unistd.h>

int main (void)
{
    printf ("Connecting to hello world server…\n");
    void *context = zmq_ctx_new ();
    void *requester = zmq_socket (context, ZMQ_REQ);
    zmq_connect (requester, "tcp://localhost:5555");

    int request_nbr;
    for (request_nbr = 0; request_nbr != 10; request_nbr++) {
        char buffer [10];
        printf ("Sending Hello %d…\n", request_nbr);
        zmq_send (requester, "Hello", 5, 0);
        zmq_recv (requester, buffer, 10, 0);
        printf ("Received World %d\n", request_nbr);
    }
    zmq_close (requester);
    zmq_ctx_destroy (context);
    return 0;
}
```

The client creates a socket of type request, connects and starts sending messages.

クライアントは、要求タイプのソケットを作成し、接続してメッセージの送信を開始します。

Both the send and receive methods are blocking (by default). For the receive it is simple: if there are no messages the method will block. For sending it is more complicated and depends on the socket type. For request sockets, if the high watermark is reached or no peer is connected the method will block.

送信メソッドと受信メソッドの両方は（デフォルトで）ブロッキングです。 受信の場合は簡単です。メッセージがない場合、メソッドはブロックします。 送信に関しては、より複雑で、ソケットのタイプに依存します。 requestソケットの場合、最高水準点に達するか、ピアが接続されていない場合、メソッドはブロックします。

# Bind vs Connect

With ZeroMQ sockets it doesn’t matter who binds and who connects. In the above you may have noticed that the server used Bind while the client used Connect. Why is this, and what is the difference?

ZeroMQソケットでは、誰がバインドし、誰が接続してもかまいません。上記で、サーバーがバインドを使用し、クライアントが接続を使用していることに気付いたかもしれません。これはなぜですか、違いは何ですか？

ZeroMQ creates queues per underlying connection. If your socket is connected to three peer sockets, then there are three messages queues behind the scenes.

ZeroMQは、基礎となる接続ごとにキューを作成します。ソケットが3つのピアソケットに接続されている場合、背後に3つのメッセージキューがあります。

With Bind, you allow peers to connect to you, thus you don’t know how many peers there will be in the future and you cannot create the queues in advance. Instead, queues are created as individual peers connect to the bound socket.

バインドを使用すると、ピアからの接続を許可するため、将来ピアの数がわからないため、事前にキューを作成することはできません。代わりに、個々のピアがバインドされたソケットに接続すると、キューが作成されます。

With Connect, ZeroMQ knows that there’s going to be at least a single peer and thus it can create a single queue immediately. This applies to all socket types except ROUTER, where queues are only created after the peer we connect to has acknowledge our connection.

Connectを使用すると、ZeroMQは少なくとも1つのピアが存在することを認識するため、単一のキューをすぐに作成できます。これは、接続先のピアが接続を確認した後にのみキューが作成されるROUTERを除くすべてのソケットタイプに適用されます。

Consequently, when sending a message to bound socket with no peers, or a ROUTER with no live connections, there’s no queue to store the message to.

そのため、ピアのないバインドされたソケット、または生きた接続のないルーターにメッセージを送信する場合、メッセージを保存するキューはありません。


## When should I use bind and when connect?

As a general rule use bind from the most stable points in your architecture, and use connect from dynamic components with volatile endpoints. For request/reply, the service provider might be the point where you bind and the clients are using connect. Just like plain old TCP.

原則として、アーキテクチャ内の最も安定したポイントからのバインドを使用し、揮発性エンドポイントを持つ動的コンポーネントからの接続を使用します。 request/replyの場合、サービスプロバイダーはバインドするポイントであり、クライアントは接続を使用している可能性があります。 昔ながらのTCPのように。

If you can’t figure out which parts are more stable (i.e. peer-to-peer), consider a stable device in the middle, which all sides can connect to.

どの部分がより安定しているかがわからない場合（つまり、ピアツーピア）は、中間にすべての側が接続できる安定したデバイスを検討してください。

You can read more about this at the ZeroMQ FAQ under the “Why do I see different behavior when I bind a socket versus connect a socket?” section.

これについて詳しくは、ZeroMQ FAQの「ソケットをバインドする場合とソケットを接続する場合の動作が異なるのはなぜですか？」セクションをご覧ください。

# High-Water-Mark

The high water mark is a hard limit on the maximum number of outstanding messages ZeroMQ is queuing in memory for any single peer that the specified socket is communicating with.

最高水準点は、指定されたソケットが通信している単一のピアに対して、ZeroMQがメモリ内でキューイングしている未処理のメッセージの最大数に対するハードリミットです。

If this limit has been reached the socket enters an exceptional state and depending on the socket type, ZeroMQ will take appropriate action such as blocking or dropping sent messages. Refer to the individual socket descriptions below for details on the exact action taken for each socket type.

この制限に達すると、ソケットは例外状態になり、ソケットの種類に応じて、ZeroMQは送信されたメッセージのブロックやドロップなどの適切なアクションを実行します。 各ソケットタイプに対して実行される正確なアクションの詳細については、以下の個々のソケットの説明を参照してください。

# Messaging Patterns

Underneath the brown paper wrapping of ZeroMQ’s socket API lies the world of messaging patterns. ZeroMQ patterns are implemented by pairs of sockets with matching types.

ZeroMQのソケットAPIの茶色の紙の包装の下には、メッセージングパターンの世界があります。 ZeroMQパターンは、一致するタイプのソケットのペアによって実装されます。

The built-in core ZeroMQ patterns are:

組み込みのコアZeroMQパターンは次のとおりです。

* Request-reply, which connects a set of clients to a set of services. This is a remote procedure call and task distribution pattern.
* Pub-sub, which connects a set of publishers to a set of subscribers. This is a data distribution pattern.
* Pipeline, which connects nodes in a fan-out/fan-in pattern that can have multiple steps and loops. This is a parallel task distribution and collection pattern.
* Exclusive pair, which connects two sockets exclusively. This is a pattern for connecting two threads in a process, not to be confused with “normal” pairs of sockets.

* Request-reply、これはクライアント群をサービス群に接続します。これは、リモートプロシージャコールおよびタスク分散パターンです。
* Pub-sub、これはパブリッシャー群をサブスクライバー群に接続します。これはデータ分散パターンです。
* Pipeline、これは複数のステップとループを持つことができるファンアウト/ファンインパターンでノードを接続します。これは、並列タスクの分散および収集パターンです。
* Exclusive pair、これは2つのソケットを排他的に接続します。これは、プロセス内の2つのスレッドを接続するためのパターンです。「通常の」ソケットペアと混同しないでください。

There are more ZeroMQ patterns that are still in draft state:

他にも、ドラフト状態のZeroMQパターンがさらにあります。

* Client-server, which allows a single ZeroMQ server talk to one or more ZeroMQ clients. The client always starts the conversation, after which either peer can send messages asynchronously, to the other.

* Radio-dish, which used for one-to-many distribution of data from a single publisher to multiple subscribers in a fan out fashion.

* Client-server、これは単一のZeroMQサーバーが1つ以上のZeroMQクライアントと通信できます。会話を開始するのは常にクライアントで。その後、どちらかのピアがもう一方に非同期にメッセージを送信できます。
* Radio-dish、これは単一のパブリッシャーから複数のサブスクライバーへのファンアウト方式でのデータの1対多の配信に使用します。

## Request-reply pattern

The request-reply pattern is intended for service-oriented architectures of various kinds. It comes in two basic flavors: synchronous (REQ and REP socket types), and asynchronous socket types (DEALER and ROUTER socket types), which may be mixed in various ways.

request-replyパターンは、さまざまな種類のサービス指向アーキテクチャを対象としています。 2つの基本的なフレーバーが存在します。すなわち、同期（REQおよびREPソケットタイプ）と非同期ソケットタイプ（DEALERおよびROUTERソケットタイプ）があり、これらはさまざまな方法で混合できます。

The request-reply pattern is formally defined by RFC 28/REQREP.

request-replyパターンは、RFC 28/REQREPで形式的に定義されています。


### REQ socket

A REQ socket is used by a client to send requests to and receive replies from a service. This socket type allows only an alternating sequence of sends and subsequent receive calls. A REQ socket maybe connected to any number of REP or ROUTER sockets. Each request sent is round-robined among all connected services, and each reply received is matched with the last issued request. It is designed for simple request-reply models where reliability against failing peers is not an issue.

クライアントは、REQソケットを使用して、サービスに要求を送信し、サービスから応答を受信します。 このソケットタイプは、送信とそれに後続する受信呼び出しの交互のシーケンスのみを許可します。 REQソケットは、任意の数のREPまたはROUTERソケットに接続できます。 送信された各リクエストは、接続されているすべてのサービス間でラウンドロビンされ、受信された各応答は最後に発行されたリクエストと照合されます。 これは、障害のあるピアに対する信頼性が問題にならない単純なrequest-replyモデル用に設計されています。

If no services are available, then any send operation on the socket will block until at least one service becomes available. The REQ socket will not discard any messages.

利用可能なサービスがない場合、少なくとも1つのサービスが利用可能になるまで、ソケットに対する送信操作はブロックされます。 REQソケットはメッセージを破棄しません。

**Summary of characteristics:**
	
Compatible peer sockets 	REP, ROUTER
Direction 	Bidirectional
Send/receive pattern 	Send, Receive, Send, Receive, …
Outgoing routing strategy 	Round-robin
Incoming routing strategy 	Last peer
Action in mute state 	Block

### REP socket

A REP socket is used by a service to receive requests from and send replies to a client. This socket type allows only an alternating sequence of receive and subsequent send calls. Each request received is fair-queued from among all clients, and each reply sent is routed to the client that issued the last request. If the original requester does not exist any more the reply is silently discarded.

REPソケットは、クライアントから要求を受信し、クライアントに応答を送信するためにサービスによって使用されます。 このソケットタイプは、受信呼び出しとそれに後続する送信呼び出しの交互のシーケンスのみを許可します。 受信した各要求は、すべてのクライアント間で公平キューに入れられ、送信された各応答は、最後の要求を発行したクライアントにルーティングされます。 元のリクエスターが既に存在しない場合、応答は無言で破棄されます。

**Summary of characteristics:**
	
Compatible peer sockets 	REQ, DEALER
Direction 	Bidirectional
Send/receive pattern 	Receive, Send, Receive, Send …
Outgoing routing strategy 	Fair-robin
Incoming routing strategy 	Last peer

### DEALER socket

The DEALER socket type talks to a set of anonymous peers, sending and receiving messages using round-robin algorithms. It is reliable, insofar as it does not drop messages. DEALER works as an asynchronous replacement for REQ, for clients that talk to REP or ROUTER servers. Message received by a DEALER are fair-queued from all connected peers.

DEALERソケットタイプは、匿名ピア群と通信し、ラウンドロビンアルゴリズムを使用してメッセージを送受信します。 メッセージをドロップしない限り、信頼できます。 DEALERは、REPまたはROUTERサーバーと通信するクライアントのREQの非同期置換として機能します。 DEALERが受信したメッセージは、接続されているすべてのピアから公平キューされます。

When a DEALER socket enters the mute state due to having reached the high water mark for all peers, or if there are no peers at all, then any send operation on the socket will block until the mute state ends or at least one peer becomes available for sending; messages are not discarded.

すべてのピアの最高水位標に達したためにDEALERソケットがミュート状態になった場合、またはピアがまったくない場合は、ミュート状態が終了するか、少なくとも1つのピアが送信使用可能になるまで、ソケットに対する送信操作がブロックされます。すなわち、メッセージは破棄されません。

When a DEALER socket is connected to a REP socket message sent must contain an empty frame as first part of the message (the delimiter), followed by one or more body parts.

DEALERソケットがREPソケットに接続される場合、送信されるメッセージには、メッセージの最初の部分（区切り文字）として空のフレームが含まれ、その後に1つ以上の本文部分が続く必要があります。

**Summary of characteristics:**
	
Compatible peer sockets 	ROUTER, REP, DEALER
Direction 	Bidirectional
Send/receive pattern 	Unrestricted
Outgoing routing strategy 	Round-robin
Incoming routing strategy 	Fair-queued
Action in mute state 	Block

### ROUTER socket

The ROUTER socket type talks to a set of peers, using explicit addressing so that each outgoing message is sent to a specific peer connection. ROUTER works as an asynchronous replacement for REP, and is often used as the basis for servers that talk to DEALER clients.

ROUTERソケットタイプは、各発信メッセージが特定のピア接続に送信されるように、明示的なアドレス指定を使用して、ピア群と通信します。 ROUTERはREPの非同期置換として機能し、多くの場合、DEALERクライアントと通信するサーバーの基盤として使用されます。

When receiving messages a ROUTER socket will prepend a message part containing the routing id of the originating peer to the message before passing it to the application. Messages received are fair-queued from among all connected peers. When sending messages a ROUTER socket will remove the first part of the message and use it to determine the routing id of the peer the message shall be routed to. If the peer does not exist anymore, or has never existed, the message shall be silently discarded.

メッセージを受信する場合、ROUTERソケットは、発信元ピアのルーティングIDを含むメッセージ部分をメッセージに付加してから、アプリケーションに渡します。受信したメッセージは、接続されているすべてのピア間で公平キューに入れられます。メッセージを送信するとき、ROUTERソケットはメッセージの最初の部分を削除し、それを使用してメッセージのルーティング先となるピアのルーティングIDを決定します。ピアがもう存在しない、または存在したことがない場合、メッセージは無言で破棄されます。

When a ROUTER socket enters the mute state due to having reached the high water mark for all peers, then any messages sent to the socket will be dropped until the mute state ends. Likewise, any messages routed to a peer for which the individual high water mark has been reached will also be dropped.

すべてのピアの最高水準点に達したためにROUTERソケットがミュート状態になると、ミュート状態が終了するまでソケットに送信されたメッセージはすべてドロップされます。同様に、個々の最高水準点に到達したピアにルーティングされたメッセージもドロップされます。

When a REQ socket is connected to a ROUTER socket, in addition to the routing id of the originating peer each message received shall contain an empty delimiter message part. Hence, the entire structure of each received message as seen by the application becomes: one or more routing id parts, delimiter part, one or more body parts. When sending replies to a REQ socket the application must include the delimiter part.

REQソケットがROUTERソケットに接続されている場合、発信ピアのルーティングIDに加えて、受信した各メッセージには空の区切りメッセージ部分が含まれます。したがって、アプリケーションから見た各受信メッセージの構造全体は、1つ以上のルーティングID部分、区切り文字部分、1つ以上の本文部分になります。 REQソケットに応答を送信する場合、アプリケーションには区切り文字部分を含める必要があります。

**Summary of characteristics:**
	
Compatible peer sockets 	DEALER, REQ, ROUTER
Direction 	Bidirectional
Send/receive pattern 	Unrestricted
Outgoing routing strategy 	See text
Incoming routing strategy 	Fair-queued
Action in mute state 	Drop (see text)

## Publish-subscribe pattern

The publish-subscribe pattern is used for one-to-many distribution of data from a single publisher to multiple subscribers in a fan out fashion.

publish-subscribeパターンは、単一のパブリッシャーから複数のサブスクライバーへのファンアウト方式でのデータの1対多の配信に使用されます。

The publish-subscribe pattern is formally defined by RFC 29/PUBSUB.

publish-subscribeパターンは、RFC 29/PUBSUBで正式に定義されています。

ZeroMQ comes with support for Pub/Sub by way of four socket types:

ZeroMQには、4つのソケットタイプによるPub/Subのサポートがあります。

* PUB Socket Type
* XPUB Socket Type
* SUB Socket Type
* XSUB Socket Type


### Topics

ZeroMQ uses multipart messages to convey topic information. Topics are expressed as an array of bytes, though you may use a string and with suitable text encoding.

ZeroMQは、マルチパートメッセージを使用してトピック情報を伝えます。 トピックはバイトの配列として表されますが、適切なテキストエンコーディング文字列を使用することもできます。

A publisher must include the topic in the message’s’ first frame, prior to the message payload. For example, to publish a status message to subscribers of the status topic:

publisherは、メッセージのペイロードの前に、メッセージ群の最初のフレームにトピックを含める必要があります。 たとえば、ステータストピックのsubscriberにステータスメッセージをpublishするには、次のようにします。

```C
//  Send a message on the 'status' topic
zmq_send (pub, "status", 5, ZMQ_SNDMORE);
zmq_send (pub, "All is well", 11, 0);
```

Subscribers specify which topics they are interested in by setting the ZMQ_SUBSCRIBE option on the subscriber socket:

subscriberは、subscriberソケットでZMQ_SUBSCRIBEオプションを設定することで、関心のあるトピックを指定します。

```C
//  Subscribe to the 'status' topic
zmq_setsockopt (sub, ZMQ_SUBSCRIBE, "status", strlen ("status"));
```

A subscriber socket can have multiple subscription filters.

subscriberソケットは、複数の購読フィルタを持つことができます。

A message’s topic is compared against subscribers’ subscription topics using a prefix check.

メッセージのトピックは、前方一致を使用して、subscriberの購読トピックと比較されます。

That is, a subscriber who subscribed to topic would receive messages with topics:

つまり、topicを購読しているsubscriberは、以下のようなトピックを含むメッセージを受信します。

* topic
* topic/subtopic
* topical

However it would not receive messages with topics:

ただし、以下のようなトピックを含むメッセージは受信しません。

* topi
* TOPIC (remember, it’s a byte-wise comparison)

A consequence of this prefix matching behaviour is that you can receive all published messages by subscribing with an empty topic string.

この前方一致動作により、空のトピック文字列で購読することで、公開されたすべてのメッセージを受信できます。

### PUB socket

A PUB socket is used by a publisher to distribute data. Messages sent are distributed in a fan out fashion to all connected peers. This socket type is not able to receive any messages.

PUBソケットは、publisherがデータを配布するために使用します。 送信されたメッセージは、接続されたすべてのピアにファンアウト方式で配信されます。 このソケットタイプはメッセージを受信できません。

When a PUB socket enters the mute state due to having reached the high water mark for a subscriber, then any messages that would be sent to the subscriber in question shall instead be dropped until the mute state ends. The send function does never block for this socket type.

subscriberの最高水位標に達したためにPUBソケットがミュート状態になると、代わりに、問題のsubscriberに送信されるメッセージは、ミュート状態が終了するまでドロップされます。 送信関数は、このソケットタイプに対してブロックしません。

**Summary of characteristics:**
	
Compatible peer sockets 	SUB, XSUB
Direction 	Unidirectional
Send/receive pattern 	Send only
Incoming routing strategy 	N/A
Outgoing routing strategy 	Fan out
Action in mute state 	Drop

### SUB socket

A SUB socket is used by a subscriber to subscribe to data distributed by a publisher. Initially a SUB socket is not subscribed to any messages. The send function is not implemented for this socket type.

SUBソケットは、publisherによって配信されるデータを購読するためにsubscriberによって使用されます。 最初は、SUBソケットはどのメッセージにも購読されていません。 このソケットタイプには送信機能は実装されていません。

**Summary of characteristics:**
	
Compatible peer sockets 	PUB, XPUB
Direction 	Unidirectional
Send/receive pattern 	Receive only
Incoming routing strategy 	Fair-queued
Outgoing routing strategy 	N/A

### XPUB socket

Same as PUB except that you can receive subscriptions from the peers in form of incoming messages. Subscription message is a byte 1 (for subscriptions) or byte 0 (for unsubscriptions) followed by the subscription body. Messages without a sub/unsub prefix are also received, but have no effect on subscription status.

着信メッセージの形式でピアから購読を受信できることを除いて、PUBと同じです。 購読メッセージは、バイト1（購読している場合）またはバイト0（購読していない場合）の後に購読本文が続きます。 sub/unsubプレフィックスのないメッセージも受信されますが、購読ステータスには影響しません。

**Summary of characteristics:**
	
Compatible peer sockets 	ZMQ_SUB, ZMQ_XSUB
Direction 	Unidirectional
Send/receive pattern 	Send messages, receive subscriptions
Incoming routing strategy 	N/A
Outgoing routing strategy 	Fan out
Action in mute state 	Drop

### XSUB socket

Same as SUB except that you subscribe by sending subscription messages to the socket. Subscription message is a byte 1 (for subscriptions) or byte 0 (for unsubscriptions) followed by the subscription body. Messages without a sub/unsub prefix may also be sent, but have no effect on subscription status.

購読メッセージをソケットに送信して購読することを除き、SUBと同じです。 購読メッセージは、バイト1（購読している場合）またはバイト0（購読していない場合）の後に購読本文が続きます。 sub/unsubプレフィックスのないメッセージも送信できますが、購読ステータスには影響しません。

**Summary of characteristics:**
	
Compatible peer sockets 	ZMQ_PUB, ZMQ_XPUB
Direction 	Unidirectional
Send/receive pattern 	Receive messages, send subscriptions
Incoming routing strategy 	Fair-queued
Outgoing routing strategy 	N/A
Action in mute state 	Drop

## Pipeline pattern

The pipeline pattern is intended for task distribution, typically in a multi-stage pipeline where one or a few nodes push work to many workers, and they in turn push results to one or a few collectors. The pattern is mostly reliable insofar as it will not discard messages unless a node disconnects unexpectedly. It is scalable in that nodes can join at any time.

The pipeline pattern is formally defined by RFC 30/PIPELINE.

ZeroMQ comes with support for pipelining by way of two socket types:

* PUSH Socket Type
* PULL Socket Type

### PUSH socket

The PUSH socket type talks to a set of anonymous PULL peers, sending messages using a round-robin algorithm. The receive operation is not implemented for this socket type.

When a PUSH socket enters the mute state due to having reached the high water mark for all downstream nodes, or if there are no downstream nodes at all, then any send operations on the socket will block until the mute state ends or at least one downstream node becomes available for sending; messages are not discarded.

**Summary of characteristics:**
	
Compatible peer sockets 	PULL
Direction 	Unidirectional
Send/receive pattern 	Send only
Incoming routing strategy 	N/A
Outgoing routing strategy 	Round-robin
Action in mute state 	Block

### PULL socket

The PULL socket type talks to a set of anonymous PUSH peers, receiving messages using a fair-queuing algorithm.

The send operation is not implemented for this socket type.

**Summary of characteristics:**
	
Compatible peer sockets 	PUSH
Direction 	Unidirectional
Send/receive pattern 	Receive only
Incoming routing strategy 	Fair-queued
Outgoing routing strategy 	N/A
Action in mute state 	Block

## Exclusive pair pattern

PAIR is not a general-purpose socket but is intended for specific use cases where the two peers are architecturally stable. This usually limits PAIR to use within a single process, for inter-thread communication.

The exclusive pair pattern is formally defined by 31/EXPAIR.

### PAIR socket

A socket of type PAIR can only be connected to a single peer at any one time. No message routing or filtering is performed on messages sent over a PAIR socket.

When a PAIR socket enters the mute state due to having reached the high water mark for the connected peer, or if no peer is connected, then any send operations on the socket will block until the peer becomes available for sending; messages are not discarded.

While PAIR sockets can be used over transports other than inproc, their inability to auto-reconnect coupled with the fact new incoming connections will be terminated while any previous connections (including ones in a closing state) exist makes them unsuitable for TCP in most cases.

## Client-server pattern

> Note: This pattern is still in draft state and thus might not be supported by the zeromq library you’re using!


The client-server pattern is used to allow a single SERVER server talk to one or more CLIENT clients. The client always starts the conversation, after which either peer can send messages asynchronously, to the other.

The client-server pattern is formally defined by RFC 41/CLISRV.

### CLIENT socket

The CLIENT socket type talks to one or more SERVER peers. If connected to multiple peers, it scatters sent messages among these peers in a round-robin fashion. On reading, it reads fairly, from each peer in turn. It is reliable, insofar as it does not drop messages in normal cases.

If the CLIENT socket has established a connection, send operations will accept messages, queue them, and send them as rapidly as the network allows. The outgoing buffer limit is defined by the high water mark for the socket. If the outgoing buffer is full, or if there is no connected peer, send operations will block, by default. The CLIENT socket will not drop messages.

**Summary of characteristics:**
	
Compatible peer sockets 	SERVER
Direction 	Bidirectional
Send/receive pattern 	Unrestricted
Outgoing routing strategy 	Round-robin
Incoming routing strategy 	Fair-queued
Action in mute state 	Block

### SERVER socket

The SERVER socket type talks to zero or more CLIENT peers. Each outgoing message is sent to a specific peer CLIENT. A SERVER socket can only reply to an incoming message: the CLIENT peer must always initiate a conversation.

Each received message has a routing_id that is a 32-bit unsigned integer. To send a message to a given CLIENT peer the application must set the peer’s routing_id on the message.

> Example clisrv is missing for libzmq. Would you like to contribute it? Then follow the steps below:
>
> ```bash
> git clone https://github.com/zeromq/zeromq.org
> cd zeromq.org && mkdir -p content/docs/examples/c/libzmq
> cp archetypes/examples/clisrv.md
> content/docs/examples/c/libzmq/clisrv.md
> ```
If the routing_id is not specified, or does not refer to a connected client peer, the send call will fail. If the outgoing buffer for the client peer is full, the send call will block. The SERVER socket will not drop messages in any case.

**Summary of characteristics:**
	
Compatible peer sockets 	CLIENT
Direction 	Bidirectional
Send/receive pattern 	Unrestricted
Outgoing routing strategy 	See text
Incoming routing strategy 	Fair-queued
Action in mute state 	Fail

## Radio-dish pattern

> Note: This pattern is still in draft state and thus might not be supported by the zeromq library you’re using!

The radio-dish pattern is used for one-to-many distribution of data from a single publisher to multiple subscribers in a fan out fashion.

Radio-dish is using groups (vs Pub-sub topics), Dish sockets can join a group and each message sent by Radio sockets belong to a group.

Groups are null terminated strings limited to 16 chars length (including null). The intention is to increase the length to 40 chars (including null). The encoding of groups shall be UTF8.

Groups are matched using exact matching (vs prefix matching of PubSub).

### RADIO socket

A RADIO socket is used by a publisher to distribute data. Each message belong to a group. Messages are distributed to all members of a group. The receive operation is not implemented for this socket type.

> Example raddsh_radio_example is missing for libzmq. Would you like to contribute it? Then follow the steps below:
>
> ```bash
> git clone https://github.com/zeromq/zeromq.org
> cd zeromq.org && mkdir -p content/docs/examples/c/libzmq
> cp archetypes/examples/raddsh_radio_example.md
>    content/docs/examples/c/libzmq/raddsh_radio_example.md
> ```

When a RADIO socket enters the mute state due to having reached the high water mark for a subscriber, then any messages that would be sent to the subscriber in question will instead be dropped until the mute state ends. The send operation will never block for this socket type.

**Summary of characteristics:**
	
Compatible peer sockets 	DISH
Direction 	Unidirectional
Send/receive pattern 	Send only
Incoming routing strategy 	N/A
Outgoing routing strategy 	Fan out
Action in mute state 	Drop

### DISH socket

A DISH socket is used by a subscriber to subscribe to groups distributed by a RADIO. Initially a DISH socket is not subscribed to any groups. The send operations are not implemented for this socket type.

> Example raddsh_dish_example is missing for libzmq. Would you like to contribute it? Then follow the steps below:
> ```bash
> git clone https://github.com/zeromq/zeromq.org
> cd zeromq.org && mkdir -p content/docs/examples/c/libzmq
> cp archetypes/examples/raddsh_dish_example.md
>    content/docs/examples/c/libzmq/raddsh_dish_example.md
> ```

**Summary of characteristics:**
	
Compatible peer sockets 	RADIO
Direction 	Unidirectional
Send/receive pattern 	Receive only
Incoming routing strategy 	Fair-queued
Outgoing routing strategy 	N/A
