代码对应的version:78ae391f73d5a275b9eb6e9b95dc054653517933

# batch的使用
客户端可以把多个命令打包在一起．读写命令也可以打包在一起. 读写命令应该可以打包在一起．
- 带事务的batch
```
ba := client.Txn.NewBatch()
ba.Put("key1", "a")
ba.Get("key2", "b")
if err := cleint.Txn.Run(ba); err != nil{
	// error hanle
}
```
- 不带事务的batch
```go
func runPut(cmd *cobra.Command, args []string) {
	if len(args) == 0 || len(args)%2 == 1 {
		mustUsage(cmd)
		return
	}

	var b client.Batch
	for i := 0; i < len(args); i += 2 {
		b.Put(
			unquoteArg(args[i], true /* disallow system keys */),
			unquoteArg(args[i+1], false),
		)
	}

	kvDB, stopper := makeDBClient()
	defer stopper.Stop()

	if err := kvDB.Run(&b); err != nil {
		panicf("put failed: %s", err)
	}
}
```
distSender负责把一个batch命令拆分开
- 第一层，把不属性的命令拆分开，拆分的代码见```func (ba BatchRequest) Split(canSplitET bool) [][]RequestUnion ```.有一些命令是不能够在一起执行的．todo:列出来哪些命令不能够在一起执行.  把不同属性分开，是靠distSender里面的sendChunk的区分
- 把向不同range发送的命令进行拆分,  分别发送至不同的```range```
这里使用```func Range(ba roachpb.BatchRequest) roachpb.RSpan```函数，来计算出来一个batch涉及的keys,实际就是计算出来这个batch里面的最大key与最小key


# batch的行为分析．
客户端生成一个batch,这个batch可是基于一个事务生成的batch,也可以是用```client.DB```来运行一个batch  

## client.DB运行一个batch```func (db *DB) Run(b *Batch) *roachpb.Error```
 如果客户端使用```func (db *DB) Run(b *Batch) *roachpb.Error```它会有以下的行为:
 - 首先，客户端会把一个batch当成一个命令发送给Txn_coordinator_sender
 以前的代码里面说明一个batch是一个客户端并发的单元，实际上现在已经没有这个功能．
 - 如果这一个batch并没有跨range, 那么这个batch会利用rocksdb的原子提交特性一起提交．
 - 如果这一个batch跨了range,那么它会被包装成一个事务，进行提交．
 - distSender会把一个batch里面所有的内容发送给replica, replica负责检查batch的哪些操作属于自己,代码参见
 ```go
 func (r *Replica) executeCmd(batch engine.Engine, ms *engine.MVCCStats, h roachpb.Header, args roachpb.Request) (roachpb.Response, []roachpb.Intent, *roachpb.Error) {
	// Verify key is contained within range here to catch any range split
	// or merge activity.
	ts := h.Timestamp

	if _, ok := args.(*roachpb.NoopRequest); ok {
		return &roachpb.NoopResponse{}, nil, nil
	}

	if pErr := r.checkCmdHeader(args.Header()); pErr != nil {
		pErr.Txn = h.Txn
		return nil, nil, pErr
	}
 ```
 
## 带事务的batch
行为基于与不带事务的batch一样，但是所有的写入行都会作为一个事务来提交

# resolveIntent
正常一个事务结束(EndTransaction成功以后)，需要resolveIntent,把所有写入行的intent清除掉，以前这个操作是txn_coordinator发起的，而现在有了优化 ,在endTransaction执行成功的node上直接进行resolveIntent,可以减少消息的来回，而且与endtransaction处于同一个replica的写命令可以与endtransaction走同一个rocksdb的原子提交
- 与endtransaction处理同一个replica的，使用同一个batch,使用rocksdb来保证原子性，同时resolveIntent
- 与事务表不在同一个replica的intent则是使用```handleSkippedIntents```函数来异步resolveIntent  


