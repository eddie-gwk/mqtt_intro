# MQTT 5.0


## 1.报文总览及快查表


### 1.1 数据表示
#### 1.1.1 二进制位 Bits

#### 1.1.2 双字节整数
16位无符号整数
#### 1.1.3 四字节整数
32位无符号整数
#### 1.1.4 UTF-8编码

每一个字符串都有一个两字节长度的前缀作为UTF-8编码的字节数。因此UTF-8编码的字节长度在0~65535。UTF-8编码必须按照Unicode规范，客户端和服务端接收到无效编码时会断开连接。

#### 1.1.5 变长字节整数

变长字节整数的编码方案对于整数的处理如下，小于128的值使用单字节编码，更大的值则用以下方案：低7位用于编码数据，最高位用于指示是否有更多数据。理论上对于4个字节的变长字节整数最大可表示的值为268,435,455 。


### 1.2 控制报文快查表

#### 1.2.1 固定报头

固定报头的第1个字节用来表示报文类型，第2个字节开始表示报文的剩余长度，包括可变报头和负载的数据。剩余长度是一个变长字节整数，意味着他可能的长度在1~4字节。

> <em>这里的命名其实存在歧义,固定报头(Fixed Header)并没有明确说明是固定什么，通常大家会认为固定报头可能指长度固定。但其实这里的固定意指报文的结构是固定的，由报文类型和剩余长度两个字段组成。</em>

##### 1.2.1.1 报文类型
<em>比特位</em> | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
--- | --- | --- | --- | --- | --- | --- | --- | --- |
byte1 | > | > | > | MQTT控制报文的类型 | 用于指定控制报文类型的标志位|
byte2 |  剩余长度... |

取第1个字节 7 ~ 4位作为无符号整数 <i>(2^4^=16)</i> 作为类型位，有如下表格:

| 名字 | 值 | 报文流动方向 | 描述 |
| ---  | --- | --- | --- |
| Reserved | 0 | 禁止 | 保留 |
| CONNECT | 1 | 客户端到服务端 | 客户端请求连接服务端 |
| CONNACK | 2 | 服务端到客户端 | 连接报文确认 |
| PUBLISH | 3 | 两个方向都允许 | 发布消息 |
| PUBACK | 4 | 两个方向都允许 | QoS 1消息发布收到确认 |
| PUBREC | 5 | 两个方向都允许 | 发布收到（保证交付第一步）|
| PUBREL | 6 | 两个方向都允许 | 发布释放（保证交付第二步）|
| PUBCOMP | 7 | 两个方向都允许 | QoS 2消息发布完成（保证交互第三步）|
| SUBSCRIBE | 8 | 客户端到服务端 | 客户端订阅请求 |
| SUBACK | 9 | 服务端到客户端 | 订阅请求报文确认 |
| UNSUBSCRIBE | 10 | 客户端到服务端 | 客户端取消订阅请求 |
| UNSUBACK | 11 | 服务端到客户端 | 取消订阅报文确认 |
| PINGREQ | 12 | 客户端到服务端 | 心跳请求 |
| PINGRESP | 13 | 服务端到客户端 | 心跳响应 |
| DISCONNECT | 14 | 两个方向都允许 | 断开连接通知 |
| AUTH | 15 | 两个方向都允许 | 认证信息交换 |

类型字段表明报文的请求类型，不同的类型将影响报文数据的内容。

<b>从以上表格可以总结出几点:</b>

    1. 连接请求只能由客户端发起
    2. 客户端和服务端都允许发布消息
    3. 只允许客户端向服务端发起订阅请求
    4. 客户端要向服务端维护心跳请求
    5. 断开和认证可以由任意一方发起

#### 1.2.2 可变报头

##### 1.2.2.1 报文标识符

在16种报文类型中只有以下几种需要报文标识符

+ PUBLISH （如果QoS>0）
+ PUBACK
+ PUBREC 
+ PUBREL 
+ PUBCOMP 
+ SUBSCRIBE 
+ SUBACK 
+ UNSUBSCRIBE 
+ UNSUBACK 


客户端每次发送一个新的SUBSCRIBE，UNSUBSCRIBE或者PUBLISH（当QoS>0时）MQTT控制报文时都必须分配一个当前未使用的非零报文标识符。在一次会话中，该报文标识符用来在每次报文中标识一组通话，在该组通话结束前标识符不能被其他会话使用。
也就是说在客户端和服务端建立通话后，PUBLISH，SUBSCRIBE和UNSUBSCRIBE的报文标识符不能相同。PUBACK，PUBREC，PUBREL，SUBACK，UNSUBACK必须包含相同的功能起始报文标识符。
客户端和服务端彼此独立分配标识符。因此，客户端服务端组合使用相同的报文标识符可以实现并发的消息交换。

##### 1.2.2.2 属性 (5.0 拓展)

属性是可变报头中用以指定功能的一段字节，其通常由属性长度和属性内容组成。属性长度是一个变长字节整数，属性长度不包含其自身，如果没有任何属性，则长度为0。

属性内容则由一个属性标识符和一段数据组成，属性标识符用于指定属性的用途。对于任何包含了无效属性的报文都属于无效报文。属性标识符及其性质如下表所示:


