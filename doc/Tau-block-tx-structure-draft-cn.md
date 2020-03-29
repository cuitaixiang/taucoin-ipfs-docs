# TAU - blockchain cloud for unlimited data publishing.

## TAU is designed to enable mobile device. Three types of modes exists, which are statefull miner, stateless miner and normal user. 
### procedures for statefull miner, which requires wifi and power plugged to download full history, and mising wifi or plug will switch to normal user mode
0. open android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance and swarm connection history; note: combine relay and peer randomness to reduce connection jam;
2. request the miner peer for the n+1 state according to CBC (correct by construction); 
3. traverse 144 history states from the miner and keep accounting of the root array; 
mining based on curent root and build&validate (n+1) state JSON; when connection timeout, go to step(1)
4. go to step (1), until half of the know mining peers are traversed. 
5. calculate the CBC safety state k, if k it out of mutable range, that is n-144, then go to step (1). 
6. random walk until connect to a next relay; random walk until connect to a next miner
7. start mining by asking the peers longest chain, and verify k to n, build&validate (n+1) state JSON, when timeout, go to step (6)
8. as full nodes, it will verify state #1 to #n in the background. 
9. when new added mining nodes increase 33% or self-disconnected 12 hours, go to step (1).
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

### Procedures for stateless miner, which requires wifi and power plugged to download partial history, and missing wifi or plug will switch to normal user mode
0. open android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance and swarm connection history; note: combine relay and peer randomness to reduce connection jam;
2. request the miner peer for the (n+1) state according to CBC (correct by construction); 
3. traverse 144 history states from the miner and keep accounting of the root array; 
mining based on curent root and build&validate (n+1) state JSON; when connection timeout, go to step(1)
4. go to step (1), until half of the know mining peers are traversed. 
5. calculate the CBC safety state k, if k it out of mutable range, that is n-144, then go to step (1). 
6. random walk until connect to a next relay; random walk until connect to a next miner
7. start mining by asking the peers longest chain, and verify k to n, build&validate (n+1) state JSON, when timeout, go to step (6) 
8. when new added mining nodes increase 33% or self-disconnected 12 hours, go to step (1).
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

### steps for normal users on battery or 4G 
0. open android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance and swarm connection history; note: combine relay and peer randomness to reduce connection jam;
2. request the miner peer for the n+1 state according to CBC (correct by construction); 
3. traverse 144 history states from the miner and keep accounting of the root array; 
mining based on curent root k and build&validate (n+1) state JSON; when connection timeout, go to step(1)
4. update the CBC safety state k, then go to step (1). 
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

## State structure with entry point of "cid" + peersID, key-values are:

### 1. stateNumber, 8; stateNumber=1234567

### 2. statJSON1234567content={ 

version,8; 

timestamp, 4; 

state number, 8; 

base target, 8; 

cumulative difficulty,8 ; 

generation signature,32;

sender/miner TAU address, 20; 

sender/miner nounce, 8, mining is treated as a tx sending to self, nounce ++;

senderProfileJSON,1024,Ta..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; }; 

txJSON; {original JSON from peers}

previous hamt state root,

32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte, when at same difficulty, high signature number wins.

}

## exporting three type Transactions Nounce JSON and its vars, exported, three type of txs: 0-coinbase, 1-wiring, 2-message.
### -coinbase tx
### 3a. sender/minerNounceJSON ={

version,8, "0x1" as default;

opt_code, 8, 0x0 is coinbase;

nounce, 8;

timestamp,4,tx expire in 12 hours;

amount,5;

sender/minerProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

stateNumber, 8;

}

### 4a. sender/miner nounce	|8;
### 5a. sender/miner balance        | 5;

### -Coins Wiring
### 3b. senderNounceJSON = {

opt_code, 8, 1;

nounce, 8;

receiver TAU address,20, type 2;

version,8, "0x1" as default;

timestamp,4,tx expire in 12 hours;

amount,5, type 1, 2;

stateNumber, 8;

txfee;

senderProfileJSON,1024,Ta..xProfile; {TAU:Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; 
};


### 4b. sender nounce	|8 			|Ta..xNounce"	10;used as power
### 5b. sender balance        | 5       	|Tsender..xBalance" | 10000 ; through senderNounce to get TXJSON, ProfielJSON
### 6b receiver balance      | 5     		|Treceiver..xBalance" | 10000

