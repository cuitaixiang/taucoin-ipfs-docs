# Chain Level State  -  keys are defined in the protocol and will be hash-stored in hamt, all global export
  field        | Size     |  Key   |  value and notes
---------------|----------|--------|--------
stateJSON provides all the vars in json format for pick up. 
stateJSONContent | * | stateNumber+"JSONContent" | statJSONcontent={ 
//timestamp    | 4        				|"timestamp"	|unix timestamp for
//version        | 8        				|"version" 	| "0x1" as initial default 
//state number   | 8        					|"stateNumber"	| e.g 0x1stateNumber = 100
//base target    | 8        				|"baseTarget"	|for POT - Proof of Transaction calculation 
//cumulative difficulty    | 8       			|"cumulativeDifficulty"|current consensus chain parameter 
//generation signature     | 32      			|"generationSignature"|for POT calculation,  x power x time
//miner TAU address    | 20       			|"minerTAUaddr"	| minerTAUaddr = "Ta..x"; 
//JSON for transactions| tx JSON #1    | 32 		| "txJSON_1"| eg. txJSON_1= {"Ta..x","",signature}		tx JSON #1    | 32 		| "txJSON_2"	| eg. txJSON_2= {"","",signature}
// miner ipld address|46|"minerTAUaddr"	| minerTAUaddr = "Ta..x"; 
// previous hamt state root
//signature | 65   |"signature"|r: 32 bytes, s: 32 bytes, v: 1 byte, when at #6 same difficulty, high signature number wins.
}
miner TAU address blance |  5 				|minerTAUaddr + "balance" |Ta..xbalance= 1000; miner balance after the block execution to get tx fee

# Transactions: assume this block only include 2 transaction
  field        | Size     |  Key   |  Notes
----------------------|----------|--------|--------
tx 1 sender nounce  | 8      	|senderTAUaddr + "nounce"| hashTa..xnounce = 10; "0x1" similar to ETH nounce to prevent replay transactions, nounce is also used as power and message text key components.eq. TaddressNounce = 100

tx 2 sender nounce  | 8      	|senderTAUaddr + "nounce"| hashTa..xnounce = 20

.. | .. |..| an example of transaction #T JSON execution result
----------------------|----------|--------|--------
//sender's tx identifier for tx 1 |32	|senderTAUaddr + nounce +"hash" | Ta..xNounceHash = hash("Ta..x"+"10"); good for history msg direct reference with changing nounce
//sender TAU address | 20 		|Ta..x100Hash + "senderTAUaddr"|e.g hashTtxsenderTAUaddr = Ta..x
//relay_ipld_addr    | 46        		|Ta..x100Hash + "relayIPLDaddr" |hash("Ta..x"+"10")relayIPLDAddr = Qm...
//receiver TAU address | 20 		|Ta..x100Hash + "receriverTAUaddr"|e.g hash("Ta..x"+"10")receiverTAUaddr = Ta..x
//version        | 8        		|Ta..x100Hash  + "version" | "0x1" as initial default 
//roothash       | 32       		|Ta..x100Hash +  "roothash"| "0x0" similar to EOS TAPOS, witness of the stateroot within the mutable range point in a chosen state, must fill in to promote community engagement for high security and basic data knowledge
//timestamp      | 4       		|Ta..x100Hash +  "timestamp" |tx timestamp, tx expire in 12 hours
//amount        | 5        		|Ta..x100Hash+ "amount" |transfer amount
////txfee           | 1        		|Ta..x100Hash + "txfee" |transaction fee
//relayfee           | 1        		|Ta..x100Hash +"relayfee"  |relay fee, current version set to zero until the relay private key is supported to do wiring
# for Message and wiring transaction
.. | .. |..| an example of transaction #T JSON execution result
----------------------|----------|--------|--------
sender tx optcode    | 32       | senderNounceOptcoide = Ta..x100Hash +"optcode" |0 means self save, 1 means thread head, 2 means comments 
msg thread hash      | 65       |senderNouceThread = Ta..x100Hash + "thread" |the referred hash(writerTAUaddr+nounce) 
msg title/content      | 1024   |sendernouncecontentTa..x100Hash + "content" |The message; all messages forms up history

# state wide results
# for User info update transaction
.. | .. |..| an example of transaction #T JSON execution result
----------------------|----------|--------|--------
sender nick name      | 32         |senderTAUaddr + "name"| e.g imorpheus
sender contact info   | 65         |senderTAUaddr+ "contact"| your telegram id or any well know social media account
sender profile        | 1024       |senderTAUaddr + "profile"| user profile 
sender profile attachementSize |8|
sender profile attachment  |46 |


.. | .. |..| an example of transaction #T JSON execution result
----------------------|----------|--------|--------
sender balance        | 5       |senderTAUaddr + "balance" |
sender nounce | 8 |
relay balance        | 5        |relay_ipld_addr + "balance" |
receiver balance        | 5     |receriverTAUaddr + "balance" |
relay_maddr           | *       |relayIPLDaddr + "maddr" |Qm...maddr = {...}; in json, for mobile node connect to relay
sender IPLD address    | 4      |senderTAUaddr + "IPLDaddr" |Ta..xIPLDaddr=QMa...x; tx sender address in IPFS system, for locating tx file in IPFS, updated in each new tx
## how tx json look like


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
