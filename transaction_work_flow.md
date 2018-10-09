# 0 概述

以`fabric-sdk-go`的使用为例，基于`1.1.0`版本的fabric代码，对社区文档的交易流程[Transaction Flow](https://hyperledger-fabric.readthedocs.io/en/release-1.2/txflow.html?highlight=transaction%20flow)进行分析，理解其工作原理。

# 1 客户端发送背书请求

```go
//NewExecuteHandler returns query handler with EndorseTxHandler, EndorsementValidationHandler & CommitTxHandler Chained
func NewExecuteHandler(next ...Handler) Handler {
    //选择背书节点，并像这些节点发送请求获取背书响应
	return NewSelectAndEndorseHandler(
        //验证所有的背书响应
		NewEndorsementValidationHandler(
            //验证背书响应的签名               提交交易到orderer
			NewSignatureValidationHandler(NewCommitHandler(next...)),
		),
	)
}
```

> sdk采用职责链模式，设计整个交互流程，阅读还比较清晰

## 选择背书节点

> 不同的sdk实现逻辑不同，可以手动通过调用配置指定背书节点，也可以通过背书策略，由sdk或者fabric的discovery service生成背书节点

`fabric-sdk-go`的选择逻辑

- 如果指定了调用目标节点，则直接使用

- auto：根据通道的Capability配置选择，如果是1.1，使用static策略，如果是1.2使用dynamic策略
- static：使用配置文件中的所有背书节点
- dynamic：调用`lscc`中的`getccdata`获取到合约的背书策略后，由sdk根据背书策略，在配置的背书节点中选择符合条件的最小集合。
- fabric：使用service discovery新特性，查询到合约背书需要的最小节点集合

```go
func getEndorsers(requestContext *RequestContext, clientContext *ClientContext, opts ...options.Opt) ([]*fab.ChaincodeCall, []fab.Peer, error) {
	var selectionOpts []options.Opt
	selectionOpts = append(selectionOpts, opts...)
	if requestContext.SelectionFilter != nil {
		selectionOpts = append(selectionOpts, selectopts.WithPeerFilter(requestContext.SelectionFilter))
	}

	ccCalls := newInvocationChain(requestContext)
    //根据策略，为合约选择背书节点
	peers, err := clientContext.Selection.GetEndorsersForChaincode(newInvocationChain(requestContext), selectionOpts...)
	return ccCalls, peers, err
}
```

## 生成背书请求并发送到背书节点

请求头内容TransactionHeader数据模型

```yaml
TransactionHeader:
  id: 交易ID  #由客户端生成，根据安全随机数、组织身份+证书通过hash算法生成
  creator: 调用者身份 #组织ID
    Mspid: 组织MSPID
    IdBytes: 调用者证书
  nonce: 生成交易ID的安全随机数
  channelID: 通道名称
```

背书请求TransactionProposal数据模型

```yaml
Header:
  ChannelHeader:
    Type: HeaderType_ENDORSER_TRANSACTION #背书交易
    TxId: 交易ID
    Timestamp: 客户端UTC时间戳
    ChannelId: 通道名称
    Extension: 
      ChaincodeId: 合约名称
      PayloadVisibility: 交易数据可见性  #可以看到交易/交易数据的hash/不可看到交易三种case
    Epoch: 0  #目前默认都是0
  SignatureHeader:
    Nonce: 生成交易ID的安全随机数
    Creator: 调用者身份  #从交易请求头中获取
      Mspid: 组织MSP
      IdBytes: 调用者证书
Payload:
  ChaincodeProposalPayload:
    Input: 
      ChaincodeInvocationSpec:
        ChaincodeSpec:
          Type: 调用合约的语言类型  #?需要确认golang、node、java
          ChainCodeID:
            Name: 合约名称
          Input: 参数列表 #例如["invoke","a","b","1"]
    TransientMap: 交易过程中的瞬态参数map
```

发送前，对请求对象使用调用者私钥进行签名

```go
// signProposal creates a SignedProposal based on the current context.
func signProposal(ctx contextApi.Client, proposal *pb.Proposal) (*pb.SignedProposal, error) {
	proposalBytes, err := proto.Marshal(proposal)
	if err != nil {
		return nil, errors.Wrap(err, "mashal proposal failed")
	}

	signingMgr := ctx.SigningManager()
	if signingMgr == nil {
		return nil, errors.New("signing manager is nil")
	}

	signature, err := signingMgr.Sign(proposalBytes, ctx.PrivateKey())
	if err != nil {
		return nil, errors.WithMessage(err, "sign failed")
	}

	return &pb.SignedProposal{ProposalBytes: proposalBytes, Signature: signature}, nil
}

```

## 发送请求到背书节点

通过**gprc调用**protobuf格式的交易请求

```go
func (c *endorserClient) ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error) {
	out := new(ProposalResponse)
	err := grpc.Invoke(ctx, "/protos.Endorser/ProcessProposal", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

# 2 背书节点生成背书响应

## peer节点启动时，启动背书gpc服务

```go
// NewEndorserServer creates and returns a new Endorser server instance.
func NewEndorserServer(privDist privateDataDistributor, s Support) pb.EndorserServer {
	e := &Endorser{
		distributePrivateData: privDist,
		s: s,
	}
	return e
}

// Server API for Endorser service

type EndorserServer interface {
	ProcessProposal(context.Context, *SignedProposal) (*ProposalResponse, error)
}

func RegisterEndorserServer(s *grpc.Server, srv EndorserServer) {
	s.RegisterService(&_Endorser_serviceDesc, srv)
}

func _Endorser_ProcessProposal_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(SignedProposal)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(EndorserServer).ProcessProposal(ctx, in)
	}
    //注册背书grpc API
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Endorser/ProcessProposal",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(EndorserServer).ProcessProposal(ctx, req.(*SignedProposal))
	}
	return interceptor(ctx, in, info, handler)
}

