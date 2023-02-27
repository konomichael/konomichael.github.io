+++
title = "Understanding the Start Process of Hyperledger Peer"
date = "2022-11-05T19:09:20+08:00"
author = "konomichael"
authorTwitter = "@konomichael_" #do not include @
tags = ["hyperledger", "peer", "blockchain"]
keywords = ["hyperledger", "peer", "blockchain"]
readingTime = true
hideComments = false

+++

In the last post, we explored the start process of the chaincode side container, which is simple: starting a gRPC stream and handling the message. However, the peer node's start process is somehow complicated, which involves more components and data structures.

<!--more-->

## Launch a peer node

It's easy to launch a  peer node using the command: `peer node start`, which will finally invoke the function `serve`. The `serve` function, however, is not well readable and contains lots of start processes of different servers and data structures:

+ System Server
+ LedgerMgr
+ gRPC Server
+ Gossip Server
+ Chaincode Support Server
+ Deliver Server
+ Chaincode Custodian Server
+ Snapshot Server
+ Gateway Server

## System Server

The `System Server` is a simple HTTP server, initialized in the `serve` function:

```go
opsSystem := newOperationsSystem(coreConfig)
err = opsSystem.Start()
```

As the function name `newOperationsSystem` indicates, the `System server` works as a system monitor, which provides
+ Health Check Handler, which is used to check the health status of the peer node.
+ Logging Spec Handler, which is used to change the log level of the peer node.
+ Metrics Provider, can be implemented by different providers, such as `Prometheus` and `StatsD`, via which we can get lots of metrics of the peer node.
+ Version Info Handler, which is used to get the version information of the peer node.

## LedgerMgr

A `LedgerMgr` is a data structure that manages the lifecycle of the ledger, it's data structure is defined as follows:

```go
type LedgerMgr struct {
    creationLock         sync.Mutex
	joinBySnapshotStatus *pb.JoinBySnapshotStatus

	lock           sync.Mutex
	openedLedgers  map[string]ledger.PeerLedger
	ledgerProvider ledger.PeerLedgerProvider

	ebMetadataProvider MetadataProvider
}
```

In the `serve` function, the `LedgerMgr` is initialized as follows:

```go
peerInstance.LedgerMgr = ledgermgmt.NewLedgerMgr(
		&ledgermgmt.Initializer{
			...
		},
	)
```

Here, we pass an `initializer` to the `NewLedgerMgr` function, which will invoke `kv.NewProvider` to take a series of initialization steps:

+ initialize ledger ID Store to create a database for ledger IDs
+ initialize block store to create a database for blocks
+ initialize private data store to create a database for private data
+ initialize history DB to create a database store for state history
+ initialize config history manager to create a database store for config history
+ initialize collElgNotifier to listen to the chaincode events
+ initialize state listeners which include `collElgNotifier` and `configHistoryMgr`
+ initialize state DB to create a database store for the state
+ initialize ledger statistics to statistics the ledger block processing time, the block storage commit time, the state DB commit time and the transaction count
+ scans for and deletes any ledger with a status of `UNDER_CONSTRUCTION` or `UNDER_DELETION`
+ initialize snapshot directory to create a directory for the snapshot

## gRPC Server

The `gRPC Server` here is a common server that can be wrapped by different services, such as `DeliverServer`, `SnapshotServer`. The `gRPC Server` is initialized in the `serve` function as follows:

```go
peerServer, err := comm.NewGRPCServer(listenAddr, serverConfig)
```

The server config contains the options such as `MaxRecvMsgSize`, `MaxSendMsgSize`, `StreamInterceptors`, `UnaryInterceptors`, etc.

## Gossip Server

The `Gossip Server` of a peer node is used to gossip the blocks and transactions to other peer nodes which based on the `gossip` protocol. The `Gossip Server` is initialized by `initGossipService` function, which does the following steps:

+ determine if TLS is enabled, if so, load and store the server's and client's TLS certificates
+ invoke `NewMCS` to create a message crypto service, which is used to authenticate the identity of the peer node and verify the signature of the message
+ invoke `NewSecurityAdvisor` to create a security advisor, which is an external auxiliary object that provides security and identity-related capabilities
+ Wrap the `gossipService` with `gPRC Server` and the other components, such as `MCS`, `SecurityAdvisor`, etc.

## Chaincode Support Server

The `Chaincode Support Server` is used to handle chaincode-related requests, such as `Install`, `Invoke`, `Query`, etc.

```go
ca, err := tlsgen.NewCA()
ccSrv, ccEndpoint, err := createChaincodeServer(coreConfig, ca, peerHost)
chaincodeSupport := &chaincode.ChaincodeSupport{
  ...
}
ccSupSrv := pb.ChaincodeSupportServer(chaincodeSupport)
pb.RegisterChaincodeSupportServer(ccSrv.Server(), ccSupSrv)
go ccSrv.Start()
```

As the above code shows, the `ChaincodeSupportServer` starts with the steps:

+ Invoke `NewCA` to create a self-signed CA for chaincode service
+ Invoke `createChaincodeServer` to create a listener(which is a gRPC server created with `comm.NewGRPCServer`) using `peer.chaincodeListneAddress`
+ cast the `chaincodeSupport` to `pb.ChiancodeSupportServer` interface which must own the method (or API) `Register(ChaincodeSupport_RegisterServer) error`
+ Register the casted server API(`ccSupSrv`) to the gRPC server(`ccSrv.Server()`)
+ Start gRPC server

