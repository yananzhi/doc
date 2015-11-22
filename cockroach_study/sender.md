# 背景
在cockroach里面,sender是一个发送kv命令的接口. 有不同的几个sender分部署在不同的位置.


sender的接口.
```
// Sender is the interface used to call into a Cockroach instance.
// If the returned *roachpb.Error is not nil, no response should be returned.
type Sender interface {
	Send(context.Context, roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error)
}
```

有以下几个方法实现了Sender interface
``` go
func (ts *txnSender) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error)
func (tc *TxnCoordSender) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error)
func (ds *DistSender) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error) 
func (cs *chunkingSender) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error) 
func (s *rpcSender) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error)
func (ls *LocalSender) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error) 
func (r *Replica) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error)
func (s *Store) Send(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, *roachpb.Error)  
todo:这里可好像没有串起来,这里少了txn client.DB如何把一个BatchRequest发送给txnCoordSender的.
```

基础数据结构

context 主要带了几个跨api接口的几个对象
// A Context carries a deadline, a cancelation signal, and other values across
// API boundaries.
//

```
/ A BatchRequest contains one or more requests to be executed in
// parallel, or if applicable (based on write-only commands and
// range-locality), as a single update.
//
// The Span should contain the Key of the first request
// in the batch. It also contains the transaction itself; individual
// calls must not have transactions specified. The same applies to
// the User and UserPriority fields.
type BatchRequest struct {
	Header   `protobuf:"bytes,1,opt,name=header,embedded=header" json:"header"`
	Requests []RequestUnion `protobuf:"bytes,2,rep,name=requests" json:"requests"`
}
这里认为一个BatchRequest可以一个或者多个request. 而这些request可以并发执行(在distSender执行?).
如果可能的话,一个BatchRequest可以作为一个single update来执行(这里可能说的就是作为一个命令在replica层去执行)
```


Batch接口
```
// Batch provides for the parallel execution of a number of database
// operations. Operations are added to the Batch and then the Batch is executed
// via either DB.Run, Txn.Run or Txn.Commit.
//
// TODO(pmattis): Allow a timestamp to be specified which is applied to all
// operations within the batch.
type Batch struct {
	// The DB the batch is associated with. This field may be nil if the batch
	// was not created via DB.NewBatch or Txn.NewBatch.
	DB *DB
	// Results contains an entry for each operation added to the batch. The order
	// of the results matches the order the operations were added to the
	// batch. For example:
	//
	//   b := db.NewBatch()
	//   b.Put("a", "1")
	//   b.Put("b", "2")
	//   _ = db.Run(b)
	//   // string(b.Results[0].Rows[0].Key) == "a"
	//   // string(b.Results[1].Rows[0].Key) == "b"
	Results    []Result
	reqs       []roachpb.Request
	resultsBuf [8]Result
	rowsBuf    [8]KeyValue
	rowsIdx    int
}
```
todo: 如果batch里面有两个put 一个get,而且操作是同一个key,那么会是什么样子?
b.Put("a",)
b.Get("a")
那么,get得到的值是put之前的值,还是put之后的值.(应该不是允许这样使用.)



# 各个对象是如何配合的
client.DB有一个Txn的方法 
```

// Txn executes retryable in the context of a distributed transaction. The
// transaction is automatically aborted if retryable returns any error aside
// from recoverable internal errors, and is automatically committed
// otherwise. The retryable function should have no side effects which could
// cause problems in the event it must be run more than once.
//
// TODO(pmattis): Allow transaction options to be specified.
func (db *DB) Txn(retryable func(txn *Txn) error) error {
	txn := NewTxn(*db)
	txn.SetDebugName("", 1)
	return txn.exec(retryable)
}
```
Txn方法就是使用client.DB去执行一个事务,事务的代码可以写到```reryable func(txn *Txn)```里面. 


Txn->DB   = copy of client.DB
Txn->wrapped = client.DB.Sender  Txn的Sender使用了DB对应的sender
把对应的DB sender修改成为一个txnSender,  txn本身就是一个sender,它实现了send interface

client.DB.sender是什么?
一般情况下,它接的都是TxnCoordSender

## TxnCoordSender下面接的是什么
一般情况下,TxnCoordSender下面接的是DistSender,
特殊情况 , 一个node在启动的时候,它的下面直接接的是LocalSender

