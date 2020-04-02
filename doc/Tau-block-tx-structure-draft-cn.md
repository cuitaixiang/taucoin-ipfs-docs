# TAU - share and preserve community data
```
User experienses:= {
data upload format include all types - directory, pictures and videos, 
data download from swarm and export to specific types eg mp4 with play media in app to avoid legal issue; 
community genesis independant coins for file sharing; community members can decide use tau relay annoucned on tau main chain or own relay setup. 
TAU mainnet does not hold files.
}
Business model:= {
TAU community decentral nodes will provide relay services charging TAU coins.
Tau nodes will have own policy for charging by annoucement; all chain address are derivative from TAU private key and use IPFS peers ID for internet connection (the association of TAUaddr and IPFSaddr is by signature on TAUaddr by ipfs private key
}
Launch steps:={
Free community creation for exchange data, such as TAUTest as initial test.
Tau nodes keeps graphyRelaySync logs in levelDB.
}
```
## Three processes exists: 
* A. Response with future ContractReceiptStateRoot; 
* B. Collect votings from peers; 
* C. Attachment Downloader.
## Backgound Context
* Safety SafetyContractReceiptStateRoot; // this is constantly updated by voting collecting process B. When most difficulty chain is found or verified after voting process.
* ContractReceiptStateRoot; // after found safety, this is the new contract state hamt node.cid
## Concept
```
. Miner is what nodes call themself, in CBC POT all miners predicting future; Sender is what nodes call other peers.
Safety is the CBC concept of the safe and agreed history milestone.
. Mutable range is one week.
. Community chain ID: Tgenesisaddress+Tgenesisaddress; new hamt node built with genesis state; if a community chain want to be searched through mainchain, it need to make an gensis annoucment with genesis address on tau chain, also with several bootstrap miners ipfs address, 
. Community chains peer address format : Tgenesisaddress address+own address; 
. TAU address: T.....
. After file published, other peers commenting is the seeding indication, which means hosting this file. 
. HamtGraphyRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replace the relay circuit, relay server will setup connection to peers. when cbor.cid is null, then asking for the prediction cid, the peer's ContractReceiptStateRoot.
. Address system: 
. TAU private key: the base for all address; TAU public Key hash to TAU address for main chain;
. AttachmentRoot and nounce, the community is designed for handle file sharing, rather than list all tx, it list files and seeders in global key-value pair
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
2. through relay, randomly request a chainPeer (get chainPeerIPFSaddr) for the future receipt state root candidate according to CBC (correct by construction); 
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
2. through relay, randomly request a chainPeer (get chainPeerIPFSaddr) for the future receipt state root candidate according to CBC (correct by construction); 
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
contract number = SafetyContractReceiptStateRoot/contract number)+1;
version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32; // for POT calc
miner TAU address, 20; Nounce, 8; // mining is treated as a tx sending to self
minerProfileJSON,1024; // e.g. TAUminerImorpheus..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

TXpoolJSON, flexible bytes; 
= { original JSON from peers for wiring and message; 
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 12 hours;
txfee;
senderProfileJSON,1024,Ta..xProfile; {
TAU: Ta..x; relay:relay multiaddress: {}; telegram:/t/...; 
IPFS private key sign TAU to proof, it is association. // verifier can decode siganture to get public key then hash to ipfs address -QM...; // tau address is the core
};

thread; if thread equal null, it is a new attachment file. otherwise, it is a seeding mark with comments link to ContractReceiptStateRoot. // in app, we provide options for not seeding or delete seeding 
attachmentDescJSON,1024;//{ "message has to have attachement"}
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
* generate TminerImopheus..xNounce++; for the sender side
* genreate TminerImppheus..xNounceRoot := ContractReceiptStateRoot
* generate TminerImopheus..xNounce++; for the receiving side
* genreate TminerImppheus..xNounceRoot := ContractReceiptStateRoot
##### output Coins Wiring tx
* generate Key 2b. hamt_update(TsenderImorpheus..xBalance,TsenderImorpheus..xBalance - amount-txfee); 
* generate Key 5b, hamt_update(Ttxreceiver..xBalance,Ttxreceiver..xBalance + amount);
* generate TsenderImopheus..xNounce++
* genreate TminerImppheus..xNounceRoot := ContractReceiptStateRoot
* generate Treceiver..xNounce++
* genreate Treceiver..xNounceRoot := ContractReceiptStateRoot
##### attachment transaction
* generate Key 2c, hamt_update(TsenderImorpheus..xBalance,TsenderImorpheus..xBalance-txfee); 
* generate TsenderImopheus..xNounce++
* genreate TsenderImppheus..xNounceRoot := ContractReceiptStateRoot
* generate key 3c,  hamt_update(attachementRootNounce ++) // new attachment nonce=1; commenting on attachment nounce ++
* generate key 4c, hamt_add(AttachmentRootTXStateJSONReceiptRoot, ContractReceiptStateRoot); // everytime seeding, the nonce ++ and easy to find seeding hosts. 



6. random walk until connect to a next relay
through graphrelaySync randomly request a chainPeer (get chainPeerIPFSaddr) for the future receipt state root candidate 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
```
7. mining by following the most difficult chain: if received futureContractReceiptStateRoot/ContractJSON shows a more difficult chain, then verify this chain's transactions for ONEWEEK
8. If verification succesful, hamt_update(SafetyContractReceiptStateRoot, ContractReceiptStateRoot), go to step (5); Else go to step (6)
9. if network-disconnected from internet 48 hours, go to step (1).

### C. Attachment Downloader
```
input (attachment root); // this is for single thread download
```
1. from attachmentroot, find nouce, loop the attachment nounce, find contractJSON/msgTx/IPFS peers id;
{ random walk on all relays
```
graphRelaySync(relay, chainID, chainPeerIPFSID, attachmentroot, selector(field:=section 1..m))
```
until finish all relays or find the chainPeer
}

* The download payment structure is charge TAUcoin per graphRelaysync block from TAUnodes. Therefore, the more peers paralell connections, the fast it is, the more expensive. 