| <b>Dec</b> | <b>Hex</b> | <b>属性名</b> | <b>数据类型</b> | <b>报文/遗嘱属性</b> |
| --- | --- | --- | --- | --- | 
|  1  |   0x01  |  载荷格式说明   | 字节    | PUBLISH, Will Properties    |
|  2  |   0x02  |  <div id="message_timeout">消息过期时间</div>   | 四字节整数    | PUBLISH, Will Properties    |
|  3  |   0x03  |  内容类型   | UTF-8编码字符串    |  PUBLISH, Will Properties   |
|  8  |   0x08  |  响应主题   | UTF-8编码字符串    |  PUBLISH, Will Properties   |
|  9  |   0x09  |  相关数据   | 二进制数据    |  PUBLISH, Will Properties   |
|  11  |  0x0B   | 定义标识符    | 变长字节整数    | PUBLISH, SUBSCRIBE    |
|  17  |  0x11   | <div id="Session_Expiry_Interval">会话过期间隔</div>    | 四字节整数    |  CONNECT, CONNACK, DISCONNECT   |
|  18  |  0x12   | 分配客户标识符    |  UTF-8编码字符串   |  CONNACK   |
|  19  |  0x13   | 服务端保活时间    |  UTF-8编码字符串   |  CONNACK   |
|  21  |  0x15   | <div id="auth_method">认证方法</div>    | UTF-8编码字符串    | CONNECT, CONNACK, AUTH    |
|  22  |  0x16   | <div id="auth_data">认证数据</div>   | 二进制数据    |  CONNECT, CONNACK, AUTH   |
|  23  |  0x17   | 请求问题信息    |  字节   |  	CONNECT   |
|  24  |  0x18   | <div id="will_delay_time">遗嘱延时间隔</div>    | 四字节整数    |  Will Properties   |
|  25  |  0x19   | 请求响应信息   | 字节    |   CONNECT  |
|  26  |  0x1A   | 请求信息    | UTF-8编码字符串    |  CONNACK   |
|  28  |  0x1C   | 服务端参考    | UTF-8编码字符串    |  CONNACK, DISCONNECT   |
|  31  |  0x1F   | <div id="reason_chars">原因字符串</div>    | UTF-8编码字符串    | CONNACK, PUBACK, PUBREC, PUBREL, PUBCOMP, SUBACK, UNSUBACK, DISCONNECT, AUTH    |
|  33  |  0x21   | <div id="Receive_Maximum">接收最大数量</div>    | 双字节整数    |  CONNECT, CONNACK   |
|  34  |  0x22   | <div id="max_alias_length">主题别名最大长度</div>    | 双字节整数    |  CONNECT, CONNACK   |
|  35  |  0x23   | <div id="topic_alias">主题别名</div>    | 双字节整数    |  PUBLISH   |
|  36  |  0x24   | 最大QoS    | 字节    |  CONNACK   |
|  37  |  0x25   | 保留属性可用性    | 字节    | CONNACK    |
|  38  |  0x26   | <div id="user_attribute">用户属性</div>    | 	UTF-8字符串对    | CONNECT, CONNACK, PUBLISH, Will Properties, PUBACK, PUBREC, PUBREL, PUBCOMP, SUBSCRIBE, SUBACK, UNSUBSCRIBE, UNSUBACK, DISCONNECT, AUTH    |
|  39  |  0x27   | 最大报文长度    | 四字节整数    |  CONNECT, CONNACK   |
|  40  |  0x28   | 通配符订阅可用性    | 字节    | CONNACK    |
|  41  |  0x29   | 订阅标识符可用性    |  字节   |  CONNACK   |
|  42  |  0x2A   | 共享订阅可用性    | 字节    | CONNACK    |

> 尽管属性标识符用变长字节整数来表示，但在此版本协议中，所有的标识符均由一个字节来表示。

##### 1.2.2.1 CONNECT报文连接标志 (Connect Flags)

|7|6|5|4|3|2|1|0|
| --- | --- | --- | --- | --- | --- | --- | --- |
| User Name Flag | Password Flag| Will Retain| > |Will QoS |Will Flag | Clean Start	|Reserved |
|用户名标志|密码标志|遗嘱保留| > | 遗嘱 QoS|遗嘱标志|新开始| 保留位 |
连接标志字节包含一些用于指定MQTT连接行为的参数。它还指出有效载荷中的字段是否存在。

##### 1.2.2.1.1  新开始 Clean Start

这个二进制位表名此次连接是一个新会话还是一个已存在的会话的延续。

如果收到Clean Start为1的连接报文，客户端和服务端必须丢弃任何已存在的会话，并开始一个新的报文。

如果索道为0的连接报文，并且存在一个关联客户的会话，服务端就会尝试恢复此会话。如果不存在相关客户标识符会话，就创建一个新的。

##### 1.2.2.1.2 遗嘱标志

