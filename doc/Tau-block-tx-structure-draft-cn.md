# TAU - share and preserve community data
```
User experienses:= {
data upload as TGZ format include all types - directory, pictures and videos, 
data download from swarm and export to specific types eg mp4 with play media in app to avoid legal issue; 
community can genesis independant coins.
}
Business model:= {
TAU coins community will provide relay routers charging TAU coins.
Community address will keep balance with TAU nodes, each tau nodes will have own policy for charging by annoucement
}
Launch steps:={
Free community creation for exchange data, such as TAUT as initial.
TAU mainchain tau nodes keeps logs.   file relay is charged and maintain a local list of accounts in levelDB. TAU mainchain does not support file attachment.  
}
```
## Three processes exists: 
* A. Response with future ContractReceiptStateRoot; 
* B. Collect votings from peers; 
* C. Attachment Downloader.
## Backgound Context
* Safety SafetyContractReceiptStateRoot; // this is constantly updated by voting collecting process B. When most difficulty chain is verified after voting process, the new SafetyContractReceiptStateRoot is equal to predicted ContractReceiptStateRoot. 
* ContractReceiptStateRoot; // this is the new block/state hamt node.cid
## Concept
```
Miner is what nodes call themself, in TAU all nodes are miners predicting future; Sender is what nodes call other peers.
Safety is the CBC concept of the safe and consensed history milestone.
Mutable range is one week
SafetyContractReceiptStateRoot is the local clock for miner; there is no definited global block clock, only global timestamp
Paraless chain: new hamt node with first tx by genesisaddress
Paraless chains peer address: genesis address+own address; 
TAU address: T.....

After file published, thread response could be seeding, which means hosting this file. 
HamtGraphyRelaySync(relay ipfs addr, remotePeerIPFS addr, cbor.cid, selector); // replace the relay circuit, relay server will setup connection to peers. 
when cbor.cid=null, it will get remote peer's ContractReceiptStateRoot.

address system: 
TAU private key: the base for all address; TAU public Key hash to TAU address for main chain;
Community chain address: "C"+ chain genesis member's TAU address + ownTAUaddress; if a community want to be searched on mainchain, it need to make an gensis annoucment with genesis address on tau chain using genesis private key to sign, also with several bootstrap miners ipfs address, 

```
### A.1 When a miner receives Graphysync request for producing future state

```
1. Receive the (genesis address); // this is the chain ID, TAU is 0x0;
2. If active on this chain, return: the future contractReceiptStateRoot for this chain, which is generated in B hamt_put
```
### A.2 For file relay, graphRelaySync（ relay, peer, cid, selector). 
File relay receive request for graphsync on content. graphsyncRelay(peerID, fileroot, selelctor_range). no need to setup connection to peer through relay, since know the peerID and root. file relay will setup connection will peers do a secondary graphysync
If successful, File relay will log this in
it will as well response with log root, log is the proof of graph sync history, each new TAU address can get a mininum service from hosts such as 1G, it is configurable in the sendersProfile json. Attachment content trie is ***another trie*** called attachmentRoot in contractJSON. each member need to pay tau to relay nodes from time to time, or provide seeding service as compenstion. several IPFS relay nodes can belong to one TAU address. Tau will use ipfs node private key to proof that relation in txJSON profile. For now, the relay service is free. 