### -Message transaction
### 3c. senderNounceJSON = {

opt_code, 8, 2;

nounce, 8;

version,8, "0x1" as default;

timestamp,4,tx expire in 12 hours;
stateNumber, 8;
txfee;

senderProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

thread = "other sender address + tx nounce"; if thread equal self sender nounce, it is a new thread.
msgJSON,1024;{}
msgAttachmentSize,8;
msgAattachmentCid,32;
}


### 4c sender nounce	|8 	
### 5c sender balance        | 5     


### 20200304
- 讨论的问题主要是解决IPFS中的Peer Routing问题
```
	1. 目前IPFS的Peer routing和Content routing是基于Kad DHT算法来实现的，在一定网络规模下效率有限

	2. 为了舍弃DHT查找矿工节点的问题，在区块链中尽可能记录已有矿工节点的信息(Peer Id+ Relay MultiAddress)

	3. 利用POT可预测出块的机制，矿工节点可以时刻进行区块的Forge, N+1区块的Forge是在其他节点请求下进行

	4. N+1区块保留Block中的Relay MultiAddress

	5. 对于打包出块的交易做如下变动:

		5.1 节点本身的交易，交易中需要添加到请求节点的Relay节点信息

		5.2 其他节点的交易，直接打包即可

		举例说明，A -> Rb <- B，B节点请求A节点N+1区块，A节点的本身交易需要添加Rb签名后打包
					其他节点通过节点B，已知Rb，进而可以Peer的链接；

				  A -> Rc <- C，C节点请求A节点N+1区块，A节点的本身交易需要添加Rc签名后打包
					其他节点通过节点C，已知Rc，进而可以Peer的链接；

	6. 增加Appendix Root Hash，其中放当前节点的其他交易信息，该Root Hash背后对应的是一个DAG结构的多笔交易

	7. N+1区块是实时请求，实时返回的结果，因为交易中添加Relay信息的不一致，导致N+1区块也是不同的结果
```

- 区块交易结构做如下变动，区块增加Appendix Root Hash，交易中再次确认Relay MutliAddress字段的重要性
---

### 20200305
- 讨论了节点连接和出块的具体实现，总结如下：
```
	1. 节点上线，利用Tau软件中配置的最初区块节点(交易信息)或者上次Mining退出时保留的最新区块，开启发现流程

		1.1 对于已在线过的节点，是否保留连接过的中继和Peers，或者优先连接

	2. 区块和交易信息中数据为上线节点提供了起始的Relay nodes和Tau链上的Peers；

	3. 连接和本节点网络通讯较好的几个中继节点后，借助Relay链路可以进一步连接Tau链上记录的Peer

		3.1 这部分的设计后期需要文档说明，Relay和Peer的挑选策略，进而组成一个高质量的Swarm网络

	4. Peer一旦建立连接，双方通过P2P数据协议获取特定节点的Block
		
		举例说明一些细节：
		
		新节点A请求B节点的下一个区块, 记做BBlock(n+ 1)
		
		4.1 BBlock(n+ 1)的区块时间是B节点下一个区块的预计出块时间

		4.2 BBlock(n+ 1)含有定制的B发出的交易，该交易中B节点需要填入和A节点连接的Relay节点MultiAddress

		B请求新节点A的下一个区块, 记做ABlock(n+ 1)，这种行为也是合法的，即便远落后于主链高度

	5. 通过Block(n+ 1)以及IPFS的Content Routing机制，可以获取每个Swarm peer的Block(n)

		5.1 对于Block(n+ 1)的处理，先拿Block(n+ 1)区块中记录的交易信息(打包的一级交易，Appendix root记录的交易),来丰富自己的交易池

		5.2 验证Block(n+ 1)合法性，不合法不放入出块时间列表

	6. 开始对Block(n)的投票流程，决定选择哪条链作为本节点的主链

	7. 对主链的确定实际更新了本地节点的Block(n),下次其他节点的请求可以回复调整Block(n+ 1)
		
		7.1 随着Block(n)的确定，以及Tau链中Stateless的设计，此时节点的Mining实际已经跟上了主链脚步

	8. 本节点同时维护一个出块时间列表，到达出块时间更新本地节点信息，本节点从N高度变更到N+1高度了，开始下一轮询问，转到Step 4
