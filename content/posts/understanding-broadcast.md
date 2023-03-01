+++
title = "Understanding Broadcast"
date = "2022-12-05T16:19:01+08:00"
author = "konomichael"
authorTwitter = "@konomichael_" #do not include @
tags = ["hyperledger", "broadcast", "blockchain"]
keywords = ["hyperledger", "broadcast", "blockchain"]
readingTime = true
hideComments = false

+++

In the previous post, we discussed how a proposal proposed by a client gets endorsed by other endorsers. When the client has collected enough endorsements for the proposal, it then needs to package these endorsements and send a message to the order service in order to broadcast the transaction wrapped in the proposal. In this post, we will first learn how endorsements are packaged into an `Envelope` that can be sent to order service later. Then we will inspect the important role `orderer` and figure out how it runs.

<!--more-->

## Create an envelope

The `Envelope` wraps a payload with a signature so that the message may be authenticated:

```go
type Envelope struct {
	// A marshaled Payload
	Payload []byte `protobuf:"bytes,1,opt,name=payload,proto3" json:"payload,omitempty"`
	// A signature by the creator specified in the Payload header
	Signature            []byte   `protobuf:"bytes,2,opt,name=signature,proto3" json:"signature,omitempty"`
}
```

We must wrap our endorsements, the original proposal, and the signer into the `Payload` above. The whole process can be concluded with the following diagram:

```go
                    ┌───────────┐
                ┌───►Signature  ├──┐   ┌────────────────┐
                │   └───────────┘  │   │                │
                │                  ├───►    Envelope    │
┌───────────┐   │   ┌───────────┐  │   │                │
│   signer  ├───┴───┤  Bytes    ├──┘   └────────────────┘
└───────────┘ Sign  └─────▲─────┘
                          │
                    ┌─────┴─────┐      ┌────────────────┐   ┌─────────────────┐
                    │  Payload  ◄──────┤TransactionByte ◄───┤   Transaction   │
                    └─────▲─────┘      └────────────────┘   └────────▲────────┘
                          │                                          │
                          │                                          │
                          │                                          │
                    ┌─────┴─────┐       ┌───────────────┐    ┌───────┴─────────┐
                  ┌─►  Header   ├───────►SignatureHeader├────►TransactionAction│
                  │ └───────────┘       └───────────────┘    └───────▲─────────┘
  ┌───────────┐   │                                                  │
  │  proposal ├───┤                                          ┌───────┴─────────┐
  └───────────┘   │                                          │     Bytes       │
                  │ ┌───────────┐        ┌──────────────┐    └───────▲─────────┘
                  └─►  Payload  ├────────► PayloadBytes ├─┐          │
                    └───────────┘        └──────────────┘ │  ┌───────┴─────────┐
                                                          ├──►ActionPayload    │
                    ┌───────────┐        ┌──────────────┐ │  └─────────────────┘
                  ┌─►  Payload  ├────────►EndorsedAction├─┘
                  │ └───────────┘        └──────▲───────┘
  ┌───────────┐   │                             │
  │endorsement├───┤                             │
  └───────────┘   │                             │
                  │ ┌───────────┐               │
                  └─►Endorsement├───┐           │
                    └───────────┘   │    ┌──────┴───────┐
                                    ├────► Endorsements │
        ...         ┌───────────┐   │    └──────────────┘
                  ┌─►Endorsement├───┘
                  │ └───────────┘
  ┌───────────┐   │
  │endorsement├───┤
  └───────────┘   │
                  │ ┌───────────┐
                  └─►  Payload  │
                    └───────────┘
```

Don't get panic about the above diagram. We first extract the `Endorsement` and `Payload` from all the endorsements the client receives. Of course, we should ensure that all the payloads are the same. Note that the `Payload` comprises `ProposalHash`, `Events`, `Results,` and `ChaincodeID`, which describe how a proposal is executed on a channel, and the endorsement gives the endorser and signature. With the `EndorsedAction`, `PayloadBytes` from the original proposal, and the `SignatureHeader` extracted from the original proposal's header, we can get the final `Transaction`, which will be sent to the order service. Still, before that, we need to do the sign trick again for this `Transaction`.

## Broadcast

