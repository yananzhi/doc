# 看一下endtransaction如何把intent带到参数上面.

# 看一下那个问题,endtransaction为什么一定需要txnKey存在?


// EndTransaction either commits or aborts (rolls back) an extant
// transaction according to the args.Commit parameter.
// TODO(tschottdorf): return nil reply on any error. The error itself
// must be the authoritative source of information.
func (r *Replica) EndTransaction(batch engine.Engine, ms *engine.MVCCStats, h roachpb.Header, args roachpb.EndTransactionRequest) (roachpb.EndTransactionResponse, []roachpb.Intent, error) {
	var reply roachpb.EndTransactionResponse
	ts := h.Timestamp // all we're going to use from the header.

	if err := verifyTransaction(h, &args); err != nil {
		return reply, nil, err
	}

	key := keys.TransactionKey(h.Txn.Key, h.Txn.ID)

	// Fetch existing transaction.
	reply.Txn = &roachpb.Transaction{}
	if ok, err := engine.MVCCGetProto(batch, key, roachpb.ZeroTimestamp, true, nil, reply.Txn); err != nil {
		return reply, nil, err
	} else if !ok {// 为什么有这一行代码
		return reply, nil, util.Errorf("transaction does not exist: %s on store %d", h.Txn, r.store.StoreID())
	}