### B. Collect votings from peers: this process has two modes: miner and non-miner: 
```
. RelayList is a level DB list of known relays
. ChainList is a level DB list of Chains to follow
. PeerList[chain ID] is level DB list of known peers 

```
#### B1. non-mining users, which are on battery power or telecome data service. 
0. release android wake-lock and wifi-lock
1. random walk to next chain id; random walk until connect to a next relay; 
2. through relay, randomly request a peer for the future receipt state root candidate according to CBC (correct by construction); 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
// when CID is NULL,  - 0 means the relay will request futureContractReceiptStateRoot from the peer via tcp
```
3. traverse mutable range history states from the peer and keep accounting of the ContractReceiptStateRoot
```
(*) stateroot = futureContractReceiptStateRoot
hamt_get(stateroot, "SafetyContractReceiptStateRoot");
do voting processing; 
goto (*) until the mutable range
goto step (2); 
```
when connection timeout or hit any error, go to step(1)
4. accounting the new voting, update the CBC safety root: hamt_update(SafetyContractReceiptStateRoot, voted SAFETY), 
5. build a future contractReceiptState; check B2(miner)- step 5.
6. then go to step (1).


#### B2. mining user, which requires wifi and power plugged, while missing wifi or plug will switch to non-mining mode B1. 

0. turn on android wake-lock and wifi-lock
1. random walk to next chain id; random walk until connect to a next relay; 
2. through relay, randomly request a peer for the future receipt state root candidate according to CBC (correct by construction); 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
// when CID is NULL,  - 0 means the relay will request futureContractReceiptStateRoot from the peer via tcp
```
3. traverse mutable range history states from the peer and keep accounting of the ContractReceiptStateRoot
```
(*) stateroot = futureContractReceiptStateRoot
hamt_get(stateroot, "SafetyContractReceiptStateRoot");
do voting processing; 
goto (*) until the mutable range
goto step (2); 
```
when connection timeout or hit any error, go to step(1)
4. accounting the new voting, update the CBC safety root: hamt_update(SafetyContractReceiptStateRoot, voted SAFETY), 
// initial voting done with a new safety
5. predict new future contract. 
* generate key 1 for ContractReceiptStateRoot, hamp_add(contractJSON, X);
```
//TAUminerImorpheus..x is an exmple of TAU miner address belong to imorpheus
X = {
SafetyContractReceiptStateRoot 32; // link to past the current state node.cid, and move to generate future
version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32; // for POT calc
miner TAU address, 20; Nounce, 8; // mining is treated as a tx sending to self
minerProfileJSON,1024; // e.g. TAUminerImorpheus..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };
TXpoolJSON, flexible bytes; 
= { original JSON from peers for wiring and message; 
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 12 hours;
txfee;
contract number = SafetyContractReceiptStateRoot(contract number)+1;
senderProfileJSON,1024,Ta..xProfile; {
TAU: Ta..x; relay:relay multiaddress: {}; telegram:/t/...; 
IPFS private key sign TAU to proof, it is association. // verifier can decode siganture to get public key then hash to ipfs address -QM...; // tau address is the core
};
thread = "other sender address + tx nounce"; if thread equal self sender nounce, it is a new thread. otherwise, it is a seeding mark with comments, seeding with comments. // in app, we provide options for not seeding or delete seeding 
msgJSON,1024;//{ "hello world", this is a message.}
attachmentRoot = newNode.hamp_put(1-10000, sections of data); 
Attachment Size,32; 
tx sender signature;
// regarding the attachment processing
// 1. use ipfs block standard size e.g. 250k to chop the data to m pieceis, TGZ each piece
// 2. newNode.hamt(1,piece(1)); loop to newNode.hamt(m,piece(m));
// 3. attachmentRoot=hamp_flush_put()
// 4. return attachmentRoot and m to transaction Json. 
}
32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte
}
```
#### contract execute results
Put the all new generated states into  cbor block, generate ContractReceiptStateRoot = hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid
##### output coinbase tx
* generate key 2a, coinbase transaction hamt_update(TminerImorpheus..xBalance,TminerImorpheus..xBalance + amount); // update balance 
* generate key 3a, hamt_add(TminerImorpheus..xNounce, nounce);// use for generate key TAUminerImorpheus..xNounceContractReceiptStateRoot
* generate key 4a, hamt_add(TminerImorpheus..xNounceContractReceiptStateRoot, ContractReceiptStateRoot);// linking to contractJSON via root, for fast link to other transactions by the same sender without go through the full chain.

##### output Coins Wiring tx
* generate Key 2b. hamt_update(TsenderImorpheus..xBalance,TsenderImorpheus..xBalance - amount-txfee); 
* generate key 3b, hamt_add(TsenderImorpheus..xNounce, nounce);
* generate key 4b, hamt_add(TsenderImorpheus..xNounceContractReceiptStateRoot, ContractReceiptStateRoot);
* generate Key 5b, hamt_update(Ttxreceiver..xBalance,Ttxreceiver..xBalance + amount);

##### Message transaction
* generate Key 2c. hamt_update(TsenderImorpheus..xBalance,TsenderImorpheus..xBalance-txfee); 
* generate key 3c, hamt_add(TsenderImorpheus..xNounce, nounce);
* generate key 4c, hamt_add(TsenderImorpheus..xNounceContractReceiptStateRoot, ContractReceiptStateRoot);
6. random walk until connect to a next relay
through graphrelaySync randomly request a peer for the future receipt state root candidate 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
```
7. mining by following the most difficult chain: if received futureContractReceiptStateRoot/ContractJSON shows a more difficult chain, then verify this chain's transactions for ONEWEEK
8. If verification succesful, hamt_update(SafetyContractReceiptStateRoot, ContractReceiptStateRoot), go to step (5); Else go to step (6)
9. if network-disconnected from internet 48 hours, go to step (1).

### C. Attachment Downloader
The downloader will use attachmentlog content to retrieve data from many peers. It will random walk on relays, but will focus on peers. Once connected, it will graphsync the data sections. The download coins payment structure is charge per attachment graphsync block. Therefore, the more peers paralell connections, the fast it is, the more expensive. 
find file relay; find peers. each node maintain one relay connection to state, and another relay to file. file relay will response with the content it holds, not blocks. 