```

## 背书节点模拟交易

背书节点，通过调用系统智能合约`escc`进行背书

其主要流程：

- 检查请求消息的合法性

  - 请求类型只允许HeaderType_ENDORSER_TRANSACTION、HeaderType_CONFIG_UPDATE、HeaderType_CONFIG、HeaderType_PEER_RESOURCE_UPDATE这四种
  - 带有正常的请求者身份证书和组织信息
  - 验证请求者证书和签名是合法的证书
  - peer使用相同的once、txid和请求者身份，重新生成txid，应当与请求中的txid完全一样，验证两者使用相同算法。
  - 请求中包含合约名称和payloadvisibility
  - 如果调用的是**系统智能合约(lscc、csss、qscc、escc、vscc)**，则检查请求者身份必须在通道的writers acl中

  ```go
  	//0 -- check and validate
  	vr, err := e.preProcess(signedProp)
  	if err != nil {
  		resp := vr.resp
  		return resp, err
  	}
  ```

- 使用TxSimulator，**调用交易请求中的智能合约**，产生读写集

  ```go
  //call specified chaincode (system or user)
  func (e *Endorser) callChaincode(ctxt context.Context, chainID string, version string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, cis *pb.ChaincodeInvocationSpec, cid *pb.ChaincodeID, txsim ledger.TxSimulator) (*pb.Response, *pb.ChaincodeEvent, error) {
  	endorserLogger.Debugf("[%s][%s] Entry chaincode: %s version: %s", chainID, shorttxid(txid), cid, version)
  	defer endorserLogger.Debugf("[%s][%s] Exit", chainID, shorttxid(txid))
  	var err error
  	var res *pb.Response
  	var ccevent *pb.ChaincodeEvent
  
  	if txsim != nil {
  		ctxt = context.WithValue(ctxt, chaincode.TXSimulatorKey, txsim)
  	}
  
  	//is this a system chaincode
  	scc := e.s.IsSysCC(cid.Name)
  
  	res, ccevent, err = e.s.Execute(ctxt, chainID, cid.Name, version, txid, scc, signedProp, prop, cis)
  	if err != nil {
  		return nil, nil, err
  	}
  
  	//per doc anything < 400 can be sent as TX.
  	//fabric errors will always be >= 400 (ie, unambiguous errors )
  	//"lscc" will respond with status 200 or 500 (ie, unambiguous OK or ERROR)
  	if res.Status >= shim.ERRORTHRESHOLD {
  		return res, nil, nil
  	}
  
  	//...其他非关键code
  	return res, ccevent, err
  }
  ```

## 对交易结果背书

- 对系统智能合约的发起的交易请求使用自带的`escc`系统智能合约
- 智能合约在初始化时，可以指定背书使用的系统智能合约，默认也使用`escc`系统智能合约
- 1.2.0以前，背书通过`escc`系统智能合约进行背书，在1.2.0以后，背书过程变为可插拔的实现，因此可以直接通过调用进行背书，但其背书流程是一样的
- 背书流程：使用节点本地证书和私钥，对模拟交易响应进行签名，表示完成背书

```go
var pResp *pb.ProposalResponse
//TODO till we implement global ESCC, CSCC for system chaincodes
//chainless proposals (such as CSCC) don't have to be endorsed
if chainID == "" {
	pResp = &pb.ProposalResponse{Response: res}
} else {
	pResp, err = e.endorseProposal(ctx, chainID, txid, signedProp, prop, res, simulationResult, ccevent, hdrExt.PayloadVisibility, hdrExt.ChaincodeId, txsim, cd)
	if err != nil {
		return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, err
	}
	if pResp != nil {
		if res.Status >= shim.ERRORTHRESHOLD {
			endorserLogger.Debugf("[%s][%s] endorseProposal() resulted in chaincode %s error for txid: %s", chainID, shorttxid(txid), hdrExt.ChaincodeId, txid)
			return pResp, &chaincodeError{res.Status, res.Message}
		}
	}
}
```
# 3 客户端验证背书响应

背书响应TransactionProposalResponse的数据模型

```yaml
TransactionProposalResponse:
  Endorser: 背书节点 #例如，grpcs://XXX:7051
  Status: 背书状态码
  ChaincodeStatus: 合约执行状态
  ProposalResponse:
    Payload:
      ProposalHash: 背书请求的hash #关联请求和响应
      Extension:
        Results: #背书响应产生的读写集
        Events: 背书响应产生的事件
        Response:
          Status: 合约响应状态
          Message: 合约异常时的消息
          Payload: 合约产生的输出
        ChaincodeId: 产生响应的合约信息  #记账节点根据此信息检查版本一致
          Name: 合约名称
          Version: 合约版本
          Path: 合约路径
    Version: 背书节点产生的版本依赖 #?用途存疑
    Timestamp: 背书响应生成UTC时间戳
    Response:
      Status: 背书节点响应状态码
      Message: 背书节点产生的响应消息
      Payload: 背书节点产生的响应
    Endorsement: 
      Endorser: #背书节点身份
        Mspid: 组织MSP
        IdBytes: 节点证书
      Signature: 背书结果签名 #背书节点使用自己私钥对响应做签名
        #?具体结构
```

## 验证背书结果的一致性

客户端需要验证背书成功并且所有背书节点的结果是一致的

```go
func (f *EndorsementValidationHandler) validate(txProposalResponse []*fab.TransactionProposalResponse) error {
	var a1 *pb.ProposalResponse
	for n, r := range txProposalResponse {
		response := r.ProposalResponse.GetResponse()
        //所有背书都成功
		if response.Status < int32(common.Status_SUCCESS) || response.Status >= int32(common.Status_BAD_REQUEST) {
			return status.NewFromProposalResponse(r.ProposalResponse, r.Endorser)
		}
		if n == 0 {
			a1 = r.ProposalResponse
			continue
		}

        //所有节点的背书结果都一致
		if !bytes.Equal(a1.Payload, r.ProposalResponse.Payload) ||
			!bytes.Equal(a1.GetResponse().Payload, response.Payload) {
			return status.New(status.EndorserClientStatus, status.EndorsementMismatch.ToInt32(),
				"ProposalResponsePayloads do not match", nil)
		}
	}

	return nil
}

