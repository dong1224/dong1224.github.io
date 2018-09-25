# libnetwork中的reaper机制

##
函数handleNetworkEvent

		n.leaving = nEvent.Type == NetworkEventTypeLeave //这个事件类型是干什么的
		if n.leaving {
			n.reapTime = reapInterval
		}

		nDB.addNetworkNode(nEvent.NetworkID, nEvent.NodeName)

		
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
		
		
