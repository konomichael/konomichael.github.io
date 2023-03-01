+++
title = "Understanding the Endorser process proposal"
date = "2022-11-25T19:09:20+08:00"
author = "konomichael"
authorTwitter = "@konomichael_" #do not include @
tags = ["hyperledger", "endorser", "blockchain"]
keywords = ["hyperledger", "endorser", "blockchain"]
readingTime = true
hideComments = false

+++

In the previous posts, we learned how the peer and chaincode start and knew there're multiple roles in a hyperledger fabric network. In this post, we will have new insights into the endorser. But before we do that, let's have a global view of how the fabric run.

<!--more-->

## The proposal, the endorser, and the order

A proposal in Hyperledger Fabric is a transaction request sent to the network for processing. It contains all the necessary information related to the transaction, including the sender's identity, the receiver's identity, the amount of the transaction, and any other relevant data. When a proposal is submitted, it is first verified by the endorsing peers to ensure that it is valid and meets the required criteria. Once the proposal is validated, it is sent to the ordering service for further processing.

```go
type Proposal struct {
	Header    []byte `protobuf:"bytes,1,opt,name=header,proto3" json:"header,omitempty"`
	Payload   []byte `protobuf:"bytes,2,opt,name=payload,proto3" json:"payload,omitempty"`
	Extension []byte   `protobuf:"bytes,3,opt,name=extension,proto3" json:"extension,omitempty"`
}
```

The endorser in Hyperledger Fabric is responsible for endorsing the validity of a transaction proposal. Endorsers are selected by the client based on the network's endorsement policy, which specifies the required number of endorsements required to execute a transaction. Endorsers examine the proposal and determine whether it meets the requirements specified by the endorsement policy. If the proposal meets the endorsement policy criteria, the endorser digitally signs the proposal and sends it back to the client for further processing.

The order in Hyperledger Fabric is responsible for processing and validating endorsed transactions. The ordering service is a separate component in the network that ensures that all transactions are executed in a consistent and sequential order. The order service takes endorsed transactions from the client and creates a block containing the transactions in the correct order. Once the block is created, it is broadcasted to all the nodes in the network for validation and verification. Once the block is validated, it is added to the blockchain, and the transaction is considered complete.

```text
       Execution Phase             │   Ordering Phase  │    Validation Phase
                                   │                   │
┌─────────────┐                    │                   │      ┌─────────────┐
│   Peer 1    │     1              │                   │      │   Peer 1    │
│ (Endorser)  ◄────────────┐       │                   │ ┌────► (Endorser)  │
└─┬───────────┘            │       │                   │ │    │   6, 7, 8   │
  │        2           ┌───┴───┐   │   ┌───────────┐   │ │    └─────────────┘
  └────────────────────►       │ 3 │   │  Ordering │ 5 │ │
                       │ client├───┼──►│           ├───┼─┤
  ┌────────────────────►       │   │   │  Service  │   │ │    ┌─────────────┐
  │        2           └───┬───┘   │   └───────────┘   │ │    │   Peer 2    │
┌─┴───────────┐            │       │        4          │ ├────► (Endorser)  │
│   Peer 2    ◄────────────┘       │                   │ │    │   6, 7, 8   │
│ (Endorser)  │     1              │                   │ │    └─────────────┘
└─────────────┘                    │                   │ │
                                   │                   │ │
                                   │                   │ │    ┌─────────────┐
┌─────────────┐                    │                   │ │    │   Peer 3    │
│   Peer 3    │                    │                   │ └────►             │
│ (Endorser)  │                    │                   │      │   6, 7, 8   │
└─────────────┘                    │                   │      └─────────────┘

 1: Send transaction for endorsement
 2: Transaction with endorser signature and read/write set
 3: Transaction with endorser response
 4: Transactions packed in blocks
 5: Block of transactions
 6: VSCC & MVCC validation
 7: World state updated
 8: Block appended to ledger
```

## The Channel

A Hyperledger Fabric `channel` is a private “subnet” of communication between two or more specific network members, for the purpose of conducting private and confidential transactions. A channel is defined by members (organizations), anchor peers per member, the shared ledger, chaincode application(s) and the ordering service node(s). Each transaction on the network is executed on a channel, where each party must be authenticated and authorized to transact on that channel. Each peer that joins a channel, has its own identity given by a membership services provider (MSP), which authenticates each peer to its channel peers and services.