如果遗嘱标志（Will Flag）被设置为1，表示遗嘱消息必须已存储在服务端与此客户标识符相关的会话中。遗嘱消息（Will Message）包含遗嘱属性，遗嘱主题和遗嘱载荷字段。遗嘱必须在网络连接被关闭、遗嘱延时间隔到期或者会话结束之后被发布，除非服务端收到包含原因码为[0x00（正常关闭）](#normal_disconncet)的DISCONNECT报文之后删除了遗嘱消息（Will Message），或者一个关于此客户标识符的新的网络连接在遗嘱迟发时间（Will Delay Interval）超时之前被创建。

##### 1.2.2.1.3 遗嘱 QoS Will QoS

3、4位一起指定了遗嘱消息的服务质量，如果遗嘱标志Will Flag为0，遗嘱服务质量也必须为0.如果遗嘱标志为1，遗嘱服务质量可以是0、1、2。

##### 1.2.2.1.4 遗嘱保留 Will Retain

此标志位指定遗嘱消息在发布时是否会保留。

0为非保留消息、1为保留消息。

##### 1.2.2.1.5 用户名标志 User Name Flag

如果用户名标志（User Name Flag）被设置为0，有效载荷中不能包含用户名字段。如果用户名标志被设置为0，有效载荷中必须包含用户名字段。

##### 1.2.2.1.6 密码标志 Password Flag

如果密码标志（Password Flag）被设置为0，有效载荷中不能包含密码字段。如果密码标志被设置为1，有效载荷中必须包含密码字段。

#### 1.2.3 有效载荷

有些MQTT报文需要在报文的最后部分包含一个有效载荷，对于publish来说有效载荷就等于应用消息。
在这里只罗列需要有效载荷的报文类型:

1. CONNECT
2. PUBLISH(可选)
3. SUBSCRIBE
4. SUBACK
5. UNSUBSCRIBE
6. UNSUBACK

#### 1.2.4 原因码

原因码是一个单字节无符号整数，用来指示一次操作结果。小于0x80的原因码指示某次操作成功，通常用0表示。大于0x80的用来表示操作失败。原因码如下所示：

| > | 原因码	| 名称 |	报文 |
| --- | --- | --- | --- |
| DEC | HEX | > | &nbsp; | 
| 0	    |0x00	| 成功	| CONNACK, PUBACK, PUBREC, PUBREL, PUBCOMP, UNSUBACK, AUTH |
|0	    |0x00|	|<div id="normal_disconncet">正常断开</div> | DISCONNECT|
|0	    |0x00	|授权的QoS 0|	SUBACK|
|1	    |0x01	|授权的QoS 1|	SUBACK|
|2	    |0x02	|授权的QoS 2|	SUBACK|
|4	    |0x04	|包含遗嘱的断开|	DISCONNECT|
|16	    |0x10	|无匹配订阅|	PUBACK, PUBREC|
|17	    |0x11	|订阅不存在|	UNSUBACK|
|24	    |0x18	|继续认证|	AUTH|
|25	    |0x19	|重新认证|	AUTH|
|128	|0x80	|未指明的错误|	CONNACK, PUBACK, PUBREC, SUBACK, UNSUBACK, DISCONNECT|
|129	|0x81	|无效报文|	CONNACK, DISCONNECT|
|130	|0x82	|<div id="protocol_error">协议错误</div>|	CONNACK, DISCONNECT|
|131	|0x83	|实现错误|	CONNACK, PUBACK, PUBREC, SUBACK, UNSUBACK, DISCONNECT|
|132	|0x84	|协议版本不支持|	CONNACK|
|133	|0x85	|客户标识符无效|	CONNACK|
|134	|0x86	|用户名密码错误|	CONNACK|
|135	|0x87	|未授权|	CONNACK, PUBACK, PUBREC, SUBACK, UNSUBACK, DISCONNECT|
|136	|0x88	|服务端不可用|	CONNACK|
|137	|0x89	|服务端正忙|	CONNACK, DISCONNECT|
|138	|0x8A	|禁止|	CONNACK|
|139	|0x8B	|服务端关闭中|	DISCONNECT|
|140	|0x8C	|无效的认证方法|	CONNACK, DISCONNECT|
|141	|0x8D	|保活超时|	DISCONNECT|
|142	|0x8E	|会话被接管|	DISCONNECT|
|143	|0x8F	|主题过滤器无效|	SUBACK, UNSUBACK, DISCONNECT|
|144	|0x90	|主题名无效|	CONNACK, PUBACK, PUBREC, DISCONNECT|
|145	|0x91	|报文标识符已被占用|	PUBACK, PUBREC, SUBACK, UNSUBACK|
|146	|0x92	|报文标识符无效|	PUBREL, PUBCOMP|
|147	|0x93	|接收超出最大数量|	DISCONNECT|
|148	|0x94	|主题别名无效|	DISCONNECT|
|149	|0x95	|报文过长|	CONNACK, DISCONNECT|
|150	|0x96	|消息太过频繁|	DISCONNECT|
|151	|0x97	|超出配额|	CONNACK, PUBACK, PUBREC, SUBACK, DISCONNECT|
|152	|0x98	|管理行为|	DISCONNECT|
|153	|0x99	|载荷格式无效|	CONNACK, PUBACK, PUBREC, DISCONNECT|
|154	|0x9A	|不支持保留|	CONNACK, DISCONNECT|
|155	|0x9B	|不支持的QoS等级|	CONNACK, DISCONNECT|
|156	|0x9C	|（临时）使用其他服务端|	CONNACK, DISCONNECT|
|157	|0x9D	|服务端已（永久）移动|	CONNACK, DISCONNECT|
|158	|0x9E	|不支持共享订阅	|SUBACK, DISCONNECT|
|159	|0x9F	|超出连接速率限制|	CONNACK, DISCONNECT|
|160	|0xA0	|最大连接时间	DISCONNECT|
|161	|0xA1	|不支持订阅标识符|	SUBACK, DISCONNECT|
|162	|0xA2	|不支持通配符订阅|	SUBACK, DISCONNECT|

#### 1.2.5 常见报文

##### 1.2.5.1 AUTH 认证交换

AUTH报文被从客户端发送给服务端，或从服务端发送给客户端，作为扩展认证交换的一部分，比如质询/响应认证。如果CONNECT报文不包含相同的认证方法，则客户端或服务端发送AUTH报文将造成协议错误（[Protocol Error](#protocol_error)）。

###### AUTH固定报头

|Bit|7|6|5|4|3|2|1|0|
|---|---|---|---|---|---|---|---|---|
|byte1|>|>|>|MQTT控制报文类型(1111)|>|>|>|保留位(0000)|
|byte2|>|>|>|>|>|>|>|剩余长度|

AUTH固定报头的3\2\1\0是保留位，必须设置为0。

###### AUTH可变报头

AUTH报文可变报头按顺序包含以下字段：认证原因码（Authentication Reason Code），属性（Properties）。

###### 认证原因码

可变报头第0字节是认证原因码（Authenticate Reason Code）。单字节无符号认证原因码字段的值如下所示。AUTH报文的发送端必须使用一种认证原因码。

|DEC|HEX|原因码名称|发送端|说明|
|---|---|---|---|---|
|0|0x00|成功|服务端|认证成功|
|24|0x18|继续认证|客户端或服务端|继续下一步认证|
|25|0x19|重新认证|客户端|开始重新认证|


###### 属性

###### 属性长度 Property Length

AUTH报文可变报头中的属性的长度被编码为变长字节整数。

###### 认证方法 Authentication Method

[21 (0x15)](#auth_method) ，认证方法（Authentication Method）标识符

###### 认证数据 Authentication Data

[22 (0x16)](#auth_data) ，认证数据（Authentication Data）标识符。

###### 原因字符串 Reason String

[31 (0x1F)](#reason_chars) ，原因字符串（Reason String）标识符。

###### 用户属性 User Property

[38 (0x26)](#user_attribute) ，用户属性（User Property）标识符。






## 2.MQTT v5 新特性

### 2.1 预览及链接

MQTT v5 添加了以下特性(部分):

<details style="cursor:pointer" open>
  <summary><b>用户属性</b></summary>
  <small>允许用户向MQTT消息添加自己的元数据，传输额外的自定义信息以扩充更多应用场景。
  <a href="#221-用户属性">点击查看</a> </small>
</details>

<details style="cursor:pointer" open>
  <summary><b>主题别名</b></summary>
  <small>通过将主题名缩写为小整数来减小MQTT报文的开销大小。客户端和服务端分别指定他们允许的主题别名的数量。 <a href="#222-主题别名">点击查看</a> </small>
</details>

<details style="cursor:pointer" open>
  <summary><b>会话过期</b></summary>
  <small>会话分为新开始(不使用现有会话，而是新建) 和 会话过期间隔标志(指会话断开之后保留的时间)。把新开始标志设置为1且会话标志设置为0，等同于v3中把清理会话设置为1。<a href="#223-会话过期">点击查看</a> </small>
</details>

<details style="cursor:pointer" open>
  <summary><b>流量控制</b></summary>
  <small>允许客户端和服务端分别指定未完成的可靠消息(Qos > 0)的数量。发送端可以暂停发送此类消息以保持消息数量低于配额。<a href="#224-流量控制">点击查看</a></small>
</details>

<details style="cursor:pointer" open>
  <summary><b>消息过期</b></summary>
  <small>允许消息在发布时设置一个过期间隔。<a href="#225-保留消息与消息过期">点击查看</a></small>
</details>

<details style="cursor:pointer" open>
  <summary><b>最大报文长度</b></summary>
  <small>允许客户端和服务端各自指定它们支持的最大报文长度。会话参与方发送过大的报文会导致错误。</small>
</details>

<details style="cursor:pointer" open>
  <summary><b>所有确认报文原因码</b></summary>
  <small>所有确认报文增加原因码和一个可选的原因字符。这是为问题定位设计的，并且不应由接收端解析。</small>
</details>

<details style="cursor:pointer" open>
  <summary><b>增强的认证</b></summary>
  <small>提供一种机制来启用包括互相认证在内的质询/响应风格的认证。<a href="#226-增强的认证">点击查看</a></small>
</details>

<details style="cursor:pointer" open>
  <summary><b>遗嘱延迟</b></summary>
  <small>指定遗嘱消息在中断后延迟发送的能力。设计此特性是为了在会话重新建立连接的情况下不发送遗嘱消息。此特性允许连接短暂中断而不通知其他客户端。<a href="#227-遗嘱延迟">点击查看</a></small>
</details>


<details style="cursor:pointer" open>
  <summary><b>共享订阅</b></summary>
  <small>添加对共享订阅的支持，以允许多个订阅消费者进行负载均衡。<a href="#228-共享订阅">点击查看</a></small>
</details>

<details style="cursor:pointer" open>
  <summary><b>载荷格式和内容类型</b></summary>
  <small>允许在消息发布时指定载荷格式(二进制、文本)和MIME样式内容类型。</small>
</details>



### 2.2 特性详解

接下来我们对几个重要的特性进行详解，配合报文我们可以清晰的了解这几个特性的性质、作用和工作原理。

#### 2.2.1 用户属性

<h5>什么是用户属性</h5>

用户属性是是一种自定义属性，包含在可变报头的 <a href="#user_attribute">属性</a> 部分中，它由用户自定义的UTF-8键值对组成，只要不超过最大报文长度，可以使用无线数量的用户属性来向MQTT消息添加元数据。
该功能类似于HTTP协议的Header，但是这些属性只对用户自定义的功能有意义。该字段在几乎所有报文中都可携带，相当于用户可以拓展任意类型报文的功能。

<h5>用户属性的使用</h5>

<h6>文件传输</h6>

MQTT 5 的用户属性，可扩展为使用其进行文件传输，而不是像之前的 MQTT 3 中将数据放到消息体的 Payload 中，用户属性使用键值对的方式。这也意味着文件可以保持为二进制，因为文件的元数据在用户属性中。代码示例可参见[[通过用户属性传输文件](#31-通过用户属性传输文件)]。

<h6>协议解析</h6>

假设我们有一些设备分别以不同的协议数据上传，例如json、modbus、bluetooth或其他自定义协议格式。我们可以在发送数据时带上用户属性:
```json
{
  "protocol_type":"Json",
  "data_length":10
}
```
然后在接收端(msg服务)根据该字段统一处理转换为json，让我们的后续的处理程序不需要在进行协议转换而专注于处理业务逻辑。

#### 2.2.2 主题别名

<h5>什么是主题别名</h5>

主题别名是属性报文中新增的一种允许替代主题名的新特性。主题别名允许使用一个双字节整数来代替较长的topic字符串在Publish报文中起作用。对于MQTTv3，当客户端在某次连接中需要发布大量相同的主题的消息，那么每条PUBLISH报文都需要携带topic字符串，同时服务端每次都需要解析UTF-8字符串，这些都是对资源的浪费。但如果使用题主别名，这将使有效降低资源的消耗。

<h5>主题别名的使用</h5>

主题别名的生命周期在一次会话当中，如果会话断开主题别名就需要重新设置。 客户端和服务端的主题别名需要各自维护，当然它们也不会互相影响，完全可以使用重复的主题别名。

要设置主题别名的最大值，需要在CONNECT报文和CONNACK报文中携带 [0x22主题别名最大长度](#max_alias_length)。要设置主题别名，需要在PUBLISH报文中携带[0x23主题别名](#topic_alias)，当第一次PUBLISH报文携带主题名和主题别名成功完成映射后，后续PUBLISH报文可以不带主题名而只发送主题别名(主题别名值范围 1~max_alias_length)。如果没有主题名且主题别名为0，服务器将断开和客户端的连接。

当在PUBLISH报文中接收到未映射的主题别名时，即接收端没有建立主题和别名映射时，对端会使用包含原因码的[0x82协议错误](#protocol_error)的DISCONNECT报文断开网络连接。

![MQTT Client 与 MQTT Broker 分别设置 topic_alias](https://assets.emqx.com/images/90b455343adc89bade35746b3bf71a88.png?imageMogr2/thumbnail/1520x)
*<center>MQTT Client 与 MQTT Broker 分别设置 topic_alias</center>*

代码示例请查看 [通过主题别名发布消息](#32-通过主题别名发布消息)。

#### 2.2.3 会话过期

会话过期由 [新开始 Clean Start](#12211-新开始-clean-start) 和 [会话过期间隔 Session Expiry Interval](#Session_Expiry_Interval) 两个字段同时控制。当连接关闭后如果存在会话过期间隔，服务端将保留此次会话信息知道过期，在发送连接报文时，如果新开始字段被置为1则会尝试恢复还没有过期的会话。

##### MQTTv5的Clean Start

>在CONNECT报文中，Clean Start 被设置为1则不论有没有存在的会话都将开启一个新的会话。设置为0则会寻找已存在会话，如果找不到则开启一个新会话。

##### MQTTv5的 Session Expiry Interval

> Session Expiry Interval是以秒为单位的四字节整数，如果未指定或设为0，会话将在网络连接关闭时结束。如果设置为0xFFFFFFFF，则会话永远不会过期。
如果连接关闭时Session Expiry Interval 大于0，则客户端和服务端必须存储会话状态，直到过期。


##### MQTTv3的clean Session

>如果 Clean Session 设置为 0，服务端必须使用与 Client ID 关联的会话来恢复与客户端的通信。如果不存在这样的会话，服务器必须创建一个新会话。客户端和服务器在断开连接后必须存储会话的状态。
如果 Clean Session 设置为 1，客户端和服务器必须丢弃任何先前的会话并创建一个新的会话。该会话的生命周期将和网络连接保持一致，其会话状态一定不能被之后的任何会话重用。

对比v3的clean session，v3的会话即使在其不需要会话功能时也会进行存储，设置clean session的功能也只是丢弃之前的会话创建一个新的，这个机制会导致服务器资源的浪费。

v5通过增加属性来使会话的使用更加灵活。在paho java的实现中，对应的属性分别为`MqttConnectionOptions`的`cleanStart` 和 `MqttProperties`的`sessionExpiryInterval`。具体的示例代码见 [会话过期](#33-会话过期)。

#### 2.2.4 流量控制

通常服务端的资源都是固定且有限的，而客户端的流量则可能是随时随地变化的。正常业务（用户集中访问、设备大量重启）、被恶意攻击、网络波动，都会导致流量出现激增，如果服务端没有对其进行任何限制，就会导致负载迅速上升，进而导致响应速度下降，影响其他业务，甚至导致系统瘫痪。

v3中没有进行流量控制，流量控制只能依赖于各种实现库。在v5中增加了[接收最大数量(Receive Maximum)](#Receive_Maximum)属性来进行最大配额设置，控制QOS > 0的消息所能发送的最大数量。

再CONNECT报文中设置好最大配额后，每次发送一个PUBLISH(QOS>0)配额就会相应减额，如果发送配额等于0，客户端和服务端不能发送任何QOS>0的PUBLISH报文，下列情况则会增加配额:
+ 每当收到一个PUBACK报文或PUBCOMP报文，不管PUBACK或PUBCOMP报文是否包含错误码。
+ 每次收到一个包含返回码大于等于0x80的PUBREC报文。

在paho java中，只要设置`MqttConnectionOptions`的`ReceiveMaximum`就可以实现最大接收值。

#### 2.2.5 保留消息与消息过期

服务端接收到 Retain(PUBLISH固定报头第一个字节第0位) 标志为1的PUBLISH报文时，会把该消息视为保留消息，除了转发给接收端，服务端也会保留一个副本。服务端最多只能保留一份消息，如果相同topic下的已存在保留消息，它将会被替换为最新的消息。

当客户端建立订阅时，如果服务端存在主题匹配的保留消息，则这些保留消息将被立即发送给该客户端。借助保留消息，新的订阅者能够立即获取最近的状态，而不需要等待无法预期的时间，这在很多场景下非常重要的。

保留消息虽然存储在服务端中，但它并不属于会话的一部分。也就是说，即便发布这个保留消息的会话终结，保留消息也不会被删除。删除保留消息只有两种方式：

1. 客户端往某个主题发送一个 Payload 为空的保留消息，服务端就会删除这个主题下的保留消息
2. 如果包含保留消息的 PUBLISH 报文设置了[消息过期间隔](#message_timeout)属性，那么保留消息在服务端存储超过过期时间后就会被删除。

##### EMQX 服务器的保留消息实现

EMQX MQTT Broker的保留消息功能由`emqx_retainer `插件实现，该插件默认开启，可以修改`emqx_retainer (etc/plugins/emqx_retainer.conf)`进行配置。提供的配置有存储消息的位置(内存、硬盘)，限制接收保留消息数量和 Payload 最大长度，以及调整保留消息的过期时间。

具体的配置方式请查看官方文档:
[EMQX官方文档-基本参数-保留消息](#https://www.emqx.io/docs/zh/v5.1/configuration/configuration-manual.html#mqtt-%E5%9F%BA%E6%9C%AC%E5%8F%82%E6%95%B0)

#### 2.2.6 增强的认证

在物联网的应用场景中，安全设计是非常重要的一个环节，敏感数据泄露或是边缘设备被非法控制等事故都是不可接受的，但是相比于其他应用场景，物联网项目还存在着以下局限：

+ 安全性和高性能的兼顾
+ 加密算法需要更多算力，而物联网设备性能有限
+ 物联网的网络条件不稳定

##### 简单的认证

简单认证是通过在CONNECT报文中传递 [用户名 User Name Flag](#12215-用户名标志-user-name-flag) 和 [密码 Password Flag](#12216-密码标志-password-flag) 来验证客户端的合法性。

简单验证的好处是对算力要求低，但是在没有信道加密的前提下，无论明文传输或哈希的方式都很容易被攻击。

##### 增强认证

增强认证包含质询/相应的风格，可以对服务端和客户端进行双向的认证，可以保证通信双方的身份真实性，提高安全性。

相较于简单认证，增强认证需要进行更多论通信，因此MQTTv5新增了[AUTH](#1251-auth-认证交换)报文。认证报文的可变报头中带有属性认证方法/认证数据用以对高级认证提供支持。认证的过程通常是客户端发起CONNECT连接报文，服务端接收后发起一次AUTH报文携带认证方法和数据确认客户端身份，客户端也用相同方式发送AUTH报文(认证方法应该一致，否则将断开连接)，最后服务端发送一个CONNACK报头给客户端，至此完成认证连接。

![认证过程](https://img-blog.csdnimg.cn/img_convert/400dd03d199579976b4728d9a6de3902.png)
*<center>增强认证报文交换过程</center>*

<strong>目前paho并不支持太多的增强认证，网上可以查到的实现大多是Mosquitto的，因为我对于安全认证这块的知识还很薄弱，因此无法给出代码示例。</strong>

<strong>ps:有大佬可以提供学习案例的可以在这里补充[[增强认证](#34-增强认证)]。</strong>

#### 2.2.7 遗嘱延迟

##### 遗嘱延迟是什么

在v3版本中已经存在遗嘱消息，及客户端在连接时设置遗嘱topic/qos/message后，当客户端断开及发布遗嘱消息，但是因为物联网设备的网络通信并不稳定，导致经常出现一些网络波动导致的"假断线"，如果频繁的上下线会导致遗嘱消息被频繁发送，这可能导致一些问题。

遗嘱延迟的作用在于通过设置[遗嘱延时间隔（Will Delay Interval）](#will_delay_time)来控制遗嘱消息的发送，在v3，遗嘱消息的发布条件，包括但不限于:

+ 服务端检测到了一个I/O错误或者网络故障
+ 客户端在保持连接（Keep Alive）的时间内未能通讯
+ 客户端在没有发送包含原因码0x00（正常关闭）的情况下关闭了网络连接
+ 服务端在没有收到包含原因码0x00（正常关闭）的情况下关闭了网络连接

在v5，出现上述情况下，服务端还会保证遗嘱消息必须在遗嘱延迟时间到达之后发布消息。也就是说从异常断开到发布遗嘱之间存在一定的延迟。一旦网络通信恢复，遗嘱消息的发送将被取消。

##### 如何使用遗嘱延迟

要使用遗嘱延迟特性需要设置消息过期间隔，该属性是一个四字节整数的以秒为单位的数值。该属性被设置在CONNECT报文有效载荷(payload)的Will Properties中。但前提是[连接标志](#1221-connect报文连接标志-connect-flags)中的遗嘱标志有效。具体的示例代码请查看 [遗嘱消息延迟发送](#35-遗嘱消息延迟发送)。

#### 2.2.8 共享订阅

##### 什么是共享订阅

共享订阅是MQTTv5引入的新特性，相当于订阅端的负载均衡功能。一般的消息发布流程都是发布端到服务端，服务端转发到接收端。如图：
![发布流程](https://assets.emqx.com/images/87f2594bb38d81feb0441a5ac54aa339.png?imageMogr2/thumbnail/1520x)
*<center>发布流程图示</center>*
这种结构下如果订阅节点发生故障，就会导致消息丢失(qos=0)或者消息堆积在服务端。如果只是单纯的增加订阅节点来解决，那么就会出现很多重复消息，可能需要订阅端自行判断，增加了业务难度。
在v5中，我们可以通过共享订阅解决上述问题，当使用共享订阅时，消息流向就会如下图:
![共享订阅](https://assets.emqx.com/images/7b172accff520fef7a48586b5aa0ba0b.png?imageMogr2/thumbnail/1520x)
*<center>共享订阅发布流程图示</center>*

##### 如何使用共享订阅

同普通订阅一样，共享订阅也包含一个主题过滤器和订阅选项，唯一区别是共享订阅的过滤格式必须是`$share/{ShareName}/{filter}`格式，其定义如下：

+ `$share` 前缀表明这将是一个共享订阅
+ `{ShareName}` 是一个不包含 "/", "+" 以及 "#" 的字符串。订阅会话通过使用相同的
  `{ShareName}` 表示共享同一个订阅，匹配该订阅的消息每次只会发布给其中一个会话
+ `{filter}` 即非共享订阅中的主题过滤器

虽然共享订阅使得订阅端可以负载均衡，但是MQTT协议并没有规定服务器应当使用什么负载均衡。作为参考，在EMQX中提供了四种策略：

+ random: 在所有共享订阅会话中随机选择一个发送消息
+ round_robin: 按照订阅顺序轮流选择
+ sticky: 使用 random 策略随机选择一个订阅会话，持续使用至该会话取消订阅或断开连接再重复这一流程
+ hash: 对发送者的 ClientID 进行 hash 操作，根据 hash 结果选择订阅会话

具体的参考文档见 [[基本参数-共享订阅-派发策略](https://www.emqx.io/docs/zh/v5.1/configuration/configuration-manual.html#mqtt-%E5%9F%BA%E6%9C%AC%E5%8F%82%E6%95%B0)] 。
示例代码见 [使用共享订阅实现负载均衡](#36-使用共享订阅实现负载均衡) 。

## 3. 代码示例

以下代码都是在本地运行的，不具有代表性，这里顺便给出本地运行环境
```
jdk 1.8
paho.mqtt 1.2.5
maven 3.8.1
```

### 3.1 通过用户属性传输文件

如果使用payload传输文件，那通常payload会充斥着大段的十六进制字符或其他非UTF-8字符数据，可以预想到payload的混乱和冗余。我们在此尝试用用户属性来传递文件。

首先来实现一个消息订阅者
```java
public class Receiver implements MqttCallback {

    //连接所需配置
    static String topic = "/topic/test/user/property";
    static String userName = "admin";
    static String passwd = "qwe123@";
    static int qos = 0;
    static String broker = "tcp://47.101.218.45:1883";
    static String clientId = "javaReceiveClient";

    public static void main(String[] args) throws Exception{
        MqttClient sampleClient = new MqttClient(broker, clientId, new MemoryPersistence());
        sampleClient.setCallback(new Receiver());

        MqttConnectionOptions connectionOptions = new MqttConnectionOptions();
        //cleanStart也属于MQTT5的新特性(详情见[会话过期])
        //true标识重新开启一个会话，false则会尝试恢复一个会话
        connectionOptions.setCleanStart(true);
        connectionOptions.setUserName(userName);
        connectionOptions.setPassword(passwd.getBytes());
        sampleClient.connect(connectionOptions);

        sampleClient.subscribe(topic, qos);

        System.out.println("waiting for message ...");

    }


    @Override
    public void messageArrived(String s, MqttMessage mqttMessage) throws Exception {
        System.out.println("receive message:");
        //取出properties中的userProperties
        //paho定义其为一个包含key/value属性对象的列表
        System.out.println(mqttMessage.getProperties().getUserProperties());
    }

    //省略部分代码
    ...


}
```

再来定义一个消息发布者

```java
public class Publisher implements MqttCallback {

    static String topic = "/topic/test/user/property";
    static String content = "file transport";
    static String userName = "admin";
    static String passwd = "qwe123@";
    static int qos = 0;
    static String broker = "tcp://47.101.218.45:1883";
    static String clientId = "javaSendClient";
    static String filePath = "D:\\test.txt";

    public static void main(String[] args) throws Exception{
        MqttClient sampleClient = new MqttClient(broker, clientId, new MemoryPersistence());
        
        //省略配置客户端代码
        ...

        //读取一个文件名及文件内容
        String fileName = FileUtil.readFileName(filePath);
        String fileContent = FileUtil.readFile(filePath);
        
        //创建一个用户属性
        List<UserProperty> userProperties = new ArrayList();
        userProperties.add(new UserProperty("fileName", fileName));
        userProperties.add(new UserProperty("fileContent", fileContent));

        MqttProperties properties = new MqttProperties();
        properties.setUserProperties(userProperties);
        MqttMessage message = new MqttMessage(content.getBytes());
        message.setQos(qos);
        //在发布消息前将属性设置到properties中
        //此处演示发布报文的用户属性定义。但它还支持在连接、订阅、验权等等报文中定义。
        message.setProperties(properties);
        sampleClient.publish(topic, message);
        sampleClient.disconnect();

    }

    ...
}
```


运行如上代码后，我们可以看到控制台打印结果如下:

![测试结果](https://s1.ax1x.com/2023/08/11/pPnYeSO.png)

我们可以看到在代码中我们是通过`mqttMessage.getProperties().getUserProperties()`获取到的文件内容。也就是说`mqttMessage.getPayload()`的负载内容可以继续保持简洁和清晰。关于协议解析的应用也可以用类似方式进行实现。用户属性的加入更多的是帮我们进行一些结构和逻辑上的优化。

### 3.2 通过主题别名发布消息

首先需要明确的是，对于主题别名的设置是针对接收端的，也就是说发送端1将携带主题别名信息的PUBLISH报文给接收端1时，在这次会话之后的所有通讯中都可以使用主题别名代替主题名。

我们在这里只查看发送端的代码:
```java
public class Publisher implements MqttCallback {
    
    ...

    public static void main(String[] args) throws Exception{
        MqttClient sampleClient = new MqttClient(broker, clientId, new MemoryPersistence());
        //省略连接选项代码
        ...

        //设置主题别名最大值，如果不设置默认未65535(双字节整数的最大值)
        connectionOptions.setTopicAliasMaximum(4096);
        sampleClient.connect(connectionOptions);

        MqttProperties properties = new MqttProperties();
        //设置主题别名
        properties.setTopicAlias(1);
        MqttMessage message = new MqttMessage("hello there".getBytes());
        message.setQos(qos);
        message.setProperties(properties);
        //先发送一次建立主题和别名的映射
        sampleClient.publish(topic, message);

        //模拟后续可能的通讯
        Scanner scanner = new Scanner(System.in);
        do{
            String input = scanner.next();
            if("send".equals(input)){
                sampleClient.publish(topic, message);
            }
        }while (scanner.hasNext());

        //代码中调用sampleClient.disconnect()后视为会话结束
        sampleClient.disconnect();

    }

    ...
}
```

上述代码先发布一次带主题名和主题别名的PUBLISH报文给接收端，以期建立映射。后续根据输入决定是否再次发送报文。我们通过`wireshark`捕获MQTT报文查看主题别名是否生效，我们这里通过EMQX作为broker，其地址为47.101.218.45，本机的ip是192.168.1.89，因为发布和订阅都开在本机ip，我使用的服务质量为0所以没有ACK报文等干扰，PUBLISH的报文流向是:
>192.168.1.89  -> 47.101.218.45
>47.101.218.45 -> 192.168.1.89

在连接成功后发送一次PUBLISH建立映射，后续再发送一次PUBLISH后如下图：
![报文信息](https://s1.ax1x.com/2023/08/11/pPnYuOH.png)

可以看到在broker发送给接收端的报文中没有主题信息，且报文中的 [0x23主题别名](#topic_alias) 后面跟着的两字节 00 01的值和我们的主题别名一致。

### 3.3 会话过期

会话过期这里我们通过改变接收端的会话过期时间来验证接收端能否收到离线期间消息。

接收端代码如下
```java {.line-numbers}
public class Receiver implements MqttCallback {

    //省略相关配置
    ...

    public static void main(String[] args) throws Exception{

        MqttClient sampleClient = new MqttClient(broker, clientId, new MemoryPersistence());
        sampleClient.setCallback(new Receiver());

        MqttConnectionOptions connectionOptions = new MqttConnectionOptions();
        //新开始设置为false，去寻找已保存的会话
        connectionOptions.setCleanStart(false);
        //设置会话过期时间为1分钟
        connectionOptions.setSessionExpiryInterval(60L);

        ...

        sampleClient.connect(connectionOptions);

        sampleClient.subscribe(topic, qos);

    }


    @Override
    public void messageArrived(String s, MqttMessage mqttMessage) throws Exception {
        System.out.println("receive message:");
        System.out.println(new String(mqttMessage.getPayload()));
    }

    ...

}
```

对于发送端，我们只需要修改服务质量>0即可(确保消息可以保存在服务端进行重发)
```java {.line-numbers, highlight=18}
public class Publisher implements MqttCallback {

    ...

    public static void main(String[] args) throws Exception{
        MqttClient sampleClient = new MqttClient(broker, clientId, new MemoryPersistence());
        sampleClient.setCallback(new Publisher());
        MqttConnectionOptions connectionOptions = new MqttConnectionOptions();
        connectionOptions.setCleanStart(true);
        connectionOptions.setUserName(userName);
        connectionOptions.setPassword(passwd.getBytes());
        connectionOptions.setTopicAliasMaximum(4096);
        sampleClient.connect(connectionOptions);

        MqttProperties properties = new MqttProperties();
        properties.setTopicAlias(1);
        MqttMessage message = new MqttMessage("message from publish".getBytes());
        message.setQos(1); //只要保证qos>0即可
        message.setProperties(properties);
        sampleClient.publish(topic, message);

        Scanner scanner = new Scanner(System.in);
        do{
            String input = scanner.next();
            if("send".equals(input)){
                sampleClient.publish(topic, message);
            }
        }while (scanner.hasNext());

        sampleClient.disconnect();
    }

    ...

}
```

之后我们就可以开始测试的流程，下面是我完成的两种测试流程:

1. 先设置接收端的会话过期为1分钟，发送端发布消息，确保接收端可以接到。关闭接收端，在1分钟内发送端继续发送消息，接收端重新上线，会话重连并接收到了离线消息。

2. 设置接收端的会话过期为5s，同样的，确保第一次可以收到消息，然后断开连接，发送端继续发送消息，5s后接收端重新上线，无法找到已存在会话且离线消息也无法接收。

上述流程验证了EMQX服务端关于会话过期的完美支持，对于协议的实现也比较完整。


### 3.4 增强认证

待补充

### 3.5 遗嘱消息延迟发送

遗嘱标志设为1后，遗嘱消息的延迟发送需要关系到两个参数的值:

1. 会话过期时间
2. 遗嘱延迟间隔

当客户端以某个非正常退出原因断连后，服务端会检查会话过期时间是否>0，如果不是遗嘱消息立刻发布。如果不是，在会话时间<延迟间隔的情况下，会话到期后遗嘱消息立刻发布，否在就在延迟间隔到达后发布。

><h5>3.1.3.2.2 遗嘱延时间隔 Property Length</h5>
>
>服务端将在遗嘱延时间隔（Will Delay Interval）到期或者会话（Session）结束时发布客户端的遗嘱消息（Will Message），<span style="color:#dd0000">取决于两者谁先发生</span>。如果某个会话在遗嘱延时间隔到期之前创建了新的网络连接，则服务端不能发送遗嘱消息。

*ps:在测试的时候没认真看协议，发现遗嘱消息一直无法延迟发送，后来重新检查才注意到这个定义...*

我们修改我们的发送端:
```java
public class Publisher implements MqttCallback {
    
    
    ...

    public static void main(String[] args) throws Exception{
        MqttClient sampleClient = new MqttClient(broker, clientId, new MemoryPersistence());
        sampleClient.setCallback(new Publisher());
        MqttConnectionOptions connectionOptions = new MqttConnectionOptions();
        connectionOptions.setCleanStart(true);
        connectionOptions.setUserName(userName);
        connectionOptions.setPassword(passwd.getBytes());
        //会话超时时间，不设置默认为0，遗嘱消息将立马发布。
        connectionOptions.setSessionExpiryInterval(20L);
        connectionOptions.setTopicAliasMaximum(4096);

        //设置遗嘱消息
        String willTopic = "/my/will/topic";
        MqttMessage willMessage = new MqttMessage("I was dead man".getBytes());
        willMessage.setQos(1);
        //不设置为保留消息
        willMessage.setRetained(false);

        //设置遗嘱延迟5s发送
        MqttProperties properties = new MqttProperties();
        properties.setWillDelayInterval(5L);

        connectionOptions.setWillMessageProperties(properties);
        connectionOptions.setWill(willTopic, willMessage);

        sampleClient.connect(connectionOptions);

        MqttMessage message = new MqttMessage("hello !".getBytes(StandardCharsets.UTF_8));
        message.setQos(0);
        message.setRetained(false);

        sampleClient.publish("/topic/test/error/occur", message);

        //保持连接中...

    }

    ...
}
```

我们还需要开启一个订阅遗嘱topic的接收端，完成后我们只要强制关闭发送端进程，等待5s后就可以接收到遗嘱消息，如果在5s内我们重新上线发送端，遗嘱消息将不会被发送。


### 3.6 使用共享订阅实现负载均衡

共享订阅基本不需要修改代码，只要在topic中加入$share并且配置EMQX就可以实现负载均衡。后续会补充对负载均衡的测试代码。


## 4. 参考文档

[MQTTv5协议中文版](http://mqtt.p2hp.com/mqtt-5-0)
[MQTT v 5.0 New Features Overview](http://www.steves-internet-guide.com/mqttv5/)
[OASIS MQTT Version 5.0](http://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html)
[MQTT5探索](https://www.emqx.com/zh/mqtt/mqtt5)
[MQTT v5 – New Opportunities For IoT Development](https://mobidev.biz/blog/mqtt-5-protocol-features-iot-development)