```

## 检查背书响应签名

检查所有响应中的签名和证书

1. 证书满足以下条件：

   - 证书在有效期内

   - 背书者的证书，可以获取到ca链，且ca链上的证书也都在有效期
   - 根据配置，还需要检查证书的OU，必须包含client、peer或者orderer等有效ou

2. 签名检查

   根据响应的Signature和响应Payload，结合响应提供的背书证书，进行验签

# 4 客户端向orderer申请生成交易

客户端生成交易对象，并向orderer请求

```go
func (c *CommitTxHandler) Handle(requestContext *RequestContext, clientContext *ClientContext) {
	txnID := requestContext.Response.TransactionID

	//Register Tx event
	reg, statusNotifier, err := clientContext.EventService.RegisterTxStatusEvent(string(txnID)) // TODO: Change func to use TransactionID instead of string
	if err != nil {
		requestContext.Error = errors.Wrap(err, "error registering for TxStatus event")
		return
	}
	defer clientContext.EventService.Unregister(reg)

    //生成交易对象，并请求获取响应
	_, err = createAndSendTransaction(clientContext.Transactor, requestContext.Response.Proposal, requestContext.Response.Responses)
	if err != nil {
		requestContext.Error = errors.Wrap(err, "CreateAndSendTransaction failed")
		return
	}

    //通过eventhub监听区块交易
	select {
	case txStatus := <-statusNotifier:
        //交易是否有效
		requestContext.Response.TxValidationCode = txStatus.TxValidationCode

		if txStatus.TxValidationCode != pb.TxValidationCode_VALID {
			requestContext.Error = status.New(status.EventServerStatus, int32(txStatus.TxValidationCode),
				"received invalid transaction", nil)
			return
		}
        //交易超时：未通过network-config.yaml超时时间的，默认3分钟
        //参考fabric-sdk-go/pkg/fab/endpointconfig.go
	case <-requestContext.Ctx.Done():
		requestContext.Error = status.New(status.ClientStatus, status.Timeout.ToInt32(),
			"Execute didn't receive block event", nil)
		return
	}

	//Delegate to next step if any
	if c.next != nil {
		c.next.Handle(requestContext, clientContext)
	}
}
```

请求的Transaction对象数据模型

```yaml
Transaction:
  Transaction:
    Actions:
    - TransactionAction:
        Header: #背书请求Header中的SignatureHeader
        Payload:
          ChaincodeProposalPayload:
            ProposalResponsePayload:
              ChaincodeProposalPayload:
                Input: #背书请求中的合约调用Input
                TransientMap: nil #有意置空，对orderer屏蔽信息
            Endorsements: #所有背书响应的Endorsement列表
          Action:
  Proposal: 背书请求对象 #客户端发送的背书请求对象
```

发送前，使用调用者私钥对请求进行签名

```go
// signPayload signs payload
func signPayload(ctx contextApi.Client, payload *common.Payload) (*fab.SignedEnvelope, error) {
	payloadBytes, err := proto.Marshal(payload)
	if err != nil {
		return nil, errors.WithMessage(err, "marshaling of payload failed")
	}

	signingMgr := ctx.SigningManager()
	signature, err := signingMgr.Sign(payloadBytes, ctx.PrivateKey())
	if err != nil {
		return nil, errors.WithMessage(err, "signing of payload failed")
	}
	return &fab.SignedEnvelope{Payload: payloadBytes, Signature: signature}, nil
}
```

随机选择一个orderer节点发送请求，请求时，使用**grpc广播**，通过**双向流方式**，实现交易

```go
// SendBroadcast Send the created transaction to Orderer.
func (o *Orderer) SendBroadcast(ctx reqContext.Context, envelope *fab.SignedEnvelope) (*common.Status, error) {
	conn, err := o.conn(ctx)
	if err != nil {
		rpcStatus, ok := grpcstatus.FromError(err)
		if ok {
			return nil, errors.WithMessage(status.NewFromGRPCStatus(rpcStatus), "connection failed")
		}

		return nil, status.New(status.OrdererClientStatus, status.ConnectionFailed.ToInt32(), err.Error(), nil)
	}
	defer o.releaseConn(ctx, conn)

	broadcastClient, err := ab.NewAtomicBroadcastClient(conn).Broadcast(ctx)
	if err != nil {
		rpcStatus, ok := grpcstatus.FromError(err)
		if ok {
			err = status.NewFromGRPCStatus(rpcStatus)
		}
		return nil, errors.Wrap(err, "NewAtomicBroadcastClient failed")
	}

	responses := make(chan common.Status)
	errs := make(chan error, 1)

	go broadcastStream(broadcastClient, responses, errs)

	err = broadcastClient.Send(&common.Envelope{
		Payload:   envelope.Payload,
		Signature: envelope.Signature,
	})
	if err != nil {
		return nil, errors.Wrap(err, "failed to send envelope to orderer")
	}
	if err = broadcastClient.CloseSend(); err != nil {
		logger.Debugf("unable to close broadcast client [%s]", err)
	}

	return wrapStreamStatusRPC(responses, errs)
}
```



# 5 orderer产生区块并记账

## orderer启动时，启动grpc广播服务

代码在`fabric/orderer/common/server/main.go`中

```go
	server := NewServer(manager, signer, &conf.Debug, conf.General.Authentication.TimeWindow, mutualTLS)

	switch cmd {
	case start.FullCommand(): // "start" command
		logger.Infof("Starting %s", metadata.GetVersionInfo())
		initializeProfilingService(conf)
		ab.RegisterAtomicBroadcastServer(grpcServer.Server(), server)
		logger.Info("Beginning to serve requests")
		grpcServer.Start()
	case benchmark.FullCommand(): // "benchmark" command
		logger.Info("Starting orderer in benchmark mode")
		benchmarkServer := performance.GetBenchmarkServer()
		benchmarkServer.RegisterService(server)
		benchmarkServer.Start()
	}