## Process the proposal 

As the above diagram shows, processing the proposal happens in the 2nd step, which invokes `Endorser.ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error)` and do the following jobs:

1. Validate the proposal
2. Simulate the execution of the proposal
3. Sign the proposal and read/write set, then return the response

### Unpack and Validate the Proposal

Since the proposal is serialized to bytes and signed, we need first unpack the proposal and verify the certification and signature. The `proto.Unmarshal` is used to deserialize the message into the `UnpackedProposal`:

```go
type UnpackedProposal struct {
	ChaincodeName   string
	ChannelHeader   *common.ChannelHeader
	Input           *peer.ChaincodeInput
	Proposal        *peer.Proposal
	SignatureHeader *common.SignatureHeader
	SignedProposal  *peer.SignedProposal
	ProposalHash    []byte
}
```

It's worth mentioning that the `proposalHash` is calculated with `sha256`:

```go
propHash := sha256.New()
propHash.Write(hdr.ChannelHeader)
propHash.Write(hdr.SignatureHeader)
propHash.Write(ppBytes)
```

Since the transaction is preprocessed on the channel, we have to find the channel first：

```go
var channel *Channel
if up.ChannelID() != "" {
  channel = e.ChannelFetcher.Channel(up.ChannelID())
  if channel == nil {
    return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: fmt.Sprintf("channel '%s' not found", up.ChannelHeader.ChannelId)}}, nil
  }
} else {
  channel = &Channel{
    IdentityDeserializer: e.LocalMSP,
  }
}
```

As the code shows, if the `channelID` is provided in the channel header, then we use it to find the channel, else we use the local membership manager to create a default channel.

The validation is done in the method `Endorser.preProcess(up *UnpackedProposal, channel *Channel) error`:

1. Validate the channel header's type and epoch
2. Validate the signature header's nonce and creator, then compare the `TxID` with `Sha256(nonce+creator)`
3. Find the creator by step 2's creator in the channel then validate it.
   1. Validate the creator's certificate chain
   2. Validate  the creator's organization unit
4. Verify signature with Cryptography algorithms
5. Find the local transaction by `TxId` provided in the channel header to avoid replay attacks.
6. Check if the chaincode is a system chaincode(`CSCC`, `LSCC`, `QSCC`), if not,  ensure the proposal complies with the channel's writers.

### Simulate the proposal

As the word `simulate` implies, this part will not change the ledger. To do the simulation, We first need `TxSimulator` and `HistoryQueryExecutor`. (Note that `txSim` acquires a shared lock(read lock) on the `stateDB`, which will impact the block commits.) Then we call `Endorser.simulateProposal` to get the result:

1. invoke `callChaincode` to call specified chaincode, which will finally invoke the `ChaincodeSupport`'s method `Invoke`:

   ```go
   func (cs *ChaincodeSupport) Invoke(txParams *ccprovider.TransactionParams, chaincodeName string, input *pb.ChaincodeInput) (*pb.ChaincodeMessage, error) {
   	ccid, cctype, err := cs.CheckInvocation(txParams, chaincodeName, input)
   	if err != nil {
   		return nil, errors.WithMessage(err, "invalid invocation")
   	}
   
   	h, err := cs.Launch(ccid)
   	if err != nil {
   		return nil, err
   	}
   
   	return cs.execute(cctype, txParams, chaincodeName, input, h)
   }
   ```

   1. We will first check the invocation and determines if, how, and to where that invocation should be routed. The function `CheckInvocation` returns the `chancodeID`, `chaincodeType`, and error. If the chaincode requires initialization, the `chaincodeType` will be `ChaincodeMessage_INIT`, otherwise `ChaincodeMessage_TRANSACTION`. Remember that we have talked about the `chaincodeType` in the [previous post](/posts/understanding-the-start-process-of-hyperledger-chaincode/).
   2. We then call `Launch()` to create and launch chaincode runtime, and finally call `dockerClient.CreateContainer` to create a docker container.
   3. Once the container is created successfully, we call `execute` to run the chaincode. To do that, we first construct the `chaincodeMessage` with the `chaincodeType`, `payload`(which is the `input` we unmarshaled from the proposal), `txID`, `channelID` and then send it to chaincode via gRPC.

