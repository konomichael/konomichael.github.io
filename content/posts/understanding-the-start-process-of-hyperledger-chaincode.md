+++
title = "Understanding the Start Process of Hyperledger Chaincode"
date = "2022-10-24T22:39:40+08:00"
author = "konomichael"
authorTwitter = "@konomichael_" #do not include @
tags = ["hyperledger", "chaincode", "blockchain"]
keywords = ["hyperledger", "chaincode", "blockchain"]
readingTime = true
hideComments = false

+++

The topic at hand is `User Chaincode (UCC)`, an essential component for application developers in the realm of blockchain technology. It provides the logic necessary to process states based on a distributed ledger, enabling developers to create complex applications.

<!--more-->

In the context of Hyperledger Fabric, Chaincode is typically executed within Docker containers. Peers utilize the Docker API to create and launch Chaincode containers. Once a Chaincode container is up and running, it establishes a bi-directional `gRPC ` connection with the Peer, enabling communication via the exchange of ChaincodeMessages. To initiate requests to the Peer, the Chaincode container utilizes the interface provided by the `shim` package.

## The typical structure of chaincode

The typical structure of the chaincode is presented below. Users only need to focus on the implementation of the `Init()` and `Invoke()` functions and use the `shim.ChaincodeStubInterface` structure in them to implement the interaction logic with the ledger.

```go
type MyChaincode struct{}

func (d *MyChaincode) Init(stub shim.ChaincodeStubInterface) peer.Response {
	return shim.Success(nil)
}

func (d *MyChaincode) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
	return shim.Success(nil)
}

func main() {
	err := shim.Start(&MyChaincode{})
	if err != nil {
		panic(err)
	}
}
```

## The start process

In the `main` function we called the `shim.Start` to start a chaincode:

```go
func Start(cc Chaincode) error {
	chaincodename := os.Getenv("CORE_CHAINCODE_ID_NAME")
	if chaincodename == "" {
		return errors.New("'CORE_CHAINCODE_ID_NAME' must be set")
	}

	if streamGetter == nil {
		streamGetter = userChaincodeStreamGetter
	}

	stream, err := streamGetter(chaincodename)
	if err != nil {
		return err
	}

	err = chaincodeAsClientChat(chaincodename, stream, cc)

	return err
}
```

The function first retrieves the value of the `CORE_CHAINCODE_ID_NAME` in order to get a `stream`, the implementation of the interface  `ClientStream`:

```go
type ClientStream interface {
	PeerChaincodeStream
	CloseSend() error
}
```

where `PeerChaincodeStream` is the common stream interface for peer-chaincode communication(send & receive chaincode message ):

```go
type PeerChaincodeStream interface {
	Send(*pb.ChaincodeMessage) error
	Recv() (*pb.ChaincodeMessage, error)
}
```

The above stream interface is default implemented with `gRPC`:

```go
conn, err := internal.NewClientConn(*peerAddress, conf.TLS, conf.KaOpts)
return internal.NewRegisterClient(conn)
```

We first establish a gRPC connection to the peer given the `peerAddress`, then we create a new stream  for the client side:

```go
func NewChaincodeSupportClient(cc *grpc.ClientConn) ChaincodeSupportClient {
	return &chaincodeSupportClient{cc}
}

func (c *chaincodeSupportClient) Register(ctx context.Context, opts ...grpc.CallOption) (ChaincodeSupport_RegisterClient, error) {
	stream, err := c.cc.NewStream(ctx, &_ChaincodeSupport_serviceDesc.Streams[0], "/protos.ChaincodeSupport/Register", opts...)
	if err != nil {
		return nil, err
	}
	x := &chaincodeSupportRegisterClient{stream}
	return x, nil
}
```

So here we can say that the stream is actually a streaming rpc.

### The chaincode side handler

As what we will do next in web programming, we get conn via dialing to a server or listening to the port, we need a handler to process the data we accept from conn. We wrap the `stream` and `chaincode` into the `handler`:

```go
type Handler struct {
	// serialLock is used to prevent concurrent calls to Send on the
	// PeerChaincodeStream. This is required by gRPC.
	serialLock sync.Mutex
	// chatStream is the client used to access the chaincode support server on
	// the peer.
	chatStream PeerChaincodeStream

	// cc is the chaincode associated with this handler.
	cc Chaincode
	// state holds the current state of this handler.
	state state

	// Multiple queries (and one transaction) with different txids can be executing in parallel for this chaincode
	// responseChannels is the channel on which responses are communicated by the shim to the chaincodeStub.
	// need lock to protect chaincode from attempting
	// concurrent requests to the peer
	responseChannelsMutex sync.Mutex
	responseChannels      map[string]chan pb.ChaincodeMessage
}
```

After being initialized, the handler sends the first message to the peer to register the chaincode:

```go
if err = handler.serialSend(&peerpb.ChaincodeMessage{Type: peerpb.ChaincodeMessage_REGISTER, Payload: payload}); err != nil {
		return fmt.Errorf("error sending chaincode REGISTER: %s", err)
	}
```

Upon successful registration, the chaincode starts a message processing loop, waiting to receive messages from the peer and messages about its own state transition.

The chaincode and the peer use the `FSM(Finite State Machine)` to complete a series of response operations to messages:

+ When the peer receives a `ChaincodeMessage_REGISTER` message from the chaincode container, it registers the message to a local Handler structure and returns a `ChaincodeMessage_REGISTERED` message to the Chaincode container. The peer then updates its status to `established` and sends a `ChaincodeMessage_READY` message to the chaincode side, updating its status to `ready`.

+ The chaincode side receives the `ChaincodeMessage_REGISTERED` message and updates its status from `created` to `established`. Upon receiving the `ChaincodeMessage_READY` message, it updates its status to 'ready'.

+ The peer sends a `ChaincodeMessage_INIT` message to the Chaincode container to trigger chaincode initialization operations. 

+ When the chaincode container receives the `ChaincodeMessage_INIT` message, it initializes the required `ChaincodeStub` structure and calls the `Init()` method in the chaincode code. After successful initialization, the chaincode container sends a `ChaincodeMessage_COMPLETED` message to the peer, indicating that it is ready to be invoked.

+ When the chaincode is invoked, the peer sends a `ChaincodeMessage_TRANSACTION` message to the Chaincode.

+  Upon receiving this message, the Chaincode calls the `Invoke()` method and sends messages, and based on the logic implemented by the user in the Invoke method, it can send messages such as

  + `ChaincodeMessage_GET_HISTORY_FOR_KEY`,
  + `ChaincodeMessage_GET_QUERY_RESULT`,
  + `ChaincodeMessage_GET_STATE`,
  + `ChaincodeMessage_GET_STATE_BY_RANGE`,
  + `ChaincodeMessage_QUERY_STATE_CLOSE`,
  + `ChaincodeMessage_QUERY_STATE_NEXT`,
  + `ChaincodeMessage_INVOKE_CHAINCODE` 

  to the peer side. The peer processes these messages and responds with a `ChaincodeMessage_RESPONSE` message. Finally, the chaincode replies with a `ChaincodeMessage_COMPLETE` message to the Peer to indicate the completion of the call.

+ During this process, the peer and chaincode sides periodically send `ChaincodeMessage_KEEPALIVE` messages to each other to ensure that they remain connected.

Overall, this system enables communication between the chaincode and the peer using a set of standardized messages and a well-defined protocol, ensuring efficient and reliable interaction.

## Summary

In this post, we explored the starting process of the hyperledger chaincode side container. We learned that this process primarily involves establishing a gRPC connection (though other methods like lib-p2p can also be used) with the peer side, and then starting a message handler to process incoming messages. By understanding the chaincode startup process, we gain a deeper understanding of how hyperledger fabric works, which can help us develop more efficient and effective blockchain applications.