```

## orderer排序

orderer接收消息，

> 代码路径`fabric/orderer/common/broadcast/broadcast.go`

```go
// Handle starts a service thread for a given gRPC connection and services the broadcast connection
func (bh *handlerImpl) Handle(srv ab.AtomicBroadcast_BroadcastServer) error {
	addr := util.ExtractRemoteAddress(srv.Context())
	logger.Debugf("Starting new broadcast loop for %s", addr)
	for {
        //双向流模式，从客户端接收消息
		msg, err := srv.Recv()
		if err == io.EOF {
			logger.Debugf("Received EOF from %s, hangup", addr)
			return nil
		}
		if err != nil {
			logger.Warningf("Error reading from %s: %s", addr, err)
			return err
		}

        //判断消息类型，由对应通道处理消息
		chdr, isConfig, processor, err := bh.sm.BroadcastChannelSupport(msg)
		if err != nil {
			channelID := "<malformed_header>"
			if chdr != nil {
				channelID = chdr.ChannelId
			}
			logger.Warningf("[channel: %s] Could not get message processor for serving %s: %s", channelID, addr, err)
			return srv.Send(&ab.BroadcastResponse{Status: cb.Status_BAD_REQUEST, Info: err.Error()})
		}

		if err = processor.WaitReady(); err != nil {
			logger.Warningf("[channel: %s] Rejecting broadcast of message from %s with SERVICE_UNAVAILABLE: rejected by Consenter: %s", chdr.ChannelId, addr, err)
			return srv.Send(&ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()})
		}

		if !isConfig {
			logger.Debugf("[channel: %s] Broadcast is processing normal message from %s with txid '%s' of type %s", chdr.ChannelId, addr, chdr.TxId, cb.HeaderType_name[chdr.Type])

			configSeq, err := processor.ProcessNormalMsg(msg)
			if err != nil {
				logger.Warningf("[channel: %s] Rejecting broadcast of normal message from %s because of error: %s", chdr.ChannelId, addr, err)
				return srv.Send(&ab.BroadcastResponse{Status: ClassifyError(err), Info: err.Error()})
			}

            //对请求进行排序
			err = processor.Order(msg, configSeq)
			if err != nil {
				logger.Warningf("[channel: %s] Rejecting broadcast of normal message from %s with SERVICE_UNAVAILABLE: rejected by Order: %s", chdr.ChannelId, addr, err)
				return srv.Send(&ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()})
			}
		} else { // isConfig
			logger.Debugf("[channel: %s] Broadcast is processing config update message from %s", chdr.ChannelId, addr)

			config, configSeq, err := processor.ProcessConfigUpdateMsg(msg)
			if err != nil {
				logger.Warningf("[channel: %s] Rejecting broadcast of config message from %s because of error: %s", chdr.ChannelId, addr, err)
				return srv.Send(&ab.BroadcastResponse{Status: ClassifyError(err), Info: err.Error()})
			}

			err = processor.Configure(config, configSeq)
			if err != nil {
				logger.Warningf("[channel: %s] Rejecting broadcast of config message from %s with SERVICE_UNAVAILABLE: rejected by Configure: %s", chdr.ChannelId, addr, err)
				return srv.Send(&ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE, Info: err.Error()})
			}
		}

		logger.Debugf("[channel: %s] Broadcast has successfully enqueued message of type %s from %s", chdr.ChannelId, cb.HeaderType_name[chdr.Type], addr)

		err = srv.Send(&ab.BroadcastResponse{Status: cb.Status_SUCCESS})
		if err != nil {
			logger.Warningf("[channel: %s] Error sending to %s: %s", chdr.ChannelId, addr, err)
			return err
		}
	}
}

