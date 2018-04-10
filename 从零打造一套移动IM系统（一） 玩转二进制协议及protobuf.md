## 从零打造一套移动IM系统（一） 玩转二进制协议及protobuf

### 本文默认您已具备以下知识：
- iOS开发的基础知识，以及swift语法
- node.js的基础语法
- TCP基础及IM相关基础知识

### 通过本文您将能收获
- 在iOS上用底层socket，服务器建立tcp连接并通讯
- 如何设计一个二进制通讯协议
- swift当中如何操作二进制网络数据流，会涉及一些unsafe类型及C指针的操作
- node.js中如何操作网络数据流
- protobuf 3.0在客户端及服务器的实际运用，以及在两个平台中的编译、序列化和反序列化
- 心跳保活机制

### 1 基于socket的TCP通讯
#### 1.1 iOS端实现
ios端采用开源库CocoaAsyncSocket，进行TCP通讯。

```
private let delegateQueue = DispatchQueue.global()
private lazy var socket :GCDAsyncSocket = {
    let socket = GCDAsyncSocket(delegate: self, delegateQueue: delegateQueue)
    return socket
}()
```
建立连接

``` 
socket.delegate   = self
try socket.connect(toHost: host, onPort: port)
socket.readData(withTimeout:-1, tag: 0)
```

发送数据

```
self.socket.write(data, withTimeout:5 * 60, tag: 0)
```

连接成功监听

```
func socket(_ sock: GCDAsyncSocket, didConnectToHost host: String, port: UInt16) {
    print("socket \(sock) didConnectToHost \(host) port \(port)")
}
```
连接失败监听

```
func socketDidDisconnect(_ sock: GCDAsyncSocket, withError err: Error?) {
    print("socketDidDisconnect \(sock) withError \(err)")
}
```
发送数据

```
self.socket.write(data, withTimeout:5 * 60, tag: 0)
```
接收数据

```
func socket(_ sock: GCDAsyncSocket, didRead data: Data, withTag tag: Int) {
	let msgArr = SocketDataBuilder.shared().parse(data: data)
	for (seq,socketData) in msgArr {
	    switch (socketData){
	    case .request(let comom):
	        handle(common: comom, seq: seq);
	    case .ping:
	        handlePing(seq: seq);
	    case .message(let msg):
	        handle(message: msg, seq: seq);
	    case .notification(let noti):
	        handle(notification: noti, seq: seq);
	    }
	}
	sock.readData(withTimeout: -1, tag: 0)
}
```
值得注意的是，在接收到数据时，或者读超时的时候需要重新调用``readData(withTimeout:tag)``方法 不然下个数据包到来时，不会再走这个方法。由于我们还有透传体系，需要不间断的监听，所以timeout是-1无穷大

### 1.2 node.js 服务器实现
```
var HOST = '0.0.0.0';
var PORT = 6969;
var server = net.createServer();
server.listen(PORT, HOST);
server.on('connection', function(sock) {

    logger.info('CONNECTED: ' + sock.remoteAddress +':'+ sock.remotePort);
    
    // 接收数据
    sock.on('data', function(data) {
    	
    }
    
    // 断开连接
    sock.on('close', function(data) {      
        logger.info('CLOSED: ' + sock.remoteAddress + ' ' + sock.remotePort);
    });
    
}
```


### 2 TCP部分的通讯协议的二进制头部设计
对于一个TCP数据包，它包含一个二进制头部，和一个包体。包体是protubuf序列化后的数据流。包头一共8个字节，从第一个字节开始依次有以下含义

- margic_num :  1个字节 UInt8 魔法数字，一个特定的数字，服务器与各个终端统一。主要作用是解析时判断包有没有损坏。如果解析出的值与设定值不同，则说明包损坏或者在拆包或解析过程中发生异常
- sequence  :   4个字节  UInt32 序列号，用于区分不同的包，客户端维护，服务器根据不同的连接session+sequence区分不同的包
- type      ：  1个字节  UInt8   包含内容的类型：1心跳包 2普通数据请求 3聊天消息 4推送 根据不同的类型路由到下级业务模块 着四种类型基本包含了一个IM系统主要的业务模块
- length     ：  2个字节  UInt16 包体的长度 通过它获取当前数据包的包体，以进行下一步解析

