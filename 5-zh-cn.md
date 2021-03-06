# 5. 流和多路复用

在一个HTTP/2的连接中, 流是用于客户端与服务器交换frame的独立双向序列. 流有几个重要的特点:

+ 一个HTTP/2连接可以包含多个并发的流, 各个端点从多个流中交换frame
+ 流可以被客户端或服务器单方面建立, 使用或共享
+ 流也可以被任意一方关闭
+ frames在一个流上的发送顺序很重要. 接收方将按照他们的接收顺序处理这些frame. 特别是`HEADERS`和`DATA` frame的顺序, 在协议的语义上显得尤为重要.
+ 流用一个整数(流标识符)标记. 端点初始化流的时候就为其分配了标识符.

## 5.1 流的状态

下图展示了流的生存周期:

```
                         +--------+
                 send PP |        | recv PP
                ,--------|  idle  |--------.
               /         |        |         \
              v          +--------+          v
       +----------+          |           +----------+
       |          |          | send H /  |          |
,------| reserved |          | recv H    | reserved |------.
|      | (local)  |          |           | (remote) |      |
|      +----------+          v           +----------+      |
|          |             +--------+             |          |
|          |     recv ES |        | send ES     |          |
|   send H |     ,-------|  open  |-------.     | recv H   |
|          |    /        |        |        \    |          |
|          v   v         +--------+         v   v          |
|      +----------+          |           +----------+      |
|      |   half   |          |           |   half   |      |
|      |  closed  |          | send R /  |  closed  |      |
|      | (remote) |          | recv R    | (local)  |      |
|      +----------+          |           +----------+      |
|           |                |                 |           |
|           | send ES /      |       recv ES / |           |
|           | send R /       v        send R / |           |
|           | recv R     +--------+   recv R   |           |
| send R /  `----------->|        |<-----------'  send R / |
| recv R                 | closed |               recv R   |
`----------------------->|        |<----------------------'
                         +--------+

   send:   发送这个frame的终端
   recv:   接受这个frame的终端

   H:  HEADERS帧 (隐含CONTINUATION帧)
   PP: PUSH_PROMISE帧 (隐含CONTINUATION帧)
   ES: END_STREAM标记
   R:  RST_STREAM帧
```

该图只展示了流的状态转换以及frame和标记如何对转换产生影响. 这方面, `CONTINUATION`frames不会导致状态的转换, 他们只是跟在`HEADERS`或`PUSH_PROMISE` frame后面的有效组成部分.

状态转换的用途, 对于设置了`END_STREAM`标记的frame来说, `END_STREAM`被当做一个分开的事件处理. 设置了`END_STREAM`标记的`HEADERS` frame会导致两次状态转换.

在传输过程中, 每个端点对流状态的主观认识可能不同. 这些终端不会协商流的创建, 都是由终端独立创建的. 端点的流状态不同会带来负面影响: 在发送了`RST_STREAM`之后流处于关闭状态, 而frame可能在流关闭之后才到达.

流有如下状态:

+ `idle`
	所有流最初状态都是`idle`.  
  下面描述了流从`idle`状态到其它状态的几种可能转换:
  
  - 发送或接收到一个`HEADERS`frame会使流状态变换`open`. 流标识符的选择参见5.1.1里的描述. 收到相同的`HEADERS`frame会导致流立即变为`half-close`状态.
  - (Sending a PUSH\_PROMISE frame on another stream reserves the idle stream that is identified for later use.)在另一个流上发送一个`PUSH_PROMISE`frame 被标识为以后使用. 预留流的状态对应转换到`reserved (local)`.
  - (Receiving a PUSH\_PROMISE frame on another stream reserves an idle stream that is identified for later use.)在另一个流上接收一个`PUSH_PROMISE`frame 被标识为以后使用. 预留流的状态对应转换到`reserved (remote)`.
  - 注意`PUSH_PROMISE`frame并不在idle流上发送, 只是promised流的ID字段引用了新的reserved流.

	在`idle`状态接收到任何非`HEADERS`或`PUSH_PROMISE`frame必须视为连接错误, 错误类型为`PROTOCOL_ERROR`
  
+ `reserved (local)`
	处于这种状态的流表示它已经发送了一个`PUSH_PROMISE`frame并成为promised流. `PUSH_PROMISE`frame通过关联一个由远程对等点初始化的流来转换idle流到reserved流.  
  处于这个状态的流, 只有下面的几种可能状态转换:
  
  - 端点发送一个`HEADERS`frame, 流进入`half-closed (remote)`状态.
  - 任何一个端点发送一个`RST_STREAM`frame, 流变成`closed`状态. 这将释放一个流保留的资源.

	端点不准发送除`HEADERS`, `RST_STREAM`或`PRIORITY`之外任何类型的frame.
  
  这一状态可能收到`PRIORITY`或`WINDOW_UPDATE`frame. 除了`RST_STREAM`, `PRIORITY`以及`WINDOW_UPDATE`frame之外, 收到其他类型的frame必须视为`PROTOCOL_EROR`类型的连接错误.

+ `reserved (remote)`
	如果一个流已被远程对等点保留, 状态就会变成`reserved(remote)`.  
  可能的转换如下:
  
  - 收到一个HEADERS frame导致状态变为half-close(local)
  - 任何端点发送一个`RST_STREAM`frame会导致状态变成`closed`, 并释放流保留的资源.
  
  端点可以发送一个PRIORITY frame以重新确定reserved流的优先级次序. 不允许发送除RST_STREAM, WINDOW_UPDATE或PRIORITY之外的frame.
  
  在一个流上拿到非HEADERS, RST_STREAM或PRIORITY的frame必须视为`PROTOCOL_EROR`类型的连接错误.