```

### solo共识

代码路径`fabric/orderer/consensus/solo/consensus.go`

solo共识通过单一chan来作为排序的途径，按进入chan的顺序作为消息的顺序。

```go
// Order accepts normal messages for ordering
func (ch *chain) Order(env *cb.Envelope, configSeq uint64) error {
	select {
	case ch.sendChan <- &message{
		configSeq: configSeq,
		normalMsg: env,
	}:
		return nil
	case <-ch.exitChan:
		return fmt.Errorf("Exiting")
	}
}
```

orderer服务启动时，会不断从chan中获取消息，按批次写入区块通知排序外逻辑。

- 如果消息总个数达到`MaxMessageCount`，产生区块
- 当前消息大小超过`PreferredMaxBytes`，则与之前的消息分离，产生两个区块
- 如果消息总大小超过`PreferredMaxBytes`，产生一个区块

```go
func (ch *chain) main() {
	var timer <-chan time.Time
	var err error

	for {
		seq := ch.support.Sequence()
		err = nil
		select {
		case msg := <-ch.sendChan:
			if msg.configMsg == nil {
				// NormalMsg
				if msg.configSeq < seq {
					_, err = ch.support.ProcessNormalMsg(msg.normalMsg)
					if err != nil {
						logger.Warningf("Discarding bad normal message: %s", err)
						continue
					}
				}
                //按当前消息大小决定是否打包为区块
				batches, _ := ch.support.BlockCutter().Ordered(msg.normalMsg)
				if len(batches) == 0 && timer == nil {
					timer = time.After(ch.support.SharedConfig().BatchTimeout())
					continue
				}
				for _, batch := range batches {
                    //写入区块到chan
					block := ch.support.CreateNextBlock(batch)
					ch.support.WriteBlock(block, nil)
				}
				if len(batches) > 0 {
					timer = nil
				}
			} else {
				// ConfigMsg
				if msg.configSeq < seq {
					msg.configMsg, _, err = ch.support.ProcessConfigMsg(msg.configMsg)
					if err != nil {
						logger.Warningf("Discarding bad config message: %s", err)
						continue
					}
				}
                //如果是配置消息，则立即打包交易为区块
				batch := ch.support.BlockCutter().Cut()
				if batch != nil {
					block := ch.support.CreateNextBlock(batch)
					ch.support.WriteBlock(block, nil)
				}

				block := ch.support.CreateNextBlock([]*cb.Envelope{msg.configMsg})
				ch.support.WriteConfigBlock(block, nil)
				timer = nil
			}
		case <-timer:
			//clear the timer
			timer = nil

			batch := ch.support.BlockCutter().Cut()
			if len(batch) == 0 {
				logger.Warningf("Batch timer expired with no pending requests, this might indicate a bug")
				continue
			}
			logger.Debugf("Batch timer expired, creating block")
			block := ch.support.CreateNextBlock(batch)
			ch.support.WriteBlock(block, nil)
		case <-ch.exitChan:
			logger.Debugf("Exiting")
			return
		}
	}
}
```



### kafka共识

代码路径`fabric/orderer/consensus/kafka/chain.go`

- 如果消息为配置消息，则立即打包消息为区块，不再管大小的问题

- 将收到的请求封装为 Kafka 消息，通过 Produce 接口发送到 Kakfa 集群对应的 topic 分区中。下列分块条件满足一个则发送分块消息给kafka：
  - 当前消息数达到 `MaxMessageCount`，默认为10 
  - 当前批次总消息尺寸过大`AbsoluteMaxBytes`，默认为10MB，要求kafka配置`message.max.bytes`和`replica.fetch.max.bytes`要比这个值大
  - 单个消息大小超过`PreferredMaxBytes`，默认为512KB
  - 超时时间达到 `BatchTimeout`，默认2s。

- Kafka 集群维护多个 topic 分区（即channel对应的topic）。Kakfa 通过共识算法来确保写入到分区后的消息的一致性。一旦写入分区，任何 Orderer 节点看到的都是相同的消息队列。

```go
// Implements the consensus.Chain interface. Called by Broadcast().
func (chain *chainImpl) Order(env *cb.Envelope, configSeq uint64) error {
	return chain.order(env, configSeq, int64(0))
}

func (chain *chainImpl) order(env *cb.Envelope, configSeq uint64, originalOffset int64) error {
	marshaledEnv, err := utils.Marshal(env)
	if err != nil {
		return fmt.Errorf("cannot enqueue, unable to marshal envelope because = %s", err)
	}
	if !chain.enqueue(newNormalMessage(marshaledEnv, configSeq, originalOffset)) {
		return fmt.Errorf("cannot enqueue")
	}
	return nil
}
```

orderer 节点在创建通道后，对通道对应的 Kafka 分区进行数据监听，不断从 Kafka 拉取（Consume）新的交易消息，并对消息进行处理，收到分块消息后，将消息打包为区块。

```go
//fabric/orderer/consensus/kafka/chain.go
func (chain *chainImpl) processTimeToCut(ttcMessage *ab.KafkaMessageTimeToCut, receivedOffset int64) error {
	ttcNumber := ttcMessage.GetBlockNumber()
	logger.Debugf("[channel: %s] It's a time-to-cut message for block %d", chain.ChainID(), ttcNumber)
	if ttcNumber == chain.lastCutBlockNumber+1 {
		chain.timer = nil
		logger.Debugf("[channel: %s] Nil'd the timer", chain.ChainID())
		batch := chain.BlockCutter().Cut()
		if len(batch) == 0 {
			return fmt.Errorf("got right time-to-cut message (for block %d),"+
				" no pending requests though; this might indicate a bug", chain.lastCutBlockNumber+1)
		}
		block := chain.CreateNextBlock(batch)
		metadata := utils.MarshalOrPanic(&ab.KafkaMetadata{
			LastOffsetPersisted:         receivedOffset,
			LastOriginalOffsetProcessed: chain.lastOriginalOffsetProcessed,
		})
        //写入区块到chan中，被排序外业务接收
		chain.WriteBlock(block, metadata)
		chain.lastCutBlockNumber++
		logger.Debugf("[channel: %s] Proper time-to-cut received, just cut block %d", chain.ChainID(), chain.lastCutBlockNumber)
		return nil
	} else if ttcNumber > chain.lastCutBlockNumber+1 {
		return fmt.Errorf("got larger time-to-cut message (%d) than allowed/expected (%d)"+
			" - this might indicate a bug", ttcNumber, chain.lastCutBlockNumber+1)
	}
	logger.Debugf("[channel: %s] Ignoring stale time-to-cut-message for block %d", chain.ChainID(), ttcNumber)
	return nil
}
```

## orderer写入区块到本地账本

orderer服务同时，从共识算法中消费消息，写入区块到本地账本

```go
//fabric/orderer/common/multichannel/blockwriter.go
// WriteBlock should be invoked for blocks which contain normal transactions.
// It sets the target block as the pending next block, and returns before it is committed.
// Before returning, it acquires the committing lock, and spawns a go routine which will
// annotate the block with metadata and signatures, and write the block to the ledger
// then release the lock.  This allows the calling thread to begin assembling the next block
// before the commit phase is complete.
func (bw *BlockWriter) WriteBlock(block *cb.Block, encodedMetadataValue []byte) {
	bw.committingBlock.Lock()
	bw.lastBlock = block

	go func() {
		defer bw.committingBlock.Unlock()
		bw.commitBlock(encodedMetadataValue)
	}()
}