## Deliver Server

The `Deliver server` is primarily used for the delivery and filter of the blocks:

```go
type DeliverServer interface {
	Deliver(Deliver_DeliverServer) error
	DeliverFiltered(Deliver_DeliverFilteredServer) error
	DeliverWithPrivateData(Deliver_DeliverWithPrivateDataServer) error
}
```

As same as the `Chaincode support Server`, we first create the instance of `Deliver Server`, then register it to the gRPC server (the peer server, as mentioned above):

```go
mutualTLS := serverConfig.SecOpts.UseTLS && serverConfig.SecOpts.RequireClientCert
policyCheckerProvider := func(resourceName string) deliver.PolicyCheckerFunc {
  return func(env *cb.Envelope, channelID string) error {
    return aclProvider.CheckACL(resourceName, channelID, env)
  }
}

metrics := deliver.NewMetrics(metricsProvider)
abServer := &peer.DeliverServer{
  DeliverHandler: deliver.NewHandler(
    &peer.DeliverChainManager{Peer: peerInstance},
    coreConfig.AuthenticationTimeWindow,
    mutualTLS,
    metrics,
    false,
  ),
  PolicyCheckerProvider: policyCheckerProvider,
}
pb.RegisterDeliverServer(peerServer.Server(), abServer)
```

The steps are shown from the above:

+ Acquire the resource check `policyCheckerProvider`, which invokes `CheckACL` to check ACL (Access Control Lists) for the given channel(`channelID`) using the `env(the 1st parameter)`. As for the resources to be checked, the deliver focuses on the `Event_FilteredBlock` and `Event_Block`
+ Create a handler to handle the events, which will parse the event, check the ACL, then delivery the blocks.
+ Register the `abServer` to the gRPC server.

## Deploy System Chaincode

> The System chaincodes are specialized chaincodes that run as part of the peer process as opposed to user chaincodes that run in separate docker containers. Examples of System Chaincodes include QSCC (Query System Chaincode) for ledger and other Fabric-related queries, CSCC (Configuration System Chaincode) which helps regulate access control, and LSCC (Lifecycle System Chaincode). Unlike a user chaincode, a system chaincode is not installed and instantiated using proposals from SDKs or CLI. It is registered and deployed by the peer at start-up.
>
> https://hyperledger-fabric.readthedocs.io/en/release-1.3/systemchaincode.html

The following code shows how the system chaincodes are deployed:

```go
// deploy system chaincodes
for _, cc := range []scc.SelfDescribingSysCC{lsccInst, csccInst, qsccInst, lifecycleSCC} {
  if enabled, ok := chaincodeConfig.SCCAllowlist[cc.Name()]; !ok || !enabled {
    logger.Infof("not deploying chaincode %s as it is not enabled", cc.Name())
    continue
  }
  scc.DeploySysCC(cc, chaincodeSupport)
}
```

The function `DeploySysCC` is finally invoked, in which the chaincode starts as we talked about [before](posts/understanding-the-start-process-of-hyperledger-chaincode/). The difference is `shim.StartInProc` is invoked instead of `shim.Start`.

## Chaincode Custodian Server

The `Chaincode Custodian` is responsible for enqueuing builds and launches of chaincodes as they become available and stops when chaincodes are no longer referenced by an active chaincode definition.

The custodian follows the `Producer-Consumer` pattern, where the requests to manage the chaincode lifecycle are added to a queue (by the Producer), and the Consumer (Custodian) retrieves and processes them.

## Snapshot Server

The `Snapshot` is introduced in `Fabric v2.3`, which gives us an alternative way to join new peers into a fabric network. The details can be found [here](https://hyperledger-fabric.readthedocs.io/en/latest/peer_ledger_snapshot.html). Just like the `Deliver`, we just need to create a snapshot server instance, and attach it to the peer gRPC server:

```go
snapshotSvc := &snapshotgrpc.SnapshotService{LedgerGetter: peerInstance, ACLProvider: aclProvider}
pb.RegisterSnapshotServer(peerServer.Server(), snapshotSvc)
```

## Gateway Server

The `Gateway` is introduced in `Fabric v2.4`, which provides a simplified, minimal API for submitting transactions to a Fabric network:

```go
type GatewayServer interface {
	Endorse(context.Context, *EndorseRequest) (*EndorseResponse, error)
	Submit(context.Context, *SubmitRequest) (*SubmitResponse, error)
	CommitStatus(context.Context, *SignedCommitStatusRequest) (*CommitStatusResponse, error)
	Evaluate(context.Context, *EvaluateRequest) (*EvaluateResponse, error)
	ChaincodeEvents(*SignedChaincodeEventsRequest, Gateway_ChaincodeEventsServer) error
}
```

The details can be found [here](https://hyperledger-fabric.readthedocs.io/en/latest/gateway.html). Also, the gateway server is registered to the peer gRPC server in the same way same as the snapshot and deliver. 

## Conclusion

In conclusion, launching a Hyperledger Fabric peer node involves initializing multiple servers and data structures such as the System Server, LedgerMgr, gRPC Server, Gossip Server, Chaincode Support Server, Deliver Server, Chaincode Custodian Server, Snapshot Server, Gateway Server, etc. While this post provides a high-level overview of these components, much more detail is required to truly understand how they work together to form a robust and secure blockchain network. In future discussions, we will delve deeper into each of these elements and explore their functions and interactions.