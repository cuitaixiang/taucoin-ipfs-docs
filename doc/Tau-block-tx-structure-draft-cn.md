# TAU - share and preserve big data
```
User experienses:= {
data upload as TGZ format include all types - directory, pictures and videos, 
data download from swarm and export to specific types eg mp4 with play media in app to avoid legal issue; 
community can genesis branch coins, or independant coins.
}
Business model:= {
community members can receive branch coin payment for hosting data; 
TAU coins community will provide relay routers charging TAU coins.
}
Launch steps:={
Free community creation for exchange data, such as TAUT as initial.
TAU mainchain for relay service
TAU branchchain for searching service
}
```
## Three processes exists: 
* A. Response with future ContractReceiptStateRoot; 
* B. Collect votings from peers; 
* C. Attachment Downloader.
## Backgound Context
* Safety SafetyReceiptStateRoot; // this is constantly updated by voting collecting process B. When most difficulty chain is verified after voting process, the new SafetyReceiptStateRoot is equal to predicted ContractReceiptStateRoot. 
* ContractReceiptStateRoot; // this is the new block/state hamt node.cid
```
Concept: Miner is what nodes call themself, in TAU all nodes are miners predicting future; Sender is what nodes call other peers.
```
### A. When a miner receives Graphysync request for producing future state, it will process following steps: 
* get CBC(consensus by construction) current safetyStateRoot, This is the result of voting by process B. If there is no safety found, such as out of mutable range, then exit with no response, continue to allow Collecting votes happenning. 
* hamt_get(safetyStateRoot,TAUminerImorpheus..xNounce); // miner's nounce is the base for coinbase transaction and new contract. If fail to get nounce, it means the node is too young and exist with no respones, continue to allow Collecting votes happenning.
input: none; backgroun context: safetyStateRoot; output: future contractReceiptStateRoot.
```
TAUminerImorpheus..x is an exmple of TAU miner address belong to imorpheus
```
* generate key 1 for the future state, hamt_update(TAUminerImorpheus..xNounce, TAUminerImorpheus..xNounce +1); 
* generate key 2 for the future state, hamp_add(contractJSON, X);

```
X = {
previousContractReceiptStateroot 32; // link to past state 
version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32; // for POT calc
miner TAU address, 20; TAUminerImorpheus..xNounce, 8; // mining is treated as a tx sending to self
minerProfileJSON,1024; // e.g. TAUminerImorpheus..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };
TXpoolJSON, flexible bytes; 
= { original JSON from peers for wiring and message; 
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 12 hours;
txfee;
contractblock number = previoushash(contract number)+1;
senderProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };
thread = "other sender address + tx nounce"; if thread equal self sender nounce, it is a new thread.
msgJSON,1024;//{ "hello world", this is a message.}
attachmentRoot = newNode.hamp_put(1-10000, sections of data); 
Attachment Size,32; 
tx sender signature;

// for attachment processing
// 1. TGZ the file; use ipfs block standard size e.g. 250k to chop the data to m pieceis
// 2. newNode.hamt(1,piece(1)); loop to newNode.hamt(m,piece(m));
// 3. attachmentRoot=hamp_flush_put()
// 4. return attachmentRoot and m to transaction Json. 

}
32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte
}
```
#### contract execute results
##### output coinbase tx

hamt_add(statejson, new block), this is new contract hash for execution  "hash98798234..sd"

