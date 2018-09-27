# libnetwork中的reaper机制

## gossip与bulksync
With the current design in libnetwork control plane events are first sent through gossip. This is through UDP which can be lossy. To account for that there is an anti entropy phase every 30 seconds that does a full state sync.
目前libnetwork的事件同步机制，首先是通过gossip同步（udp可能会丢失），再加上30秒一次的状态同步bulksync。
可以理解为gossip可以快速的散播消息，bulksync比较慢，但是准确。

## 初始化
可以看到 不管是收割机reapstat还是消息同步gossip与bulksynctables都靠定时器触发。

	for _, trigger := range []struct {
		interval time.Duration
		fn       func()
	}{
		{reapPeriod, nDB.reapState},
		{config.GossipInterval, nDB.gossip},
		{config.PushPullInterval, nDB.bulkSyncTables},
		{retryInterval, nDB.reconnectNode},
		{nodeReapPeriod, nDB.reapDeadNode},
	} {
		t := time.NewTicker(trigger.interval)
		go nDB.triggerFunc(trigger.interval, t.C, nDB.stopCh, trigger.fn)
		nDB.tickers = append(nDB.tickers, t)
	}
	
## reapState
收割机reapstate会定期清理到期的数据库内容，也就是reaptime < 0时清数据库


## 关于故障 1944 
[pr1944](https://github.com/docker/libnetwork/pull/1944)
故障点 tableentry没有 填入reaptime字段。
那么一旦进入reapTableEntries 数据库就会被删除，
也就是说最终的entry状态如果在这之前没有传播出去，那么最终的状态其实就丢了。剩下堵在网络中的状态全是过期的。

func (nDB *NetworkDB) reapTableEntries() {

		if entry.reapTime > 0 {      //这里永远不会成立，往下执行下去就是删数据库
			entry.reapTime -= reapPeriod
			return false
		}

	})


##消息接收

	func (nDB *NetworkDB) handleMessage(buf []byte, isBulkSync bool) {
		mType, data, err := decodeMessage(buf)
		if err != nil {
			logrus.Errorf("Error decoding gossip message to get message type: %v", err)
			return
		}

		switch mType {
		case MessageTypeNodeEvent:
			nDB.handleNodeMessage(data)
		case MessageTypeNetworkEvent:
			nDB.handleNetworkMessage(data)
		case MessageTypeTableEvent:
			nDB.handleTableMessage(data, isBulkSync)
		case MessageTypeBulkSync:
			nDB.handleBulkSync(data)
		case MessageTypeCompound:
			nDB.handleCompound(data, isBulkSync)
		default:
			logrus.Errorf("%s: unknown message type %d", nDB.config.NodeName, mType)
		}
	}

进到消息处理里面gossip和bulksync入口是一样的。区别在于gossip会根据需要进行转发，而bulkysync定点发送，不需要转发。

	if rebroadcast := nDB.handleTableEvent(&tEvent); rebroadcast && !isBulkSync {
	}
	


唯一删除entry的地方

		func (nDB *NetworkDB) reapTableEntries() {
			var paths []string

			nDB.RLock()
			nDB.indexes[byTable].Walk(func(path string, v interface{}) bool {
				entry, ok := v.(*entry)
				if !ok {
					return false
				}

				if !entry.deleting {
					return false
				}
				if entry.reapTime > 0 {
					entry.reapTime -= reapPeriod
					return false
				}
				paths = append(paths, path)
				return false
			})
			nDB.RUnlock()

			nDB.Lock()
			for _, path := range paths {
				params := strings.Split(path[1:], "/")
				tname := params[0]
				nid := params[1]
				key := params[2]

				if _, ok := nDB.indexes[byTable].Delete(fmt.Sprintf("/%s/%s/%s", tname, nid, key)); !ok {
					logrus.Errorf("Could not delete entry in table %s with network id %s and key %s as it does not exist", tname, nid, key)
				}

				if _, ok := nDB.indexes[byNetwork].Delete(fmt.Sprintf("/%s/%s/%s", nid, tname, key)); !ok {
					logrus.Errorf("Could not delete entry in network %s with table name %s and key %s as it does not exist", nid, tname, key)
				}
			}
			nDB.Unlock()
		}
		
