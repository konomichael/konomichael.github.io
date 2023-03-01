+++
title = "Understanding the Behavior of Orderer"
date = "2022-12-17T20:09:10+08:00"
author = "konomichael"
authorTwitter = "@konomichael_" #do not include @
tags = ["hyperledger", "orderer", "blockchain"]
keywords = ["hyperledger", "orderer", "blockchain"]
readingTime = true
hideComments = false

+++

The default orderer server in fabric composes of the `Registrar`, which serves as a point of access and control for the individual channel resources. When the orderer is instantiated, the `Registrar.StartChannels()` is invoked:

```go
func (r *Registrar) startChannels() {
	for _, chainSupport := range r.chains {
		chainSupport.start()
	}
	for _, fChain := range r.followers {
		fChain.Start()
	}

	if r.systemChannelID == "" {
		logger.Infof("Registrar initializing without a system channel, number of application channels: %d, with %d consensus.Chain(s) and %d follower.Chain(s)",
			len(r.chains)+len(r.followers), len(r.chains), len(r.followers))
	}
}
```

The `chains` and `followers` above are different fields of the `Registrar`. Both of them are `map`s, the key is `ChannelID`, the value, however, is `ChainSupport` for `chains` and `Chain` for `followers`.  When the orderer starts, it scans local ledgers and determines if it is a member of each channel. If it is, add a `ChainSupport` to the map `chains`, otherwise, add a `Chain` to the map `followers`. The `ChainSupport`, of course, composes of the `Chain`, but can do more things than later.

In this post, we will inspect the `Chain` and `ChainSupport`, and dig out what they have done when the orderer starts.

<!--more-->

## Chain.Start

The `Chain.Start` is called when the orderer is not a member of the channel which means the `Chain` is responsible for just copying the blocks as the following codes show:

```go
func (c *Chain) Start() {
...
	go c.run()
	c.logger.Info("Started")
}
func (c *Chain) run() {
	...
	if err := c.pull(); err != nil {
		c.logger.Warnf("Pull failed, error: %s", err)
	}
}
```

```go
func (c *Chain) pull() error {
	var err error
	if c.joinBlock != nil {
		err = c.pullUpToJoin()
		if err != nil {
			return errors.WithMessage(err, "failed to pull up to join block")
		}
		c.logger.Info("Onboarding finished successfully, pulled blocks up to join-block")
	}

	if c.joinBlock != nil && !proto.Equal(c.ledgerResources.Block(c.joinBlock.Header.Number).Data, c.joinBlock.Data) {
		c.logger.Panicf("Join block (%d) we pulled mismatches block we joined with", c.joinBlock.Header.Number)
	}

	err = c.pullAfterJoin()
	if err != nil {
		return errors.WithMessage(err, "failed to pull after join block")
	}

	// Trigger creation of a new consensus.Chain.
	c.logger.Info("Block pulling finished successfully, going to switch from follower to a consensus.Chain")
	c.halt()
	c.chainCreator.SwitchFollowerToChain(c.ledgerResources.ChannelID())

	return nil
}
```

The `Chian.pull` runs in a goroutine until it encounters an error. It does the following jobs:

1. Pull the channel's blocks to the latest if the `joinBlock` the follower started with is not nil.
2. Check if pulled mismatches block
3. Pull the blocks continuously with `Chain.pullAfterjoin`
4. When pulling the blocks, we may add the current orderer to the membership of the channel, that's why `Chain.pullAfterjoin` can exit. Then we call `c.halt()` to stop the `follower`, and switch it to `chain`.

It's necessary to dig into more detail about the `pullAfterJoin`.

### pullAfterJoin

The method `pullAfterJoin` is called by `Chain` to pull blocks until the `Chain` is not a `follower`.