# 问题:EndTransaction里面的intents从哪里获得的．
这里有一个场景．  
客户端发送一个BatchRequest, 这个batch里面包含了两条put操作，key1 ,key2 ,而且key1与key2应该处于两个不同的range.  
使用commitInBatch进行提交.  
EndTransacton里面的Intents应该从哪里来?    
答案:Txn_coordinator会在把命令发送之前，把所有要写入的key作为intent，放到EndTransaction的参数里面．以前有一个错误的认识，就是intent必须从命令返回值里面拿到，但是实际上intent可以直接从一个request里面拿到,只要是写入操作，如果正常执行，都会产生对应的intent
从以前的代码看来，txn_coordinator在txnMeta里面记录了intents,那么Endtransaction命令经过它的时候，会带上txnMeta里面所有的intents　　
下面这一段代码在txn_coordinator把batch发送给dist_sender之前．
```
if rArgs, ok := ba.GetArg(roachpb.EndTransaction); ok {
			et := rArgs.(*roachpb.EndTransactionRequest)
			if len(et.Key) != 0 {
				return nil, roachpb.NewErrorf("EndTransaction must not have a Key set")
			}
			et.Key = ba.Txn.Key
			// Remember when EndTransaction started in case we want to
			// be linearizable.
			startNS = tc.clock.PhysicalNow()
			if len(et.IntentSpans) > 0 {
				// TODO(tschottdorf): it may be useful to allow this later.
				// That would be part of a possible plan to allow txns which
				// write on multiple coordinators.
				return nil, roachpb.NewErrorf("client must not pass intents to EndTransaction")
			}
			tc.Lock()
			txnMeta, metaOK := tc.txns[id]
			if id != "" && metaOK {
				et.IntentSpans = txnMeta.intentSpans()
			}
			tc.Unlock()

			if intentSpans := ba.GetIntentSpans(); len(intentSpans) > 0 {
				// Writes in Batch, so EndTransaction is fine. Should add
				// outstanding intents to EndTransaction, though.
				// TODO(tschottdorf): possible issues when the batch fails,
				// but the intents have been added anyways.
				// TODO(tschottdorf): some of these intents may be covered
				// by others, for example {[a,b), a}). This can lead to
				// some extra requests when those are non-local to the txn
				// record. But it doesn't seem worth optimizing now.
				et.IntentSpans = append(et.IntentSpans, intentSpans...)
			} else if !metaOK {
				// If we don't have the transaction, then this must be a retry
				// by the client. We can no longer reconstruct a correct
				// request so we must fail.
				//
				// TODO(bdarnell): if we had a GetTransactionStatus API then
				// we could lookup the transaction and return either nil or
				// TransactionAbortedError instead of this ambivalent error.
				return nil, roachpb.NewErrorf("transaction is already committed or aborted")
			}
			if len(et.IntentSpans) == 0 {
				// If there aren't any intents, then there's factually no
				// transaction to end. Read-only txns have all of their state in
				// the client.
				return nil, roachpb.NewErrorf("cannot commit a read-only transaction")
			}
			if log.V(1) {
				for _, intent := range et.IntentSpans {
					trace.Event(fmt.Sprintf("intent: [%s,%s)", intent.Key, intent.EndKey))
				}
			}
		}
```

# batch本身的数据结构


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
	// The Txn the batch is associated with. This field may be nil if the batch
	// was not created via Txn.NewBatch.
	txn *Txn
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
	reqs       []roachpb.Request　//这里其实把所有的request放到这里了．
	resultsBuf [8]Result
	rowsBuf    [8]KeyValue
	rowsIdx    int
}
```
#### 针对batch有一个专门的flat, isAlone, 如果这个字段被设置，说明这个request必须被单独放到一个batch里面．
里面发现EndTransaction AdminSplitRequet AdminMergeRequest必须是isAlone的 ,上面讲到，distSender会把不同属性的命令分拆，为什么需要分拆?todo:

```go
const (
	isAdmin    = 1 << iota // admin cmds don't go through raft, but run on leader
	isRead                 // read-only cmds don't go through raft, but may run on leader
	isWrite                // write cmds go through raft and must be proposed on leader
	isTxn                  // txn commands may be part of a transaction
	isTxnWrite             // txn write cmds start heartbeat and are marked for intent resolution
	isRange                // range commands may span multiple keys
	isReverse              // reverse commands traverse ranges in descending direction
    isAlone                // requests which must be alone in a batch  // Here!!!!!!!!!!!!!!!!!!!!!!!!!!!!
	flagMax                // sentinel
)

func (*EndTransactionRequest) flags() int     { return isWrite | isTxn | isAlone }
func (*AdminSplitRequest) flags() int         { return isAdmin | isAlone }
func (*AdminMergeRequest) flags() int         { return isAdmin | isAlone }

```