+ `open`
	任何一对等方可以使用open状态的流发送任意类型的frame. 这一状态下, 发送方会监视给出的流级别和流控范围.
  
  在任意一方发送设置了END_STREAM标记的frame后, 流状态会变为half-closed的其中一个状态: 如果一方发送了该frame, 其流变为half-closed(local); 如果一方收到该frame, 流变为half-closed(remote).
  
  在这个状态发送RST_STREAM frame可以使状态立即变成closed.

+ `half-closed (local)`
	处于这个状态的流不能发送除WINDOW\_UPDATE, PRIORITY以及RST\_STREAM之外的frame.
  
  收到一个标记了END_STREAM的frame或者发送一个RST_STREAM frame, 都会使状态变成closed.
  
  端点允许接收任意类型的frame. 便于后续接收用于流控的frame, 使用WINDOW_UPDATE frame提供流控credit很有必要. 接收方可以选择忽略WINDWO_UPDATE frame, (which might arrive for a short period after a frame bearing the END_STREAM flag is sent.)
  
  收到的PRIORITY frame用于重定流的优先级次序(依据流的标记而定)

+ `half-closed (remote)`
	处于这个状态的流不能发送frame了. 并且端点也无需继续维护接收方流控窗口.
  
  如果端点收到额外的frame,并且不是WINDOW_UPDATE, PRIORITY或RST_STREAM,那么必须响应一个类型为STREAM_CLOSED的流错误.
  
  这一状态下的流可以发送任意类型的frame. 端点仍会继续监视已知的流级别和流控范围.
  
  发送一个END_STERAM标记的frame或任意一个对等方发送了RST_STREAM frame都会使流变为closed.

+ `closed`
	closed标识终止状态.
  
  在一个closed的流上不允许发送PRIORITY之外的其他frame. 端点在收到RST_STREAM frame后又收到非PRIORITY的frame的话, 一定被视为流错误对待(类型STREAM_CLOSED). 
  
  同样, 收到END_STREAM标记后又收到**非如下描述**的frame, 会触发一个连接错误(类型STREAM_CLOSED):
  
  发送了包含END_STREAM标记的DATA或HEADERS frame后的一小段时间内, 允许WINDOW_UPDATE或RST_STREAM frame被接收. 直到远程对等端收到并处理了RST_STERAM或包含END_STREAM标记的frame, 才可以发送这些类型的frame. 
  假如在发送了END_STREAM后已明显过了超时时间, 这时却再次收到frame, 尽管终端可以选择把这个frame当成PROTOCOL_ERROR类型的连接错误来处理, 但无论如何最终**必须**忽略这种情况下收到的WINDOW_UPDATE或RST_STREAM frame.
  
  PRIORITY帧可从closed流上发到优先级更高的流(取决于closed流). 终端应该处理PRIORITY帧, 尽管他们可能因为流已经从依赖树中移除而被忽略.
  
  如果是发送RST_STREAM帧的原因让状态转换到了closed, 收到RST_STREAM的对等端这时可能已经发送了RST_STREAM或者入队等待发送中, 但是已经在流上传输的帧是不可以被撤销的. 这时, 终端必须忽略从closed的流上再取得的帧, 如果这个closed流已经发送了RST_STREAM帧. 终端也可以选择一个超时时间, 忽略在此之后到达的帧, 并一律视作错误.
  
  在发送了RST_STREAM之后收到的流控帧(比如DATA帧)也会被用于计算当前连接的流控窗口.(are counted toward the connection flow-control window.) 尽管这些帧有可能被忽略掉, 但是因为他们在发送方收到RST_STREAM之前被发送了, 所以发送方仍有可能会根据这些帧计算流控窗口大小.
  
  终端发送了RST_STREAM帧之后可以再接收一个PUSH_PROMISE帧. PUSH_PROMISE帧会将流状态变为reserved即使相关的流已经被重置. 因此需要一个RST_STREAM帧去关闭不再需要的promised流.
  
本文档中没有给出更具体说明的地方, 对于收到的那些未在上述状态描述中明确认可的帧, 协议实现上应该视这种情况为一个类型为PROTOCOL_ERROR的连接错误. 另外注意PRIORITY帧可以在流的任何一个状态被发送/接收. 忽略未知类型的帧.
  
### 5.1.1 Stream标识符

每个流都用31位无符号整型标识. 客户端初始化流时必须使用奇数做标识, 而那些被服务器初始化的流则要使用偶数做标识. 注: 流标识符0x0用于连接控制消息, 不能用于建立一个新的流.

用于升级到HTTP/2的HTTP/1.1请求会以一个0x1为标识的流响应. 升级完成后, 客户端流0x1处于half-closed(local)状态. 因此, 0x1不能被客户端(从HTTP/1.1升级过来的)选作新的流标示符.

### 5.1.2 流的并发性

## 5.2 流控

### 5.2.1 流控原则

### 5.2.2 恰当使用流控

## 5.3 流的优先级

### 5.3.1 流的依赖关系

### 5.3.2 依赖权重

### 5.3.3 优先级重拍

### 5.3.4 优先级状态管理

### 5.3.5 默认优先级

## 5.4 错误处理

### 5.4.1 连接错误

### 5.4.2 流错误

### 5.4.3 连接终止

## 5.5 HTTP/2扩展