```go
func (c *Chain) pullAfterJoin() error {
	...
	err := c.loadLastConfig()
	if err != nil {
		return errors.WithMessage(err, "failed to load last config block")
	}

	c.blockPuller, err = c.blockPullerFactory.BlockPuller(c.lastConfig, c.stopChan)
	if err != nil {
		return errors.WithMessage(err, "error creating block puller")
	}
	defer c.blockPuller.Close()

	heightPollInterval := c.options.HeightPollMinInterval
	for {
		// Check membership
		isMember, errMem := c.clusterConsenter.IsChannelMember(c.lastConfig)
		if errMem != nil {
			return errors.WithMessage(err, "failed to determine channel membership from last config")
		}
		if isMember {
			c.setConsensusRelation(types.ConsensusRelationConsenter)
			return nil
		}

		// Poll for latest network height to advance beyond ledger height.
		var latestNetworkHeight uint64
	heightPollLoop:
		for {
			endpoint, networkHeight, errHeight := cluster.LatestHeightAndEndpoint(c.blockPuller)
			if errHeight != nil {
				c.logger.Errorf("Failed to get latest height and endpoint, error: %s", errHeight)
			} else {
				c.logger.Infof("Orderer endpoint %s has the biggest ledger height: %d", endpoint, networkHeight)
			}

			if networkHeight > c.ledgerResources.Height() {
				// On success, slowly decrease the polling interval
				c.decreaseRetryInterval(&heightPollInterval, c.options.HeightPollMinInterval)
				latestNetworkHeight = networkHeight
				break heightPollLoop
			}

			c.logger.Infof("My height: %d, latest network height: %d; going to wait %v for latest height to grow",
				c.ledgerResources.Height(), networkHeight, heightPollInterval)
			select {
			case <-c.stopChan:
				c.logger.Debug("Received a stop signal")
				return ErrChainStopped
			case <-c.timeAfter(heightPollInterval):
				// Exponential back-off, to avoid calling LatestHeightAndEndpoint too often.
				c.increaseRetryInterval(&heightPollInterval, c.options.HeightPollMaxInterval)
			}
		}

		// Pull to latest height or chain stop signal
		err = c.pullUntilLatestWithRetry(latestNetworkHeight, true)
		if err != nil {
			return err
		}
	}
}
```

1. Load the last config block to create a `BlockPuller`

2. Open a pulling loop where we do three jobs continuously:

   1. Check if the current node is a member of the channel. If it is, change the `consensusRelation` to `consenter` and break the loop.

      There are 4 consensus relations: `consenter`, `follower`, `config-tracker`, `other`(for non-cluster node).

   2. Pull for latest network height to advance beyond ledger height. Here, if the height we pulled is not beyond the local height, we block the goroutine for `heightPollInterval`, then increase the interval exponentially to avoid calling the pull height function too often:

      ```go
      func (c *Chain) increaseRetryInterval(retryInterval *time.Duration, upperLimit time.Duration) {
      	if *retryInterval == upperLimit {
      		return
      	}
      	*retryInterval = time.Duration(1.5 * float64(*retryInterval))
      	if *retryInterval > upperLimit {
      		*retryInterval = upperLimit
      	}
      }
      ```

   3. Pull the blocks to the latest height. In the method `Chain.pullUntilLatestWithRetry`, `pullUntilTarget` is called to pull blocks to a given target height. The following code shows part of it:

      ```go
      n := seq - firstBlockToPull
      nextBlock := c.blockPuller.PullBlock(seq)
      if nextBlock == nil {
        return n, errors.WithMessagef(cluster.ErrRetryCountExhausted, "failed to pull block %d", seq)
      }
      reportedPrevHash := nextBlock.Header.PreviousHash
      if (nextBlock.Header.Number > 0) && !bytes.Equal(reportedPrevHash, actualPrevHash) {
        return n, errors.Errorf("block header mismatch on sequence %d, expected %x, got %x",
          nextBlock.Header.Number, actualPrevHash, reportedPrevHash)
      }
      actualPrevHash = protoutil.BlockHeaderHash(nextBlock.Header)
      if err := c.ledgerResources.Append(nextBlock); err != nil {
        return n, errors.WithMessagef(err, "failed to append block %d to the ledger", nextBlock.Header.Number)
      }
      
      if protoutil.IsConfigBlock(nextBlock) {
        c.logger.Debugf("Pulled blocks from %d to %d, last block is config", firstBlockToPull, nextBlock.Header.Number)
        c.lastConfig = nextBlock
        if err := c.blockPullerFactory.UpdateVerifierFromConfigBlock(nextBlock); err != nil {
          return n, errors.WithMessagef(err, "failed to update verifier from last config,  block number: %d", nextBlock.Header.Number)
        }
        if updateEndpoints {
          endpoints, err := cluster.EndpointconfigFromConfigBlock(nextBlock, c.cryptoProvider)
          if err != nil {
            return n, errors.WithMessagef(err, "failed to extract endpoints from last config,  block number: %d", nextBlock.Header.Number)
          }
          c.blockPuller.UpdateEndpoints(endpoints)
        }
      }
      ```

      + If the block we just pulled is nil, return the number of pulled blocks with an error.
      + If this block's `PreviousHash` is not the same as the previously pulled block's, return. Note that, the pulling is from back to front.
      + update `actualPrevHash` to this block's hash, then write this block to the ledger.
      + If this block is a config block, then we update `c.lastConfig` to this block, and we may want to update the verifier and endpoint according to this block.

