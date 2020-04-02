# TAU - Decentralized File Sharing Community
```
User experienses:= {
- File upload format is TGZ, which zips all types - directory, pictures and videos. TGZ will be choped and added hamt node. Downloaded file, which is a hamt node, export to specific types, eg mp4, without native media play to avoid legal issue. 
- Community creates independant chains with new coins for file sharing.
- TAU mainnet does not hold files, only provide relay meta data service. 
- All chain addresses are derivative from TAU private key and use IPFS peers ID for internet connection (the association of TAUaddr and IPFSaddr is by signature by ipfs RSA private key)
}
Business model:= {
- Tau foundation will develop TAU App and provide public relays for free, in return for admob/mopub income to cover AWS cost. Any one can add relay permission-lessly on TAU to share profit. 
- On TAU, members can install and announce own relay restricted to serve certain chain ID, configuerable for fee & service policy. Relay generates data flow, and it will be paid in taucoin. Eventually, all relay nodes are provided by community and get paid by advertisement income. 
- Individual nodes will see ads to keep the data for free, the more data upload, the less ads to see. In app, show a balance of uploaded data and ads time. 
}
Launch steps:={
- Free community creation for file sharing. TAUTest coin is an initial test. At this stage, TAU provide static relay service via AWS
- Tau main net turn on for community relay nodes joinning profit sharing.
}
```
## Three processes exists 
* A. Response with future ContractReceiptStateRoot, which is a cbor.cid and an anchor for hamt state retrieve. 
* B. Collect votings from peers to find the safety state root for the chain. 
* C. File Downloader.
## Global Context: store in levelDB
It helps to make process internal data exchange efficient. 
```
* SafetyContractReceiptStateRoot; // this is constantly updated by voting process B. When most difficulty chain is found or verified after voting process.
* ContractReceiptStateRoot; // after found safety, this is the new contract state, which is a hamt node.cid
* RelayList is a list of known relays from Kademlia scope and TAU chain info; 
* ChainList is a list of Chains to follow/mine by users.
* PeerList[chain ID] is list of known peers for the chain by users.
```
## Concept
```
- Miner is what nodes call itself, in CBC POT all miners predicting future; Sender is what nodes call other peers. 
- Safety is the CBC concept of the safe and agreed history milestone.
- Mutable range is one week for now.
- After file published, other peers commenting is considerred as seeding indication. 
- HamtGraphyRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replace the relay circuit, relay server will maintain connection to two peers. When cbor.cid is null, it means asking for the prediction cid, the target's future ContractReceiptStateRoot.
- Address system: 
- TAU public key: the base for all address generation;
- Community chain ID: Tgenesisaddress + random number; new hamt node built with genesis state
- Community chains peer address format : chain ID + own address; 
- FileRoot and nounce, the community is designed for handle file sharing, it list fileRoot nounce and related seeders in global key-value state.
- Principle of traverse, the relay random walk is based on Kademlia, peer random walk is real random. Once in a relay+peer communication, we will not incur another recursive process to a new relay+peer to get supporting evidence. if some vars are missing, just abort process to go next randomness contact. depth priority.  However for the file search, it is the width priority to do paralell download. 
```
## Wormhole
In each stateroot, which contractReceiptStateRoot, we will setup wormhole for some future var request. Wormhole prevents the future user screening the whole blockchain. The wormholes are:
```
TsenderNounce, power
TsenderNounceRoot, the entry for the transaction
TsenderBalance

FileRootNounce
FileRootNounceContractReceiptStateRoot

ChainRelayNounce
ChainRelayNounceRoot
```
# I. community chain - supports file sharing
genesis parameters: block size in number of txs, frequency, chain nick name, coins total default is 1 million, initial peers ipfs address. // relay bootstrap is on tau chain via Kademlia, before that it is written in software. 
```
hamt_new node().
hamt_add(contractJson, {genesis});
hamt_put  -> ContractReceitpStateRoot
add(genesisStateRoot,ContractReceitpStateRoot)
add(genesisAddress, Tminer..x)
nounce, balance, ... TBD
```
## A One miner receives GraphSync request from a relay. 
Miner does not know which peer contacting, because of the relay covers the peers. Two types of requests: stateRoot and file.  In the random walk on relay, no recursively switching on relay, it relies on top random working, which is based on Kademlia distance. 
### A.1 for future ContractReceiptStateRoot
```
1. Receive the (genesis address); // this is the chain ID, TAU is 0x0;
2. If active on this chain, return: the future contractReceiptStateRoot for this chain, which is generated in B hamt_put
```
### A.2 For file relay, from a graphSync of a graphRelaySync（ relay, peer, chainID, cid, selector). 
File relay receive request for graphsync on content. graphsyncRelay(peerID, fileroot, selelctor_range). no need to setup connection to peer through relay, since know the peerID and root. file relay will setup connection will peers do a secondary graphysync
If successful, File relay will log this in
it will as well response with log root, log is the proof of graph sync history, each new TAU address can get a mininum service from hosts such as 1G, it is configurable in the sendersProfile json. File content trie is ***another trie*** called FileRoot in contractJSON. each member need to pay tau to relay nodes from time to time, or provide seeding service as compenstion. several IPFS relay nodes can belong to one TAU address. Tau will use ipfs node private key to proof that relation in txJSON profile. For now, the relay service is free. 


### B. Collect votings from peers: this process has two modes: miner and non-miner: 
In the peer randome walking, no recursively switching peers inside the loop, it relies on top random working. In the process of voting, the loose coupling along time is good practise to keep the new miners learning without influcence from external. This process is for multiple chain, multiple relay and mulitple peers.  
#### B1. non-mining users, which are on battery power or telecom data service. 