* generate key 3a, coinbase transaction 
``` 
hamt_add(TAUminerImorpheus..xNounceTXOutputJSON =
{
opt_code, 0;// 0 - coinbase tx, 1 - means coin wiring, 2 - means data upload and message 
amount,5; // sum of transaction fees from the mining
}
```
* generate Key 4a. hamt_update(TAUminerImorpheus..xBalance,TAUminerImorpheus..xBalance + amount); // update balance
* generate Key 5a. hamt_add(TAUminerNounceAttachmentsLogJSON={ atachment1:graphsync request from VisitorTAUaddr; attachement 2...} // attachment host will charge coins for each graphsync on attachment as compensation, log is the proof, each new TAU address can get a mininum service from hosts such as 1G, it is configurable in the sendersProfile json. Attachment content trie is ***another trie*** called attachmentRoot in contractJSON
```
 hamt_add(TAUminerImorpheus..x11AttachmentsLogJSON={ "starwar:graphsync request from VisitorTAUaddr; attachement 2...} 
```

##### output Coins Wiring tx
* generate Key-3b. TAUsenderNounceTXOutputJSON = {}
``` 
hamt_add(TAUsender..xNounceTXOutputJSON =
{
opt_code, 1; // this is a wiring, very simple
}
```
* generate Key-4b. hamt_update(TAUsender..xBalance,amount); // update balance
* generate Key-5b hamt_update(TAUtxreceiver..xBalance,amount);

##### Message transaction
* generate Key 3c. TAUsenderNounceTXOutputJSON
``` 
hamt_add(TAUsender..xNounceTXOutputJSON =
{
opt_code, 2; // this is an attachment or message
}
```
* generate Key 4c hamt_update(TAUsender..xBalance,amount); // update balance


### Put the result ContractReceiptStateRoot = hamt_put(cbor); // this the for returning to requestor for future state prediction, "this state is my prediction"

### B. Collect votings from peers has two modes: miner and non-miner: 
#### B1. non mining users, which are on battery power or telecome data service. 
* node will maitain three sets of hamt keys
```
. TminerRelayTotal is a count of how many relays encountred. When found new relay information, hamt_update(TminerRelayTotal, TminerRelayTotal++);
. hamt_add(TminerRelayTminerrelayTotal, QmRelay..x); 
. TminerPeersTotal is a count of how many peers encountred. When found new peer information with (TAU and IPFS addr), hamt_update(TminerPeersTotal, TminerPeersTotal++);
. hamt_add(TminerPeerTAUTminerPeersTotal, Tsender..x); hamt_add(TminerPeerIPFSTminerPeersTotal, QmSender..x); 
```
0. release android wake-lock and wifi-lock
1. random walk until connect to a next relay; random walk until connect to a next peer; 
```
x=random(TminerRelaysTotal), y =random(TminerPeersTotal); 
multiaddress:= TminerRelay"x"/p2pcircuitRelay/TminerPeerIPFSTminer"y";
```
2. request the miner peer for the future receipt state root according to CBC (correct by construction); 
get contractJSON, get previous root, 
```
graphsync( multiaddress, CID, selector(field:=contractJSON)); 
// when CID is NULL,  - 0 means the first graphsync retrieve the future state cid
```
**problem** we do not know remote future cid at the moment, but we know the contractJSON as key from protocol definition.

3. traverse ONEWEEK history states from the miner and keep accounting of the root array; mining based on curent safty k to propose k+1 state with own tx uppon request; 
```
hamt_get(stateroot, "previous state root");
goto step (2); this time, we have the CID, that is the (future - 1) 
```
when connection timeout or hit any error, go to step(1)
4. accounting the new voting cid set, update the CBC safety root: SAFETYStateroot, hamt(SAFETYroot), then go to step (1).// think about put some keys into memory context.




#### B2. background procedures for mining user, which requires wifi and power plugged, while missing wifi or plug will switch to non-mining mode B1. 
```
input: votes of future; output: safetystateRoot
```

##### node will maitain three sets of hamt keys
```
. TminerRelayTotal is a count of how many relays encountred. When found new relay information, hamt_update(TminerRelayTotal, TminerRelayTotal++);
. hamt_add(TminerRelayTminerrelayTotal, QmRelay..x); 
. TminerPeersTotal is a count of how many peers encountred. When found new peer information with (TAU and IPFS addr), hamt_update(TminerPeersTotal, TminerPeersTotal++);
. hamt_add(TminerPeerTAUTminerPeersTotal, Tsender..x); hamt_add(TminerPeerIPFSTminerPeersTotal, QmSender..x); 
```
0. turn ON android wake-lock and wifi-lock

1. random walk until connect to a next relay; random walk until connect to a next peer; 
```
x=random(TminerRelaysTotal), y =random(TminerPeersTotal); 
multiaddress:= TminerRelay"x"/p2pcircuitRelay/TminerPeerIPFSTminer"y";
```
2. request the miner peer for the future receipt state root according to CBC (correct by construction); 
get contractJSON, get previous root, 
```
graphsync( multiaddress, CID, selector(field:=contractJSON)); 
// when CID is NULL,  - 0 means the first graphsync retrieve the future state cid
```
**problem** we do not know remote future cid at the moment, but we know the contractJSON as key from protocol definition.

3. traverse ONEWEEK history states from the miner and keep accounting of the root array; mining based on curent safty k to propose k+1 state with own tx uppon request; 
```
hamt_get(stateroot, "previous state root");
goto step (2); this time, we have the CID, that is the (future - 1) 
```
when connection timeout or hit any error, go to step(1)

4. accounting the new voting cid set, update the CBC safety root: SAFETYStateroot, hamt(SAFETYStateroot), then go to step (1)until half of the know mining peers are traversed.

5. if SAFETYStateRoot is out of mutable range, that is ONEWEEK, then go to step (1). 

6. random walk until connect to a next relay; random walk until connect to a next miner

7. start mining by following the most difficult chain: if received ContractReceiptStateRoot/ContractJSON shows a more difficult chain, then verify this chain's transactions ONEWEEK to the received future ContractReceiptStateRoot

8. If veification succesful, hamt(SAFETYStateroot, ContractReceiptStateRoot), go to step (6)

9. if self-disconnected from internet 48 hours, go to step (1).

10. goto step (1)

### C. Attachment Downloader
The downloader will use attachmentlog content to retrieve data from many peers. It will random walk on relays, but will focus on peers. Once connected, it will graphsync the data sections. The download coins payment structure is charge per attachment graphsync block. Therefore, the more peers paralell connections, the fast it is, the more expensive. 