The client first calls the function `GetBroadcastClientFunc` to get a broadcast client:

1. Create an instance of an `OrderClient` containing `orderers' addresses` and `clientConfig` from the environment
2. Create a gRPC connection object `conn` for each of the orderers' addresses, with which create an `atomicBroadcastClient`
3. Invoke the `actomicBroadcastClient`'s method `Broadcast` to build a new stream on the method `/orderer.AtomicBroadcast/Broadcast`,  yields the final working client `atomicBroadcastBroadcastClient`

A `BroadcastOrdererClient` owns multiple such `atomicBroadcastBroadcastClient` which guarantees we can send the transaction to multiple `orderer`s.

When a signed transaction is broadcasted to an orderer, the orderer immediately invokes its broadcast handler that was created when the server started to process it:

```go
func (bh *Handler) Handle(srv ab.AtomicBroadcast_BroadcastServer) error {
	...
	for {
		msg, err := srv.Recv()
		...
		resp := bh.ProcessMessage(msg, addr)
		err = srv.Send(resp)
		...
	}
}
```

To handle the transaction, the handler opens a `broadcast loop` since there may be multiple rounds of message passing between the orderer and the client. Here we just focus on the key method `ProcessMessage`:

```go
func (bh *Handler) ProcessMessage(msg *cb.Envelope, addr string) (resp *ab.BroadcastResponse) {
	...
	chdr, isConfig, processor, err := bh.SupportRegistrar.BroadcastChannelSupport(msg)
	...
	if !isConfig {
		configSeq, err := processor.ProcessNormalMsg(msg)
		...
		if err = processor.WaitReady(); err != nil {
			logger.Warningf("[channel: %s] Rejecting broadcast of message from %s with SERVICE_UNAVAILABLE: rejected by Consenter: %s", chdr.ChannelId, addr, err)
			return &ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()}
		}

		err = processor.Order(msg, configSeq)
		if err != nil {
			logger.Warningf("[channel: %s] Rejecting broadcast of normal message from %s with SERVICE_UNAVAILABLE: rejected by Order: %s", chdr.ChannelId, addr, err)
			return &ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()}
		}
	} else { // isConfig
		config, configSeq, err := processor.ProcessConfigUpdateMsg(msg)
	  ...
		if err = processor.WaitReady(); err != nil {
			logger.Warningf("[channel: %s] Rejecting broadcast of message from %s with SERVICE_UNAVAILABLE: rejected by Consenter: %s", chdr.ChannelId, addr, err)
			return &ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()}
		}

		err = processor.Configure(config, configSeq)
		if err != nil {
			logger.Warningf("[channel: %s] Rejecting broadcast of config message from %s with SERVICE_UNAVAILABLE: rejected by Configure: %s", chdr.ChannelId, addr, err)
			return &ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()}
		}
	}

	...

	return &ab.BroadcastResponse{Status: cb.Status_SUCCESS}
}
```

It contains several functions we should care about:

+  `BroadcastChannelSupport` parses the message and gives the `ChannelHeader`, `ChainSupport` which can deal with a `channel` according to the `channel`, and `IsConfig` which shows whether the message is a config update and the channel resources.
+ `processor.WaitReady` blocks waiting for the consenter to be ready to accept new messages.
+ `processor.ProcessNormalMsg` process normal transaction message
+ `processor.ProcessConfigUpdateMsg` process config transaction message
+ `processor.Order` submits the transaction message consisting of `LastValidationSeq`, `Payload,` and `ChannelID` for ordering. The message goes to the local run goroutine if the current orderer is a leader, otherwise forwarded to the actual leader via RPC. This method finally invokes `Chain.Submit` .
+ `processor.Configure` submits the config transaction message for ordering, which works exactly the same way with `processor.Order`.

Now, let's take a closer look at the `ProcessNormalMsg`, `ProcessConfigUpdateMsg,` and `Chain.Submit`.

### ProcessNoramlMsg

The `ProcessNormalMsg` is declared as a method of `StandardChannel`:

```go
func (s *StandardChannel) ProcessNormalMsg(env *cb.Envelope) (configSeq uint64, err error) {
	oc, ok := s.support.OrdererConfig()
	if !ok {
		logger.Panicf("Missing orderer config")
	}
	if oc.Capabilities().ConsensusTypeMigration() {
		if oc.ConsensusState() != orderer.ConsensusType_STATE_NORMAL {
			return 0, errors.WithMessage(
				ErrMaintenanceMode, "normal transactions are rejected")
		}
	}

	configSeq = s.support.Sequence()
	err = s.filters.Apply(env)
	return
}
```

The first thing it does is to gain the `config Sequence`. In a Hyperledger Fabric network, the channel configuration includes information about the members, policies, etc. When a channel configuration is updated, a new configuration is created with a new sequence number that is incremented by one from the previous configuration sequence number. This allows the network to keep track of the latest channel configuration and ensure that all participating peers are on the same page regarding the channel state.

The Next thing to do is filter the message using the `filters.Apply`. To be filtered means the message should obey the filter's rule such as `MaxBytesRule`, `expirationRejectRule`, `EmptyRejectRule`, `SigFilter`, etc. The filters are created at the start of the orderer server, with the initialization of a `registrar`.

### ProcessConfigUpdateMsg

The `ProcessConfigUpdateMsg` is also declared in the `StandardChannel`:

```go
func (s *StandardChannel) ProcessConfigUpdateMsg(env *cb.Envelope) (config *cb.Envelope, configSeq uint64, err error) {
	...
	seq := s.support.Sequence()
	err = s.filters.Apply(env)
	if err != nil {
		return nil, 0, errors.WithMessage(err, "config update for existing channel did not pass initial checks")
	}

	configEnvelope, err := s.support.ProposeConfigUpdate(env)
	if err != nil {
		return nil, 0, errors.WithMessagef(err, "error applying config update to existing channel '%s'", s.support.ChannelID())
	}

	config, err = protoutil.CreateSignedEnvelope(cb.HeaderType_CONFIG, s.support.ChannelID(), s.support.Signer(), configEnvelope, msgVersion, epoch)
	if err != nil {
		return nil, 0, err
	}

	err = s.filters.Apply(config)
	if err != nil {
		return nil, 0, errors.WithMessage(err, "config update for existing channel did not pass final checks")
	}

	err = s.maintenanceFilter.Apply(config)
	if err != nil {
		return nil, 0, errors.WithMessage(err, "config update for existing channel did not pass maintenance checks")
	}

	return config, seq, nil
}
```

It first gets the sequence and filters the message just like `ProcessNormalMsg`, then it takes in the envelope of the `CONFIG_UPDATE` and produces a `ConfigEnvelope`:

```go
return &cb.ConfigEnvelope{
  Config: &cb.Config{
    Sequence:     vi.sequence + 1,
    ChannelGroup: channelGroup,
  },
  LastUpdate: configtx,
}, nil
```

The `configtx` above is the `env` we passed to `s.support.ProposeConfigUpdate`. This new envelope is later wrapped in the `HeaderType_CONFIG` and signed to orderer.

### Submit

The `Submit` is a method of `Chain`, which defines a way to inject messages for ordering, however, it is not the `blockchain`.

```go
func (c *Chain) Submit(req *orderer.SubmitRequest, sender uint64) error {
	...
	leadC := make(chan uint64, 1)
	select {
	case c.submitC <- &submit{req, leadC}:
		lead := <-leadC
		if lead == raft.None {
			c.Metrics.ProposalFailures.Add(1)
			return errors.Errorf("no Raft leader")
		}

		if lead != c.raftID {
			if err := c.forwardToLeader(lead, req); err != nil {
				return err
			}
		}

...
	}

	return nil
}
```

It sends the request to go channel `submitC` and receives the current leader in the raft network.

+ When `lead==raft.None`, the leader is not elected yet, and the consensus service is unavailable.
+ When `lead!=c.raftID`, the current orderer is not the leader, so it just forwards the message to the leader instead of processing it.
+ When `lead==c.raftID`, the current orderer can process the message, which we will discuss in the next post.

## Summary

In this post, we explored how the client wraps the endorsements to request then sends it to the orderer, and the orderer validates the message and then processes or forwards it. Still, we haven't stepped into the core domain of the orderer. Be patient, we will talk about it in the next post.