// commitBlock should only ever be invoked with the bw.committingBlock held
// this ensures that the encoded config sequence numbers stay in sync
func (bw *BlockWriter) commitBlock(encodedMetadataValue []byte) {
	// Set the orderer-related metadata field
	if encodedMetadataValue != nil {
		bw.lastBlock.Metadata.Metadata[cb.BlockMetadataIndex_ORDERER] = utils.MarshalOrPanic(&cb.Metadata{Value: encodedMetadataValue})
	}
	bw.addBlockSignature(bw.lastBlock)
	bw.addLastConfigSignature(bw.lastBlock)

	err := bw.support.Append(bw.lastBlock)
	if err != nil {
		logger.Panicf("[channel: %s] Could not append block: %s", bw.support.ChainID(), err)
	}
	logger.Debugf("[channel: %s] Wrote block %d", bw.support.ChainID(), bw.lastBlock.GetHeader().Number)
}

```

# 6 peer索取新区块记入本地账本

## leader peer发送deliver消息

peer节点再加入通道后，会在peer和orderer之间通过**grpc双向流模式建立连接**，然后发送`deliver`消息到orderer上读取账本。

peer节点启动服务时，根据本地账本的通道配置中的orderer地址和leader配置，向orderer发送`delivery`消息，索要新的区块。

```go
//fabric/core/peer/peer.go
// createChain creates a new chain object and insert it into the chains
func createChain(cid string, ledger ledger.PeerLedger, cb *common.Block) error {
	chanConf, err := retrievePersistedChannelConfig(ledger)
	if err != nil {
		return err
	}

	var bundle *channelconfig.Bundle

	if chanConf != nil {
		bundle, err = channelconfig.NewBundle(cid, chanConf)
		if err != nil {
			return err
		}
	} else {
		// Config was only stored in the statedb starting with v1.1 binaries
		// so if the config is not found there, extract it manually from the config block
		envelopeConfig, err := utils.ExtractEnvelope(cb, 0)
		if err != nil {
			return err
		}

		bundle, err = channelconfig.NewBundleFromEnvelope(envelopeConfig)
		if err != nil {
			return err
		}
	}

	capabilitiesSupportedOrPanic(bundle)

	channelconfig.LogSanityChecks(bundle)

	gossipEventer := service.GetGossipService().NewConfigEventer()

	gossipCallbackWrapper := func(bundle *resourcesconfig.Bundle) {
		ac, ok := bundle.ChannelConfig().ApplicationConfig()
		if !ok {
			// TODO, handle a missing ApplicationConfig more gracefully
			ac = nil
		}
		gossipEventer.ProcessConfigUpdate(&gossipSupport{
			Validator:   bundle.ChannelConfig().ConfigtxValidator(),
			Application: ac,
			Channel:     bundle.ChannelConfig().ChannelConfig(),
		})
		service.GetGossipService().SuspectPeers(func(identity api.PeerIdentityType) bool {
			// TODO: this is a place-holder that would somehow make the MSP layer suspect
			// that a given certificate is revoked, or its intermediate CA is revoked.
			// In the meantime, before we have such an ability, we return true in order
			// to suspect ALL identities in order to validate all of them.
			return true
		})
	}

	trustedRootsCallbackWrapper := func(bundle *resourcesconfig.Bundle) {
		updateTrustedRoots(bundle.ChannelConfig())
	}

	mspCallback := func(bundle *resourcesconfig.Bundle) {
		// TODO remove once all references to mspmgmt are gone from peer code
		mspmgmt.XXXSetMSPManager(cid, bundle.ChannelConfig().MSPManager())
	}

	ac, ok := bundle.ApplicationConfig()
	if !ok {
		ac = nil
	}
	cs := &chainSupport{
		Application: ac, // TODO, refactor as this is accessible through Manager
		ledger:      ledger,
		fileLedger:  fileledger.NewFileLedger(fileLedgerBlockStore{ledger}),
	}

	peerSingletonCallback := func(bundle *resourcesconfig.Bundle) {
		ac, ok := bundle.ChannelConfig().ApplicationConfig()
		if !ok {
			ac = nil
		}
		cs.Application = ac
		cs.Resources = bundle.ChannelConfig()
	}

	resConf := &common.Config{ChannelGroup: &common.ConfigGroup{}}
	if ac != nil && ac.Capabilities().ResourcesTree() {
		iResConf, err := retrievePersistedResourceConfig(ledger)

		if err != nil {
			return err
		}

		if iResConf != nil {
			resConf = iResConf
		}
	}

	rBundle, err := resourcesconfig.NewBundle(cid, resConf, bundle)
	if err != nil {
		return err
	}

	cs.bundleSource = resourcesconfig.NewBundleSource(
		rBundle,
		gossipCallbackWrapper,
		trustedRootsCallbackWrapper,
		mspCallback,
		peerSingletonCallback,
	)

	vcs := struct {
		*chainSupport
		*semaphore.Weighted
		Support
	}{cs, validationWorkersSemaphore, GetSupport()}
	validator := txvalidator.NewTxValidator(vcs)
	c := committer.NewLedgerCommitterReactive(ledger, func(block *common.Block) error {
		chainID, err := utils.GetChainIDFromBlock(block)
		if err != nil {
			return err
		}
		return SetCurrConfigBlock(block, chainID)
	})

	ordererAddresses := bundle.ChannelConfig().OrdererAddresses()
	if len(ordererAddresses) == 0 {
		return errors.New("No ordering service endpoint provided in configuration block")
	}

	// TODO: does someone need to call Close() on the transientStoreFactory at shutdown of the peer?
	store, err := transientStoreFactory.OpenStore(bundle.ConfigtxValidator().ChainID())
	if err != nil {
		return errors.Wrapf(err, "Failed opening transient store for %s", bundle.ConfigtxValidator().ChainID())
	}
	simpleCollectionStore := privdata.NewSimpleCollectionStore(&collectionSupport{
		PeerLedger: ledger,
	})
	service.GetGossipService().InitializeChannel(bundle.ConfigtxValidator().ChainID(), ordererAddresses, service.Support{
		Validator: validator,
		Committer: c,
		Store:     store,
		Cs:        simpleCollectionStore,
	})

	chains.Lock()
	defer chains.Unlock()
	chains.list[cid] = &chain{
		cs:        cs,
		cb:        cb,
		committer: c,
	}

	return nil
}

