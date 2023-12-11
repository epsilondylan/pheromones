# pheromones
Peer to Peer network.

## 使用
### 短连接
#### 创建p2p节点

``` go
r1 := p2p.NewSRouter(timeout) // 短连接路由
p1 := pto.NewProtocal("luda", r1, timeout)
s1 := p2p.NewServer(p1, timeout)
println("h1 监听 12345")
go s1.ListenAndServe("127.0.0.1:12345")

r2 := p2p.NewSRouter(timeout) // 短连接路由
p2 := pto.NewProtocal("yoghurt", r2, timeout)
s2 := p2p.NewServer(p2, timeout)
println("h2 监听 12345")
go s2.ListenAndServe("127.0.0.1:12346")
```
#### 添加路由

``` go
p1.Add("yoghurt", "127.0.0.1:12346")
```

#### 发送数据
由于为了完成协议状态机，因此需要循环对返回结果进行协议解析，直到返回结果为空

``` go
for msg != nil {
    b, err := p1.Dispatch("yoghurt", msg)
    if err != nil {
        println("操作失败", err.Error())
        break
    }
    msg = nil
    msg, err = p1.Handle(nil, b)
    fmt.Println(string(msg), err)
}
```

## 实现
### Server层 通常意义上的Server功能
``` go
// 开启接口监听，将读到的数据传输给prtocal层解析
ListenAndServe(addr string) error
```

### Protocal层 handler功能
提供一个空的接口，需要用户来实现协议
``` go
type Protocal interface {
    // 解析请求通信内容,并返回数据,双工协议
    Handle(c net.Conn, msg []byte) ([]byte, error)
}
```
_example 中实现支持了一个长/短的协议状态机器
状态机为：
``` go
func (p *Protocal) Handle(c net.Conn, msg []byte) ([]byte, error) {
    cType := p.Router.GetConnType()
    req := &p2p.MsgPto{}
    resp := &p2p.MsgPto{}
    err := json.Unmarshal(msg, req)
    if err != nil {
        resp.Name = p.HostName
        resp.Operation = UnknownOp
        ret, _ := json.Marshal(resp)
        return ret, p2p.Error(p2p.ErrMismatchProtocalReq)
    }
    resp.Name = p.HostName
    switch req.Operation {
    case ConnectReq:
        subReq := &MsgGreetingReq{}
        err := json.Unmarshal(req.Data, subReq)
        if err != nil {
            return nil, p2p.Error(p2p.ErrMismatchProtocalResp)
        }
        if cType == p2p.ShortConnection {
            err = p.Router.AddRoute(req.Name, subReq.Addr)
        } else {
            if p.Router.AddRoute(req.Name, c) == nil {
                go p.IOLoop(c)
            }
        }
        if err != nil {
        }
        resp.Operation = ConnectResp
    case GetReq:
        resp.Operation = GetResp
    case FetchReq:
        resp.Operation = FetchResp
    case NoticeReq:
        resp.Operation = NoticeResp
    case ConnectResp:
        resp.Operation = GetReq
    case GetResp:
        resp.Operation = FetchReq
    case FetchResp:
        resp.Operation = NoticeReq
    case NoticeResp:
        return nil, nil
    default:
        resp.Operation = UnknownOp
    }
    ret, err := json.Marshal(resp)
    return ret, nil
}
```
### Router层 Client功能：发送数据
同样是接口，只不过这次提供了一套长/短连接的默认实现。
``` go
type Router interface {
    // 添加路由：短链接传的是地址；长链接传的是net.Conn
    AddRoute(name string, addr interface{}) error
    // 删除路由
    Delete(name string) error
    // 获取连接类型
    GetConnType() ConnType
    // 广播发送信息
    DispatchAll(msg []byte) map[string][]byte
    // 单点发送信息
    Dispatch(name string, msg []byte) ([]byte, error)
}
```