## DistSender下面接是什么
从代码上看它使用一个钩子函数.
```
// A DistSender provides methods to access Cockroach's monolithic,
// distributed key value store. Each method invocation triggers a
// lookup or lookups to find replica metadata for implicated key
// ranges. RPCs are sent to one or more of the replicas to satisfy
// the method invocation.
type DistSender struct {
	// nodeDescriptor, if set, holds the descriptor of the node the
	// DistSender lives on. It should be accessed via getNodeDescriptor(),
	// which tries to obtain the value from the Gossip network if the
	// descriptor is unknown.
	nodeDescriptor unsafe.Pointer
	// clock is used to set time for some calls. E.g. read-only ops
	// which span ranges and don't require read consistency.
	clock *hlc.Clock
	// gossip provides up-to-date information about the start of the
	// key range, used to find the replica metadata for arbitrary key
	// ranges.
	gossip *gossip.Gossip
	// rangeCache caches replica metadata for key ranges.
	rangeCache           *rangeDescriptorCache
	rangeLookupMaxRanges int32
	// leaderCache caches the last known leader replica for range
	// consensus groups.
	leaderCache *leaderCache
	// RPCSend is used to send RPC calls and defaults to rpc.Send
	// outside of tests.
	rpcSend         rpcSendFn  // 这个钩子函数
	rpcRetryOptions retry.Options
}

### 钩子函数
// rpcSendFn is the function type used to dispatch RPC calls.
type rpcSendFn func(rpc.Options, string, []net.Addr,
	func(addr net.Addr) proto.Message, func() proto.Message,
	*rpc.Context) ([]proto.Message, error)
```
DistSender使用这个钩子函数来发送消息.

这个钩子函数钩子函数实际上为rpc.Send
```
// NewDistSender returns a batch.Sender instance which connects to the
// Cockroach cluster via the supplied gossip instance. Supplying a
// DistSenderContext or the fields within is optional. For omitted values, sane
// defaults will be used.
func NewDistSender(ctx *DistSenderContext, gossip *gossip.Gossip) *DistSender {
	if ctx == nil {
		ctx = &DistSenderContext{}
	}
	clock := ctx.Clock
	if clock == nil {
		clock = hlc.NewClock(hlc.UnixNano)
	}
	ds := &DistSender{
		clock:  clock,
		gossip: gossip,
	}
	if ctx.nodeDescriptor != nil {
		atomic.StorePointer(&ds.nodeDescriptor, unsafe.Pointer(ctx.nodeDescriptor))
	}
	rcSize := ctx.RangeDescriptorCacheSize
	if rcSize <= 0 {
		rcSize = defaultRangeDescriptorCacheSize
	}
	rdb := ctx.RangeDescriptorDB
	if rdb == nil {
		rdb = ds
	}
	ds.rangeCache = newRangeDescriptorCache(rdb, int(rcSize))
	lcSize := ctx.LeaderCacheSize
	if lcSize <= 0 {
		lcSize = defaultLeaderCacheSize
	}
	ds.leaderCache = newLeaderCache(int(lcSize))
	if ctx.RangeLookupMaxRanges <= 0 {
		ds.rangeLookupMaxRanges = defaultRangeLookupMaxRanges
	}
	ds.rpcSend = rpc.Send
	if ctx.RPCSend != nil {
		ds.rpcSend = ctx.RPCSend
	}
	ds.rpcRetryOptions = defaultRPCRetryOptions
	if ctx.RPCRetryOptions != nil {
		ds.rpcRetryOptions = *ctx.RPCRetryOptions
	}

	return ds
}
```

## rpcServer接收命令？

而rpc.Send直接使用rpc的客户端把命令发送给rpcServer了.
## 一些疑问

```
// NewTxn returns a new txn.
func NewTxn(db DB) *Txn {
	txn := &Txn{
		db:      db,
		wrapped: db.sender,
	}
	txn.db.sender = (*txnSender)(txn)
	return txn
	todo: 这里修改了db.sender对吗?
	DB上面的txn不能并发访问?  没有问题,这里传递的是DB的一个实例,而不是一个指针.
}
```

client.DB
它并没有实现sender的接口,但是它提供的接口与sender差不多.
func (db *DB) send(reqs ...roachpb.Request) (*roachpb.BatchResponse, *roachpb.Error) 

而且db也是一个wraper类,(相对于sender来讲)
```
// DB is a database handle to a single cockroach cluster. A DB is safe for
// concurrent use by multiple goroutines.
type DB struct {
	sender Sender // 它包括了一个sender, 如果在客户端,这个sender应该是httpSender , 现在可能修改成了rpcSender? todo:

	// userPriority is the default user priority to set on API calls. If
	// userPriority is set non-zero in call arguments, this value is
	// ignored.
	userPriority    int32
	txnRetryOptions retry.Options
}
```