0. release android wake-lock and wifi-lock
```
code section H:
1. random walk to next followed chain id; random walk until connect to a next relay using Kademlia and TAU chain info; chain id has relationship with Kademlia distance.  
2. through relay, randomly request a chainPeer for the future receipt state root candidate according to CBC (correct by construction); 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
// when CID is NULL,  - 0 means the relay will request futureContractReceiptStateRoot from the peer via tcp
```
3. traverse mutable range history states
```
stateroot = futureContractReceiptStateRoot
(*) 
hamt_get(stateroot, "contractJSON");
stateroot= contractJSON/SafetyContractReceiptStateRoot // recursive getting previous stateRoot to move into history
goto (*) until the mutable range; // 
goto step (2); 
```
when connection timeout or hit any error, go to step(1)

4. accounting the new voting, update the CBC safety root: levelDB_update(SafetyContractReceiptStateRoot, voted SAFETY), 
``` 
5. build a future contractReceiptState; use B2 - step 5.
6. then go to step (1).


#### B2. mining user, which requires wifi and power plugged, while missing wifi or plug will switch to non-mining mode B1. 

0. turn on android wake-lock and wifi-lock
```
copy code section H from B1.
```
5. predict new future contract. 
```
//TAUminerImorpheus..x is an exmple of TAU miner address belong to imorpheus
X = {
SafetyContractReceiptStateRoot 32; // link to past the current state node.cid, and move to generate future
contract number = SafetyContractReceiptStateRoot/contract number +1;
version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32; // for POT calc
miner TAU address, 20; Nounce, 8; // mining is treated as a tx sending to self
minerProfileJSON,1024; // e.g. TAUminer..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; IPFS signature on TAU to proof its association. // verifier can decode siganture to get public key then hash to ipfs address-QM...; };
TXsJSON, flexible bytes; 
= { original JSON from peers for wiring and message; 
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 12 hours;
txfee;
senderProfileJSON,1024,Ta..xProfile; { TAU: Ta..x; relay:relay multiaddress: {}; telegram:/t/...; IPFS signature on TAU to proof its association. // verifier can decode siganture to get public key then hash to ipfs address-QM...; };

thread; if thread equal null, it is a new File. otherwise, it is a seeding mark with comments link to FileRoot/Nounce. // in app, we provide options for pause seeding or delete seeding file
FileDescJSON,1024;//{ "file tx has to have File upload"}, no support for indepandent nick name tx, these info is sent along other tx. 
FileRoot = newNode.hamp_put(1-10000, sections of data); 
File Size,32; 
tx sender signature;
// regarding the File processing
// 1. tgz then use ipfs block standard size e.g. 250k to chop the data to m pieceis
// 2. newNode.hamt(1,piece(1)); loop to newNode.hamt(m,piece(m));
// 3. FileRoot=hamp_flush_put()
// 4. return FileRoot and m to transaction Json. 
}
32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte
}
```
* generate key 1 for hamp_add(contractJSON, X);
Put the all new generated states into  cbor block, generate ContractReceiptStateRoot = hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid

#### contract execute results
##### output coinbase tx
* generate key 2a, coinbase transaction hamt_update(Tminer..xBalance,Tminer..xBalance + amount); // update balance 
* generate TminerImopheus..xNounce++; for the sender side
* genreate TminerImppheus..xNounceRoot := ContractReceiptStateRoot
* generate TminerImopheus..xNounce++; for the receiving side
* genreate TminerImppheus..xNounceRoot := ContractReceiptStateRoot
##### output Coins Wiring tx, both sender and receive increase power, this is good for new users to produce contract.
* generate Key 2b. hamt_update(Tsender..xBalance,Tsender..xBalance - amount-txfee); 
* generate Key 5b, hamt_update(Ttxreceiver..xBalance,Ttxreceiver..xBalance + amount);
* generate Tsender..xNounce++
* genreate Tsender..xNounceRoot := ContractReceiptStateRoot
* generate Treceiver..xNounce++
* genreate Treceiver..xNounceRoot := ContractReceiptStateRoot
##### File transaction
* generate Key 2c, hamt_update(Tsender..xBalance,Tsender..xBalance-txfee); 
* generate Tsender..xNounce++
* genreate Tsender..xNounceRoot := ContractReceiptStateRoot
* generate key 3c,  hamt_update(FileRootNounce ++) // new File tx nonce=1; commenting on File nounce ++
* generate key 4c, hamt_add(FileRootNouceTXStateJSONReceiptRoot, ContractReceiptStateRoot); // everytime seeding/commenting, the nonce ++ and easy to find seeding hosts.

6. random walk until connect to a next relay
through graphrelaySync randomly request a chainPeer (get chainPeerIPFSaddr) for the future receipt state root candidate 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
```
7. mining by following the most difficult chain: if received futureContractReceiptStateRoot/ContractJSON shows a more difficult chain, then verify this chain's transactions for ONEWEEK
8. If verification succesful, levelDB_update(SafetyContractReceiptStateRoot, futureContractReceiptStateRoot), go to step (5) to get a new state prediction; Else go to step (6)
9. if network-disconnected from internet 48 hours, go to step (1).
### C. File Downloader
```
input (File root); // this is for single thread download

From Fileroot, find nouce, loop the File nounce, find contractJSON/msgTx/IPFS peers id;
{ random walk on all relays

graphRelaySync(relay, chainID, chainPeerIPFSID, File, selector(field:=section 1..m))

until finish all relays or find the chainPeer
}
```

# II. TAU Chain
TAU has only function that is relay configuration and related data payment settlement.
- wormhole
relay nounce/ relaynounce = ...
* hamt_update(relayNounce, relayNounce +1)
* hamt_add(RelayNounceAddr, new relay info)
