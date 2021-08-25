# compact 源码解析

当你通过 API 发起一个 compact 请求后： 

1.  KV Server 收到 Compact 请求提交到 raft 模块处理 
2. 在 raft 模块中提交后，apply 模块就会通过 MVCC 模块的 Compact 接口执行 此压缩任务



 Compact 接口首先会更新当前 server 已压缩的版本号，并将耗时昂贵的压缩任务保存到 FIFO 队列中异步执行：

1. 压缩任务执行时，它首先会压缩 treeIndex 模块中的 keyIndex 索引 
2. 其次会遍历 boltdb 中的 key ，删除已废弃的 key



raft 模块对于 memoryStorage 定义：

注意 ents 这个变量，它是一个\* 结构体类型的变量，其中字段含义如下：

* Term
* Index
* Type
* Data

```go
// MemoryStorage implements the Storage interface backed by an
// in-memory array.

type MemoryStorage struct {
	// Protects access to all fields. Most methods of MemoryStorage are
	// run on the raft goroutine, but Append() is run on an application
	// goroutine.
	sync.Mutex

	hardState pb.HardState
	snapshot  pb.Snapshot
	
	// ents[i] has raft log position i+snapshot.Metadata.Index
	ents []pb.Entry

```

```go
type Entry struct {
   Term  uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`
   Index uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
   Type  EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
   Data  []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
}
```

**memoryStorage Compact 实现逻辑：**

1. 计算当前 \* 中的索引位置  `offset := ms.ents[0].Index`
2. 判断压缩的版本是否小于等于这个位置
   1. 是：报错并返回
   2. 否：继续
3. 判断要压缩的版本，是否高于当前最大的版本
   1. 是：报错并
   2. 否：继续
4. 计算本次要压缩版本的偏移量，也就是要压缩的版本在这个数组中的下标  `i := compactIndex - offset`
5. 定义一个与原 ents 类型相同的空数组，长度为 1，容量为要保留的元素个数 + 1 `ents := make([]pb.Entry, 1, 1+uint64(len(ms.ents))-i)`
6. 将该数组下标为 0 的元素（即：第一个元素）的 Index 和 Term 赋值为第 i 个元素的 Index 和 Term 
7. 将原 ents 第 i 个元素后的所有内容（不包括第 i 个元素），填充到该数组的后部分（即：除了第一个元素外）`ents = append(ents, ms.ents[i+1:]...)`
8. 用新的 ents 整体替换旧的 ，这样即便 compact 操作过程耗时，也不会影响到其他操作`ms.ents = ents` 

```go
// Compact discards all log entries prior to compactIndex.
// It is the application's responsibility to not attempt to compact an index
// greater than raftLog.applied.