2. For operations deploying or upgrading user chaincode through LSCC, it is necessary to use the `Execute()` method again to initialize or upgrade the user chaincode instance.

The whole simulation may involve multiple rounds of gRPC communication between the endorser and the chaincode and these communications may want to change the ledger, as we said before, the ledger should not be modified during the simulation. So We put the mediate result to the simulator's `rwsetBuilder`:

```go
// SetState implements method in interface `ledger.TxSimulator`
func (s *txSimulator) SetState(ns string, key string, value []byte) error {
	if err := s.checkWritePrecondition(key, value); err != nil {
		return err
	}
	s.rwsetBuilder.AddToWriteSet(ns, key, value)
	// if this has a key level signature policy, add it to the interest
	return s.checkStateMetadata(ns, key)
}
```

3. Once the simulation is done, we can get the results via `GetTxSimulationResults`.
4. With the simulation results, we can build the `chaincodeInterest` that the client can pass to the discovery service to get the correct endorsement policy for the chaincode.

### Sign the endorsement

We have gotten the simulation result, the chaincode's response, the chaincode event, and chaincodeID, we need to wrap them into an endorsement response and signature. A `ProposalResonsePayload` is shown below:

```go
type ProposalResponsePayload struct {
	ProposalHash []byte `protobuf:"bytes,1,opt,name=proposal_hash,json=proposalHash,proto3" json:"proposal_hash,omitempty"`
	Extension            []byte   `protobuf:"bytes,2,opt,name=extension,proto3" json:"extension,omitempty"`
}
```

The field `Extension` is where we need to put the things in:

```go
cAct := &peer.ChaincodeAction{
  Events: event,
  Results: result,
  Response: response,
  ChaincodeId: ccid,
}
cActBytes, err := proto.Marshal(cAct)
```

After that, we use `EndorsementPlugin` to sign the payload:

```go
func (e *DefaultEndorsement) Endorse(prpBytes []byte, sp *peer.SignedProposal) (*peer.Endorsement, []byte, error) {
	signer, err := e.SigningIdentityForRequest(sp)
	if err != nil {
		return nil, nil, errors.Wrap(err, "failed fetching signing identity")
	}
	// serialize the signing identity
	identityBytes, err := signer.Serialize()
	if err != nil {
		return nil, nil, errors.Wrapf(err, "could not serialize the signing identity")
	}

	// sign the concatenation of the proposal response and the serialized endorser identity with this endorser's key
	signature, err := signer.Sign(append(prpBytes, identityBytes...))
	if err != nil {
		return nil, nil, errors.Wrapf(err, "could not sign the proposal response payload")
	}
	endorsement := &peer.Endorsement{Signature: signature, Endorser: identityBytes}
	return endorsement, prpBytes, nil
}
```

It's clear the endorsement is composed of two parts: the signature of `ProposalResponsePayload+EndorserIdentity` and the `EndorserIdentity`.

## The ProposalResponse

The final `ProposalResponse` returned to the client is shown below:

```go
type ProposalResponse struct {
	// Version indicates message protocol version
	Version int32 `protobuf:"varint,1,opt,name=version,proto3" json:"version,omitempty"`
	// Timestamp is the time that the message
	// was created as  defined by the sender
	Timestamp *timestamppb.Timestamp `protobuf:"bytes,2,opt,name=timestamp,proto3" json:"timestamp,omitempty"`
	// A response message indicating whether the
	// endorsement of the action was successful
	Response *Response `protobuf:"bytes,4,opt,name=response,proto3" json:"response,omitempty"`
	// The payload of response. It is the bytes of ProposalResponsePayload
	Payload []byte `protobuf:"bytes,5,opt,name=payload,proto3" json:"payload,omitempty"`
	// The endorsement of the proposal, basically
	// the endorser's signature over the payload
	Endorsement *Endorsement `protobuf:"bytes,6,opt,name=endorsement,proto3" json:"endorsement,omitempty"`
	// The chaincode interest derived from simulating the proposal.
	Interest             *ChaincodeInterest `protobuf:"bytes,7,opt,name=interest,proto3" json:"interest,omitempty"`
}
```