## bulksync
	
bulkSyncTables

		// When a node leaves a network on the last task removal cleanup the
		// local entries for this network & node combination. When the tasks
		// on a network are removed we could have missed the gossip updates.
		// Not doing this cleanup can leave stale entries because bulksyncs
		// from the node will no longer include this network state.
	
	
bulksync只是随机找一个node同步过去

	func (nDB *NetworkDB) bulkSync(nodes []string, all bool) ([]string, error) {
		if !all {
			// If not all, then just pick one.
			nodes = nDB.mRandomNodes(1, nodes)
		}
	
handleNetworkEvent

	func (nDB *NetworkDB) handleNetworkEvent(nEvent *NetworkEvent) bool {
	
		n.leaving = nEvent.Type == NetworkEventTypeLeave
		if n.leaving {
			n.reapTime = reapInterval
		}

## 故障1704	
[pr1704](https://github.com/docker/libnetwork/pull/1704/)
如果gossip丢失的情况下，刚好发生了leavenetwork。那么bulksync中就没有此节点的信息了，消息彻底丢失。
	
		n.leaving = nEvent.Type == NetworkEventTypeLeave //这个事件类型是由dockerd触发，收到此消息后会删除当前节点上的关于此network的所有内容
		if n.leaving {
			n.reapTime = reapInterval
		}

		nDB.addNetworkNode(nEvent.NetworkID, nEvent.NodeName)

// LeaveNetwork leaves this node from a given network and propagates
// this event across the cluster. This triggers this node leaving the
// sub-cluster of this network and as a result will no longer
// participate in the network-scoped gossip and bulk sync for this
// network. Also remove all the table entries for this network from
// networkdb
func (nDB *NetworkDB) LeaveNetwork(nid string) error {
		
leaveNetwork -> NetworkEventTypeLeave

堆栈

WARNING: Missing unwind data for a module, rerun with 'stap -d (unknown; retry with -DDEBUG_UNWIND)'
 0x8afbf8 : github.com/docker/docker/vendor/github.com/docker/libnetwork/networkdb.(*NetworkDB).LeaveNetwork+0x68/0xe40 [/usr/bin/dockerd]
 0x83642a : github.com/docker/docker/vendor/github.com/docker/libnetwork.(*network).leaveCluster+0x9a/0x100 [/usr/bin/dockerd]
 0x86366a : github.com/docker/docker/vendor/github.com/docker/libnetwork.(*network).delete+0x4ea/0xd50 [/usr/bin/dockerd]
 0x863150 : github.com/docker/docker/vendor/github.com/docker/libnetwork.(*network).Delete+0x30/0x60 [/usr/bin/dockerd]
 0x595cf7 : github.com/docker/docker/daemon.(*Daemon).deleteNetwork+0x107/0x510 [/usr/bin/dockerd]
 0x595b54 : github.com/docker/docker/daemon.(*Daemon).DeleteManagedNetwork+0x44/0x70 [/usr/bin/dockerd]
 0xad2e2a : github.com/docker/docker/daemon/cluster/executor/container.(*containerAdapter).removeNetworks+0xaa/0x3b0 [/usr/bin/dockerd]
 0xadcb7f : github.com/docker/docker/daemon/cluster/executor/container.(*controller).Remove+0xdf/0x3a0 [/usr/bin/dockerd]
 0xa98322 : github.com/docker/docker/vendor/github.com/docker/swarmkit/agent.reconcileTaskState.func1+0xb2/0x360 [/usr/bin/dockerd]
 0x472871 : runtime.goexit+0x1/0x10 [/usr/bin/dockerd]
 0xc4214accc0 : 0xc4214accc0
------------------




