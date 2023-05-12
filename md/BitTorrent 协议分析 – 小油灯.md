> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.zhaoyingze.com](https://www.zhaoyingze.com/bittorrent%E5%8D%8F%E8%AE%AE%E5%88%86%E6%9E%90/)

> BitTorrent (简称 BT) 协议是一种点对点 (Peer-to-Peer, 或简写为 P2P) 传输协议, 它被设计用来高效地分发文件 (尤其是对于大文件、多人同时下载时效率非常高), 在传统的场景下, 用户希望下载一个文件, 一般都会通过比如 HTTP / FTP 的方式从目标站点的服务器上下载, 服务器的带宽通常都是有限的, 当同时下载的用户过多时, 将超出服务器的带宽限制, 这时大家都会下载地很慢甚至是无法继续下载, 而 BitTorrent 协议便是为了解决这个问题, 它的创新点在于将文件分片, 每个终端用户下载文件分片, 在下载的同时也会互相分发自己已下载的文件分片给其它正在下载的用户, 从而将部分原本应从服务器拉取数据所造成的带宽压力分散给了终端用户, 所有正在同时下载同一文件的终端用户构成一个图结构, 互相点对点式地分发文件片段, 因此对于大文件、多人同时下载时有非常好的文件分发效率, BitTorrent 协议不是 IETF 制定的互联网标准化协议, 因而没有 RFC 文档, 关于 BitTorrent 协议标准《The BitTorrent Protocol Specification》可以从 bittorrent.org 上获得。

概述
--

BitTorrent (简称 BT) 协议是一种点对点 (Peer-to-Peer, 或简写为 P2P) 传输协议, 它被设计用来高效地分发文件 (尤其是对于大文件、多人同时下载时效率非常高), 在传统的场景下, 用户希望下载一个文件, 一般都会通过比如 HTTP / FTP 的方式从目标站点的服务器上下载, 服务器的带宽通常都是有限的, 当同时下载的用户过多时, 将超出服务器的带宽限制, 这时大家都会下载地很慢甚至是无法继续下载, 而 BitTorrent 协议便是为了解决这个问题, 它的创新点在于将文件分片, 每个终端用户下载文件分片, 在下载的同时也会互相分发自己已下载的文件分片给其它正在下载的用户, 从而将部分原本应从服务器拉取数据所造成的带宽压力分散给了终端用户, 所有正在同时下载同一文件的终端用户构成一个图结构, 互相点对点式地分发文件片段, 因此对于大文件、多人同时下载时有非常好的文件分发效率, BitTorrent 协议不是 IETF 制定的互联网标准化协议, 因而没有 RFC 文档, 关于 BitTorrent 协议标准[《The BitTorrent Protocol Specification》](http://bittorrent.org/beps/bep_0003.html)可以从 [bittorrent.org](http://bittorrent.org/beps/bep_0003.html) 上获得。BitTorrent 协议是架构于 TCP/IP 协议之上的一个 P2P 文件传输通信协议，是一个应用层协议。

1.  **资源描述**：一个文件资源的下载是从一个 BT 种子（.torrent 文件）或磁力链接（magnet link）开始的，在 BT 协议中，也被称为元数据（metainfo），元数据中包含该文件资源如何被分片，以及每个分片的大小，校验哈希等信息。
2.  **资源定位**：用户从数据发布者获取到文件的元数据后，接下来需要通过资源定位的机制，来找到拥有该资源的 peer，如果无法找到拥有该资源的 peer，也就意味着下载失败。通常一个 peer 只包含一个文件资源的部分分片信息，所以需要找到多个 peer 才能完整地拼接出整个文件。
3.  **数据下载**：在获取到拥有资源的 peer 列表后，需要通过 peer 传输协议来沟通协商各自拥有的分片资源，请求下载各自所需的分片，来完成整个资源的下载。

![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-12.png)

资源描述
----

### BT 种子（.torrent）的文件格式

BT 协议的 metainfo file 用来描述资源的元信息, 如资源的 URL, 资源的 SHA1 哈希等, 它的文件后缀为 .torrent, 也就是常所说的 BT 种子文件 (BT 种子文件中的文本内容统一使用 UTF-8 编码), BT 种子文件使用 bencoding 编码来描述信息的。

*   **Bencoding 编码**（[BEP_0003](http://bittorrent.org/beps/bep_0003.html)）

Bencoding 共有 4 种基本数据类型, 分别是 String, Integer, List 和 Dictionary

<table><thead><tr><th><strong>类型</strong></th><th><strong>定义</strong></th><th><strong>示例</strong></th></tr></thead><tbody><tr><td>String：字符串类型</td><td>使用形如 length-prefix : string-content 的形式来描述字符串, 其中 length-prefix 是字符串的长度, string-content 是字符串内容</td><td>4:spam 表示字符串 \'spam\'</td></tr><tr><td>Integer：整数类型</td><td>使用字符 i 作为前缀, 使用字符 e 作为后缀, 在前缀 i 和后缀 e 中间的是整数本身 (整数都使用十进制表达)</td><td>i3e 表示整数 3, i-3e 代表整数 -3</td></tr><tr><td>List：列表类型</td><td>使用字符 l 作为前缀, 使用字符 e 作为后缀, 在前缀 l 和后缀 e 之间的是列表的元素 (列表元素也都使用 bencoding 编码方法)</td><td>l4:spam4:eggse 表示 [\'spam\', \'eggs\']</td></tr><tr><td>Dictionary：字典类型</td><td>使用字符 d 作为前缀, 使用字符 e 作为后缀, 在前缀 d 和后缀 e 之间的是 k-v 对, 其中字典的键 (key) 必须是 bencoding 编码的 String 形式, 而字典键所对应的值可以是 bencoding 编码的任何一种类型 (String, Integer, List, Dictionary)</td><td>d3:cow3:moo4:spam4:eggse 表示 {\'cow\': \'moo\', \'spam\':\'eggs\'} d4:spaml1:a1:bee 表示 {\'spam\':[\'a\', \'b\']}</td></tr></tbody></table>

*   **文件格式**

BT 种子文件实际上就是一个 bencoding 编码的 Dictionary, [BEP_0003](http://bittorrent.org/beps/bep_0003.html) 定义了两个 Key, 分别是 announce 和 info。 [BEP_0012（Metadata Extension）](http://bittorrent.org/beps/bep_0012.html)扩展协议增加了 announce-list 字段。另外，还有一些常用，但不在协议定义范围内的字段，如注释、创建时间等。

<table><thead><tr><th><strong>key</strong></th><th><strong>value</strong></th><th><strong>协议版本</strong></th></tr></thead><tbody><tr><td>announce</td><td>BT Tracker 的 URL</td><td><a href="http://bittorrent.org/beps/bep_0003.html">BEP_0003</a></td></tr><tr><td>announce-list</td><td>可选, 为备用 BT Tracker 的 URL</td><td><a href="http://bittorrent.org/beps/bep_0012.html">BEP_0012</a></td></tr><tr><td>nodes</td><td>可选，对于无 tracker 的种子文件（没有 announce 字段），可以给出 DHT 的临近的 K 个 nodes 列表，来查找 peer 信息</td><td><a href="http://bittorrent.org/beps/bep_0005.html">BEP_0005</a></td></tr><tr><td>creation date</td><td>可选, 种子的创建时间 (UNIX 时间戳), 如 13:creation datei1542799851e</td><td></td></tr><tr><td>comment</td><td>可选, 种子制作者添加的备注信息</td><td></td></tr><tr><td>created by</td><td>可选, 该关键字对应的值为生成种子文件的 BT 客户端软件的信息, 如客户端名、版本号等</td><td></td></tr><tr><td>info_hash</td><td>info 字段内容的 SHA1 哈希值，用于唯一标识该资源</td><td></td></tr><tr><td>info</td><td>是一个 dictionary, 有如下 key<table><tbody><tr><td><strong>Key</strong></td><td><strong>Value</strong></td></tr><tr><td>name</td><td>建议值，UTF-8 编码的字符串, 代表资源名称（单文件）。种子格式为多文件时，该字段表示目录名称。</td></tr><tr><td>piece length</td><td>文件以字节为单位的每个分片的长度，除了最后一个分片, 其它分片的长度都是相等的, 分片的长度通常取 2 的整数次幂, 按照最新版 BT 协议, 分片长度的值为 218 字节, 即 256 KB, 对应的 B 编码为 i262144e</td></tr><tr><td>pieces</td><td>对应的值为字节序列, 字节序列的长度为 20 的整数倍, 若该将字节序列按 20 个字节为一组切分开, 则每组都是文件相对应 piece 的 SHA1 哈希值 , 哈希值是二进制形式记录的, 无法用文本编辑器查看</td></tr><tr><td>private</td><td>可选，整数值，当取值为 1 时，表示只能从 meta 文件中指定的 tracker 获取 peer 信息，不得从其它途径获取，并且只能向指定的 tracker 发布自己的信息，以形成一个封闭的共享圈子。该字段定义在 <a href="http://bittorrent.org/beps/bep_0027.html">BEP_0027（Private Torrents） </a>协议中。private tracker（PT）将统计每个 peer 上传数据量，对于那些上传数据不足的 peer，将限制获取数据。</td></tr><tr><td>length（单文件） / files（多文件）</td><td>这两个键不能同时存在, 有且仅有其中一个, 若 .torrent 文件描述的文件只有一个, 则 length 键存在, 其值为该文件以字节为单位的长度 若 .torrent 文件描述的文件有多个, 则 files 键存在, 其值为一个字典的列表, 即列表的每一个元素都是一个字典结构, 每个字典都描述了其中的一个文件, 每个字典结构中包含两个 key, 分别是 <strong>length</strong>, 代表该文件以字节为单位的长度, <strong>path</strong>, 是一个字符串列表，表示该文件的相对路径名，最后一个字符串为实际的文件名。</td><td></td></tr></tbody></table></td><td><a href="http://bittorrent.org/beps/bep_0003.html">BEP_0003</a></td></tr></tbody></table>

### 磁力链的格式（magnet URI format）

磁力链接是一种特殊链接，通过不同文件内容的 Hash 结果生成一个纯文本的 “数字指纹”，并用它来识别文件。例如 `magnet:?xt=urn:btih:a27463ca05b1e87ecf89f95a07c13f158b5f2437`  
磁力链非常类似于 WEB 服务中用于定位资源的 URL，其中使用资源的哈希码即可唯一标识资源，相比 BT 种子文件，非常简单。任何人都可以分发一个磁力链接来确保该链接指向的资源就是他想要的，而和得到该资源的方式无关。因为磁力链中并不需要包含 tracker 服务器信息，因此相比于 BT 种子，更难于封锁。磁力链在 [BEP_0009（Extension for Peers to Send Metadata Files）](http://bittorrent.org/beps/bep_0009.html)

*   协议中定义有 2 种格式：  
    `magnet:?xt=urn:btih:<info-hash>&dn=<name>&tr=<tracker-url>&x.pe=<peer-address>`  
    `magnet:?xt=urn:btmh:<tagged-info-hash>&dn=<name>&tr=<tracker-url>&x.pe=<peer-address>`
    
*   字段含义
    

<table><thead><tr><th><strong>字段</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>magnet</td><td>必须，协议名</td></tr><tr><td>xt</td><td>必须，exact topic 的缩写，包含文件哈希值的统一资源名称。bthi（BitTorrent Info Hash）表示哈希方法名，使用 40 个字符的 16 进制编码字符串，表示 160 位的哈希值。info-hash 是对. torrent 文件中的 info-hash 字段进行 hash 计算，在磁力链中的 metadata 也是指 info-hash 的部分。btmh 是 <a href="https://github.com/multiformats/multihash">multihash</a> ，使用较少。</td></tr><tr><td>dn</td><td>可选，display name 的缩写，表示向用户显示的文件名</td></tr><tr><td>tr</td><td>可选，tracker 服务器的地址</td></tr><tr><td>x.pe</td><td>可选，用于获取 metadata 的 peer 地址，格式为 hostname:port, ipv4-literal:port 或 [ipv6-literal]:port</td></tr></tbody></table>

在磁力链协议中，metadata 指. torrent 文件中的 info 字段的字典信息。如果参数中不包含 tracker 信息，则需要从 DHT（[BEP_0005](http://www.bittorrent.org/beps/bep_0005.html)）获取 peer。

资源定位机制
------

有了 BT 种子文件或磁力链接后，接下来我们就需要找到拥有资源的 peer，来完成数据的下载。BT 下载的定位机制相对简单，在种子文件中的 announce 字段，给出了 tracker 地址，客户端可以通过与 tracker 通信的方式获取拥有资源的 peer，完成下载。对于无 tracker 的磁力链接来说，需要通过 DHT，找到拥有 metadata（包含文件分配及描述信息）的 peer，获取到 metadata 后，再进行数据的下载。

### BT 的资源定位机制 - Tracker 协议

BT Tracker 是一个注册服务, 用来协调 BT 协议的文件分发, 通过 BT Tracker 可以知道当前有哪些终端在同时下载目标资源, 当终端执行下载任务时应首先请求 BT Tracker 服务获取当前正在下载同一资源的对等结点信息, 终端用户通过 Tracker GET 请求 BT Tracker 服务。(可以是 HTTP 或 [BEP_0015 UDP Tracker Protocol for BitTorrent](http://bittorrent.org/beps/bep_0015.html) 或其它, 具体使用的协议由 .torrent 文件的 announce 对应的 Value 值给出)  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-13.png)  
Tracker 协议定义在 [BEP_0003](http://bittorrent.org/beps/bep_0003.html)  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-14.png)

*   GET 请求字段

<table><thead><tr><th><strong>字段名</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>info_hash</td><td>20 字节, 将 .torrent 文件中的 info 键对应的值生成的 SHA1 哈希, 该哈希值可作为所要请求的资源的标识符</td></tr><tr><td>peer_id</td><td>终端生成的 20 个字符的唯一标识符, 每个进行 BT 下载的终端随机生成的 20 个字符的字符串作为其标识符 (终端应在每次开始一个新的下载任务时重新随机生成一个新的 peer_id)，只需保证在客户端唯一即可</td></tr><tr><td>ip</td><td>可选，该终端的 IP 地址（或域名）, 一般情况下该参数没有必要, 因为传输层 (Transport Layer, 如 TCP) 本身可以获取 IP 地址, 但比如 BT 下载器通过 Proxy 与 Tracker 交互时, 该在该字段中设置源端的真实 IP</td></tr><tr><td>port</td><td>终端正在监听的端口 (因为 BT 协议是 P2P 的, 所以每一个下载终端也都会暴露一个端口, 供其它结点下载), BT 下载器首先尝试监听 6881 端口, 若端口被占用被继续尝试监听 6882 端口, 若仍被占用则继续监听 6883, 6884 ... 直到 6889 端口, 若以上所有端口都被占用了, 则放弃尝试</td></tr><tr><td>uploaded</td><td>当前已经上传的文件的字节数 (十进制数字表示)</td></tr><tr><td>downloaded</td><td>当前已经下载的文件的字节数 (十进制数字表示)</td></tr><tr><td>left</td><td>当前仍需要下载的文件的字节数 (十进制数字表示)，因为存在下载分片校验失败的情况，该值不能从文件长度和已下载字节数推算出来</td></tr><tr><td>numwant</td><td>可选，希望 BT Tracker 返回的 peer 数目, 若不填, 默认返回 50 个 IP 和 Port</td></tr><tr><td>event</td><td>可选, 该参数的值可以是 started, completed, stopped, empty 其中的一个, 该参数的值为 empty 与该参数不存在是等价的。当开始下载时, 该参数的值设置为 started, 当下载完成时, 该参数的值设置为 completed, 当下载终止时, 该参数的值设置为 stopped</td></tr><tr><td>compact</td><td>可选，compact=1 时返回紧凑格式的 peers 列表（默认），在 <a href="http://bittorrent.org/beps/bep_0023.html">BEP_0023</a> 中定义</td></tr></tbody></table>

在向 BT Tracker 发送包含以上参数的 GET 请求后, 正常情况下, BT Tracker 会返回一个 bencoding 编码的字典, 如果字典中含有 failure reason 的 Key, 则说明此次请求失败, 同时该 Key 对应的值是一个可读性的字符串, 用于向终端用户展示此次请求失败的原因, 在请求失败的情况下, BT Tracker 返回的字典里有且仅有这个一个 Key, 请求成功时, BT Tracker 返回的字典中应含有以下 Key

<table><thead><tr><th><strong>Key</strong></th><th><strong>Value</strong></th></tr></thead><tbody><tr><td>warnging message</td><td>当发生非致命性错误时, Tracker 返回的可读的警告信息</td></tr><tr><td>interval</td><td>必须，终端在下一次请求 BT Tracker 前应等待的时间 (以秒为单位)</td></tr><tr><td>min interval</td><td>终端在下一次请求 BT Tracker 前的最短等待时间 (以秒为单位)</td></tr><tr><td>complete</td><td>当前已完成整个资源下载的 peer 的数量</td></tr><tr><td>incomplete</td><td>当前未完成整个资源下载的 peer 的数量</td></tr><tr><td>peers</td><td>必须，一个 peers 的字典列表, 列表中每个字典包含有如下 Key<table><tbody><tr><td><strong>Key</strong></td><td><strong>Value</strong></td></tr><tr><td>peer id</td><td>peer 结点的 Id，与 GET 请求中的 peer id 对应</td></tr><tr><td>ip</td><td>peer 结点的 IP 地址</td></tr><tr><td>port</td><td>peer 结点的端口</td><td></td></tr></tbody></table><p>为了缩减 tracker 返回的 peers 列表大小，<a href="http://bittorrent.org/beps/bep_0023.html">BEP_0023</a> 将 peers 表示为一个字符串，每个 peer 表示为 6 字节的字符串，其中 4 字节 ip 和 2 字节 port，去掉了 peer id 字段。默认情况下，tracker 返回该紧凑格式的 peer 列表，只有当请求中的 compact=0 时，返回原始的 peer 列表格式</p></td></tr></tbody></table>

### 效率更优的 UDP Tracker 协议

UDP tracker 协议（[BEP_0015 UDP Tracker Protocol for BitTorrent](http://bittorrent.org/beps/bep_0015.html)）是一种高性能、低消耗的 tracker 协议，这种协议的 URL 一般是这样：udp://tracker:port。 数据开销的优势：基于 HTTP 的 tracker 协议，通常一个请求和响应的数据包含 10 个包，1206 个字节。如采用 UDP 协议，则可以将数据传输量减少到 4 个包，618 个字节，数据传输量约减少了 50%。对于一个客户端而言，这些数据量的减少影响可能不大，但对于一个同时服务数百万客户端的 tracker 服务器来说，节省的带宽开销非常可观。另外一个优势是，基于 UDP 的协议，采用了二进制传输方式，省去了 tracker 解析数据的工作，也使得 tracker 的代码变得更为简单。 相比于 TCP 协议，UDP 属于无连接协议，无需维护连接状态。一方面减少了服务器维护连接的开销，提升了系统性能，另一方面也带来了不可靠的问题：

*   **UDP 包的伪造问题**：由于无需进行 TCP 的握手连接过程，服务端收到数据包可以对来源 IP 进行伪造。为此，需要在 UDP 协议的基础上，增加建立连接的机制，服务端通过给客户端分配 connection_id 的机制，来避免伪造数据包来源的问题（伪造方无法接收到服务端的 connection_id 信息）
    
*   **超时重传问题**：由于 UDP 协议是无连接的不可靠协议，需要由应用方来实现数据的可靠传输。如果客户端经过秒后，还没有接收到来自服务端的响应数据，则重新发送数据请求，n 取值从 0 到 8。
    
*   UDP Tracker 协议有如下 4 种类型的请求
    

<table><thead><tr><th><strong>action</strong></th><th><strong>请求类型</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>0</td><td>connect</td><td>客户端与服务器建立连接，从服务器端获取 connection_id</td></tr><tr><td>1</td><td>announce</td><td>向 tracker 请求 peer 列表</td></tr><tr><td>2</td><td>scrape</td><td>获取某个种子当前的统计信息</td></tr><tr><td>3</td><td>error</td><td>错误消息</td></tr></tbody></table>

![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-15.png)

对于 tracker 来说，是 2 个完全不同的 url。UDP 协议中扩展出 option 数据，来描述路径和参数信息。option 数据附加在 announce 请求的数据包后，包含有 3 个字段。一个请求中可以包含有多个 option。  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-20.png)

<table><thead><tr><th><strong>option type</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>0</td><td>EndOfOptions，固定长度 1 个字节，没有后续的 2 个字段。表示 option 的解析结束的位置。</td></tr><tr><td>1</td><td>NOP，固定长度 1 个字节，没有后续的字段。用作占位符</td></tr><tr><td>2</td><td>URLData，包含请求的 PATH 和 QUERY 参数，后续的 1 个字节表示字符串的长度，例如：URL（1）中的参数字符串可以表示如下 <code>0x2, 0xC, '/', 'd', 'i', 'r', '?', 'a', '=', 'b', '&amp;', 'c', '=', 'd'</code> 如果参数长度大于 255，可以使用多个 URLData 的 option 数据进行拼接。</td></tr></tbody></table>

### 磁力链的资源定位机制 - DHT

磁力链协议属于无 tracker 协议，定义在 [BEP_0005 DHT Protocol](http://bittorrent.org/beps/bep_0005.html)。无 tracker 并不是没有 tracker，而是没有中心式 tracker。磁力链将 tracker 分散存储到 DHT（Distributed Hash Table，分布式哈希表），解决了集中式的 tracker 一旦出现宕机或封禁，将使整个网络失效的问题。每个节点负责一个小范围的路由，并负责存储一小部分数据，从而实现整个 DHT 网络的寻址和存储，相当于所有人一起构成了一个庞大的分布式存储数据库。 一个 BT 的客户端拥有 2 个角色：

*   作为 BT 下载的节点，实现 bittorrent 协议来进行上传和下载资源，我们称为 peer。
*   作为 DHT 网络中的一员，实现 DHT 协议，保存一部分其他 Peer 的地址信息，作为 tracker，我们称为 node。

当需要进行下载的时候，先根据自己本地存的路由表找其他节点，其他节点再去找他们保存的其他节点，直到找到拥有文件的人。一传十十传百、千、万，最后通过 N 个人的中转，找到应该连上的人。

*   **Kademila 网络**

DHT 是基于 Kademila 网络（简称 Kad），在 UDP 上实现的协议。在 Kademlia 网络中，所有信息均以哈希表条目形式加以存储，这些条目被分散地存储在各个节点上，从而以全网方式构成一张巨大的分布式哈希表。Kademlia 中规定所有的节点都具有一个 160bit 的 node ID，为了方便资源的查找，该节点 ID 与种子文件中的 info_hash 相同。节点必须维护一个路由表，路由表中含有一部分其它节点的联系信息。其它节点距离自己越近时，路由表信息越详细。因此每个节点都知道 DHT 中离自己很近的节点的联系信息，而离自己非常远的 ID 的联系信息却知道的很少。

数据的冗余存储：由于在实际的 Kad 网络当中，并不能保证在任一时刻目标节点 N 均一定存在或者在线，因此 Kad 网络规定：任一条目，依据其 key 的具体取值，该条目将被复制并存放在节点 ID 距离 key 值最近 (即当前距离目标节点 N 最近) 的 k 个节点当中；之所以要将重复保存 k 份，这完全是考虑到整个 Kad 系统稳定性而引入的冗余。k 的取值准则为：“在当前规模的 Kad 网络中任意选择至少 k 个节点，令它们在任意时刻同时不在线的几率几乎为 0”；k 的典型取值为 20，即，为保证在任何时刻我们均能找到至少一份某条目的拷贝，我们必须事先在 Kad 网络中将该条目复制至少 20 份。 由上述可知，对于某一条目，在 Kad 网络中 ID 越靠近 key 的[节点](https://baike.baidu.com/item/%E8%8A%82%E7%82%B9)区域，该条目保存的份数就越多，存储得也越集中；事实上，为了实现较短的查询响应延迟，在条目查询的过程中，任一条目可被 cache 到任意节点之上；同时为了防止过度 cache、保证信息足够新鲜，必须考虑条目在节点上存储的时效性：越接近目标结点 N，该条目保存的时间将越长，反之，其超时时间就越短；保存在目标节点之上的条目最多能够被保留 24 小时，如果在此期间该条目被其发布源重新发布的话，其保存时间还可以进一步延长。 距离的度量：在 Kademlia 网络中，距离是通过异或 (XOR) 计算的，结果为无符号整数。distance(A, B) = |A xor B|，值越小表示越近。

Peer 信息的查找流程如下：  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-21.png)

*   **数据通信协议 - KRPC 协议**

KRPC 协议是由 bencode 编码组成的一个简单的 RPC 结构，他使用 UDP 报文发送。一个独立的请求包被发出去然后一个独立的包被回复。这个协议没有重发。它包含 3 种消息类型：请求（query），回复（response）和错误（error）。一条 KRPC 消息由一个独立的字典组成，其中有 2 个关键字是所有的消息都包含的，其余的附加关键字取决于消息类型。  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-22.png)

<table><thead><tr><th><strong>key</strong></th><th><strong>value</strong></th></tr></thead><tbody><tr><td>t</td><td>transaction ID，编码为 2 字节的二进制字符串。transaction ID 由请求节点产生，并且回复中要包含回显该字段。</td></tr><tr><td>y</td><td>消息的类型，1 个字节。y 对应的值有三种情况：q 表示请求，r 表示回复，e 表示错误</td></tr></tbody></table>

**Queries** 请求消息（y=q）它包含 2 个附加的关键字 q 和 a

<table><thead><tr><th><strong>key</strong></th><th><strong>value</strong></th></tr></thead><tbody><tr><td>q</td><td>字符串类型，包含了请求的方法名字</td></tr><tr><td>a</td><td>字典类型，包含了请求所附加的参数</td></tr></tbody></table>

其中 q 有 4 种方法：ping，find_node，get_peers 和 announce_peer。

<table><thead><tr><th><strong>请求类型 q</strong></th><th><strong>定义</strong></th><th><strong>示例</strong></th></tr></thead><tbody><tr><td>ping</td><td>Ping 请求包含一个参数 id，它是一个 20 字节的字符串包含了发送者网络字节序的节点 ID。对应的 ping 回复也包含一个参数 id，包含了回复者的节点 ID。</td><td><code>ping Query = {t:aa, y:q, q:ping, a:{id:abcdefghij0123456789}} bencoded = d1:ad2:id20:abcdefghij0123456789e1:q4:ping1:t2:aa1:y1:qe Response = {t:aa, y:r, r: {id:mnopqrstuvwxyz123456}} bencoded = d1:rd2:id20:mnopqrstuvwxyz123456e1:t2:aa1:y1:re</code></td></tr><tr><td>find_node</td><td>被用来查找给定 ID 的节点的联系信息。包含 2 个参数，第一个参数是 id，包含了请求节点的 ID。第二个参数是 target，包含了请求者正在查找的节点的 ID。当一个节点接收到了 find_node 的请求，他应该给出对应的回复，回复中包含 2 个关键字 id 和 nodes，nodes 是字符串类型，包含了被请求节点的路由表中最接近目标节点的 K(8) 个最接近的节点的联系信息。</td><td><code>find_node Query = {t:aa, y:q, q:find_node, a: {id:abcdefghij0123456789, target:mnopqrstuvwxyz123456}}</code> <code>bencoded = d1:ad2:id20:abcdefghij01234567896:target20:mnopqrstuvwxyz123456e1:q9:find_node1:t2:aa1:y1:qe</code> <code>Response = {t:aa, y:r, r: {id:0123456789abcdefghij, nodes: def456...}} bencoded = d1:rd2:id20:0123456789abcdefghij5:nodes9:def456...e1:t2:aa1:y1:re</code></td></tr><tr><td>get_peers</td><td>请求包含 2 个参数。 第一个参数是 id，包含了请求节点的 ID。 第二个参数是 info_hash，它代表 torrent 文件的 infohash。如果被请求的节点有对应 info_hash 的 peers，他将返回一个关键字 values，这是一个列表类型的字符串。每一个字符串包含了 CompactIP-address/portinfo 格式的 peers 信息。如果被请求的节点没有这个 infohash 的 peers，那么他将返回关键字 nodes，这个关键字包含了被请求节点的路由表中离 info_hash 最近的 K 个节点，使用 Compactnodeinfo 格式回复。在这两种情况下，关键字 token 都将被返回。token 关键字在今后的 annouce_peer 请求中必须要携带。token 是一个短的二进制字符串。</td><td><code>get_peers Query = {t:aa, y:q, q:get_peers, a: {id:abcdefghij0123456789, info_hash:mnopqrstuvwxyz123456}}</code> <code>bencoded = d1:ad2:id20:abcdefghij01234567899:info_hash20:mnopqrstuvwxyz123456e1:q9:get_peers1:t2:aa1:y1:qe</code> <code>Response with peers = {t:aa, y:r, r: {id:abcdefghij0123456789, token:aoeusnth, values: [axje.u, idhtnm\]}}</code> <code>bencoded = d1:rd2:id20:abcdefghij01234567895:token8:aoeusnth6:valuesl6:axje.u6:idhtnmee1:t2:aa1:y1:re</code> <code>Response with closest nodes = {t:aa, y:r, r: {id:abcdefghij0123456789, token:aoeusnth, nodes: def456...}}</code> <code>bencoded = d1:rd2:id20:abcdefghij01234567895:nodes9:def456...5:token8:aoeusnthe1:t2:aa1:y1:re</code></td></tr><tr><td>announce_peer</td><td>这个请求用来表明发出 announce_peer 请求的节点，正在某个端口下载 torrent 文件。announce_peer 包含 4 个参数。 第一个参数是 id，包含了请求节点的 ID； 第二个参数是 info_hash，包含了 torrent 文件的 infohash； 第三个参数是 port 包含了整型的端口号，表明 peer 在哪个端口下载； 第四个参数数是 token，这是在之前的 get_peers 请求中收到的回复中包含的。收到 announce_peer 请求的节点必须检查这个 token 与之前我们回复给这个节点 get_peers 的 token 是否相同。如果相同，那么被请求的节点将记录发送 announce_peer 节点的 IP 和请求中包含的 port 端口号在 peer 联系信息中对应的 infohash 下。</td><td><code>announce_peers Query = {t:aa, y:q, q:announce_peer, a: {id:abcdefghij0123456789, implied_port: 1, info_hash:mnopqrstuvwxyz123456, port: 6881, token: aoeusnth}}</code> <code>bencoded = d1:ad2:id20:abcdefghij012345678912:implied_porti1e9:info_hash20:mnopqrstuvwxyz1234564: porti6881e5:token8:aoeusnthe1:q13:announce_peer1:t2:aa1:y1:qe</code> <code>Response = {t:aa, y:r, r: {id:mnopqrstuvwxyz123456}}</code> <code>bencoded = d1:rd2:id20:mnopqrstuvwxyz123456e1:t2:aa1:y1:re</code></td></tr></tbody></table>

![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-23.png)

**Responses** 回复消息（y=r） **Errors** 错误消息（y=e），包含一个附加的关键字 e。关键字 e 是列表类型。第一个元素是数字类型，表明了错误码。第二个元素是字符串类型，表明了错误信息。 错误码有 4 种：

<table><thead><tr><th><strong>错误码</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>201</td><td>一般错误</td></tr><tr><td>202</td><td>服务错误</td></tr><tr><td>203</td><td>协议错误，比如不规范的包，无效的参数，或者错误的 token</td></tr><tr><td>204</td><td>未知方法</td></tr></tbody></table>

示例： `generic error = {t:aa, y:e, e:[201, A Generic Error Ocurred\]} bencoded = d1:eli201e23:A Generic Error Ocurrede1:t2:aa1:y1:ee`

*   **Node 的冷启动问题**

当一个新节点首次试图加入 DHT 网络时，它必须做三件事：

1.  如果一个新节点要加入到 DHT 网络中，它必须要先认识一个人带你进去。这样的人我们把他叫做 bootstrap node，常见的 bootstrap node 有：[router.bittorrent.com](http://router.bittorrent.com/)，[router.utorrent.com](http://router.utorrent.com/)，[router.bitcomet.com](http://router.bitcomet.com/)，[dht.transmissionbt.com](http://dht.transmissionbt.com/)  
    等等，我们可以称之为节点 A ，并将其加入自己的路由表
2.  向该节点发起一次针对自己 ID 的节点查询请求，从而通过节点 A 获取一系列与自己距离邻近的其他节点的信息
3.  刷新所有的路由表，保证自己所获得的节点信息全部都是新鲜的。

### 局域网服务发现机制 - LSD（Local Service Discovery）

在局域网内 BT 客户端可以通过类似于 SSDP（简单服务发现协议），采用多播的方式发现局域网内拥有资源的 peer。这种方法简单高效，易于实现，在 [BEP_0014](http://bittorrent.org/beps/bep_0014.html) 中定义。LSD 使用的多播地址为

*   IPv4：239.192.152.143:6771 (org-local)
*   IPv6：[ff15::efc0:988f]:6771 (site-local)

IPv4 的多播地址，属于 D 类地址，范围 224.0.0.0 到 239.255.255.255，或表示为 224.0.0.0/4。IPv6 的多播地址为 ff00::/8 LSD 协议在多播地址上，基于 UDP 协议发送如下的 HTTP 报文

```
BT-SEARCH * HTTP/1.1\r\n
Host: <host>\r\n
Port: <port>\r\n
Infohash: <ihash>\r\n
cookie: <cookie (optional)>\r\n
\r\n
\r\n
```

*   报文中 header 的含义：

<table><thead><tr><th><strong>header</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>Host</td><td>发送的多播地址，即 239.192.152.143 或 ff15::efc0:988f</td></tr><tr><td>Port</td><td>BT 客户端监听的端口</td></tr><tr><td>Infohash</td><td>40 字节的 info 哈希值，允许一个客户端发多个 Infohash，但需要注意不能超过 1400 字节的 MTU 限制</td></tr><tr><td>cookie</td><td>可选。用于在多播消息中，过滤掉自己发出的消息。</td></tr></tbody></table>

在加入网络后，BT 客户端应当每 5 分钟发送一次 LSD 多播消息，为了避免多播风暴，每个客户端每分钟不能发送超过一个 LSD 报文。当一个客户端有超过 5 个种子时，可以选择轮流多播各个种子信息，或者一次多播多个种子信息。接收到 LSD 消息的客户端，从 UDP 报文中获取来源 IP，并进一步确认该 IP 是否为可连接的 peer。

数据下载
----

Peer Protocol （[BEP_0003](http://bittorrent.org/beps/bep_0003.html) ）是当前正在下载同一资源的对等结点 (peer) 之间进行数据传输使用的协议, BT 协议的 Peer Protocol 基于 TCP 或 [uTP（BEP_0029）](http://bittorrent.org/beps/bep_0029.html), 在使用 BT 协议下载时, 所有正在下载同一资源的结点都是对等的, 结点之间相互建立的连接也是对等的, 对等结点建立的连接的数据传输方向是双向的, 数据可以由任何一端发往另一端, 对等结点每接收到一个完整的 piece 之后, 接收方便计算该 piece 的 SHA1 哈希并与 .torrent 文件中的 info 对应的字典的 piece 对应的 Value 的相应分段做对比, 若哈希值相等, 则说明该分片传输成功, 此时这两个对等结点都拥有了资源文件关于该 piece 的数据。

一次完整的数据获取流程  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-24.png)

### Peer 握手协议

握手消息的格式为 `<pstrlen><pstr><reserved><info_hash><peer_id>`, 对等结点一方在传输层连接建立以后便发送握手信息, 另一方收到握手信息后也回复一个握手信息, 任何一方当收到非法的握手信息 (如 pstrlen 不等于 19, pstr 缺失或其值不是 BitTorrent protocol, info_hash 不相等, peer_id 与预期值不同等) 应立即断开连接。需要连接几个 peer，就需要完成几次握手。  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-25.png)

*   **扩展协议**（[BEP_0010](http://www.bittorrent.org/beps/bep_0010.html)）：如果握手消息中的 reserved 字段，右数第 20 bit（即 `reserved_byte[5] & 0x10 == 1`）设置为 1，表示该 peer 支持扩展协议消息。扩展协议中，同样有握手消息，和数据传输消息，在下节中详细说明
*   **扩展协议**（[BEP_0005](http://bittorrent.org/beps/bep_0005.html)）：Peer 如果支持 DHT 协议，可以将 BitTorrent 协议握手消息的 reserved 字段的第 8 字节的最后一位置为 1。后续的消息中，将发送 DHT 的监听端口给 peer

### Peer 数据传输协议

若校验没有问题, 则握手成功, 之后对等双方开始进行数据传输, 数据的通用格式为 `<length prefix><type id><payload>`, 其中 length prefix 顾名思义是消息的长度 (即 len(type id) + len(payload)), type id (十进制整数) 指示消息的类型, 特别地, length prefix 为 0 (此时消息没有 type id 也没有 payload) 代表 Keep Alive 消息, BT 协议标准规定 Keep Alive 消息每 2 min 发送一次, 接收端忽略该消息即可。  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-26.png)

对于其它类型的消息, 其对应的 type id 如下

<table><thead><tr><th><strong>消息类型</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>0</td><td>choke 消息, choke 消息没有 payload, 其 length prefix 等于 1</td></tr><tr><td>1</td><td>unchoke 消息, unchoke 消息没有 payload, 其 length prefix 等于 1</td></tr><tr><td>2</td><td>interested 消息, interested 消息没有 payload, 其 length prefix 等于 1</td></tr><tr><td>3</td><td>not interested 消息, not interested 消息没有 payload, 其 length prefix 等于 1</td></tr><tr><td>4</td><td>have 消息, have 消息的 paylod 只包含一个整数, 该整数对应的是该终端刚刚接收完成并校验通过 SHA1 哈希的 piece 索引 (终端在接收到并校验完一个 piece 后, 就向它所知道的所有 peer 都发送 have 消息以宣示它拥有了这个 piece, 其它终端接收到 have 消息之后便可以知道对方已经有该 piece 的文件数据了, 因而可以向其发送 request 消息来获取该 piece)</td></tr><tr><td>5</td><td>bitfield 消息, bitfield 消息只作为对等结点进行通信时所发送的第一个消息 (即在握手完成之后, 其它类型消息发送之前), 若文件没有分片, 则不发送该消息, 它的 Payload 是一个字节序列, 逻辑上是一个 bitmap 结构, 指示当前该终端已下载的文件分片, 其中第一个字节的 8 位分别表示文件的前 8 个分片, 第二个字节的 8 位分别表示文件的第 9 至 16 个分片, 以此类推, 已下载的分片对应的位的值为 1, 否则为 0, 由于文件分片数不一定是 8 的整数倍, 所以最后一个分片可能有冗余的比特位, 对于这些冗余的比特位都设置为 0</td></tr><tr><td>6</td><td>request 消息, request 消息用于一方向另一方请求文件数据, 它的 Payload 含有 3 个字段, 分别是 index, begin 和 length. 其中 index 指示文件分片的索引 (索引从 0 开始), begin 指示 index 对应的 piece 内的字节索引 (索引从 0 开始), length 指定请求的长度, length 一般都取 2 的整数次幂, 现在所有的 BitTorrent 实现中, length 的值都取 214214, 即 16 KB, BitTorrent 协议规定结点通过随机的顺序请求下载文件 piece</td></tr><tr><td>7</td><td>piece 消息, piece 消息是对 request 消息的响应, 即返回对应的文件片段, 它的 Payload 含有 3 个字段, 分别是 index, begin 和 piece, 其中 index 和 begin 字段与 request 消息中的 index 与 begin 含义相同, 而 piece 是所请求的文件片段</td></tr><tr><td>8</td><td>cancel 消息, cancel 消息与 request 消息的 Payload 字段完全相同, 但作用相反, 用于取消对应的下载请求</td></tr><tr><td>9</td><td>port 消息，payload 为 2 字节的端口号。当使用 DHT 作为 tracker 时，port 指定 DHT 结点监听的端口，peer 收到该消息后，将该结点加入自己的路由表中。（<a href="http://bittorrent.org/beps/bep_0005.html">BEP_0005</a>） 该消息是在 <a href="http://bittorrent.org/beps/bep_0005.html">BEP_0005</a> 中定义的 BitTorrent 扩展协议，可以支持 DHT 协议发现 peer，实现无 tracker 下载，或同时支持 tracker 和 DHT。Peer 如果支持 DHT 协议，可以将 BitTorrent 协议握手消息的保留位的第 8 字节的最后一位置为 1。相应地，一个无 tracker 的 torrent 文件字典不包含 announce 关键字，而使用 nodes 关键字来替代。这个关键字对应的内容应该设置为 torrent 创建者的路由表中 K 个最接近的节点。可供选择的，这个关键字也可以设置为一个已知的可用节点，比如这个 torrent 文件的创建者。例如： nodes = [[<host>, <port>], [<host>, <port>], ...] nodes = [[127.0.0.1, 6881], [your.router.node, 4804], [2001:db8:100:0:d5c8:db3f:995e:c0f7, 1941]]</port></host></port></host></td></tr><tr><td>20</td><td>扩展协议消息（<a href="http://www.bittorrent.org/beps/bep_0010.html">BEP_0010</a>），扩展协议消息的消息类型取值固定为 20，相比普通的消息格式，在 payload 前增加了一个字节，来表示扩展消息的 id。消息的格式如下： <img class="" src="https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-27.png"> extended message ID 为 0 的消息是扩展协议的握手消息，消息中的 payload 是一个 bencode 字典。字典中的所有键都是可选的，peer 需要忽略所有自己不支持的键。可选的键（还可以有更多）如下:<table><tbody><tr><td><strong>key</strong></td><td><strong>value</strong></td></tr><tr><td>m</td><td>字典类型，表示支持的扩展消息，键是扩展消息名称，值是扩展消息的 id（与 extended message ID 对应）。peer 通过这个键通告其他 peer 自己支持的消息类型，且该 peer 之后发送其他的扩展消息就会使用在这里对应的 extended_messag_id。例如：m: {ut_metadata: 2, ut_pex: 1}，其中键名通常表示为 “客户端缩写_消息名称” 的格式，实现时确保全局唯一，扩展消息 id 由客户端定义，只要保证在字典中不出现重复即可，不要求全局唯一，如果取值为 0，则明确表示不支持该协议。</td></tr><tr><td>p</td><td>整型，表示本地监听端口。帮助另一方了解自己的端口信息。连接的接收方是不需要发送这个扩展消息的，因为接收方的端口是已知的。</td></tr><tr><td>v</td><td>UTF-8 编码的字符串，表示客户端的名称与版本。例如 µTorrent 1.2</td></tr><tr><td>yourip</td><td>紧凑格式的 ip 地址，表示在 peer 视角中对方 peer 的 IP，一般客户端通过此获取自己的公网 IP。ipv4 是 4 Bytes，ipv6 是 16 Bytes</td></tr><tr><td>ipv6</td><td>紧凑格式的 ipv6 地址（16 Bytes），客户端可以选择该地址连接到该 peer</td></tr><tr><td>ipv4</td><td>紧凑格式的 ipv4 地址（4 Bytes），客户端可以选择该地址连接到该 peer</td></tr><tr><td>reqq</td><td>整型，表示自身的在不丢弃消息情况下可以保留的未处理的消息数量。在 libtorrent 中这个值是 250。</td><td></td></tr></tbody></table></td></tr></tbody></table>

这个消息需要在普通 handshake 成功后立即发送，在连接的生命周期内这个 Extend Handshake 多次发送都是有效的，但实际实现中有可能被忽略。 |

*   **扩展协议（**[**BEP_0009**](http://bittorrent.org/beps/bep_0009.html)**）：Peer 间传输 metadata 协议**

在磁力链协议中，磁力链本身不包含任何文件相关的信息，如文件如何进行分片，每个分片的校验哈希等，需要通过 DHT 找到拥有资源的 peer，再通过 peer 获取文件的 metadata 数据。Peer 间传输文件的 metadata 数据协议，属于扩展协议的一种。在扩展的握手消息中，m 字典内容指定 ut_metadata 这个键，同时保持其对应的消息 id 在 m 字典中唯一。metadata 数据按照 16KB 大小进行分片（最后一片的大小可以小于 16KB），分片索引以 0 开始。一个支持 ut_metadata 消息的 peer 将发送类似如下的消息：{\'m\': {\'ut_metadata\': 3}, \'metadata_size\': 31235}，除了 m 字典外，还需额外发送 metadata_size 字段，用于表示 metadata 数据的大小（字节数）。

ut_metadata 消息的 payload 是一个 bencode 的字典类型

<table><thead><tr><th><strong>key</strong></th><th><strong>value</strong></th></tr></thead><tbody><tr><td>msg_type</td><td>消息类型，取值为 1. request：数据请求消息，piece 参数表示请求第几个分片。对应返回消息类型为 data 或 reject，一个没有完整 metadata 数据的 peer 因为无法校验其 infohash，必须返回 reject 消息。例如：{\'msg_type\': 0, \'piece\': 0} 2. data：返回 metadata 数据的消息，metadata 的分片数据附在消息字典的后面，大小计入消息的总长度中。该消息增加一个 total_size 字段，与 metadata_size 的含义相同。每次返回的数据分片大小为固定的 16KB，最后一个分片可以小于 16KB。例如：{\'msg_type\': 1, \'piece\': 0, \'total_size\': 3425} 3. reject：表示被请求的 peer 没有序号为 piece 的 metadata 片段。也有可能是一定时间内请求超过数量限制，为了防止洪泛攻击，直接表示拒绝。例如：{\'msg_type\': 2, \'piece\': 0}</td></tr><tr><td>piece</td><td>数据分片的索引，从 0 开始</td></tr><tr><td>total_size</td><td>与握手消息中的 metadata_size 的含义相同。在 data 类型的消息中使用</td></tr></tbody></table>

*   **扩展协议（**[**BEP_0011**](http://bittorrent.org/beps/bep_0011.html) **）：Peer Exchange (PEX)**

Peer Exchange(PEX) 协议用于在 peer 间交换 peer 列表。通过在 Extend Handshake 消息的 字典 m 中加入 ut_pex 这个键来表明支持。PEX 消息的 payload 是一个 bencode 字典，格式如下：

<table><thead><tr><th><strong>key</strong></th><th><strong>value</strong></th></tr></thead><tbody><tr><td>added</td><td>当前连接的 IPv4 peer 压缩格式列表，告知对方进行添加</td></tr><tr><td>added.f</td><td>当前连接的 IPv4 peer 标志位，每个 peer 一个字节</td></tr><tr><td>added6</td><td>当前连接的 IPv6 peer 压缩格式列表，告知对方进行添加</td></tr><tr><td>added6.f</td><td>当前连接的 IPv6 peer 压缩格式列表标志位</td></tr><tr><td>dropped</td><td>过去断开连接的 IPv4 peer 压缩格式列表，告知对方进行删除</td></tr><tr><td>dropped6</td><td>过去断开连接的 IPv6 peer 压缩格式列表，告知对方进行删除</td></tr></tbody></table>

1 字节的标志位定义如下：

<table><thead><tr><th><strong>标志位</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>0x01</td><td>加密连接</td></tr><tr><td>0x02</td><td>属于 seed 或者 partial seed，仅上传数据</td></tr><tr><td>0x04</td><td>支持 uTP 协议</td></tr><tr><td>0x08</td><td>支持 ut_holepunch 扩展协议</td></tr><tr><td>0x10</td><td>可达连接</td></tr></tbody></table>

PEX 消息每分钟发送不能超过一条，除了最初的 PEX 消息之外，每条消息中添加的 peer 数量或者 删除的 peer 数量均不能超过 50 条。通过 PEX 获得的 peer 应该视为不可信的。攻击者可能通多伪造 PEX 消息来攻击这个 swarm。攻击者也可能通过 PEX 消息诱导 BT 客户端对特定 IP 进行尝试连接而引发 DDoS 攻击。为了缓解这些问题，peer 应该避免从单个 PEX 源获取其所有连接作为候选连接。应忽略具有不同的端口的重复 IP，还可以根据 peer 的优先级来进行排序。

**Peer 连接的状态管理：**在进行 BT 下载时, 建立了 P2P 连接的对等结点两端各维护一个状态机, 状态机的状态除了要知道自己的状态外，还需要知道对方 peer 对自己的处置状态，可以用四个比特位来表征:

*   am_chocking：该值若为 1 代表当前终端将另一端阻塞 (通过向对端发送 choke 消息), 将另一端阻塞期间将不再接受对方的请求, 对方无法从当前终端下载数据, 反之, 则没有阻塞对方, 则允许对方下载数据
*   am_interested：该值若为 1 代表当前终端对另一端 感兴趣 (通过向对方发送 interested 消息), 所谓 感兴趣 即是对方拥有自己所没有的文件 piece, 因此当前终端希望从对方下载对应的数据
*   peer_chocking：该值为 1 代表对方将当前终端阻塞 (收到了对端发来的 choke 消息), 此时当前终端无法继续向对方请求数据
*   peer_interested： 该值为 1 代表对方对自己感兴趣 (收到了对端发来的 interested 消息), 即当前终端拥有另一端所没有的文件片段  
    ![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-28.png)

当前终端从另一端下载数据当且仅当当前终端对另一端感兴趣且另一端没有将当前终端阻塞, 反过来, 另一端从当前终端下载数据当且仅当另一端对当前终端感兴趣且当前终端没有将另一端阻塞, 初始化状态下, BT 客户端对所有其它的 peer 结点都是 choked 状态。choked 状态的存在用于控制每个连接 peer 的下载速度，维护 peer 网络中的公平性，对于那些贡献少索取多的 peer 进行相应的惩罚。

### 数据传输的加密 / 混淆

由于 BT 的流量对运营商的带宽消耗很大，在某些国家（或地区）互联网服务供应商限制 BitTorrent 流量或用户，他们认为 BitTorrent 流量占用过多网络资源（增加运营成本）、干扰网络正常运行，或认为或限制 “非法的” 文件共享。为此，可以采用混淆和加密的方法，使 BT 流量更难以被检测和控制。消息流加密（[Message Stream Encryption](http://wiki.vuze.com/w/Message_Stream_Encryption)，简称 MSE），被很多 BT 客户端支持，用来规避运营商的流量检测。

MSE 使用密钥交换结合 torrent 的 infohash 创建一个 RC4 加密密钥。密钥交换有助于最小化被动监听器的风险，而 infohash 有助于避免中间人攻击。选择 RC4 是为了速度更快。该规范允许用户选择仅加密报头或者完全加密整个连接。加密整个连接提供更强的混淆能力，但也消耗更多的 CPU 时间。为确保与不支持此规范的其他客户端的兼容性，用户还可选择是否仍允许未加密的传入或传出连接。支持的客户端通过节点交换（PEX）和分布式散列表（DHT）通告它们已启用 MSE。

MSE 的数据加密传输过程主要有如下 5 个步骤  
![](https://www.zhaoyingze.com/wp-content/uploads/2022/04/word-image-29.png)

**DH 秘钥交换协议：**MSE 协议中使用的 RC4 加密算法为对称加密算法，即加密和解密使用的是同一个秘钥，双方使用 Diffie-Hellman 密钥交换协议 / 算法 (Diffie-Hellman Key Exchange/Agreement Algorithm)，简称 DH 算法来交换秘钥。DH 算法交换秘钥的过程如下：

选择一个 768bit 的大素数 P，选择底数 G，例如：

```
G = 2
P = 0xFFFFFFFFFFFFFFFFC90FDAA22168C234C4C6628B80DC1CD129024E088A67CC74020BBEA63B139B22514A08798E3404DDEF9519B3CD3A431B302B0A6DF25F14374FE1356D6D51C245E485B576625E7EC6F44C42E9A63A36210000000000090563
```

交换数据双方，分别生成一个只有自己知道的随机整数 Xa 和 Xb，使用这两个随机数生成各自的公钥

```
Pubkey of A: Ya = (G^Xa) mod P
Pubkey of B: Yb = (G^Xb) mod P
```

利用发布的公钥信息，交换数据双方，可以各自生成相同的加密秘钥。

```
Sa = (Yb^Xa) mod P = (G^(Xb*Xa)) mod P
Sb = (Ya^Xb) mod P = (G^(Xa*Xb)) mod P
S = Sa = Sb
P, S, Ya and Yb are 768bits long
```

DH 密钥交换算法的安全性依赖于这样一个事实：虽然计算以一个素数为模的指数相对容易，但计算离散对数却很困难. 对于大的素数，计算出离散对数几乎是不可能的。第三方在已知公钥的情况下，很难推算出 Xa，Xb，进而也很难得到加密秘钥 S

DH 算法也存在着一些明显的缺陷。例如容易遭受中间人的攻击，第三方 C 在和 A 通信时扮演 B，和 B 通信时扮演 A，A 和 B 都与 C 协商了一个密钥，然后 C 就可以监听和传递或篡改通信数据，而 A、B 却无法感知。可以考虑在交换秘钥的环节中，增加 hash 签名的机制来对对方的身份进行验证，例如使用秘钥 S 和 info_hash 计算 SHA1 的 hash 值，来校验对方是否伪造身份，由于中间人 C 与 A、B 协商的秘钥是不同的，因此 A、B 接收到的签名信息无法通过校验。

**完整的 MSE 握手过程：**

```
A->B: Diffie Hellman Ya, PadA 
B->A: Diffie Hellman Yb, PadB 
A->B: HASH(\'req1\', S), HASH(\'req2\', SKEY) xor HASH(\'req3\', S), ENCRYPT(VC, crypto_provide, len(PadC), PadC, len(IA)), ENCRYPT(IA)
B->A: ENCRYPT(VC, crypto_select, len(padD), padD), ENCRYPT2(Payload Stream) 
A->B: ENCRYPT2(Payload Stream)
```

上述流程中的常量定义

<table><thead><tr><th><strong>常量名</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>PadA, PadB</td><td>长度为 0-512 字节的随机数据，用于填充。由于 PadA，PadB 对于接收方来说，长度未知，因此依赖于后续的数据来定位下个消息。B 使用 HASH(\'req1\', S) 来定位，A 使用 ENCRYPT(VC) 来定位下个消息的开始。</td></tr><tr><td>PadC, PadD</td><td>用于未来扩展握手协议，当前未长度 0-512 字节的填充数据，</td></tr><tr><td>SKEY</td><td>数据流标识符，在 BT 协议中使用 info_hash</td></tr><tr><td>VC</td><td>verification constant，用于校验数据发送方是否拥有私钥 S 及 SKEY 的 hash 值。在当前版本中为 8 字节字符串，取值为 0x00</td></tr><tr><td>crypto_provide</td><td>32-bit 的标志位，每一位表示数据发送方可以支持的加密算法。当前版本中 0x01 表示未加密文本， 0x02 表示 RC4 加密算法，其余字段保留扩展。</td></tr><tr><td>crypto_select</td><td>32-bit 的标志位，接收方从 crypto_provide 中选择一位置 1，表示自己选择的通信加密算法。</td></tr><tr><td>IA</td><td>initial payload data from A。在 BT 协议中，通常用来传输 peer 握手消息数据。</td></tr></tbody></table>

函数定义

<table><thead><tr><th><strong>函数名</strong></th><th><strong>定义</strong></th></tr></thead><tbody><tr><td>len(X)</td><td>x 的长度，用 2 字节的整数表示，最大值 65535 字节</td></tr><tr><td>ENCRYPT()</td><td>RC4 加密函数</td></tr><tr><td>ENCRYPT2()</td><td>双方协商后选定的数据加密算法</td></tr><tr><td>HASH()</td><td>SHA1 哈希函数，输出 20 字节的哈希值</td></tr></tbody></table>

### NAT 穿越问题

由于 IPv4 地址不足，大多数的家庭网络都使用网络地址转换（NAT）技术建立了一个网关，使用内网 IP 来上网。采用 NAT 上网的技术很好地解决了 IPv4 地址不足的问题，但对于 P2P 应用来说，却导致外网的用户无法连接到内网的客户端，导致数据传输链路不畅。

NAT 穿越技术允许网络应用程序使用 UPnP（Universal Plug and Play）协议与 NAT 路由器进行交互，通过配置端口映射的方式，来实现 NAT 外部端口的数据包转发到应用程序使用的内部端口上。所有这一切都是自动完成的，用户无需手动映射端口或者进行其它工作。

### 数据传输控制算法

BT 的数据传输控制算法，由保证网络数据可用性的分片算法，及客户端下载效率的阻塞算法 2 部分组成。

**分片选择算法（Piece selection algorithm）**

BT 的数据传输是以分片 Piece 为单位，在 Peers 间有多个分片数据可供选择下载时，Peers 又是动态变化的，可能存在随时下线。如何选择下载的分片，来确保网络中数据的可用性，是由分片选择机制来决定的。常见的分片选择机制有如下几种：

*   **Rarest first**：最少优先策略。为了确保网络数据的可用性，最直观的策略就是让最稀缺的数据得到尽快复制，避免丢失，导致整个网络下载失败。最少优先策略即是选择最稀缺的数据分片加以复制。
*   **Strict Policy**：严格优先级策略。Peers 间在传输分片数据时，又会将分片再进行划分为更小的子分片进行下载（如将 256KB 的分片，切分为 16KB 的单元下载）。当一个分片的子分片开始启动下载后，后续的下载严格执行该分片数据优先的原则，这样可以保证一个分片尽快地被完整下载。
*   **Random first piece**：随机选择第一个分片。“最少优先”的一个例外是在下载刚开始的时候。此时，下载者没有任何片断可供上传，所以，需要尽快的获取一个完整的分片。而最少的分片，通常只有某一个 peer 拥有，所以，它可能比多个 peers 都拥有的那些片断下载的要慢。因此，第一个分片是随机选择的，尽快完成第一分片的下载，并开始上传，才切换到 “最少优先” 的策略。
*   **Endgame mode**：终局模式。在下载过程中，可能会出现从一个速率很慢的 peer 那里请求一个分片，导致下载很慢。在下载即将完成时，这种情况可能会进一步放大，最后的数据分片出现下载的瓶颈。为了防止这种情况，在最后阶段，peer 向它的所有的 peers 们都发送某分片的子分片请求，一旦某些子分片到了，那么就会向其它 peer 发送 cancel 消息，取消对这些子分片的请求，以避免带宽的浪费。因为只在下载临近结束时采用这种方法，实际上并没有浪费多少带宽，而文件的结束部分也会下载的非常快，尽快地在网络中提供一份完整的数据。

**阻塞控制算法（Choking）**

由于 P2P 是对等协议，客户端之间没有相互控制的权力，所以在选择从谁下载，或者为谁提供数据时，存在相互博弈的现象，一般会采用针锋相对（tit-for-tat）的策略来进行数据传输。对于那些不贡献数据的 peer 予以惩罚，对于贡献多的 peer，也提供更多的上传资源。

*   **Choking：**BitTorrent 设计的初衷是希望结点之间公平地进行文件分发, 保证所有结点的对等性, 为了避免某些结点只下载而不上传, BitTorrent 协议采用特定的 choke 算法来维护平衡, 当阻塞（choke）一个 peer 时，可以从对方那里下载数据，但不再为对方上传数据。BT 终端每 10s 统计一次将当前下载速度最快并且对其感兴趣的 4 个 peer 结点, 向它们发送 unchoke 消息, 解除对其的阻塞, choke 算法将直接决定 BT 下载的公平性与效率。
*   **Optimistic Unchoking**：开放检测。如果只是简单的为提供最好的下载速率的 peers 们提供上载，那么就没有办法来发现那些空闲的连接是否比当前正使用的连接更好。为了解决这个问题，在任何时候，每个 peer 都拥有一个称为 “optimistic unchoking” 的连接，这个连接总是保持连接状态，而不管它的下载速率是怎样。每隔 30 秒，重新计算一次哪个连接应该是“optimistic unchoking”。30 秒足以让上传能力达到最大，下载能力也相应的达到最大。
*   **Anti-snubbing**：反对歧视。某些情况下，一个 peer 可能被它所有的 peers 都阻塞了，这种情况下，它将会保持较低的下载速率直到通过 “optimistic unchoking” 找到更好 peers。为了减轻这种问题，如果一段时间过后，从某个 peer 那里一个片断也没有得到，那么这个 peer 认为自己被对方 “怠慢” 了，于是不再为对方提供上传，除非对方是 “optimistic unchoking”。这种情况下，会允许多于一个的 “optimistic unchoking”，以保证能够发现可以连接的 peer。
*   **Upload Only**：仅仅上传。一旦某个 peer 完成了下载，它不能再通过下载速率（因为下载速率已经为 0 了）来决定为哪些 peers 提供上载了。目前采用的解决办法是，优先选择那些从它这里得到更好的上载速率的 peers。这样的理由是可以尽可能的利用上载带宽。

参考资料
----

*   [BEP 协议官网](http://bittorrent.org/)
*   wiki 介绍 [BitTorrentSpecification](https://wiki.theory.org/BitTorrentSpecification)
*   ucla 课程资料 [cs217/05BitTorrent.pdf](http://web.cs.ucla.edu/classes/cs217/05BitTorrent.pdf)
*   [BitTorrent 协议及其工作原理解析](https://sunyunqiang.com/blog/bittorrent_protocol/)
*   知乎文章《[BitTorrent 简介](https://zhuanlan.zhihu.com/p/364041702)》
*   百度百科 [DHT](https://baike.baidu.com/item/DHT/1007999)
*   [Incentives Build Robustness in BitTorrent](http://bittorrent.org/bittorrentecon.pdf)
*   [BT 增强建议之 Peer](https://0ranga.com/2018/08/30/bt-peer/)
*   [Message Stream Encryption(MSE)](http://wiki.vuze.com/w/Message_Stream_Encryption)
*   百度百科 [BitTorrent 协议加密](https://baike.baidu.com/item/BitTorrent%E5%8D%8F%E8%AE%AE%E5%8A%A0%E5%AF%86/22911780)
*   百度百科 [Diffie-Hellman](https://baike.baidu.com/item/Diffie-Hellman/9827194) 秘钥交换协议
*   百度百科 [RC4](https://baike.baidu.com/item/RC4/3454548) 加密算法
*   Java 开源客户端 [https://github.com/atomashpolskiy/bt](https://github.com/atomashpolskiy/bt)
*   开源客户端 [transmission](https://github.com/transmission/transmission)
*   开源客户端 [qBittorrent](https://github.com/qbittorrent/qBittorrent)
*   开源客户端（包含开源的 webui） [aria2](https://aria2.github.io/)