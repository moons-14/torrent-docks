# BitTorrent Protocol Specification v1.0
ソース: https://wiki.theory.org/BitTorrentSpecification
ライセンス: CC0 1.0 Universal (https://creativecommons.org/publicdomain/zero/1.0/)

## 識別

BitTorrentは、Bram Cohenによって設計されたpeer-to-peerファイル共有プロトコルです。彼のページは[http://www.bittorrent.com](http://www.bittorrent.com/)で訪問できます。BitTorrentは、信頼性の低いネットワーク上で複数のpeer間でのファイル転送を促進するために設計されています。

## 目的

この仕様書の目的は、BitTorrentプロトコル仕様のバージョン1.0を詳細に文書化することです。Bramの[プロトコル仕様ページ](http://bittorrent.org/beps/bep_0003.html)では、プロトコルをやや一般的な用語で概説していますが、一部の領域では動作の詳細が不足しています。この文書が、将来の議論と実装の基礎として使用できる、明確で曖昧さのない用語で書かれた**正式な**仕様書となることが期待されています。

この文書は、BitTorrent開発コミュニティによって維持・使用されることを意図しています。すべての人がこの文書への貢献を歓迎されていますが、ここでの内容は、すでに多数の既存のclient実装で展開されている現在のプロトコルを表すことを意図していることを理解してください。

これは機能要求を提案する場所ではありません。そのためには、[メーリングリスト](http://lists.ibiblio.org/mailman/listinfo/bittorrent)にアクセスしてください。

## 適用範囲

この文書は、BitTorrentプロトコル仕様の最初のバージョン（すなわち、バージョン1.0）に適用されます。現在、これはtorrentファイル構造、peer wireプロトコル、およびTracker HTTP/HTTPSプロトコル仕様に適用されます。各プロトコルの新しい改訂版が定義される際は、それらは**ここではなく**、独自の別々のページで指定されるべきです。

### 関連文書

- [公式プロトコル仕様](http://bittorrent.org/beps/bep_0003.html)
- [開発者およびユーザーのウィッシュリスト](BitTorrentWishList "BitTorrentWishList")
- [Trackerプロトコル拡張](BitTorrentTrackerExtensions "BitTorrentTrackerExtensions")

## 記法

この文書では、情報を簡潔かつ明確に提示するため、多数の記法が使用されています。

- *peer* v/s *client*: この文書では、*peer*はダウンロードに参加する任意のBitTorrentクライアントです。*client*もpeerですが、ローカルマシンで実行されているBitTorrentクライアントです。この仕様の読者は、多数の*peer*に接続する*client*として自分自身を考えることができます。
- *piece* v/s *block*: この文書では、*piece*はmetainfoファイルで記述され、SHA1ハッシュで検証できるダウンロードデータの一部を指します。*block*は、*client*が*peer*に要求する可能性のあるデータの一部です。2つ以上の*block*が1つの完全な*piece*を構成し、その後検証される場合があります。
- *defacto standard*: *イタリック*で表示された大きなテキストブロックは、BitTorrentの様々なclient実装で非常に一般的な慣行であり、デファクトスタンダードと見なされることを示します。

この文書に対して行われた最近の変更を他の人が見つけやすくするため、編集を行う際は**Summary:**フィールドに記入してください。これには、文書に加えた各主要な変更について簡潔な（つまり1行の）エントリを含める必要があります。

## Bencoding

Bencodingは、データを簡潔な形式で指定・整理する方法です。以下のタイプをサポートします：バイト文字列、整数、リスト、辞書。

### バイト文字列

バイト文字列は以下のようにエンコードされます: *\<base 10 ASCIIでエンコードされた文字列長>**:**\<文字列データ>*\
定数の開始区切り文字や終了区切り文字がないことに注意してください。

  - **例**: ***4:*** *spam* は文字列 "spam" を表します
  - **例**: ***0:*** は空文字列 "" を表します

### 整数

整数は以下のようにエンコードされます: ***i**\<base 10 ASCIIでエンコードされた整数>**e***\
最初の**i**と末尾の**e**は開始と終了の区切り文字です。

  - **例**: ***i***3***e*** は整数 "3" を表します
  - **例**: ***i***-3***e*** は整数 "-3" を表します

***i***-0***e*** は無効です。***i***03***e***のような先頭ゼロを持つすべてのエンコーディングは、もちろん整数 "0" に対応する***i***0***e***を除いて無効です。

  - *注意:* この整数のビット数の最大値は指定されていませんが、「大きなファイル」（つまり4Gバイト以上の.torrent）を処理するには、signed 64bit整数として処理することが必須です。

### リスト

リストは以下のようにエンコードされます: ***l**\<bencodingされた値>**e***\
最初の**l**と末尾の**e**は開始と終了の区切り文字です。リストには、整数、文字列、辞書、さらには他のリスト内のリストを含む、任意のbencodingされたタイプを含めることができます。

  - **例**: ***l***4:spam4:eggs***e*** は2つの文字列のリストを表します: \[ "spam", "eggs" ]
  - **例**: ***le*** は空のリストを表します: \[]

### 辞書

辞書は以下のようにエンコードされます: ***d**\<bencodingされた文字列>\<bencodingされた要素>**e***\
最初の**d**と末尾の**e**は開始と終了の区切り文字です。キーはbencodingされた文字列でなければならないことに注意してください。値は、整数、文字列、リスト、その他の辞書を含む、任意のbencodingされたタイプを取ることができます。キーは文字列でなければならず、ソート順（英数字ではなく、生文字列としてソート）で表示される必要があります。文字列は、文化固有の「自然な」比較ではなく、バイナリ比較を使用して比較する必要があります。

  - **例**: ***d***3:cow3:moo4:spam4:eggs***e*** は辞書 { "cow" => "moo", "spam" => "eggs" } を表します
  - **例**: ***d***4:spaml1:a1:be***e*** は辞書 { "spam" => \[ "a", "b" ] } を表します
  - **例**: ***d***9:publisher3:bob17:publisher-webpage15:www\.example.com18:publisher.location4:home***e*** は { "publisher" => "bob", "publisher-webpage" => "www\.example.com", "publisher.location" => "home" } を表します
  - **例**: ***de*** は空の辞書 {} を表します

### 実装

- [C](https://sourceforge.net/p/funzix/code/ci/master/tree/bencode/) by Mike Frysinger
- [C](https://github.com/willemt/CHeaplessBencodeReader)
- [C](https://github.com/cwyang/bencode) by [Chul-Woong Yang](https://github.com/cwyang)
- [C#](https://github.com/Krusen/BencodeNET) by [Søren Kruse](https://twitter.com/sorenkrusen)
- [C#](http://snipplr.com/view/37790/bencoding-encoder-and-decoder/) by SuprDewd
- [C#](http://bencode.codeplex.com/) by LordMike
- [Clojure](http://nakkaya.com/2009/11/02/decoding-bencoded-streams-in-clojure/) by nakkaya
- [Common Lisp](https://gist.github.com/2021424) by osa1
- [Elixir](https://github.com/folz/bento) by [Rodney Folz](https://twitter.com/rodneyfolz)
- [Erlang](Decoding_encoding_bencoded_data_with_erlang "Decoding encoding bencoded data with erlang")
- [Erlang](https://github.com/galina/bencoded) by ladybug
- [Go](https://github.com/marksamman/bencode) by [Mark Samman](https://twitter.com/marksamman)
- [Decoding encoding bencoded data with haskell](Decoding_encoding_bencoded_data_with_haskell "Decoding encoding bencoded data with haskell") by Edi
- [Haskell](http://hackage.haskell.org/package/bencode) by Mhitza
- [Java](https://bitbucket.org/gyuriX/bencoder) by gyurix
- [Java](https://bitbucket.org/frazboyz/bencoder) by Frazboyz
- [JavaScript](http://demon.tw/my-work/javascript-bencode.html) by [Demon](http://demon.tw/)
- [JavaScript](https://github.com/benjreinhart/bencode-js) by [Ben Reinhart](http://benreinhart.com/)
- [JScript](JScript__Converting_a_torrent_file_to_a_JScript_dictionary "JScript: Converting a torrent file to a JScript dictionary") by Sergej B.
- [Objective-C](http://www.stupendous.net/projects/bencoding-obj-c-class/) by Chrome
- [OCaml](http://cvs.savannah.gnu.org/viewvc/mldonkey/mldonkey/src/networks/bittorrent/bencode.ml?view=markup) by [MLDonkey](http://mldonkey.sourceforge.net/Main_Page)
- [Perl](http://search.cpan.org/perldoc?Net::BitTorrent::Protocol::BEP03::Bencode)
- [PHP](Decoding_encoding_bencoded_data_with_PHP "Decoding encoding bencoded data with PHP")
- [PHP](http://github.com/jesseschalken/pure-bencode) by [Jesse Schalken](http://jesseschalken.com/)
- [PHP Extension](https://github.com/Frederick888/php-bencode) by [Frederick Zhang](https://blog.onee3.org/)
- [PHP Extension](http://code.google.com/p/php-bencode-extension/)
- [Pixie](https://github.com/stuarth/pixie-bencode)
- [Pony](https://github.com/mlajszczak/pony_bencode)
- [Prolog](https://github.com/mndrix/bencode) by mndrix
- [Python](Decoding_bencoded_data_with_python "Decoding bencoded data with python") by Hackeron
- [Scala](https://github.com/andreafey/torrent/blob/master/src/main/scala/torrent/Bcodr.scala) by Andrea Fey
- [Scheme](https://bitbucket.org/mahcuz/bencode-scheme) by [Mark Skilbeck](http://iammark.us/)
- [VBScript](http://demon.tw/my-work/vbs-bencode.html) by [Demon](http://demon.tw/)
- [Elixir](https://github.com/patrickgombert/bencodex) by Patrick Gombert
- [Ruby](https://github.com/kholbekj/bencoder) by [Kasper Holbek Jensen](http://kasper.codes/)

## Metainfoファイル構造

metainfoファイル内のすべてのデータはbencodingされています。bencodingの仕様は上記で定義されています。

metainfoファイル（".torrent"で終わるファイル）の内容は、以下にリストされたキーを含むbencoding辞書です。すべての文字列値はUTF-8でエンコードされています。

  - **info**: torrentのファイルを記述する辞書。2つの可能な形式があります：ディレクトリ構造のない「single-file」torrentの場合と、「multi-file」torrentの場合（詳細については以下を参照）
  - **announce**: trackerのannounce URL（文字列）
  - **announce-list**: （オプション）これは公式仕様への拡張で、後方互換性を提供します（文字列のリストのリスト）。
    - 仕様変更の公式要求は[こちら](http://bittorrent.org/beps/bep_0012.html)です。
  - **creation date**: （オプション）torrentの作成時刻、標準UNIX epoch形式（整数、1970年1月1日00:00:00 UTCからの秒数）
  - **comment**: （オプション）作者の自由形式のテキストコメント（文字列）
  - **created by**: （オプション）.torrentを作成するために使用されたプログラムの名前とバージョン（文字列）
  - **encoding**: （オプション）.torrent metafileの**info**辞書の**pieces**部分を生成するために使用された文字列エンコーディング形式（文字列）

### Info辞書

このセクションでは、「single file」と「multiple file」の両方のモードに共通するフィールドについて説明します。

  - **piece length**: 各pieceのバイト数（整数）

  - **pieces**: すべての20バイトSHA1ハッシュ値の連結からなる文字列、pieceごとに1つ（バイト文字列、つまりurlエンコードされていない）

  - **private**: （オプション）このフィールドは整数です。「1」に設定されている場合、clientはmetainfoファイルで明示的に記述されたtracker経由でのみ他のpeerを取得するためにその存在を公開しなければなりません。このフィールドが「0」に設定されているか存在しない場合、clientはPEX peer exchange、dhtなどの他の手段からpeerを取得することができます。ここで、「private」は「外部peer sourceなし」として読むことができます。

    - **注意:** private trackerを巡って多くの議論があります。
    - 仕様変更の公式要求は[こちら](http://bittorrent.org/beps/bep_0027.html)です。
    - Azureusは、private trackerを尊重する最初のclientでした。詳細については[彼らのwiki](http://wiki.vuze.com/w/Private_torrent)をご覧ください。

#### Single File ModeでのInfo

**single-file**モードの場合、**info**辞書には以下の構造が含まれます：

- **name**: ファイル名。これは純粋に助言的です（文字列）
- **length**: ファイルの長さ（バイト単位）（整数）
- **md5sum**: （オプション）ファイルのMD5サムに対応する32文字の16進文字列。これはBitTorrentでは全く使用されませんが、より大きな互換性のために一部のプログラムで含まれています。

#### Multiple File ModeでのInfo

**multi-file**モードの場合、**info**辞書には以下の構造が含まれます：

- **name**: すべてのファイルを格納するディレクトリの名前。これは純粋に助言的です（文字列）

- **files**: ファイルごとに1つずつの辞書のリスト。このリスト内の各辞書には以下のキーが含まれます：

  - **length**: ファイルの長さ（バイト単位）（整数）
  - **md5sum**: （オプション）ファイルのMD5サムに対応する32文字の16進文字列。これはBitTorrentでは全く使用されませんが、より大きな互換性のために一部のプログラムで含まれています。
  - **path**: パスとファイル名を一緒に表す1つ以上の文字列要素を含むリスト。リスト内の各要素は、ディレクトリ名または（最終要素の場合は）ファイル名のいずれかに対応します。例えば、ファイル "dir1/dir2/file.ext" は3つの文字列要素で構成されます：「dir1」、「dir2」、「file.ext」。これは **l4:**&#x64;ir1**4:**&#x64;ir2**8:**&#x66;ile.ext**e** のような文字列のbencodingリストとしてエンコードされます

### 注意

- **piece length**は名目上のpiece sizeを指定し、通常は2の累乗です。piece sizeは通常、torrent内のファイルデータの総量に基づいて選択され、大きすぎるpiece sizeが非効率性を引き起こし、小さすぎるpiece sizeが大きな.torrent metadataファイルを引き起こすという事実によって制約されます。歴史的に、piece sizeは.torrentファイルが約50-75kB以下になるように選択されていました（おそらくtorrentファイルをホストするサーバーの負荷を軽減するため）。

  - 現在のベストプラクティスは、*より大きな.torrentファイルになったとしても、8-10GB程度のtorrentでは、piece sizeを512KB以下に保つ*ことです。これにより、ファイル共有のためのより効率的なswarmが実現されます。最も一般的なサイズは256KB、512KB、1MBです。
  - 最終pieceを除いて、すべてのpieceは等しい長さです。最終pieceは不規則です。したがって、pieceの数は「ceil（総長／piece size）」によって決定されます。
  - multi-fileの場合のpiece境界の目的のため、ファイルデータを*files*リストに列挙された順序で各ファイルの連結で構成された1つの長い連続ストリームとして考えてください。pieceの数とその境界は、単一ファイルの場合と同じ方法で決定されます。pieceはファイル境界を跨ぐことがあります。

- 各pieceには、そのpiece内に含まれるデータの対応するSHA1ハッシュがあります。これらのハッシュは連結されて、上記の*info*辞書の*pieces*値を形成します。これはリストではなく、単一の文字列であることに注意してください。文字列の長さは20の倍数でなければなりません。

## Tracker HTTP/HTTPSプロトコル

trackerは、HTTP GETリクエストに応答するHTTP/HTTPSサービスです。リクエストには、trackerがtorrentに関する全体的な統計を保持するのに役立つclientからのメトリクスが含まれます。レスポンスには、clientがtorrentに参加するのに役立つpeerリストが含まれます。ベースURLは、metainfo（.torrent）ファイルで定義された「announce URL」で構成されます。その後、標準CGIメソッド（つまり、announce URLの後に「?」、その後「&」で区切られた「param=value」シーケンス）を使用して、このURLにパラメータが追加されます。

URL内のすべてのバイナリデータ（特にinfo\_hashとpeer\_id）は適切にエスケープされている必要があることに注意してください。これは、0-9、a-z、A-Z、「.」、「-」、「\_」、「\~」のセットにない任意のバイトは、「%nn」形式を使用してエンコードされる必要があることを意味します。ここで、nnはバイトの16進値です。（詳細については[RFC1738](http://www.faqs.org/rfcs/rfc1738.html)を参照してください。）

\x12\x34\x56\x78\x9a\xbc\xde\xf1\x23\x45\x67\x89\xab\xcd\xef\x12\x34\x56\x78\x9aの20バイトハッシュの場合、\
正しいエンコード形式は %124Vx%9A%BC%DE%F1%23Eg%89%AB%CD%EF%124Vx%9A です

### Tracker リクエストパラメータ

client->tracker GETリクエストで使用されるパラメータは以下の通りです：

- **info\_hash**: metainfoファイルの*info*キーの*値*のurlエンコードされた20バイトSHA1ハッシュ。上記の*info*キーの定義により、*値*はbencoding辞書になることに注意してください。

- **[peer\_id](#peer_id)**: clientによって開始時に生成される、clientのユニークIDとして使用されるurlエンコードされた20バイト文字列。これは任意の値を許可し、バイナリデータでも構いません。*現在、このpeer IDを生成するためのガイドラインはありません。ただし、少なくともローカルマシンに対してユニークでなければならず、したがってプロセスIDや開始時に記録されたタイムスタンプのようなものを組み込む必要があると正しく推測できます。一般的なclientエンコーディングについては、以下の[peer\_id](#peer_id)を参照してください。*

- **port**: clientがリッスンしているポート番号。BitTorrent用に予約されているポートは通常6881-6889です。clientは、この範囲内でポートを確立できない場合、諦めることを選択することがあります。

- **uploaded**: （clientがtrackerに「started」イベントを送信してから）アップロードされた総量（base 10 ASCII）。公式仕様で明示的に述べられていませんが、コンセンサスでは、これはアップロードされた総バイト数であるべきです。

- **downloaded**: （clientがtrackerに「started」イベントを送信してから）ダウンロードされた総量（base 10 ASCII）。公式仕様で明示的に述べられていませんが、コンセンサスでは、これはダウンロードされた総バイト数であるべきです。

- **left**: このclientがまだダウンロードする必要があるバイト数（base 10 ASCII）。*説明：100％完了し、torrentに含まれるすべてのファイルを取得するためにダウンロードする必要があるバイト数。*

- **compact**: これを1に設定すると、clientがcompactレスポンスを受け入れることを示します。peerリストは、peerあたり6バイトのpeer文字列に置き換えられます。最初の4バイトはホスト（ネットワークバイト順）、最後の2バイトはポート（再びネットワークバイト順）です。一部のtracker（帯域幅節約のため）はcompactレスポンスのみをサポートし、「compact=1」なしのリクエストを拒否するか、リクエストに「compact=0」が含まれていない限り単純にcompactレスポンスを送信することに注意してください（その場合、リクエストを拒否します）。

- **no\_peer\_id**: trackerがpeer辞書でpeer idフィールドを省略できることを示します。compactが有効になっている場合、このオプションは無視されます。

- **event**: 指定された場合、*started*、*completed*、*stopped*のいずれか（または空で、指定されていないのと同じ）でなければなりません。指定されていない場合、このリクエストは定期的な間隔で実行されるものです。

  - **started**: trackerへの最初のリクエストは、この値を持つeventキーを含む*必要があります*。
  - **stopped**: clientが正常にシャットダウンしている場合、trackerに送信される*必要があります*。
  - **completed**: ダウンロードが完了したときにtrackerに送信される*必要があります*。ただし、clientの開始時にダウンロードがすでに100％完了していた場合は送信してはいけません。おそらく、これはtrackerがこのイベントのみに基づいて「完了したダウンロード」メトリクスを増加させることを可能にするためです。

- **ip**: オプション。clientマシンの真のIPアドレス、ドット4組形式またはrfc3513で定義された16進IPv6アドレス。*注意：一般的に、このパラメータは、HTTPリクエストが来たIPアドレスからclientのアドレスを決定できるため、必要ありません。このパラメータは、リクエストが来たIPアドレスがclientのIPアドレスでない場合にのみ必要です。これは、clientがproxy（または透明Webproxy/cache）を通してtrackerと通信している場合に発生します。また、clientとtrackerの両方がNATゲートウェイの同じローカル側にある場合にも必要です。その理由は、そうでなければtrackerがclientの内部（RFC1918）アドレスを与えることになり、これはルート可能ではないからです。したがって、clientは外部peerに与えられるその（外部、ルート可能な）IPアドレスを明示的に述べる必要があります。さまざまなtrackerはこのパラメータを異なって扱います。一部は、リクエストが来たIPアドレスがRFC1918空間にある場合にのみこれを尊重します。他のものは無条件にこれを尊重し、まだ他のものは完全にこれを無視します。IPv6アドレス（例：2001:db8:1:2::100）の場合、clientがIPv6経由で通信できることのみを示します。*

- **numwant**: オプション。clientがtrackerから受信したいpeerの数。この値はゼロにすることが許可されています。省略された場合、通常はデフォルトで50 peerになります。

- **key**: オプション。他のpeerと共有されない追加の識別。IPアドレスが変更された場合に、clientが自分のアイデンティティを証明することを可能にすることを意図しています。

- **trackerid**: オプション。以前のannounceがtracker idを含んでいた場合、ここに設定されるべきです。

### Trackerレスポンス

trackerは、以下のキーを持つbencoding辞書からなる「text/plain」文書で応答します：

- **failure reason**: 存在する場合、他のキーは存在してはいけません。値は、リクエストが失敗した理由の人間が読める形式のエラーメッセージです（文字列）。

- **warning message**: （新しい、オプション）failure reasonと似ていますが、レスポンスは通常通り処理されます。warning messageはエラーのように表示されます。

- **interval**: clientがtrackerに定期的なリクエストを送信する間に待機すべき秒数の間隔

- **min interval**: （オプション）最小announceインターバル。存在する場合、clientはこれより頻繁に再announceしてはいけません。

- **tracker id**: clientが次のannounceで送り返すべき文字列。存在せず、以前のannounceがtracker idを送信した場合は、古い値を破棄しないでください；それを使い続けてください。

- **complete**: ファイル全体を持つpeerの数、つまりseeder（整数）

- **incomplete**: seederでないpeerの数、別名「leecher」（整数）

- **peers**: （辞書モデル）値は辞書のリストで、それぞれに以下のキーがあります：

  - **peer id**: tracker リクエストについて上記で説明されているように、peerの自己選択ID（文字列）
  - **ip**: peerのIPアドレス、IPv6（16進）またはIPv4（ドット4組）またはDNS名（文字列）
  - **port**: peerのポート番号（整数）

- **peers**: （バイナリモデル）上記で説明した辞書モデルを使用する代わりに、**peers**値は6バイトの倍数からなる文字列である場合があります。最初の4バイトはIPアドレスで、最後の2バイトはポート番号です。すべてネットワーク（ビッグエンディアン）記法です。

上記で述べたように、peerのリストはデフォルトで長さ50です。torrentにより少ないpeerがいる場合、リストはより小さくなります。そうでなければ、trackerはレスポンスに含めるpeerをランダムに選択します。*trackerは、リクエストに応答するときにpeer選択のためのより知的なメカニズムを実装することを選択することがあります。例えば、seedに他のseederを報告することは避けることができます。*

clientは、イベントが発生した場合（つまり、stoppedまたはcompleted）、またはclientがより多くのpeerについて学習する必要がある場合、指定された間隔よりも頻繁にtrackerにリクエストを送信することがあります。ただし、複数のpeerを取得するためにtrackerを「ハンマー」することは悪い慣行と見なされます。clientがレスポンスで大きなpeerリストを望む場合は、**numwant**パラメータを指定すべきです。

***実装者の注意**: 30 peerでも**十分**です。公式clientバージョン3は実際、30 peer未満の場合にのみ新しい接続を積極的に形成し、55がある場合は接続を拒否します。**この値はパフォーマンスにとって重要**です。新しいpieceのダウンロードが完了すると、HAVEメッセージ（以下を参照）をほとんどのアクティブなpeerに送信する必要があります。その結果、ブロードキャストトラフィックのコストはpeerの数に直接比例して増加します。25を超えると、新しいpeerがダウンロード速度を向上させる可能性は非常に低くなります。UIデザイナーは、これを変更するのが有用であることは非常にまれであるため、これを曖昧で変更しにくくすることを**強く推奨**します。*

## Tracker 'scrape' Convention

慣例により、ほとんどのtrackerは別の形式のリクエストをサポートしており、これはtrackerが管理している特定のtorrent（またはすべてのtorrent）の状態を照会します。これは「scrapeページ」と呼ばれ、そうでなければtrackerの統計ページを「スクリーンスクレイピング」する面倒なプロセスを自動化するためです。

scrape URLもHTTP GETメソッドで、上記で説明したものと似ています。ただし、ベースURLは異なります。scrape URLを導出するには、以下の手順を使用してください：announce URLから始めます。その中の最後の「/」を見つけます。その「/」の直後のテキストが「announce」でない場合、そのtrackerがscrape conventionをサポートしていないというサインとして扱われます。サポートしている場合は、「announce」を「scrape」に置き換えてscrapeページを見つけます。

例：（announce URL -> scrape URL）

```uri
  ~http://example.com/announce          -> ~http://example.com/scrape
  ~http://example.com/x/announce        -> ~http://example.com/x/scrape
  ~http://example.com/announce.php      -> ~http://example.com/scrape.php
  ~http://example.com/a                 -> (scrapeサポートなし)
  ~http://example.com/announce?x2%0644  -> ~http://example.com/scrape?x2%0644
  ~http://example.com/announce?x=2/4    -> (scrapeサポートなし)
  ~http://example.com/x%064announce     -> (scrapeサポートなし)
```

特に、エンティティアンクォートは*実行されない*ことに注意してください。この標準は、BitTorrent開発リストアーカイブでBramによって文書化されています：<http://groups.yahoo.com/group/BitTorrent/message/3275>

scrape URLは、上記で説明した20バイト値である、オプションパラメータ*info\_hash*によって補完される場合があります。これにより、trackerのレポートがその特定のtorrentに制限されます。そうでなければ、trackerが管理しているすべてのtorrentの統計が返されます。ソフトウェア作者は、trackerの負荷と帯域幅を削減するため、可能な限り*info\_hash*パラメータを使用することを強く推奨されます。

また、それをサポートするtrackerに複数のinfo\_hashパラメータを指定することもできます。これは公式仕様の一部ではありませんが、やや事実上の標準になっています - 例えば：

```uri
http://example.com/scrape.php?info_hash=aaaaaaaaaaaaaaaaaaaa&info_hash=bbbbbbbbbbbbbbbbbbbb&info_hash=cccccccccccccccccccc
```

このHTTP GETメソッドのレスポンスは、以下のキーを含むbencoding辞書からなる「text/plain」または時にはgzip圧縮文書です：

- **files**: 統計があるtorrentごとに1つのキー/値ペアを含む辞書。*info\_hash*が提供され、有効だった場合、この辞書は単一のキー/値を含みます。各キーは20バイトバイナリ*info\_hash*で構成されます。各エントリの値は以下を含む別の辞書です：

  - **complete**: ファイル全体を持つpeerの数、つまりseeder（整数）
  - **downloaded**: trackerが完了を登録した総回数（「event=complete」、つまりclientがtorrentのダウンロードを完了した）
  - **incomplete**: seederでないpeerの数、別名「leecher」（整数）
  - **name**: （オプション）.torrentファイルのinfoセクションの「name」ファイルで指定された、torrentの内部名

このレスポンスには3レベルの辞書ネストがあることに注意してください。例：

`d5:filesd20:....................d8:completei5e10:downloadedi50e10:incompletei10eeee`

ここで `....................` は20バイトのinfo\_hashで、5個のseeder、10個のleecher、50個の完了したダウンロードがあります。

### scrapeへの非公式拡張

以下のレスポンスキーが非公式に使用されています。非公式であるため、これらはすべてオプションです。

- **failure reason**: リクエストが失敗した理由の人間が読める形式のエラーメッセージ（文字列）。このキーを処理することが知られているclient：Azureus。
- **flags**: その他のフラグを含む辞書。flagsキーの値は別のネストされた辞書で、以下を含む可能性があります：
  - **min\_request\_interval**: このキーの値は、clientがtrackerを再びscrapeする前に待機すべき最小秒数を指定する整数です。このキーを送信することが知られているtracker：BNBT。このキーを処理することが知られているclient：Azureus。

## Peer wireプロトコル（TCP）

### 概要

peerプロトコルは、'*metainfo*'ファイルで説明されているpieceの交換を促進します。

*ここで、元の仕様もpeerプロトコルを説明する際に「piece」という用語を使用していたが、metainfoファイルの「piece」とは異なる用語としてであったことに注意してください。そのため、この仕様では、wire上でpeer間で交換されるデータを説明するために「block」という用語を使用します。*

clientは、リモートpeerとの各接続に対して状態情報を維持する必要があります：

- **choked**: リモートpeerがこのclientをchokeしているかどうか。peerがclientをchokeすると、clientがunchokeされるまで要求に応答されないという通知です。clientはblockの要求を送信しようとするべきではなく、すべての保留中の（未回答の）要求がリモートpeerによって破棄されたと見なすべきです。
- **interested**: リモートpeerがこのclientが提供するものに興味があるかどうか。これは、clientがそれらをunchokeしたときにリモートpeerがblockを要求し始めるという通知です。

*これは、clientがリモートpeerに興味があるかどうか、リモートpeerをchokeまたはunchokeしているかどうかも追跡する必要があることを意味することにも注意してください。したがって、実際のリストは次のようになります：*

- **am\_choking**: このclientがpeerをchokeしている
- **am\_interested**: このclientがpeerに興味がある
- **peer\_choking**: peerがこのclientをchokeしている
- **peer\_interested**: peerがこのclientに興味がある

client接続は「choked」および「not interested」として開始されます。つまり：

- **am\_choking** = 1
- **am\_interested** = 0
- **peer\_choking** = 1
- **peer\_interested** = 0

blockは、clientがpeerに興味があり、そのpeerがclientをchokeしていないときにclientによってダウンロードされます。blockは、clientがpeerをchokeしておらず、そのpeerがclientに興味があるときにclientによってアップロードされます。

clientがpeerに興味があるかどうかをpeerに知らせ続けることが重要です。この状態情報は、clientがchokeされている場合でも、各peerで最新に保たれるべきです。これにより、peerはclientがunchokeされたときにダウンロードを開始するか（およびその逆）を知ることができます。

### データタイプ

特に指定がない限り、peer wireプロトコルのすべての整数は4バイトビッグエンディアン値としてエンコードされます。これには、handshake後のすべてのメッセージの長さプレフィックスが含まれます。

### メッセージフロー

peer wireプロトコルは初期handshakeで構成されます。その後、peerは長さプレフィックス付きメッセージの交換を通じて通信します。長さプレフィックスは上記で説明した整数です。

### Handshake

handshakeは必須メッセージで、clientによって送信される最初のメッセージでなければなりません。長さは(49+len(pstr))バイトです。

*handshake: \<pstrlen>\<pstr>\<reserved>\<info\_hash>\<peer\_id>*

- **pstrlen**: \<pstr>の文字列長、単一の生バイト
- **pstr**: プロトコルの文字列識別子
- **reserved**: 8つの予約バイト。現在のすべての実装はすべてゼロを使用します。これらのバイトの各ビットは、プロトコルの動作を変更するために使用できます。*Bramからのメールによると、末尾ビットが最初に使用されるべきなので、先頭ビットが末尾ビットの意味を変更するために使用される可能性があります。*
- **info\_hash**: metainfoファイルのinfoキーの20バイトSHA1ハッシュ。これは、tracker リクエストで送信されるのと同じinfo\_hashです。
- **peer\_id**: clientのユニークIDとして使用される20バイト文字列。これは通常、tracker リクエストで送信されるのと同じpeer\_idですが（ただし常にそうとは限りません。例：Azureusの匿名オプション）。

BitTorrentプロトコルのバージョン1.0では、pstrlen = 19、pstr = "BitTorrent protocol"です。

接続の開始者は、handshakeを即座に送信することが期待されます。受信者は、複数のtorrentを同時に提供できる場合、開始者のhandshakeを待つことがあります（torrentはinfo\_hashによって一意に識別されます）。ただし、受信者はhandshakeのinfo\_hash部分を見るとすぐに応答する必要があります（peer idは受信者が自分のhandshakeを送信した後に送信されると推定されます）。trackerのNATチェック機能は、handshakeのpeer\_idフィールドを送信しません。

clientが現在提供していないinfo\_hashを持つhandshakeを受信した場合、clientは接続を切断する必要があります。

接続の開始者が期待されるpeer\_idと一致しないpeer\_idを持つhandshakeを受信した場合、開始者は接続を切断することが期待されます。*開始者はおそらく、peerが登録したpeer\_idを含むpeer情報をtrackerから受信したと推定されます。trackerからのpeer\_idとhandshakeでのpeer\_idは一致することが期待されます。*

#### peer\_id

peer\_idは正確に20バイト（文字）の長さです。

peer\_idにclientとclientバージョン情報をエンコードする主に2つの慣例があります：Azureus-styleとShadow's-style。

Azureus-styleは以下のエンコーディングを使用します：「-」、client idの2文字、バージョン番号の4桁のASCII数字、「-」、その後にランダムな数字。

例：'-AZ2060-'...

このエンコーディングスタイルを使用することが知られているclient：

- '7T' - [aTorrent for Android](https://play.google.com/store/apps/details?id=com.mobilityflow.torrent\&hl=en)
- 'AB' - [AnyEvent::BitTorrent](http://search.cpan.org/dist/AnyEvent-BitTorrent/)
- 'AG' - [Ares](http://aresgalaxy.sourceforge.net/)
- 'A\~' - [Ares](http://aresgalaxy.sourceforge.net/)
- 'AR' - [Arctic](http://dev.int64.org/arctic.html)
- 'AV' - [Avicora](http://sourceforge.net/projects/avicora/)
- 'AT' - [Artemis](http://www.cyberartemis.com/)
- 'AX' - [BitPump](http://www.analogx.com/contents/download/network/bitpump.htm)
- 'AZ' - [Azureus](http://azureus.sf.net/)
- 'BB' - [BitBuddy](http://www.btvampire.com/)
- 'BC' - [BitComet](http://www.bitcomet.com/)
- 'BE' - [Baretorrent](http://sourceforge.net/projects/baretorrent/)
- 'BF' - [Bitflu](http://bitflu.workaround.ch/)
- 'BG' - [BTG (uses Rasterbar libtorrent)](http://btg.berlios.de/)
- 'BL' - [BitCometLite (uses 6 digit version number)](http://www.bitcomet.com/tools/bitcometlite/)
- 'BL' - [BitBlinder](https://web.archive.org/web/20100407144429/http://www.bitblinder.com/)
- 'BP' - [BitTorrent Pro (Azureus + spyware)](http://www.intelpeers.com/)
- 'BR' - [BitRocket](http://www.bitrocket.org/)
- 'BS' - [BTSlave](http://btslave.sourceforge.net/)
- 'BT' - [mainline BitTorrent (versions >= 7.9)](http://bittorrent.com/)
- 'BT' - [BBtor](http://bbtor.net/)
- 'Bt' - [Bt](https://github.com/atomashpolskiy/bt)
- 'BW' - [BitWombat](http://bitwombat.com/)
- 'BX' - \~Bittorrent X
- 'CD' - [Enhanced CTorrent](http://www.rahul.net/dholmes/ctorrent/)
- 'CT' - [CTorrent](http://ctorrent.sourceforge.net/)
- 'DE' - [DelugeTorrent](http://www.deluge-torrent.org/)
- 'DP' - [Propagate Data Client](https://web.archive.org/web/20110128203704/http://propagatedata.com/)
- 'EB' - [EBit](http://dywt.com.cn/)
- 'ES' - [electric sheep](http://electricsheep.org/)
- 'FC' - [FileCroc](http://www.filecroc.com/)
- 'FD' - [Free Download Manager (versions >= 5.1.12)](http://www.freedownloadmanager.org/)
- 'FT' - [FoxTorrent](http://www.foxtorrent.com/)
- 'FX' - [Freebox BitTorrent](http://dev.freebox.fr/)
- 'GS' - [GSTorrent](http://sourceforge.net/projects/gstorrent)
- 'HK' - [Hekate](http://www.pps.jussieu.fr/~jch/software/hekate/)
- 'HL' - [Halite](http://www.binarynotions.com/halite.php)
- 'HM' - [hMule (uses Rasterbar libtorrent)](https://gforge.inria.fr/projects/hmule/)
- 'HN' - [Hydranode](http://hydranode.com/)
- 'IL' - [iLivid](http://www.ilivid.com/)
- 'JS' - [Justseed.it client](https://justseed.it/)
- 'JT' - [JavaTorrent](https://github.com/Johnnei/JavaTorrent)
- 'KG' - [KGet](http://kget.sourceforge.net/)
- 'KT' - [KTorrent](http://ktorrent.org/)
- 'LC' - [LeechCraft](http://leechcraft.org/)
- 'LH' - [LH-ABC](http://code.google.com/p/lh-abc)
- 'LP' - [Lphant](http://www.lphant.com/)
- 'LT' - [libtorrent](http://libtorrent.sf.net/)
- 'lt' - [libTorrent](http://libtorrent.rakshasa.no/)
- 'LW' - [LimeWire](http://www.limewire.org/)
- 'MK' - [Meerkat](http://www.themeerkat.net/)
- 'MO' - [MonoTorrent](http://monotorrent.blogspot.com/)
- 'MP' - [MooPolice](http://www.moopolice.de/)
- 'MR' - [Miro](http://www.getmiro.com/)
- 'MT' - [MoonlightTorrent](http://www.moonlighttorrent.com/)
- 'NB' - [Net::BitTorrent](http://search.cpan.org/dist/Net-BitTorrent)
- 'NX' - [Net Transport](http://www.xi-soft.com/)
- 'OS' - [OneSwarm](http://oneswarm.cs.washington.edu/)
- 'OT' - [OmegaTorrent](http://www.omegatorrent.com/)
- 'PB' - [Protocol::BitTorrent](http://search.cpan.org/perldoc?Protocol::BitTorrent)
- 'PD' - [Pando](http://www.pando.com/)
- 'PI' - [PicoTorrent](http://www.picotorrent.org/)
- 'PT' - [PHPTracker](http://php-tracker.org/)
- 'qB' - [qBittorrent](http://www.qbittorrent.org/)
- 'QD' - [QQDownload](http://im.qq.com/cyclone/)
- 'QT' - Qt 4 Torrent example
- 'RT' - [Retriever](http://www.halogenware.com/software/retriever.html)
- 'RZ' - [RezTorrent](https://launchpad.net/reztorrent)
- 'S\~' - [Shareaza alpha/beta](http://shareaza.sourceforge.net/)
- 'SB' - \~Swiftbit
- 'SD' - [Thunder (aka XùnLéi)](http://www.xunlei.com/)
- 'SM' - [SoMud](http://www.somud.com/)
- 'SP' - [BitSpirit](http://www.bitspirit.cc/en/)
- 'SS' - SwarmScope
- 'ST' - [SymTorrent](http://symtorrent.aut.bme.hu/)
- 'st' - [sharktorrent](http://sharktorrent.com/)
- 'SZ' - [Shareaza](http://shareaza.sourceforge.net/)
- 'TB' - [Torch](http://www.torchbrowser.com/)
- 'TE' - [terasaur Seed Bank](http://terasaur.org/)
- 'TL' - [Tribler (versions >= 6.1.0)](http://www.tribler.org/)
- 'TN' - TorrentDotNET
- 'TR' - [Transmission](https://transmissionbt.com/)
- 'TS' - [Torrentstorm](http://www.torrentstorm.com/)
- 'TT' - [TuoTu](http://www.tuotu.com/)
- 'UL' - uLeecher!
- 'UM' - [µTorrent for Mac](http://mac.utorrent.com/)
- 'UT' - [µTorrent](http://www.utorrent.com/)
- 'VG' - [Vagaa](http://www.vagaa.com/)
- 'WD' - [WebTorrent Desktop](http://webtorrent.io/desktop)
- 'WT' - [BitLet](http://www.bitlet.org/)
- 'WW' - [WebTorrent](http://webtorrent.io/)
- 'WY' - [FireTorrent](http://www.wyzo.com/firetorrent/)
- 'XF' - [Xfplay](http://www.xfplay.com/)
- 'XL' - [Xunlei](http://www.xunlei.com/)
- 'XS' - XSwifter
- 'XT' - [XanTorrent](http://www.xantorrent.pwp.blueyonder.co.uk/xantorrent.zip)
- 'XX' - [Xtorrent](http://www.xtorrent.com/)
- 'ZT' - [ZipTorrent](http://www.ziptorrent.com/)

野生で見られ、識別が必要なclient：

- 'BD' (例: -BD0300-)
- 'NP' (例: -NP0201-)
- 'wF' (例: -wF2200-)
- 'hk' (例: -hk0010-) 中国のIPアドレス、メッセージ0xAで未要求でinfo dictを送信、切断された後即座に再接続、reserved bytes = 01,01,01,01,00,00,02,01

Shadow's styleは以下のエンコーディングを使用します：client識別のためのASCII英数字1文字、バージョン番号のための最大5文字（5文字未満の場合は「-」でパッド）、その後3文字（一般的には「---」ですが、常にそうとは限りません）、その後ランダム文字。バージョン文字列の各文字は、0から63の数字を表します。「0」=0、...、「9」=9、「A」=10、...、「Z」=35、「a」=36、...、「z」=61、「.」=62、「-」=63。

エンコーディングスタイルについてのShad0wによる完全な説明（バージョン文字列後の3文字の使用に関する既存の慣例の情報を含む）は[ここ](http://forums.degreez.net/viewtopic.php?t=7070)で見つけることができます。

例：Shadow's 5.8.11の場合 'S58B-----'...

このエンコーディングスタイルを使用することが知られているclient：

- 'A' - [ABC](http://pingpong-abc.sourceforge.net/)
- 'O' - [Osprey Permaseed](http://osprey.ibiblio.org/)
- 'Q' - [BTQueue](http://btqueue.sourceforge.net/)
- 'R' - [Tribler (versions < 6.1.0)](http://www.tribler.org/)
- 'S' - [Shadow's client](http://bt.degreez.net/)
- 'T' - [BitTornado](http://bittornado.com/)
- 'U' - [UPnP NAT Bit Torrent](http://aaron2003.myftp.org/upnpclient.html)

Bramのclientは現在このスタイルを使用しています... 'M3-4-2--'または'M4-20-8-'。

[BitComet](http://www.bitcomet.com/)はまた別のことをします。そのpeer\_idは、4つのASCII文字「exbc」、その後2バイトxとy、その後ランダム文字で構成されます。バージョン番号は、小数点前のxが10進数で、小数点後の2桁の10進数としてのyです。[BitLord](http://www.bitlord.com/)は同じスキームを使用しますが、バージョンバイト後に「LORD」を追加します。BitComet用の[非公式パッチ](http://solidox.org/bc/)は、かつて「exbc」を「FUTB」に置き換えました。BitComet Peer IDのエンコーディングは、BitCometバージョン0.59時点でAzureus-styleに変更されました。

[XBT Client](http://xbtt.sourceforge.net/client/)には独自のスタイルもあります。そのpeer\_idは、3つの大文字「XBT」とその後のバージョン番号を表す3つのASCII数字で構成されます。clientがデバッグビルドの場合、7番目のバイトは小文字「d」、そうでなければ「-」です。その後「-」、その後ランダムな数字、大文字小文字の文字が続きます。例：最初の「XBT054d-」は、バージョン0.5.4のデバッグビルドを示します。

[Opera 8プレビューおよびOpera 9.xリリース](http://www.opera.com/)は以下のpeer\_idスキームを使用します：最初の2文字は「OP」で、次の4桁はビルド番号と等しくなります。すべての続く文字はランダムな小文字16進数字です。

[MLdonkey](http://mldonkey.sourceforge.net/Main_Page)は以下のpeer\_idスキームを使用します：最初の文字は「-ML」、その後ドット付きバージョン、その後「-」、その後ランダムネス。例：'-ML2.7.2-kgjjfkd'

[Bits on Wheels](http://www.bitsonwheels.com/)はパターン「-BOWxxx-yyyyyyyyyyyy」を使用します。ここで、yはランダム（大文字）で、xはバージョンに依存します。バージョン1.0.6はxxx = A0Cを持ちます。

[Queen Bee](http://queenbee.se/)はBramの新しいスタイルを使用します：「Q1-0-0--」または「Q1-10-0-」の後にランダムバイト。

[BitTyrant](http://bittyrant.cs.washington.edu/)はAzureusフォークで、1.1バージョンでpeer IDとして「AZ2500BT」+ランダムバイトを単純に使用します。ダッシュが不足していることに注意してください。

[TorrenTopia](http://www.torrentopia.org/)バージョン1.90は、Mainline 3.4.6を装うか、そこから派生しています。そのpeer IDは「346------」で始まります。

[BitSpirit](http://www.167bt.com/intl/)は、peer IDのためのいくつかのモードを持っています。1つのモードでは、peerのIDを読み取り、最初の8バイトを自分のIDの基礎として使用して再接続します。その実際のIDは、バージョン3.xでは「\0\3BS」（C記法）、バージョン2.xでは「\0\2BS」を最初の4バイトとして使用するようです。すべてのモードで、IDは「UDP0」で終わることがあります。BitSpirit 3.6以降、peer IDは文字「SP」でAzureusスタイルを使用しますが、末尾の「-」はありません（FlashGetのように）。

[Rufus](http://rufus.sourceforge.net/)は、最初の2バイトに10進ASCII値としてそのバージョンを使用します。3番目と4番目のバイトは「RS」です。その後、ユーザーのニックネームといくつかのランダムバイトが続きます。

[G3 Torrent](http://g3torrent.sourceforge.net/)は、peer IDを「-G3」で開始し、ユーザーのニックネームの最大9文字を追加します。

[FlashGet](http://www.flashget.com/)は「FG」でAzureusスタイルを使用しますが、末尾の「-」はありません。バージョン1.82.1002は、バージョン桁「0180」をまだ使用しています。

[BT Next Evolution](http://www.btnext.com/)はBitTornadoから派生していますが、Azureusスタイルを模倣しようとします。その結果、peer IDは「-NE」で始まり、4桁のバージョン番号で続き、その後Shad0wのpeer IDスタイルでclientのタイプを記述する3文字で直接続きます。

[AllPeers](http://www.allpeers.com/)は、ユーザー依存文字列のsha1ハッシュを取り、最初の数文字を「AP」+バージョン文字列+「-」に置き換えます。

[Qvod](http://www.qvod.com/)は、IDを4文字「QVOD」で開始し、4桁の10進数でビルド番号（現在「0054」）を続けます。残りの12文字はランダムな大文字16進数字です。中国で、開始の4文字をランダムバイトに置き換える人気の改造clientが存在するようです。

[SpywareTerminator](http://www.spywareterminator.com/)は、ユーザー間でシグネチャ更新を共有するためにlibtorrentを使用します。「CS」とバージョン桁「2500」でAzureusスタイルを使用します。

多くのclientは、すべてランダムな数字または12個のゼロの後にランダムな数字を使用しています（[Bramのclient](http://www.bittorrent.com/)の古いバージョンのように）。

### メッセージ

プロトコルの残りのメッセージはすべて、\<length prefix>\<message ID>\<payload>の形式を取ります。length prefixは4バイトビッグエンディアン値です。message IDは単一の10進バイトです。payloadはメッセージに依存します。

#### keep-alive: \<len=0000>

**keep-alive**メッセージは、length prefixがゼロに設定されたゼロバイトのメッセージです。message IDとpayloadはありません。peerは、特定の時間期間（**keep-alive**またはその他のメッセージ）メッセージを受信しない場合、接続を閉じることがあるため、一定時間コマンドが送信されていない場合は、接続を*生きた*状態に保つためにkeep-aliveメッセージを送信する必要があります。この時間は一般的に2分です。

#### choke: \<len=0001>\<id=0>

**choke**メッセージは固定長で、payloadはありません。

#### unchoke: \<len=0001>\<id=1>

**unchoke**メッセージは固定長で、payloadはありません。

#### interested: \<len=0001>\<id=2>

**interested**メッセージは固定長で、payloadはありません。

#### not interested: \<len=0001>\<id=3>

**not interested**メッセージは固定長で、payloadはありません。

#### have: \<len=0005>\<id=4>\<piece index>

**have**メッセージは固定長です。payloadは、ダウンロードが成功し、ハッシュによって検証されたpieceのゼロベースインデックスです。

*実装者の注意：これが厳密な定義ですが、実際にはいくつかのゲームが行われることがあります。特に、peerは既に持っているpieceをダウンロードする可能性が極めて低いため、peerはすでにそのpieceを持っているpeerにpieceを持っていることを宣伝しないことを選択することがあります。最低限「HAVE抑制」により、HAVEメッセージの50％削減がもたらされ、これはプロトコルオーバーヘッドの約25-35％削減に変換されます。同時に、そのpieceを既に持っているpeerにHAVEメッセージを送信することは、どのpieceが希少かを決定するのに有用であるため、価値があることがあります。*

*悪意のあるpeerは、peerが決してダウンロードしないことを知っているpieceを持っていると宣伝することも選択するかもしれません。この情報を使用してpeerをモデル化しようとすることは**悪いアイデア**です。*

#### bitfield: \<len=0001+X>\<id=5>\<bitfield>

**bitfield**メッセージは、handshakeシーケンスが完了した直後、他のメッセージが送信される前にのみ送信される場合があります。オプションで、clientがpieceを持たない場合は送信する必要がありません。

**bitfield**メッセージは可変長で、Xはbitfieldの長さです。payloadは、正常にダウンロードされたpieceを表すbitfieldです。最初のバイトの最上位ビットは、pieceインデックス0に対応します。クリアされたビットは不足しているpieceを示し、セットされたビットは有効で利用可能なpieceを示します。最後のスペアビットはゼロに設定されます。

一部のclient（例：Deluge）は、すべてのデータを持っていても、不足しているpieceで**bitfield**を送信します。その後、残りのpieceを**have**メッセージとして送信します。これは、BitTorrentプロトコルのISPフィルタリングに対して役立つと言っています。これは**lazy bitfield**と呼ばれます。

*間違った長さのbitfieldはエラーと見なされます。clientは、正しいサイズでないbitfieldを受信した場合、またはbitfieldにスペアビットがセットされている場合、接続を切断すべきです。*

#### request: \<len=0013>\<id=6>\<index>\<begin>\<length>

**request**メッセージは固定長で、blockを要求するために使用されます。payloadには以下の情報が含まれます：

- **index**: ゼロベースpieceインデックスを指定する整数
- **begin**: piece内のゼロベースバイトオフセットを指定する整数
- **length**: 要求される長さを指定する整数

***このセクションは議論中です！これを解決するために[ディスカッションページ](https://wiki.theory.org/Talk_BitTorrentSpecification.html#Messages:_request)を使用してください！***

***View #1*** 公式仕様によると、「現在のすべての実装は2^15（32KB）を使用し、2^17（128KB）を超える量を要求する接続を閉じます。」バージョン3または2004年の早期に、この動作は2^14（16KB）ブロックを使用するように変更されました。バージョン4.0または2005年中期時点で、mainlineは2^14（16KB）を超える要求で切断しました；一部のclientもこれに従いました。ブロック要求はpiece（>=2^18バイト）より小さいため、完全なpieceをダウンロードするには複数の要求が必要になることに注意してください。

*厳密に言えば、仕様は2^15（32KB）要求を許可します。現実は、ほぼすべてのclientが2^14（16KB）要求を使用することです。そのサイズを強制するclientのために、実装はそのサイズの要求を行うことが推奨されます。より小さな要求がより多くの要求を追跡することによるオーバーヘッドの増加をもたらすため、実装者は2^14（16KB）を下回ることを避けることが推奨されます。*

*要求blockサイズ制限の強制の選択は、それほど明確ではありません。mainlineバージョン4が16KB要求を強制することで、ほとんどのclientがそのサイズを使用します。同時に2^14（16KB）が現在の*&#x73;em&#x69;*-公式（公式プロトコル文書が更新されていないため*&#x73;em&#x69;*のみ）制限であるため、それを強制することは間違いではありません。同時に、より大きな要求を許可することは可能なpeerのセットを拡大し、非常に低帯域幅接続（<256kbps）以外では、1つのchoke-timeperiodで複数のblockがダウンロードされるため、古い制限のみを強制することは最小限のパフォーマンス低下を引き起こします。この要因により、古い2^17（128KB）最大サイズ制限のみを強制することが推奨されます。*

***View #2*** このセクションは、このページが存在していた期間の大部分で虚偽を含んでいました。これは、同じセクションで間違った情報が追加されたため、私（uau）が修正する3回目なので、再び壊される可能性があるため、完全に書き直すことはしません... 現在のバージョンには少なくとも以下のエラーがあります：Mainlineは、まだ存在する唯一のclientだった時に2^14（16384）バイト要求の使用を開始しました；「公式仕様」のみが、実際にはデフォルトサイズでも許可される最大値でもなかった廃止された32768バイト値について話していました。バージョン4では要求動作は変更されませんでしたが、許可される最大サイズはデフォルトサイズと等しくなるように変更されました。最新のmainlineバージョンでは、maxは32768に変更されました（これは、最初の古いバージョン以降、デフォルトサイズまたは最大サイズのいずれかで32768が初めて登場することに注意してください）。「ほとんどの古いclientは32KB要求を使用する」は偽です。より大きな要求についての議論は、レイテンシ効果を考慮できていません。

#### piece: \<len=0009+X>\<id=7>\<index>\<begin>\<block>

**piece**メッセージは可変長で、Xはblockの長さです。payloadには以下の情報が含まれます：

- **index**: ゼロベースpieceインデックスを指定する整数
- **begin**: piece内のゼロベースバイトオフセットを指定する整数
- **block**: indexで指定されたpieceのサブセットであるデータのblock

#### cancel: \<len=0013>\<id=8>\<index>\<begin>\<length>

**cancel**メッセージは固定長で、block要求をキャンセルするために使用されます。payloadは「request」メッセージのものと同一です。これは通常、「End Game」中に使用されます（以下のAlgorithmsセクションを参照）。

#### port: \<len=0003>\<id=9>\<listen-port>

**port**メッセージは、DHTtrackerを実装するMainlineの新しいバージョンによって送信されます。listenポートは、このpeerのDHTノードがリッスンしているポートです。このpeerは、ローカルルーティングテーブルに挿入されるべきです（DHTtrackerがサポートされている場合）。

## アルゴリズム

### キューイング

***このセクションは議論中です！これを解決するために[ディスカッションページ](https://wiki.theory.org/Talk_BitTorrentSpecification.html#Algorithms:_Queuing)を使用してください！***

***View #1*** 一般的に、peerは各接続でいくつかの未履行の要求を保持することが推奨されます。これは、そうでなければ1つのblockのダウンロードから新しいblockのダウンロード開始まで完全なround tripが必要になるためです（PIECEメッセージと次のREQUESTメッセージ間のround trip）。高BDP（bandwidth-delay-product、高レイテンシまたは高帯域幅）リンクでは、これは実質的なパフォーマンス損失をもたらす可能性があります。

*実装者の注意：これは**最も重要なパフォーマンス項目**です。50msレイテンシの5mbpsリンクで16KBblockに対する10要求の静的キューは合理的です。より大きな帯域幅のリンクが非常に一般的になっているため、UIデザイナーはこれを変更のために容易に利用できるようにすることを強く推奨されます。特に、ケーブルモデムはトラフィックポリシングで知られており、これを増加させることで、これによって引き起こされた問題の一部を軽減したかもしれません。*

***View #2*** 注意：この「キューイング」セクションの情報の多くは偽または誤解を招くものです。「デフォルトで5つの未処理要求」は長い間真実ではなく、「32KBブロック」は通常32KBブロックを使用しないため誤解を招き、変更してその効果を測定しようとすることによってキュー長を調整することは悪いアイデアであることを指摘するだけにとどめます。

### Super Seeding

*（これは元の仕様の一部ではありませんでした）*

*S-5.5以降のsuper-seed機能は、限られた帯域幅を持つtorrent開始者が大きなtorrentを「pump up」するのを助けるために設計された新しいseedingアルゴリズムで、torrentで新しいseedを生成するためにアップロードする必要があるデータ量を削減します。*

*seedingclientが「super-seed mode」に入ると、標準seedとして動作せず、データを持たない通常のclientとして偽装します。clientが接続すると、送信されなかったpiece、またはすべてのpieceがすでに送信されている場合は非常に希少なpieceを受信したことを通知します。これにより、clientはそのpieceのみをダウンロードしようとするように誘導されます。*

*clientがpieceのダウンロードを完了すると、seedは、以前に送信したpieceが少なくとも1つの他のclientに存在することを確認するまで、他のpieceについて通知しません。それまで、clientはseedの他のpieceにアクセスできず、したがってseedの帯域幅を無駄にしません。*

*この方法により、peerに最も希少なデータのみを取るように誘導し、冗長なデータの送信量を削減し、swarmに貢献しないpeerに送信されるデータ量を制限することで、大幅に高いseeding効率が実現されました。これ以前は、seedは他のclientがseedになる前にtorrentの総サイズの150％から200％をアップロードする必要があったかもしれません。しかし、super-seed modeで実行されている単一のclientでseedされた大きなtorrentは、データの105％のみをアップロードした後にそれを行うことができました。これは、標準seedを使用する場合よりも150-200％効率的です。*

*Super-seed modeは一般使用には推奨されません。希少なデータのより広い配布には役立ちますが、clientがダウンロードできるpieceの選択を制限するため、すでに部分的に取得したpieceのデータをダウンロードするそれらのclientの能力も制限します。したがって、super-seed modeは初期seedingサーバーにのみ推奨されます。*

### Piece ダウンロード戦略

clientはランダム順でpieceをダウンロードすることを選択することがあります。

*より良い戦略は、*[rarest first](Availability "Availability")*順でpieceをダウンロードすることです。clientは、各peerから初期bitfieldを保持し、すべての**have**メッセージでそれを更新することでこれを決定できます。その後、clientはこれらのpeer bitfieldで最も少なく現れるpieceをダウンロードできます。Rarest First戦略には、多くのclientが同じ「最も一般的でない」pieceに飛び乗ろうとすることが逆効果になるため、少なくとも最も一般的でないいくつかのpieceの中でのランダム化を含めるべきであることに注意してください。*

### End Game

ダウンロードがほぼ完了すると、最後のいくつかのblockがゆっくりと入ってくる傾向があります。これを速めるために、clientは不足しているすべてのblockの要求をすべてのpeerに送信します。これが恐ろしく非効率になることを防ぐため、clientはblockが到着するたびに他の全員にキャンセルも送信します。

*ここでガイドまたは推奨ベストプラクティスとして使用できる文書化された閾値、推奨パーセンテージ、またはblock数はありません。*

*end gameモードに入るタイミングは議論の領域です。一部のclientは、すべてのpieceが要求されたときにend gameに入ります。他のものは、残りのblock数が転送中のblock数を下回り、20以下になるまで待ちます。保留中のblock数を少なく（1または2block）保ってオーバーヘッドを最小化し、要求されるblockをランダム化すると、重複をダウンロードする可能性が低くなることに同意があるようです。プロトコルオーバーヘッドについて詳しくは：* <http://hal.inria.fr/inria-00000156/en>。

### Choking と Optimistic Unchoking

Chokingはいくつかの理由で行われます。TCP輻輳制御は、同時に多くの接続を介して送信するときに非常に悪く動作します。また、chokingにより、各peerは一貫したダウンロード率を確保するためにtit-for-tat風のアルゴリズムを使用できます。

以下で説明するchokingアルゴリズムは、現在展開されているものです。すべての新しいアルゴリズムが、完全に自分自身からなるネットワークと、主にこれからなるネットワークの両方でうまく動作することが非常に重要です。

良いchokingアルゴリズムが満たすべきいくつかの基準があります。良いTCPパフォーマンスのために同時アップロード数をキャップすべきです。「fibrillation」として知られる、chokeとunchokeを素早く行うことを避けるべきです。ダウンロードさせてくれるpeerに対して相互主義を示すべきです。最後に、現在使用されているものよりも良いかもしれないかを見つけるため、未使用の接続を時々試すべきです。これはoptimistic unchokingとして知られています。

現在展開されているchokingアルゴリズムは、10秒に1回だけchoked peerを変更することでfibrillationを避けます。

相互主義とアップロード数のキャッピングは、最高のアップロード率を持ち、興味を持っている4つのpeerをunchokeすることで管理されます。これにより、clientのダウンロード率が最大化されます。これらの4つのpeerは、clientからのダウンロードに興味があるため*downloader*と呼ばれます。

より良いアップロード率（*downloader*と比較して）を持つが興味を持たないpeerはunchokeされます。彼らが興味を持つようになると、最悪のアップロード率を持つ*downloader*がchokeされます。clientが完全なファイルを持つ場合、unchokeするpeerを決定するためにダウンロード率ではなくアップロード率を使用します。

optimistic unchokingのため、いつでもアップロード率に関係なくunchokeされる単一のpeer（興味がある場合、許可された4つの*downloader*の1つとしてカウント）があります。optimistic unchokeされるpeerは30秒ごとにローテーションします。新しく接続されたpeerは、ローテーションの他の場所よりも現在のoptimistic unchokeとして開始する可能性が3倍高くなります。これにより、アップロードする完全なpieceを取得するまともな機会が与えられます。

#### Anti-snubbing

時々、BitTorrent peerは以前にダウンロードしていたすべてのpeerによってchokeされます。そのような場合、optimistic unchokeがより良いpeerを見つけるまで、通常は貧弱なダウンロード率を継続して取得します。この問題を軽減するため、peerからダウンロードしながら1分以上pieceデータを取得しない場合、BitTorrentはそのpeerによって「snubbed」されていると仮定し、optimistic unchokeとしてを除いてそれにアップロードしません。これにより、頻繁に同時に複数のoptimistic unchoke（上記で言及した正確に1つのoptimistic unchokeルールの例外）が発生し、ダウンロード率がfalterしたときにずっと速く回復します。

## プロトコルへの公式拡張

現在、プロトコルにはいくつかの公式拡張があります。

#### Fast Peers Extensions

- Reserved Bit: 8番目の予約バイトの3番目の最下位ビット、つまりreserved\[7] |= 0x04

これらの拡張は複数の目的を果たします。choke状態に関係なくダウンロードが許可される特定のpieceセットをpeerに与えることで、peerがswarmにより迅速にbootstrapできるようにします。HaveAllとHaveNoneメッセージを追加し、以前は暗黙的な拒否のみが可能だったpiece要求の明示的な拒否を可能にすることで、メッセージオーバーヘッドを削減します。つまり、peerは決して配信されないpieceを待ち続ける可能性がありました。

仕様は、BitTorrentサイトで文書化されています：<http://bittorrent.org/beps/bep_0006.html>。

#### Distributed Hash Table

- Reserved Bit: 8番目の予約バイトの最後のビット、つまりreserved\[7] |= 0x01

この拡張は、標準trackerを使用せずにtorrentをダウンロードしているpeerの追跡を可能にするためのものです。このプロトコルを実装するpeerは「tracker」となり、新しいpeerを見つけるために使用できる他のnode/peerのリストを保存します。

仕様は、BitTorrentサイトで文書化されています：<http://bittorrent.org/beps/bep_0005.html>。

BEP-32は、IPv6サポートでDHTを拡張し、いくつかの軽微な方法で仕様を更新します。<http://www.pps.jussieu.fr/~jch/software/bittorrent/bep-dht-ipv6.html>

#### Connection Obfuscation

この拡張により、peer間で難読化（暗号化）された接続の作成が可能になります。これは、BitTorrentトラフィックを制限するISPを回避するために使用できます。

仕様は<http://wiki.vuze.com/w/Message_Stream_Encryption>で文書化されています。

*文書はかなり完全ですが、理想的には、暗号化接続を試行するタイミング、通常の接続へのフォールバック手順などについていくつかの点で明確化される必要があります。*

## プロトコルへの非公式拡張

#### Azureus Messaging Protocol

- Reserved Bit: 1

それ自体がプロトコル - 2つのclientがプロトコルをサポートすることを示した場合、それらはそれを使用するように切り替えるべきです。通常のBitTorrentと拡張メッセージの両方をその上で送信することを可能にし、[ここ](http://wiki.vuze.com/w/Azureus_messaging_protocol)で文書化されています。現在、AzureusとTransmissionによって実装されています。

このプロトコルとLibTorrent拡張プロトコルを同時に使用することは不可能です - 両方のclientが両方をサポートすることを示した場合、[Extension Negotiation Protocol](http://wiki.vuze.com/w/Extension_negotiation_protocol)で定義されたセマンティクスに従うべきです。

#### WebSeeding

HTTPサーバー経由でtorrentをseedする可能性は、一般的にWebSeedingと呼ばれます。これにより、HTTPサーバーがBitTorrentネットワークでpeerとして動作できます。

torrentダウンロードとHTTPダウンロードを組み合わせる方法については、少なくとも2つの仕様があります。BitTornadoで実装された最初の標準は、clientでの実装は非常に簡単ですが、サーバー側でリクエストを処理するスクリプトを必要するという点でHTTPに侵入的です。つまり、単純なファイルを提供するだけの普通のHTTPサーバーでは不十分です。利点は、スクリプトがより悪用耐性を持てることです。この仕様は[http://bittornado.com/docs/webseed-spec.txt](http://www.bittornado.com/docs/webseed-spec.txt)または[BEP-17](http://bittorrent.org/beps/bep_0017.html)で見つけることができます。

2番目の仕様では、clientからやや多くが要求されますが、普通のHTTPサーバーからダウンロードします。これは<http://www.getright.com/seedtorrent.html>または[BEP-19](http://bittorrent.org/beps/bep_0019.html)で指定されています。GetRight、libtorrent、Mainline、BitComet、Vuzeによって実装されています。

#### Extension protocol

- Reserved Bit: 44、6番目の予約バイトの4番目の最上位ビット、つまりreserved\[5] |= 0x10

これは拡張情報を交換するためのプロトコルで、azureusの拡張プロトコルの初期バージョンから派生しました。定義された拡張メッセージを含む任意のhandshake情報を交換するための1つのメッセージを追加し、拡張を特定のメッセージIDにマッピングします。これは<http://www.libtorrent.org/extension_protocol.html>で文書化されており、少なくともlibtorrent、uTorrent、Mainline、Transmission、Azureus、BitCometによって実装されています。

このプロトコルとAzureus Messaging Protocolを同時に使用することは不可能です - 両方のclientが両方をサポートすることを示した場合、[Extension Negotiation Protocol](http://wiki.vuze.com/w/Extension_negotiation_protocol)で定義されたセマンティクスに従うべきです。

#### Extension Negotiation Protocol

- Reserved bits: 47と48

これらのビットは、Azureus Messaging ProtocolとLibTorrentの拡張プロトコルの両方をサポートする2つのclientが、通信に使用すべき2つの拡張のうちどちらを決定できるようにするために使用され、[ここ](http://wiki.vuze.com/w/Extension_negotiation_protocol)で定義されています。

#### BitTorrent Location-aware Protocol 1.0

- Reserved Bit: 21

より良いパフォーマンスのためにpeerの位置（地理的条件）を考慮するプロトコル。仕様は[ここ](https://wiki.theory.org/BitTorrent_Location-aware_Protocol_1)で見つけることができます。

#### SimpleBT Extension Protocol

- Reserved Bits: 最初の予約バイト = 0x01、以下のバイトはゼロに設定する必要がある場合があります

peer交換と接続統計交換を追加するためにメッセージid 9を使用する拡張。仕様は[ここ](http://web.archive.org/web/20031002201124/btfans.3322.org/simplebt/ProtocalExtension)で見つけることができます。この拡張は、SimpleBT 0.32から0.36.1で使用されていました。SimpleBTの後のバージョンはBitCometと呼ばれ、似ているが互換性のないBitComet Extension Protocolを使用しました。

#### BitComet Extension Protocol

- Reserved Bits: 最初の2つの予約バイト = "ex"

公式文書が存在しないようです。

このプロトコルでは、peerはメッセージ\<len=0001+X>\<id=0xA0>\<extension 1>...\<extension X>を送信することでサポートされている拡張を発表します。ここで\<extension n>は（通常）サポートされている拡張のメッセージidです。拡張が複数のメッセージで構成される場合、すべてのidを言及する必要があります。

現在使用されている拡張（TODO：セマンティクスのリバースエンジニアリング）：

- 0xA0 (EXT\_SUPPORT) 上記を参照、パラメータリストに含める必要があります
- 0xA1 (EXT\_PEERREQ) peer交換を要求、EXT\_PEERSと連携して使用
- 0xA2 (EXT\_PEERS) EXT\_PEERREQへの返信とその後の更新のため
- 0xA3 (EXT\_AUTH\_SEED) BitComet 0.53で登場、EXT\_AUTH\_CRYPTOEDと連携して使用
- 0xA4 (EXT\_AUTH\_CRYPTOED)
- 0xA5 (EXT\_CONNGRANT) BitComet 0.48で登場、EXT\_CONNACCEPTと連携して使用
- 0xA6 (EXT\_CONNACCEPT)
- 0x06 (?) EXT\_CONNACCEPTの代わりにBitSpiritによって発表
- 0xA7 (EXT\_CHAT\_MESSAGE) BitComet 0.53で登場、0.71で消失
- 0xA9 (EXT\_HASH\_REQ) BitComet 0.54で登場、0.71で消失、EXT\_HASHと連携して使用
- 0xAA (EXT\_HASH)
- 0xAB (EXT\_REPORT\_RATE\_old) BitComet 0.54で登場、0.57でEXT\_REPORT\_RATE\_newに置き換えられました
- 0xAC (EXT\_REPORT\_INFO) BitComet 0.54で登場、0.71で消失、0.82で再登場
- 0xAD (EXT\_REPORT\_RATE\_new) BitComet 0.57で登場、0.75で消失、0.82で再登場
- 0xAE (EXT\_BC\_PASSPORT) BitComet 0.75で登場
- 0xAF (EXT\_DHE\_PREFERRED) BitComet 0.75で登場
- 0xB0 (?) BitComet 0.86で登場
- 0xC0 (?) メッセージidに対応しない、BitComet 0.49で登場

最小実装はEXT\_SUPPORTを受け入れるだけで済みますが、EXT\_PEERREQとEXT\_PEERSは既知のすべての実装でサポートされています。

## Reserved Bytes

*以下の表では、識別しやすくするため予約ビットに1-64の番号が付けられています。ビット1は最初の予約バイトの最上位ビットに対応します。ビット8は最初の予約バイトの最下位ビット（つまりbyte\[0] |= 0x01）に対応します。ビット64は最後の予約バイトの最下位ビット、つまりbyte\[7] |= 0x01です*

*オレンジのビットは既知の非公式拡張、赤のビットは未知の非公式拡張です。*

| Bit     | Use                                    | Azureus | BitComet | MainLine | MonoTorrent | µTorrent | libtorrent | KTorrent | BitLord | XBT | Transmission |
| ------- | -------------------------------------- | ------- | -------- | -------- | ----------- | -------- | ---------- | -------- | ------- | --- | ------------ |
| 1       | Azureus Extended Messaging             | Yes     | Unknown  | Unknown  | Unknown     | Unknown  | No         | No       | No      | No  | Yes          |
| 1-16    | BitComet Extension protocol            | No      | Yes      | No       | No          | No       | No         | No       | Yes     | No  | No           |
| 21      | BitTorrent Location-aware Protocol 1.0 | No      | No       | No       | No          | No       | No         | No       | No      | No  | No           |
| 44      | Extension protocol                     | Yes     | Yes      | Yes      | Yes         | Yes      | Yes        | No       | No      | Yes | Yes          |
| 47 - 48 | Extension Negotiation Protocol         | Yes     | No       | No       | No          | No       | No         | No       | No      | No  | Yes          |
| 61      | NAT Traversal                          | No      | Unknown  | Yes      | Unknown     | Unknown  | No         | Unknown  | Unknown | No  | Unknown      |
| 62      | Fast Peers                             | No      | Unknown  | Yes      | Yes         | Unknown  | Yes        | Yes      | No      | No  | Unknown      |
| 63      | XBT Peer Exchange                      | No      | Unknown  | No       | Unknown     | Unknown  | No         | Unknown  | Unknown | Yes | Unknown      |
| 64      | DHT                                    | No      | Yes      | Yes      | Yes         | Yes      | Yes        | Yes      | No      | No  | Unknown      |
| 64      | XBT Metadata Exchange                  | No      | Unknown  | No       | Unknown     | Unknown  | No         | Unknown  | Unknown | Yes | Unknown      |