func (ms *MemoryStorage) Compact(compactIndex uint64) error {
	ms.Lock()
	defer ms.Unlock()
	offset := ms.ents[0].Index
	if compactIndex <= offset {
		return ErrCompacted
	}
	if compactIndex > ms.lastIndex() {
		getLogger().Panicf("compact %d is out of bound lastindex(%d)", compactIndex, ms.lastIndex())
	}

	i := compactIndex - offset
	ents := make([]pb.Entry, 1, 1+uint64(len(ms.ents))-i)
	ents[0].Index = ms.ents[i].Index
	ents[0].Term = ms.ents[i].Term
	ents = append(ents, ms.ents[i+1:]...)
	ms.ents = ents
	return nil
}
```





 在 server 启动时，会通过  `func (s *EtcdServer) run()` 实时的从队列中获去需要执行的信息 `case ap := <-s.r.apply()`

从 KVStore 中删除已 compact 的 version，就是其中一个分类。

那什么时候往队列里放东西呢？

当 raftNode ready 的时候。这个状态，我们后续再详细分析。

## KVStore 获得和应用 compact entry 的方式

### func \(s \*EtcdServer\) apply

compact 属于 normal entry 的一种

```go
func (s *EtcdServer) apply(
  
   ...
   
   for i := range es {
     
     ...
     
      switch e.Type {
      case raftpb.EntryNormal:
         s.applyEntryNormal(&e)
        
        ...
}
```

### func \(s \*EtcdServer\) applyEntryNormal

这里的 index 应该不是指 compact 的 Index 吧

```go
func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry) {
	shouldApplyV3 := membership.ApplyV2storeOnly
	index := s.consistIndex.ConsistentIndex()
	if e.Index > index {
		// set the consistent index of current executing entry
		s.consistIndex.SetConsistentIndex(e.Index, e.Term)
		shouldApplyV3 = membership.ApplyBoth
	}
	
	...
	
	if needResult || !noSideEffect(&raftReq) {
	
		...
		
		ar = s.applyV3.Apply(&raftReq, shouldApplyV3)
	}
	
	...
	
}
```

### func \(a \*applierV3backend\) Apply

r 是什么类型？通道吗？

```go
func (a *applierV3backend) Apply(r *pb.InternalRaftRequest, shouldApplyV3 membership.ShouldApplyV3) *applyResult {
   
   ...

   switch {
   
   ...
   
   case r.Compaction != nil:
      op = "Compaction"
      ar.resp, ar.physc, ar.trace, ar.err = a.s.applyV3.Compaction(r.Compaction)
  
   ...
   
}
```

### func \(a \*applierV3backend\) Compaction

```go
func (a *applierV3backend) Compaction(compaction *pb.CompactionRequest) (*pb.CompactionResponse, <-chan struct{}, *traceutil.Trace, error) {
	...
	
	ch, err := a.s.KV().Compact(trace, compaction.Revision)
	
	...
}
```

### func \(s \*store\) Compact

```go
func (s *store) Compact(trace *traceutil.Trace, rev int64) (<-chan struct{}, error) {
	s.mu.Lock()

	ch, err := s.updateCompactRev(rev)
	trace.Step("check and update compact revision")
	if err != nil {
		s.mu.Unlock()
		return ch, err
	}
	s.mu.Unlock()

	return s.compact(trace, rev)
}
```

### func \(s \*store\) compact

```go
func (s *store) compact(trace *traceutil.Trace, rev int64) (<-chan struct{}, error) {
	ch := make(chan struct{})
	var j = func(ctx context.Context) {
		if ctx.Err() != nil {
			s.compactBarrier(ctx, ch)
			return
		}
		start := time.Now()
		keep := s.kvindex.Compact(rev)
		indexCompactionPauseMs.Observe(float64(time.Since(start) / time.Millisecond))
		if !s.scheduleCompaction(rev, keep) {
			s.compactBarrier(context.TODO(), ch)
			return
		}
		close(ch)
	}

	s.fifoSched.Schedule(j)
	trace.Step("schedule compaction")
	return ch, nil
}
```

### func \(ti \*treeIndex\) Compact

```go
func (ti *treeIndex) Compact(rev int64) map[revision]struct{} {
	available := make(map[revision]struct{})
	ti.lg.Info("compact tree index", zap.Int64("revision", rev))
	ti.Lock()
	clone := ti.tree.Clone()
	ti.Unlock()

	clone.Ascend(func(item btree.Item) bool {
		keyi := item.(*keyIndex)
		//Lock is needed here to prevent modification to the keyIndex while
		//compaction is going on or revision added to empty before deletion
		ti.Lock()
		keyi.compact(ti.lg, rev, available)
		if keyi.isEmpty() {
			item := ti.tree.Delete(keyi)
			if item == nil {
				ti.lg.Panic("failed to delete during compaction")
			}
		}
		ti.Unlock()
		return true
	})
	return available
}
```

### func \(ki \*keyIndex\) compact

```go
func (ki *keyIndex) compact(lg *zap.Logger, atRev int64, available map[revision]struct{}) {
	if ki.isEmpty() {
		lg.Panic(
			"'compact' got an unexpected empty keyIndex",
			zap.String("key", string(ki.key)),
		)
	}

	genIdx, revIndex := ki.doCompact(atRev, available)

	g := &ki.generations[genIdx]
	if !g.isEmpty() {
		// remove the previous contents.
		if revIndex != -1 {
			g.revs = g.revs[revIndex:]
		}
		// remove any tombstone
		if len(g.revs) == 1 && genIdx != len(ki.generations)-1 {
			delete(available, g.revs[0])
			genIdx++
		}
	}

	// remove the previous generations.
	ki.generations = ki.generations[genIdx:]
}
```

### func \(ki \*keyIndex\) doCompact

这个看起来，也是先操作索引，没有真正的删除

```go
func (ki *keyIndex) doCompact(atRev int64, available map[revision]struct{}) (genIdx int, revIndex int) {
	// walk until reaching the first revision smaller or equal to "atRev",
	// and add the revision to the available map
	f := func(rev revision) bool {
		if rev.main <= atRev {
			available[rev] = struct{}{}
			return false
		}
		return true
	}

	genIdx, g := 0, &ki.generations[0]
	// find first generation includes atRev or created after atRev
	for genIdx < len(ki.generations)-1 {
		if tomb := g.revs[len(g.revs)-1].main; tomb > atRev {
			break
		}
		genIdx++
		g = &ki.generations[genIdx]
	}

	revIndex = g.walk(f)

	return genIdx, revIndex
}
```

### func \(g \*generation\) walk

```go
func (g *generation) walk(f func(rev revision) bool) int {
	l := len(g.revs)
	for i := range g.revs {
		ok := f(g.revs[l-i-1])
		if !ok {
			return l - i - 1
		}
	}
	return -1
}
```

## 技巧

从源码上，我们除了可以看出 etcd 是如何实现 compact 的以外，还可以学到一些小技巧。

比如在给切片做 append 时，一定要格外注意长度（len）和容量（cap）的区别。我们分别创建容量为 5， 和长度为 5 的切片，观察在它们之后做 append 的效果：

```go
package main

import "fmt"

func main() {

	capCase := make([]int, 0, 5)
	fmt.Printf("%v\n", append(capCase,[]int{3,4}...))

	lenCase := make([]int,5)
	fmt.Printf("%v", append(lenCase,[]int{3,4}... ))
}
```

输出：

\[3 4\] 

\[0 0 0 0 0 3 4\]