## ChainSupport.Start

The `ChainSupport.Start` finally creates a goroutine with `go c.run()`, which will block for 5 kinds of events we will talk respectively:

```go
for {
select {
  case s := <-submitC:
  ...
  case app := <-c.applyC:
  ...
  case <-timer.C():
  ...
  case sn := <-c.snapC:
  ...
  case <-c.doneC:
  ...
}
}
```

### case <-submitC

We order the endorsements in this case as discussed in the [previous post](/posts/understanding-broadcast/) if the current node is a raft leader:

```go
batches, pending, err := c.ordered(s.req)
```

In the `ordered` function, we first get the config `sequence` of our channel and determine if the message is a config message. If it is, we take the following  steps:

1. If the `sequence` carried by the message is less than the current, call `ProcessCOnfigMsg` to re-validate and reproduce config message.
2. Call `checkForEvictionNCertRotation` check for node eviction and certificate rotation by comparing the `config.Consenters` packed in the message with `Chain.opts.Consenters`. We say a node is rotated when its current certificate is not showing in the ` config.Consenters`  and evicted when it is not showing in the `config.Consenters`.
 1. A node is evicted or its certificate is rotated means that it's not capable of being a leader and we should transfer the leadership to another node.
 2. After the transfer, we forward the transaction message to the new leader.
3. Call `BlockCutter.Cut()` to package the pending transactions with the config transaction into a batch since one block should consist of only one config transaction.

If the message is a normal transaction, then we call `BlockCutter.Ordered` to add the message to the pending batch or the cutted batch according to the following rules:

1. The message's size should not be larger than the `PreferredMaxBytes`.(Note that we will get two blocks: pending message and this message)
2. Enqueuing the message into the pending batch should not cause the total bytes size larger than the `PreferredMaxBytes`.
3. Enqueuing the message into the pending batch should not cause the message count larger than `MaxMessageCount`.

If the message is pending, then we start the `timer` (we will talk about it in the case `<-timer.C()`) and then returns, otherwise, we stop timer and call `propose`:

#### Propose

```go
func (c *Chain) propose(ch chan<- *common.Block, bc *blockCreator, batches ...[]*common.Envelope) {
	for _, batch := range batches {
		b := bc.createNextBlock(batch)
		c.logger.Infof("Created block [%d], there are %d blocks in flight", b.Header.Number, c.blockInflight)

		select {
		case ch <- b:
		default:
			c.logger.Panic("Programming error: limit of in-flight blocks does not properly take effect or block is proposed by follower")
		}

		// if it is config block, then we should wait for the commit of the block
		if protoutil.IsConfigBlock(b) {
			c.configInflight = true
		}

		c.blockInflight++
	}
}
```

As the above code shows, we create a block for each batch and send the block to the channel whose buffer's size is `MaxInflightBlocks`. The other goroutine will take the block from the channel and process it. Then we add one to the counter `configInflight` if the block is a config block and one to the `blockInflight`.

#### CreateNextBlock

A `Block` comprises three components: `Header`, `Data`, and `Metadata`, which are represented by the following data structures:

```go
type BlockHeader struct {
	Number               uint64   `protobuf:"varint,1,opt,name=number,proto3" json:"number,omitempty"`
	PreviousHash         []byte   `protobuf:"bytes,2,opt,name=previous_hash,json=previousHash,proto3" json:"previous_hash,omitempty"`
	DataHash             []byte   `protobuf:"bytes,3,opt,name=data_hash,json=dataHash,proto3" json:"data_hash,omitempty"`
}
type BlockData struct {
	Data                 [][]byte `protobuf:"bytes,1,rep,name=data,proto3" json:"data,omitempty"`
}
type BlockMetadata struct {
	Metadata             [][]byte `protobuf:"bytes,1,rep,name=metadata,proto3" json:"metadata,omitempty"`
}
```

To create a block given a batch of transactions, we first marshal each of the transactions into the `[]byte` form and assemble them to the `BlockData`:

```go
data := &cb.BlockData{
  Data: make([][]byte, len(envs)),
}
for i, env := range envs {
  data.Data[i], err = proto.Marshal(env)
  ...
}
```

Then we construct a block with no data and no metadata given the block's `seqNum` and `previousHash`:

```go
block := &cb.Block{}
block.Header = &cb.BlockHeader{}
block.Header.Number = seqNum
block.Header.PreviousHash = previousHash
block.Header.DataHash = []byte{}
block.Data = &cb.BlockData{}

var metadataContents [][]byte
for i := 0; i < len(cb.BlockMetadataIndex_name); i++ {
  metadataContents = append(metadataContents, []byte{})
}
block.Metadata = &cb.BlockMetadata{Metadata: metadataContents}
```

The `BlockMetadataIndex` above indicates the metadata can be the `SIGNATURES`, `LAST_CONFIG`(deprecated due to the config block) , `TRANSACTIONS_FILTER`, `ORDERER`(deprecated), or `COMMIT_HASH`.

The header's `DataHash` can be derived from `BlockData`: `block.Header.DataHash = protoutil.BlockDataHash(data)`.

#### Leader Propose Block

When a node becomes a leader, it owns the capability to process the yield blocks by giving a channel to receive the blocks:

```go
becomeLeader := func() (chan<- *common.Block, context.CancelFunc) {
  c.blockInflight = 0
...
  ch := make(chan *common.Block, c.opts.MaxInflightBlocks)
  ctx, cancel := context.WithCancel(context.Background())
  go func(ctx context.Context, ch <-chan *common.Block) {
    for {
      select {
      case b := <-ch:
        data := protoutil.MarshalOrPanic(b)
        if err := c.Node.Propose(ctx, data); err != nil {
          c.logger.Errorf("Failed to propose block [%d] to raft and discard %d blocks in queue: %s", b.Header.Number, len(ch), err)
          return
        }
        c.logger.Debugf("Proposed block [%d] to raft consensus", b.Header.Number)

      case <-ctx.Done():
        c.logger.Debugf("Quit proposing blocks, discarded %d blocks in the queue", len(ch))
        return
      }
    }
  }(ctx, ch)
  return ch, cancel
}
```

After accepting a block, the leader proposes appending it to the blockchain using the `raft` consensus algorithm.

### case <-c.applyC

The block is sent to the other orderers or enqueued into the local raft node's unstable entries. Once the entries are committed, the `n.chain.applyC <- apply{rd.CommittedEntries, rd.SoftState}` instruction is executed. Here, the `CommittedEntries` are those blocks the majority of orderers agree to add to the blockchain. The `SoftState` describes the node status and the current leader.

We first check the `SoftState` since it determines whether the current node is the leader or not:

```go
newLeader := atomic.LoadUint64(&app.soft.Lead) // etcdraft requires atomic access
if newLeader != soft.Lead {
  c.logger.Infof("Raft leader changed: %d -> %d", soft.Lead, newLeader)
  c.Metrics.LeaderChanges.Add(1)

  atomic.StoreUint64(&c.lastKnownLeader, newLeader)

  if newLeader == c.raftID {
    propC, cancelProp = becomeLeader()
  }

  if soft.Lead == c.raftID {
    becomeFollower()
  }
}
foundLeader := soft.Lead == raft.None && newLeader != raft.None
quitCandidate := isCandidate(soft.RaftState) && !isCandidate(app.soft.RaftState)

if foundLeader || quitCandidate {
  c.errorCLock.Lock()
  c.errorC = make(chan struct{})
  c.errorCLock.Unlock()
}
```