### 3 二进制头部解析
#### 3.1 iOS中二进制头部处理
先定义一个数据结构来处理头部信息

```
 struct BaseHeader {
    private let margic_num : UInt8 = 0b10000001
    var seq   : UInt32
    var type  : UInt8
    var length : UInt16
}
```
#### 3.1.1 swift的序列化方法

序列化方法

```
func toData()->Data{
    var marg = margic_num.bigEndian
    var seq  = self.seq.bigEndian
    var type =  self.type.bigEndian
    var length = self.length.bigEndian
    
    let mp = UnsafeBufferPointer(start: &marg, count: 1)
    let sp = UnsafeBufferPointer(start: &seq, count: 1)
    let tp = UnsafeBufferPointer(start: &type, count: 1)
    let lp = UnsafeBufferPointer(start: &length, count: 1)

    var data = Data(mp)
    data.append(sp)
    data.append(tp)
    data.append(lp)
    
    return data
}
```
代码比较简单，值得注意的是两点

- 基本数据类型转化为Data必须先转化为UnsafePointer，再转化为UnsafeBufferPointer，再转化为预期的Data数据。最后用data进行拼接
- 再转化为UnsafePointer之前，必须做Big-Endian转化，swift中对应bigEndian计算属性。对于这个问题可以参考[这篇文章](https://blog.csdn.net/kingmax54212008/article/details/46043179)

如果你对位运算比较熟悉，也可以采用下面这种方式。将原始数据转化为UInt8数组再进行拼接

```
var buf = [UInt8]()
append(margic_num, bufArr: &buf)
append(self.seq, bufArr: &buf)
append(self.type, bufArr: &buf)
append(self.length, bufArr: &buf)
let result = Data(buf)

func append<T:FixedWidthInteger>(_ value:T, bufArr:inout [UInt8]){
    let size = MemoryLayout<T>.size
    for i in 1...size {
        let distance = (size - i) * 8;
        let sub  = (value >> distance) & 0xff
        let value = UInt8(sub & 0xff)
        bufArr.append(value)
    }
}
```

#### 3.1.1 swift的反序列化方法
对应的反序列化如下。

```
init?(data:Data){
    if data.count < header_length {
        return nil
    }
    var headerData  = Data(data)
    let tag : UInt8 = headerData[0..<1].withUnsafeBytes{ $0.pointee }
    if tag != margic_num {
        return nil
    }
    let seq : UInt32 = headerData[1..<5].withUnsafeBytes({$0.pointee })
    let typeValue : UInt8  =  headerData[5..<6].withUnsafeBytes({$0.pointee })
    let length : UInt16    =  headerData[6..<8].withUnsafeBytes({$0.pointee })
    
    self.seq  = seq.bigEndian
    self.type  = typeValue.bigEndian
    self.length = length.bigEndian
  
}
```
Data结构体提供了很方便的下标索引方法

```
public subscript(bounds: Range<Data.Index>) -> Data
```
得到的新的Data与原来的数据共用一块内存，只是改变指针的偏移。也就是说，相比原始数据，代表存储结构的``_backing:_DataStorage``属性指向的是同一个对象，只是 ``_sliceRange:Range<Data.Index>``不同

```
public func withUnsafeBytes<ResultType, ContentType>(_ body: (UnsafePointer<ContentType>) throws -> ResultType) rethrows -> ResultType
```
利用这个带范型的方法，可以很容易，对data里面数据进行处理，提取出所需要类型的数据

同样的你也可以在UInt8数组上做文章

```
var index  : Int  = 0
let margic : UInt8   = getValue(data: headerData, index: &index)
let seqv   : UInt32  = getValue(data: headerData, index: &index)
let typev  : UInt8   = getValue(data: headerData, index: &index)
let len    : UInt16  = getValue(data: headerData, index: &index)

func getValue<T:FixedWidthInteger>(data:Data,index:inout Int)->T{
    let size = MemoryLayout<T>.size
    var value:T = 0
    for i in index..<(index+size) {
        let distance = size - (i - index) - 1
        value  += T(data[i]) << distance
    }
    index += size
    return value
}
```

#### 3.2 node.js中二进制头部处理

下面是反序列化代码，data是tcp接收到的数据

```
var header = data.slice(0,8)
var margic = header.readUInt8(0)
var seq    = header.readUInt32BE(1)
var type   = header.readUInt8(5)
var lenth  = header.readUInt16BE(6)
```
序列化方法如下,body为需要发送的包体数据

```
var margic = 129;
var lenth  = body.length;
var header = new Buffer(8);
header.writeUInt8(margic);
header.writeUInt32BE(seq,1);
header.writeUInt8(type,5);
header.writeInt16BE(lenth,6);
```
node.js中，从socket中读取或写入的数据，都是Buffer。调用对应的read或write的方法，很容易从二进制读取或填充所需数据类型的数据。值得注意的是，除了UInt8之外，其余方法都有BE后缀，这也和之前所说的Big-Endian有关

### 4 Protobuf的运用，及数据包体的解析

#### 4.1 .proto文件的编写
采用最新的protobuf3.0的语法，去除了required、optional关键字，枚举类型统一从0开始。

根据从请求头返回的type字段，除了心跳包包体为空外，其他类型包体分别解析为响应的protobuf类型。

其中type=2，被解析为Common类型，对应的是普通数据请求。实际上这部分业务应该作为普通HTTP请求处理。这里统一归入TCP通讯自定义协议体系中。

```
syntax = "proto3";
import  "error.proto";

enum Common_method {
    common_method_user = 0;
    common_method_message = 1;
    common_method_friend   = 2;
    common_method_p2p_connect = 3;
    common_method_respond   = 4;
}

message Common {
    Common_method method = 1;
    bytes body = 2;
}

message CommonRespon {
    bool isSuc = 1;
    bytes respon = 2;
    ErrorMsg error  = 3;
}
```

```
syntax = "proto3";


enum error_type {
    comom_err  = 0;
    invalid_params = 2;
}

message ErrorMsg {
    error_type type = 1;
    string msg = 2;
}
```
``Comon``根据不同的``type``，他的``body``又可以被解析为对应的字类型数据，如``signin_request``、``login_request``、``User_info_request``等等

```
syntax = "proto3"
import "base.proto";

enum User_cmd {
	User_cmd_sign_in = 0;
	User_cmd_login   = 2;
	User_cmd_logout  = 3;
	User_cmd_user_info = 4;
}

message User_msg {
	User_cmd cmd = 1;
	bytes body  = 2;
}

message signin_request {
	 string nick_name = 1;
	 string pwd = 2;
}

message login_request {
	string nick_name = 1; // 用户名
	string pwd = 2;       // 密码
	string ip = 3;        // 设备当前的ip
	int32  port = 4;      // 设备绑定的端口
	string device_name = 5; // iOS/Andoird
	string device_id = 6;   // 设备标识符
	string version  = 7;    // 软件版本
}

message logout_request {
	 int32 uid = 1;
}

// 注册成功 必须进行登录 统一返回uid token
message sigin_response {
	uint32 uid   = 1;
	string token = 2;
}

message login_response {
	 uint32 uid   = 1;
	 string token = 2;
}

// 查询用户资料
message User_info_request {
	uint32 uid = 1; // 所要查询用户的uid
}

message User_info_response {
	User_info user_info = 1;
}
```
type = 3时，对应的是Base_msg类型，对应正儿八经的即时通讯业务模块

type=4时，Notification_msg类型，对应推送模块，及服务器向客户端发送的通知

由于代码量还算比较大，就不贴了。大家自己看源码

#### 4.2 iOS上protobuf的使用
##### 4.2.1 准备工作
将protobuf-swift库导入工程中，在Podfile中加上

```
pod 'ProtocolBuffers-Swift', '4.0.1'
```
电脑上安装protobuf

```
brew install protobuf
```

cd到.proto文件目录，编译出swift平台代码

```
protoc *.proto --swift_out="./"
```

将得到的*.pb.swift文件导入到项目工程当中
##### 4.2.1 序列化方法

以登录请求的包体构建为例为例子

```
let loginReq = LoginRequest().setPwd(pwd).setNickName(user)
let bodyData = try body.build().data()
let user  =  try UserMsg.Builder().setCmd(.userCmdLogin).setBody(bodyData).build().data()
let comom =  try Common.Builder().setMethod(.commonMethodUser).setBody(user).build()

let data = comom.data()
```
##### 4.2.2 反序列化方法

4.2.1 示例代码对应的反序列化，应该是这样子的

```
do {
	let comon =  try Common.parseFrom(data:data)
	switch comon.type {
		case .commonMethodUser:
			let user  =  try UserMsg.parseFrom(data:comon.body)
			switch user.cmd {
				case .userCmdLogin:
					let login = try LoginRequest.parseFrom(data:user.body)
				...
			}
		...
	}
}catch let err {
	print(err)
}
```

#### 4.2.3 完整数据包的构建及解析

无论序列化还是反序列化，都要用到一个中间桥架的结构体

```
enum RTPMessageGenerates {

    case ping
    case request(Common?)
    case message(Message?)
    case notification(NotificationMsg?)

    init?(type:UInt8,data:Data){
        switch type {
        case 1:
            self = .ping
        case 2:
            let comon =  try? Common.parseFrom(data:data)
            self = .request(comon)
        case 3:
            let msg = Message(data: data)
            self = .message(msg)
        case 4:
            let noti = try? NotificationMsg.parseFrom(data: data)
            self = .notification(noti)
        default:
            return nil
        }
    }

    var type : UInt8 {
        switch self {
        case .ping:
            return 1
        case .request(_):
            return 2
        case .message(_):
            return 3
        case .notification(_):
            return 4
        }
    }

    var data : Data? {
        switch self {
        case .ping:
            return Data()
        case .request(let req):
            return  req?.data()
        case .message(let msg):
            return  msg?.data
        case .notification(let noti):
            return noti?.data()
        }
    }

}
```

构建过程如下

```
func rtpData(seq:UInt32,body:RTPMessageGenerates)->Data?{
    guard let bodyData = body.data  else  { return nil }
    let header = BaseHeader(seq: seq, type: body.type, length: UInt16(bodyData.count)).toData()
    let data = header + bodyData
    return data
}
```
解析过程略微复杂点，需要进行拆包处理

```
func parse(data:Data)->[(seq:UInt32,body:RTPMessageGenerates)]{
    var curIndex : UInt16 = 0
    var temp = [(seq:UInt32,body:RTPMessageGenerates)]()
    while curIndex < data.count{
        if curIndex+header_length > data.count {
            break
        }
        let headData = data[curIndex..<curIndex+header_length]
        if let header = BaseHeader(data: headData) {
            let body = data[8..<8+header.length]
            if let msg = RTPMessageGenerates(type: header.type,data: body){
                temp.append((header.seq,msg))
            }
            curIndex += header.length + 8
        }else{
            break;
        }
    }
    return temp
}
```

#### 4.3 node.js服务器protobuf的使用
##### 4.3.1 准备工作
环境配置，包含数据库及日志库环境

```
npm install log4js
npm install mysql
npm install google-protobuf
sudo npm install protobufjs
pm2 install pm2-intercom
```
编译.proto文件

```
protoc --js_out=import_style=commonjs,binary:. *.proto
```
将*_pb.js文件导入项目工程当中
##### 4.3.2 probubuf的解析
需要导入对应模块文件

```
var builder = require("../impb/common_pb"),
    Common = builder.Common;
var MethodType = builder.Common_method;
```

```
try {
    var datas  = Uint8Array(body);
    var common = new Common.deserializeBinary(datas);
    var method = common.getMethod();
    var body   = common.getBody();
}catch (err){
    console.log(err);
}
```
需要留意以下几点：

- socket返回的数据都是Buffer类型的，而protobuf所生成的js文件，相应方法接收的是Uint8Array类型数据，需要做一下转化
- 访问属性变量时不能用点语法，要用对应的get、set方法
- 某些字符做了相应转化，转化为平台的风格。``_``都被转化为驼峰命名法；枚举类型所有字符都被转化为了大写

##### 4.3.3 protobuf的序列化

```
var comon = new Common();
comon.setMethod(MethodType.COMMON_METHOD_RESPOND);
comon.setBody(respond.serializeBinary());

var resData = comon.serializeBinary();
```
主要是serializeBinary()方法的使用。注意赋值的时候要用set方法。得到的是Uint8Array，如果要进行下一步操作需要转化为Buffer类型

##### 4.3.4 完整数据包的解析与构建

完整数据包解析

```
var tempData = new Buffer(data)
	while (tempData.length){
	    var header = data.slice(0,8)
	    var margic = header.readUInt8(0)
	    var seq    = header.readUInt32BE(1)
	    var type   = header.readUInt8(5)
	    var lenth  = header.readUInt16BE(6)
	    var body =   tempData.slice(8,lenth+8)
	    var lest = tempData.length - ( lenth + 8 )
	    logger.info("Receive data :" + "margic=" + margic + " seq=" + seq + " type=" + type + " legth=" + lenth )
	    var bodyData  = new  Uint8Array(body)
	    routeWithReiceData(type,header,bodyData)
	    if (lest.length > 0){
	        logger.info("Has one more data packetge");
	        tempData = data.slice(lenth+8,lest)
	    }else {
	        tempData = lest;
	        break
	    }
	}
}
```
数据包的构建

```
var margic = 129;
var lenth  = body.length;
var header = new Buffer(8);
header.writeUInt8(margic);
header.writeUInt32BE(seq,1);
header.writeUInt8(type,5);
header.writeInt16BE(lenth,6);
var buf = Buffer(body);
var result = Buffer.concat([header,buf])
```

### 5 心跳保活机制
由于存在NAT超时，我们必要在长时间没有数据交互时，主动发送数据包，来维持TCP连接。根据一些博客资料，NAT的超时时间最低的在5分钟左右。关于这些，可以参考[这篇文章](http://www.52im.net/thread-209-1-1.html)

我们设计的心跳间隔是3分钟。心跳由客户端控制，服务器只负责再收到心跳包之后原样返回。当心跳包的响应超时的时候，或重试三次，三次都失败证明与服务器连接中断。主动断开连接再尝试重新连接。

心跳包大小是8个字节，即一个只有包头，包体为空的tcp数据包。

客户端代码如下

```
extension  SocketManager {
    private var  pingDuration : TimeInterval  {  return 60 * 3 }

    static var reTryCount = 0;
    private func sentPing(){
        sentPing { (isSuc) in
            if isSuc {
                SocketManager.reTryCount = 0;
            }else{
                if SocketManager.reTryCount < 3 {
                    self.sentPing()
                    SocketManager.reTryCount += 1
                }else{
                    // 三次失败 连接已经断开 断开再重连
                    self.disconnect()
                    self.reconect(){_ in }
                }
            }
        }
    }
    
    private func sentPing(completion:@escaping (Bool)->()){
        self.sent(msg: .ping, completion: SocketManager.SentMsgCompletion.ping(completion))
    }
    
    func stopPing(){
        self.pingTimer?.invalidate()
        self.pingTimer  = nil;
    }
    
    func startPing(){
        sentPing()
        if pingTimer == nil {
            pingTimer  = Timer(timeInterval:pingDuration , repeats: true, block: {[weak self] (timer) in
                self?.sentPing()
            })
        }
    }
    
}
```

服务器代码：

```
 function routeWithReiceData(type,header,body) {
    switch (type){
        case 1:
            // 收到心跳包原样返回 客户端控制发送频率 必要时断开重连
            sock.write(data)
            break;
    }
 }
```

### 6 源码位置
客户端代码：[https://github.com/chaocaiwei/IMPBSocket-iOS](https://github.com/chaocaiwei/IMPBSocket-iOS)

服务器代码：[https://github.com/chaocaiwei/IMPBSocket-Server](https://github.com/chaocaiwei/IMPBSocket-Server)

这两个项目正在开发初期，尤其是客户端，界面太简陋请忽略。同时，欢迎start，欢迎提issue，欢迎PR。

水平有限，时间仓促，如有错漏，万望海涵。同时，欢迎提出各种意见。

- 后期争取完善IM部分单聊群聊的逻辑，完善各种消息形式的互通，加入消息安全机制。
- 尽量保障隐私，服务器除了对离线消息的缓存，原则上只转发不存储。位于同一网段的设备，直接走p2p不经过服务器。
- 可能的话加上p2p打洞逻辑。只有走p2p失败服务器再转发。这部分服务器参考[coturn](https://github.com/coturn/coturn)。客户端参考webrtc的实现
- 加上流媒体模块，实现实时语音对讲、视频聊天。这部分打算用webrtc

感觉埋的坑很多，后面会一边填坑，一边总结成文章。这也是对这些年来做开发，基础方面的一次总结提升吧。

以上