```

## orderer处理deliver消息返回区块

接收delivery消息，并读取本地通道账本，产生区块

```go
// Handle used to handle incoming deliver requests
func (ds *deliverHandler) Handle(srv *DeliverServer) error {
	addr := util.ExtractRemoteAddress(srv.Context())
	logger.Debugf("Starting new deliver loop for %s", addr)
	for {
		logger.Debugf("Attempting to read seek info message from %s", addr)
        //接收消息
		envelope, err := srv.Recv()
		if err == io.EOF {
			logger.Debugf("Received EOF from %s, hangup", addr)
			return nil
		}

		if err != nil {
			logger.Warningf("Error reading from %s: %s", addr, err)
			return err
		}

        //从本地的通道账本中读取开始读取区块，返回给peer
		if err := ds.deliverBlocks(srv, envelope); err != nil {
			return err
		}

		logger.Debugf("Waiting for new SeekInfo from %s", addr)
	}
}
```

## leader peer收到区块并gossip分发

从orderer服务收到区块后，通过gossip分发到其他peer

```go
//fabric/core/deliverservice/blocksprovider/blocksprovider.go
// DeliverBlocks used to pull out blocks from the ordering service to
// distributed them across peers
func (b *blocksProviderImpl) DeliverBlocks() {
	errorStatusCounter := 0
	statusCounter := 0
	defer b.client.Close()
	for !b.isDone() {
		msg, err := b.client.Recv()
		if err != nil {
			logger.Warningf("[%s] Receive error: %s", b.chainID, err.Error())
			return
		}
		switch t := msg.Type.(type) {
		case *orderer.DeliverResponse_Status:
			if t.Status == common.Status_SUCCESS {
				logger.Warningf("[%s] ERROR! Received success for a seek that should never complete", b.chainID)
				return
			}
			if t.Status == common.Status_BAD_REQUEST || t.Status == common.Status_FORBIDDEN {
				logger.Errorf("[%s] Got error %v", b.chainID, t)
				errorStatusCounter++
				if errorStatusCounter > b.wrongStatusThreshold {
					logger.Criticalf("[%s] Wrong statuses threshold passed, stopping block provider", b.chainID)
					return
				}
			} else {
				errorStatusCounter = 0
				logger.Warningf("[%s] Got error %v", b.chainID, t)
			}
			maxDelay := float64(maxRetryDelay)
			currDelay := float64(time.Duration(math.Pow(2, float64(statusCounter))) * 100 * time.Millisecond)
			time.Sleep(time.Duration(math.Min(maxDelay, currDelay)))
			if currDelay < maxDelay {
				statusCounter++
			}
			if t.Status == common.Status_BAD_REQUEST {
				b.client.Disconnect(false)
			} else {
				b.client.Disconnect(true)
			}
			continue
		case *orderer.DeliverResponse_Block:
			errorStatusCounter = 0
			statusCounter = 0
			seqNum := t.Block.Header.Number

            //验证区块
			marshaledBlock, err := proto.Marshal(t.Block)
			if err != nil {
				logger.Errorf("[%s] Error serializing block with sequence number %d, due to %s", b.chainID, seqNum, err)
				continue
			}
			if err := b.mcs.VerifyBlock(gossipcommon.ChainID(b.chainID), seqNum, marshaledBlock); err != nil {
				logger.Errorf("[%s] Error verifying block with sequnce number %d, due to %s", b.chainID, seqNum, err)
				continue
			}

            //分发到同一通道的其他peer
			numberOfPeers := len(b.gossip.PeersOfChannel(gossipcommon.ChainID(b.chainID)))
			// Create payload with a block received
			payload := createPayload(seqNum, marshaledBlock)
			// Use payload to create gossip message
			gossipMsg := createGossipMsg(b.chainID, payload)

			logger.Debugf("[%s] Adding payload locally, buffer seqNum = [%d], peers number [%d]", b.chainID, seqNum, numberOfPeers)
			// Add payload to local state payloads buffer
			if err := b.gossip.AddPayload(b.chainID, payload); err != nil {
				logger.Warning("Failed adding payload of", seqNum, "because:", err)
			}

			// Gossip messages with other nodes
			logger.Debugf("[%s] Gossiping block [%d], peers number [%d]", b.chainID, seqNum, numberOfPeers)
			b.gossip.Gossip(gossipMsg)
		default:
			logger.Warningf("[%s] Received unknown: ", b.chainID, t)
			return
		}
	}
}
```

## committing peer存储账本到本地账本

节点启动gossip服务，从leader peer接收到区块消息后，开始提交区块

```go
//fabric/gossip/state/state.go
func (s *GossipStateProviderImpl) deliverPayloads() {
	defer s.done.Done()

	for {
        //经历一系列代码流转，接收消息从中取出区块内容
		select {
		// Wait for notification that next seq has arrived
		case <-s.payloads.Ready():
			logger.Debugf("Ready to transfer payloads to the ledger, next sequence number is = [%d]", s.payloads.Next())
			// Collect all subsequent payloads
			for payload := s.payloads.Pop(); payload != nil; payload = s.payloads.Pop() {
				rawBlock := &common.Block{}
				if err := pb.Unmarshal(payload.Data, rawBlock); err != nil {
					logger.Errorf("Error getting block with seqNum = %d due to (%+v)...dropping block", payload.SeqNum, errors.WithStack(err))
					continue
				}
				if rawBlock.Data == nil || rawBlock.Header == nil {
					logger.Errorf("Block with claimed sequence %d has no header (%v) or data (%v)",
						payload.SeqNum, rawBlock.Header, rawBlock.Data)
					continue
				}
				logger.Debug("New block with claimed sequence number ", payload.SeqNum, " transactions num ", len(rawBlock.Data.Data))

				// Read all private data into slice
				var p util.PvtDataCollections
				if payload.PrivateData != nil {
					err := p.Unmarshal(payload.PrivateData)
					if err != nil {
						logger.Errorf("Wasn't able to unmarshal private data for block seqNum = %d due to (%+v)...dropping block", payload.SeqNum, errors.WithStack(err))
						continue
					}
				}
                //记账
				if err := s.commitBlock(rawBlock, p); err != nil {
					if executionErr, isExecutionErr := err.(*vsccErrors.VSCCExecutionFailureError); isExecutionErr {
						logger.Errorf("Failed executing VSCC due to %v. Aborting chain processing", executionErr)
						return
					}
					logger.Panicf("Cannot commit block to the ledger due to %+v", errors.WithStack(err))
				}
			}
		case <-s.stopCh:
			s.stopCh <- struct{}{}
			logger.Debug("State provider has been stopped, finishing to push new blocks.")
			return
		}
	}
}
```

提交区块进入账本，提交过程中对区块进行校验。

- 检查交易格式、证书有效性和数据的签名合法性
- 检查交易TxID是否重复
- 系统智能合约`vscc`检查交易有效性和背书策略是否满足
- 执行`mvcc`，检查并发的冲突
- 存储区块：无论其中的交易是否有效合法
- 更新state db：只根据**有效交易中的读写集**更新

```go
// StoreBlock stores block with private data into the ledger
func (c *coordinator) StoreBlock(block *common.Block, privateDataSets util.PvtDataCollections) error {
	if block.Data == nil {
		return errors.New("Block data is empty")
	}
	if block.Header == nil {
		return errors.New("Block header is nil")
	}
	logger.Infof("Received block [%d]", block.Header.Number)

	logger.Debugf("Validating block [%d]", block.Header.Number)
	err := c.Validator.Validate(block)
	if err != nil {
		logger.Errorf("Validation failed: %+v", err)
		return err
	}

	blockAndPvtData := &ledger.BlockAndPvtData{
		Block:        block,
		BlockPvtData: make(map[uint64]*ledger.TxPvtData),
	}

	ownedRWsets, err := computeOwnedRWsets(block, privateDataSets)
	if err != nil {
		logger.Warning("Failed computing owned RWSets", err)
		return err
	}

	privateInfo, err := c.listMissingPrivateData(block, ownedRWsets)
	if err != nil {
		logger.Warning(err)
		return err
	}

	retryThresh := viper.GetDuration("peer.gossip.pvtData.pullRetryThreshold")
	var bFetchFromPeers bool // defaults to false
	if len(privateInfo.missingKeys) == 0 {
		logger.Debug("No missing collection private write sets to fetch from remote peers")
	} else {
		bFetchFromPeers = true
		logger.Debug("Could not find all collection private write sets in local peer transient store.")
		logger.Debug("Fetching", len(privateInfo.missingKeys), "collection private write sets from remote peers for a maximum duration of", retryThresh)
	}
	start := time.Now()
	limit := start.Add(retryThresh)
	for len(privateInfo.missingKeys) > 0 && time.Now().Before(limit) {
		c.fetchFromPeers(block.Header.Number, ownedRWsets, privateInfo)
		time.Sleep(pullRetrySleepInterval)
	}

	// Only log results if we actually attempted to fetch
	if bFetchFromPeers {
		if len(privateInfo.missingKeys) == 0 {
			logger.Debug("Fetched all missing collection private write sets from remote peers")
		} else {
			logger.Warning("Could not fetch all missing collection private write sets from remote peers. Will commit block with missing private write sets:", privateInfo.missingKeys)
		}
	}

	// populate the private RWSets passed to the ledger
	for seqInBlock, nsRWS := range ownedRWsets.bySeqsInBlock() {
		rwsets := nsRWS.toRWSet()
		logger.Debugf("Added %d namespace private write sets for block [%d], tran [%d]", len(rwsets.NsPvtRwset), block.Header.Number, seqInBlock)
		blockAndPvtData.BlockPvtData[seqInBlock] = &ledger.TxPvtData{
			SeqInBlock: seqInBlock,
			WriteSet:   rwsets,
		}
	}

	// populate missing RWSets to be passed to the ledger
	for missingRWS := range privateInfo.missingKeys {
		blockAndPvtData.Missing = append(blockAndPvtData.Missing, ledger.MissingPrivateData{
			TxId:       missingRWS.txID,
			Namespace:  missingRWS.namespace,
			Collection: missingRWS.collection,
			SeqInBlock: int(missingRWS.seqInBlock),
		})
	}

	// commit block and private data
	err = c.CommitWithPvtData(blockAndPvtData)
	if err != nil {
		return errors.Wrap(err, "commit failed")
	}

	if len(blockAndPvtData.BlockPvtData) > 0 {
		// Finally, purge all transactions in block - valid or not valid.
		if err := c.PurgeByTxids(privateInfo.txns); err != nil {
			logger.Error("Purging transactions", privateInfo.txns, "failed:", err)
		}
	}

	seq := block.Header.Number
	if seq%c.transientBlockRetention == 0 && seq > c.transientBlockRetention {
		err := c.PurgeByHeight(seq - c.transientBlockRetention)
		if err != nil {
			logger.Error("Failed purging data from transient store at block", seq, ":", err)
		}
	}

	return nil
}
```

最终，当区块存储到账本后，peer通过eventhub，发布区块相关的所有事件。

# 7 客户端收到交易事件

客户端收到orderer节点响应TransactionResponse。

TransactionResponse数据模型：

```yaml
TransactionResponse:
  Odererer: 产生响应的orderer节点 #例如grpcs://XYYYY:7050
```

根据交易ID，从创建peer的eventhub来监听交易事件，获取交易结果，或者一直等待到区块生成请求超时。