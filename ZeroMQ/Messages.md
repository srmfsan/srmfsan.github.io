
Messages
========

A ZeroMQ message is a discrete unit of data passed between applications or components of the same application. From the point of view of ZeroMQ itself messages are considered to be opaque binary data.

ZeroMQメッセージは、アプリケーション間または同じアプリケーションのコンポーネント間で渡されるデータの個別の単位です。 ZeroMQ自体の観点からは、メッセージは不透明なバイナリデータと見なされます。

On the wire, ZeroMQ messages are blobs of any size from zero upwards that fit in memory. You do your own serialization using protocol buffers, msgpack, JSON, or whatever else your applications need to speak. It’s wise to choose a data representation that is portable, but you can make your own decisions about trade-offs.

通信上、ZeroMQメッセージは、メモリに収まるゼロ以上の任意のサイズのBLOBです。protocol buffers, msgpack, など、アプリケーションが話す必要のあるものを使用して、独自のシリアル化を行います。移植可能なデータ表現を選択するのが賢明ですが、トレードオフについて独自の決定を下すことができます。

The simplest ZeroMQ message consist of one frame (also called message part). Frames are the basic wire format for ZeroMQ messages. A frame is a length-specified block of data. The length can be zero upwards. ZeroMQ guarantees to deliver all the parts (one or more) for a message, or none of them. This allows you to send or receive a list of frames as a single on-the-wire message.

最も単純なZeroMQメッセージは、1つのフレーム（メッセージ部分とも呼ばれます）で構成されます。フレームは、ZeroMQメッセージの基本的な通信フォーマットです。フレームは、長さ指定のデータブロックです。長さはゼロ以上にすることができます。 ZeroMQは、メッセージの（1つ以上の）すべての部分を配信すること保証し、さもなくば全く送りません。これにより、フレームのリストを単一の送信中メッセージとして送受信できます。

A message (single or multipart) must fit in memory. If you want to send files of arbitrary sizes, you should break them into pieces and send each piece as separate single-part messages. Using multipart data will not reduce memory consumption.

メッセージ（シングルまたはマルチパート）はメモリに収まる必要があります。任意のサイズのファイルを送信する場合は、それらを断片に分割し、各断片を個別のシングルパートメッセージとして送信する必要があります。マルチパートデータを使用しても、メモリ消費は削減されません。

<!-- TOC -->

- [Working with strings](#working-with-strings)
- [More](#more)

<!-- /TOC -->

# Working with strings

Passing data as strings is usually the easiest way to get a communication going as serialization is rather straightforward. For ZeroMQ we established the rule that strings are length-specified and are sent on the wire without a trailing null.

データを文字列として渡すことは、通常、シリアル化が比較的簡単なので、通信を行う最も簡単な方法です。 ZeroMQの場合、文字列は長さ指定され、末尾のヌルなしで回線上で送信されるというルールを確立しました。

The following function sends a string to a socket where the string’s length equals frame’s length.

次の関数は、ソケットに文字列を送信します。その文字列の長さはフレームの長さと等価です。

```C
static void
s_send_string (void *socket, const char *string) {
	zmq_send (socket, strdup(string), strlen(string), 0);
}
```

To read a string from a socket we have to provide a buffer and its length. The zmq_recv method write as much data into the buffer as possible. If there’s more data it will get discarded. We use the returned frame’s size to set appropriate null-terminator and return a duplicate of the retrieved string.

ソケットから文字列を読み取るには、バッファーとその長さを指定する必要があります。 zmq_recvメソッドは、できるだけ多くのデータをバッファーに書き込みます。データがさらにある場合は破棄されます。返されたフレームのサイズを使用して適切なヌルターミネータを設定し、取得した文字列の複製を返します。

```C
//  Receive string from socket and convert into C string
//  Chops string at 255 chars, if it's longer
static char *
s_recv_string (void *socket) {
    char buffer [256];
    int size = zmq_recv (socket, buffer, 255, 0);
    if (size == -1)
        return NULL;
    if (size > 255)
        size = 255;
    buffer [size] = \0;
    /* use strndup(buffer, sizeof(buffer)-1) in *nix */
    return strdup (buffer);
}
```

Because we utilise the frame’s length to reflect the string’s length we can send mulitple strings by putting each of them into a seperate frame.

フレームの長さを使用して文字列の長さを反映しているため、複数の文字列を別々のフレームに入れて送信できます。

The following function sends an array of string to a socket. The ZMQ_SNDMORE flag tells ZeroMQ to postpone sending until all frames are ready.

次の関数は、文字列の配列をソケットに送信します。 ZMQ_SNDMOREフラグは、すべてのフレームの準備ができるまで送信を延期するようZeroMQに指示します。

```C
static void
s_send_strings (void *socket, const char[] *strings, int no_of_strings) {
    for (index = 0; index < no_of_strings; index++) {
        int FLAG = (index + 1) == no_of_strings ? 0 : ZMQ_SNDMORE;
        zmq_send (socket, strdup(strings[index]), strlen(strings[index]), FLAG);
    }
}
```

To retrieve a string frames from a multi-part messages we must use the ZMQ_RCVMORE zmq_getsockopt() option after calling zmq_recv() to determine if there are further parts to receive.

マルチパートメッセージから文字列フレームを取得するには、zmq_recv（）を呼び出した後にZMQ_RCVMORE zmq_getsockopt（）オプションを使用して、さらに受信する部分があるかどうかを判断する必要があります。

```C
char *strings[25];
int rcvmore;
size_t option_len = sizeof (int);
int index = 0;
do {
    strings[index++] = s_recv_string (socket);
    zmq_getsockopt (socket, ZMQ_RCVMORE, &rcvmore, &option_len);
} while (rcvmore);
```

# More

Find out more about working with messages:

メッセージの操作に関する詳細をご覧ください。

* Initialise a message:
    * メッセージの初期化
    * zmq_msg_init()
    * zmq_msg_init_size()
    * zmq_msg_init_data()
* Sending and receiving a message:
    * メッセージの送受信
    * zmq_msg_send()
    * zmq_msg_recv()
* Release a message:
    * メッセージのリリース
    * zmq_msg_close()
* Access message content:
    * メッセージの内容にアクセス
    * zmq_msg_data()
    * zmq_msg_size()
    * zmq_msg_more()
* Work with message properties:
    * メッセージプロパティの操作
    * zmq_msg_get()
    * zmq_msg_set()
* Message manipulation:
    * メッセージ操作
    * zmq_msg_copy()
    * zmq_msg_move()