+ When the current node is a leader, if leadership changes, it becomes a follower.
+ When the current node is a follower, if leadership changes and the new leader is itself, it becomes a follower.
+ When the current node is a candidate, but changes to another role or a leader is found, it quit electing:

We next call `apply` to write blocks.

```go
case raftpb.EntryNormal:
    if len(ents[i].Data) == 0 {
      break
    }

    position = i
    c.accDataSize += uint32(len(ents[i].Data))

    if ents[i].Index <= c.appliedIndex {
      c.logger.Debugf("Received block with raft index (%d) <= applied index (%d), skip", ents[i].Index, c.appliedIndex)
      break
    }

    block := protoutil.UnmarshalBlockOrPanic(ents[i].Data)
    c.writeBlock(block, ents[i].Index)
if ents[i].Index > c.appliedIndex {
    c.appliedIndex = ents[i].Index
  }
```

Here, we `ents[i].Index` is the entry's index in the `Storage` of the raft node, we only write those blocks whose index is not larger than `appliedIndex` to avoid writing the same block twice. After calling `writeBlock`, we update the `appliedIndex`.

```go
func (c *Chain) writeBlock(block *common.Block, index uint64) {
	if block.Header.Number > c.lastBlock.Header.Number+1 {
		c.logger.Panicf("Got block [%d], expect block [%d]", block.Header.Number, c.lastBlock.Header.Number+1)
	} else if block.Header.Number < c.lastBlock.Header.Number+1 {
		c.logger.Infof("Got block [%d], expect block [%d], this node was forced to catch up", block.Header.Number, c.lastBlock.Header.Number+1)
		return
	}

	if c.blockInflight > 0 {
		c.blockInflight-- // only reduce on leader
	}
	c.lastBlock = block

	c.logger.Infof("Writing block [%d] (Raft index: %d) to ledger", block.Header.Number, index)

	if protoutil.IsConfigBlock(block) {
		c.writeConfigBlock(block, index)
		return
	}

	c.raftMetadataLock.Lock()
	c.opts.BlockMetadata.RaftIndex = index
	m := protoutil.MarshalOrPanic(c.opts.BlockMetadata)
	c.raftMetadataLock.Unlock()

	c.support.WriteBlock(block, m)
}
```

We want the number of the block to be written is not less than the previous block's. Then we modify the `blockInflight` for the leader to avoid closing the `submitC`. For the config block, call the `writeConfigBlock`. In the end, we add the raft index to the block's metadata.

### case <- timer.C()

Remember that when the messages in the pending batch are not enough for a block, a timer starts. Every time the timer ticks, we package the messages to a block despite the number or size of the messages. And that's all it does.

### sn := <-c.snapC

The `snapC` is a signal to catch up with snapshot, it's used in the following scenes:

+ The node is evicted
+ The node is just started
+ The node's snapshot is not empty

```go
case sn := <-c.snapC:
    if sn.Metadata.Index != 0 {
      if sn.Metadata.Index <= c.appliedIndex {
        c.logger.Debugf("Skip snapshot taken at index %d, because it is behind current applied index %d", sn.Metadata.Index, c.appliedIndex)
        break
      }

      c.confState = sn.Metadata.ConfState
      c.appliedIndex = sn.Metadata.Index
    } else {
      c.logger.Infof("Received artificial snapshot to trigger catchup")
    }

    if err := c.catchUp(sn); err != nil {
      c.logger.Panicf("Failed to recover from snapshot taken at Term %d and Index %d: %s",
        sn.Metadata.Term, sn.Metadata.Index, err)
    }
```

When the index in the snapshot is large than `appliedIndex`, we should catch up with the snapshot. We can create a `BlockPuller` to pull all the blocks behind the snapshot.

### case <-c.doneC

In this case, we shut down the chain gracefully:

```go
stopTimer()
cancelProp()

select {
case <-c.errorC: // avoid closing closed channel
default:
  close(c.errorC)
}
c.periodicChecker.Stop()
```

+ Stop the timer
+ cancel proposing block
+ close error channel
+ stop the periodic checker

## Summary

In this post, we learned how transactions are broadcasted from the leader to other followers with the consensus algorithm. We should note that the key point for the ordering service to function properly is the consensus algorithm, though we haven't discussed it too much.
