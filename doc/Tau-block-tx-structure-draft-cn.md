# TAU - Decentralized File Sharing Community
```
User experienses:= {
- File upload format is TGZ, which zips all types - directory, pictures and videos. TGZ will be choped and added into AMT with a filtAMTroot as return. TAU app does not provide native media player to avoid legal issue.
- Community creates community chains with own coins for file sharing. Each community chain ID is the creator TAUaddress+random(1,000,000,000)
- TAU mainnet does not hold files, only provide relay management service and settle payment on relay income. 
- All chain addresses are derivative from TAU private key. Nodes use IPFS peers ID for internet connection (the association of TAUaddr and IPFS address is by signature using ipfs RSA private key)
}
Business model:= {
- Tau foundation will develop TAU App and provide public relays for free, in return for admob/mopub income to cover AWS cost. Any one can add relay permission-lessly on TAU chain to share profit of the mobile ads. 
- On TAU, members can install and announce own relay to serve certain chain ID, configuerable for fee & service policy. Relay generates data flow to be paid in taucoin. Eventually, all relay nodes are provided by community and get paid by advertisement income. 
- Individual nodes will see ads to keep the data for free, the more data upload, the less ads to see. In app, show a stats of uploaded data, download data and ads time. 
}
Launch steps:={
- Free community creation for file sharing. TAUTest coin is an initial test. At this stage, TAU provide static relay service via AWS. 
- Tau main net turn on for community relay nodes joinning profit sharing.
}
```
## Three processes exist 
* A. Response with predicted `ChainID`ContractResultStateRoot, which is a hamt cbor.cid. 
* B. Collect votings from peers to find the chain id safety state root. 
* C. File Downloader.

## Tries
On community chain:
* chain contract: `ChainID`ContractAMTroot is the root for AMT trie for store chain contracts vector
* chain contract result: `ChainID`ContractResultStateRoot is the root for **HAMT** tree for the chain state; contract and receipt are interconnected in each state transition on that chain. 
* file amt: e.g Falksd898..x is the root for AMT trie for chopping and store the file.

On TAU chain: 
* relay: RelayAMTroot is the root for AMT trie for relay, relay is not chain specific

Trie elements linking
* future state/`ChainID`ContractAMTroot -> Contract AMT
* contract amt/`count`/state -> safety HAMT root

## Global Context: store in levelDB
It helps to make process internal data exchange efficient. 
* `ChainID`SafetyContractResultStateRoot; // this is constantly updated by voting process B. When most difficulty chain is found or verified after voting process.
* `ChainID`ContractResultStateRoot; // after found safety, this is the new contract state, which is a hamt node.cid
* UserChainList; // a list of Chains to follow/mine by users.
* UserFileAMTlist; // a list for storing user interested files 
* `ChainID`PeerList; // list of known peers for the chain by users.
* RelayList; // a list of known relays from TAU chain; initially will be hard-coded.

## Concept explain
```
- Miner is what nodes call itself, in CBC POT all miners predicting future; Sender is what nodes call other peers. 
- Safety is the CBC concept of the safe and agreed history milestone. The future contract result is a CBD prediction. 
- Mutable range is one week for now. 
- HamtGraphyRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replace the relay circuit, relay server will maintain connection to two peers. When cbor.cid is null, it means asking peer for the prediction chain CID, the target's future ContractResultStateRoot.
- AMTgraphRelaySync(relay multiaddress, remote ipfs peer, CID, selector); CID can not be null. 
- Address system: 
- TAU public key: the base for all address generation;
- Community chain ID: Tgenesisaddress + random number; new hamt node built with genesis state
- Community chains peer address format : chain ID + own address; 
- File operation transaction, FileRoot and nounce, the community is designed for handle file sharing, it list fileRoot nounce and related seeders in wormhole. After file published, other peers will use basic contract command to operate file, the first command is "seeding -i info".
File operation command:
//  rootCreate: create hamt node for the file, return the root IPLD cid and number of blocks, set rootNounce = 1
//  - cat file| rootCreate -compress -chunk size -type file/video/voice -video_preview
//  e.g  cat starwar.mp4 | rootCreate -type video -video_preview; // return root hash and size of the preview
//  rootSeeding: graphRelaySync hamt node to local, and provide turn on seeding or off, set rootNounce ++
//  - rootSeeding fileRoot -seeding on/off
- Principle of traverse, once in a relay+peer communication, we will not incur another recursive process to a new relay+peer to get supporting evidence. if some vars are missing, just abort process to go next randomness contact. depth priority.  However for the fileAMT search, it is the width priority to do paralell download. 
```
## Wormhole - Keys in the HAMT, hashed keys are wormhole inito contract AMT trie to get history proof. 
```
Chain specific:
`ChainID`contractAMTRoot  // for indexing the contract AMT trie entrie, `ChainID`ContractResultStateRoot/`ChainID`contractAMTRoot

`ChainID``Tsender/receiver`Nounce // indexing balance and pot power for each address
`ChainID``Tsender/receiver`Balance

`FileAMTroot``ChainID``SeedingNounce // for each file, this is the history of the seeding, first seeding is the creation. e.g. `Fiuweh87..x`SeedingNounce = 00189
`FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer // the seeding peer id for the file. eg. `Fiuweh87..x`Seeding`00187`IPFSpeer= QM....
```
# I. community chain - supports file sharing commands
**Genesis** state generate, parameters: block size in number of txs, block time, chain nick name, coins total default is 1 million, initial peers ipfs address. // relay bootstrap is initially written in software until TAU mainet on.  
```
* `ChainID`contractAMTroot = amt_new node(). // root for contact AMT
* generate genesis contract, 
ChainID = `Tminer`+ random(1,000,000,000)
X = `ChainID`contractAMTRoot.add({
blocksize;
blocktime;
initial difficulty;
chain nickname;
total coins
initial IPFS peers json;
}); 
* new `ChainID`SafetyContractResultStateRoot = new hamtnode()
* hamt_add(ChainID,`ChainID`);
* hamt_add(`ChainID`contractAMTroot, X); 
* hamt_add(`Tminer`Balance, 1,000,000); 
* hamt_add( `Tminer`Nounce=1
* hamt_add(genesisAddress, `Tminer`) // add genesis address wormhole
* hamt_add(other KVs)
* `ChainID`ContractReceitpStateRoot= hamt_put() 
* levelDB vars update
* levelDB.`ChainID`SafetyContractResultStateRoot = `ChainID`ContractReceitpStateRoot
* levelDB.`ChainID`ContractResultStateRoot=`ChainID`SafetyContractResultStateRoot; 
* levelDB.UserChainList.add(`ChainID`)
* levelDB.`ChainID`PeerList.add(`Tminer);
* levelDB.RelayList.add({aws relays by taucoin dev});

```
## A. One miner receives GraphSync request from a relay. 
Miner does not know which peer requesting them, because the relay shields the peers. Two types of requests: "chainIDContractResultStateRoot" and `fileAMTroot`. 
### A.1 for future `ChainID`ContractResultStateRoot
```
1. Receive the `ChainID` from a graphRelaySync call
2. If active on this chain, return leveldb.`ChainID`contractResultStateRoot, which was generated in B process hamt_put
```
### A.2 For file relay, from a file downloader running a graphRelaySync（ relay, peer, chainID, `fileAMTroot`, selector). 
If the `fileAMTroot` exists, then return the blocks. 

### B. Collect votings from peers: this process has two modes: miner and non-miner: 
In the peer randome walking, no recursively switching peers inside the loop, it relies on top random working. In the process of voting, the loose coupling along time is good practise to keep the new miners learning without influcence from external. This process is for multiple chain, multiple relay and mulitple peers.  
#### B1. non-mining users, which are on battery power or telecom data service. 

0. release android wake-lock and wifi-lock
```
code section H:
1. random walk to next followed chain id; random walk connect to a next relayAMT in the chainlist using Kademlia selection.  
2. through relay, randomly request a chainPeer from chainIDpeerlist for the future contract result state root.  

graphRelaySync( Relay, peerID, chainID, null, selector(field:=`ChainID`contractAMTRoot)); 
// when CID is NULL,  - 0 means the relay will request `ChainID`ContractResultStateRoot from the peer via tcp

3. traverse history contract and states until mutable range.

stateroot = `ChainID`ContractResultStateRoot
(*) 
graphsyncAMT(`ChainID`contractAMTroot) 
contractJson = amt_get(contractAMTroot )
stateroot= contractJSON/SafetyContractResultStateRoot // recursive getting previous stateRoot to move into history
graphsyncHAMT(SafetyContractResultStateRoot) 
contractAMTroot = hamt_get(stateroot, contractAMTRoot)
goto (*) until the mutable range; // 
goto step (2); 

when connection timeout or hit any error, go to step(1)

4. accounting the new voting, update the CBC safety root: levelDB_update(SafetyContractResultStateRoot, voted SAFETY), 
``` 
5. build a future contractResultState; use B2 - step 5.
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
SafetyContractResultStateRoot 32; // link to current safety state node.cid, and move to generate future
contractNumber = AMT_get_count(SafetyContractResultStateRoot/contractAMTRoot number) +1;
version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32; // for POT calc
miner TAU address, 20; Nounce, 8; // mining is treated as a tx sending to self
minerProfileJSON,1024; // e.g. TAUminer..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; IPFS signature on TAU to proof its association. // verifier can decode siganture to get public key then hash to ipfs address-QM...; };
TXsJSON, flexible bytes; 
= { original JSON from peers for wiring and message; 
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 12 hours;
txfee;
senderProfileJSON,1024; { TAU: Ta..x; relay:relay multiaddress: {}; telegram:/t/...; IPFS signature on TAU to proof its association. // verifier can decode siganture to get public key then hash to ipfs address-QM...; };

seeding command; if command is "create", it is a new File. otherwise, it is command, it is a seeding -l FileRoot/Nounce. // in app, we provide options for pause seeding or delete seeding file - i FileDescJSON,1024;//{ "file tx has to have File upload"}, no support for indepandent nick name tx, these info is sent along other tx. 
FileAMTRoot;
tx sender signature;
// regarding the File processing
// 1. tgz then use ipfs block standard size e.g. 250k to chop the data to m pieceis
// 2. newNode.amt(1,piece(1)); loop to newNode.hamt(m,piece(m));
// 3. FileAMTroot=AMT_flush_put()
// 4. return FileAMTroot to contract Json. 
}
32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte
}
```
* hamt_update(`ChainID`contractAMTroot, chainIDcontractAMTroot.add(X)); 

#### contract execute results
##### output coinbase tx
* hamt_update(`Tminer`Balance,`Tminer`Balance + amount); // update balance 
* hamt_update(`Tminer`Nounce,`Tminer`Nounce++); // for the coinbase tx nounce increase
##### output Coins Wiring tx, both sender and receive increase power, this is good for new users to produce contract.
Account operation
* hamt_update(`Tsender`Balance,`Tsender`Balance - amount-txfee); 
* hamt_update(`Ttxreceiver`Balance,`Ttxreceiver`Balance + amount);
* hamt_update(`Tsender`Nounce,`Tsender`Nounce++);
* hamt_update(Treceiver..xNounce,Treceiver..xNounce++);
##### File creation and seeding transaction
Account operation
* hamt_update(`Tsender`Balance,`Tsender`Balance-txfee); 
* hamt_update(`Tsender`Nounce++)
File operation
* fileAMTroot = new AMTnode().put(file) // tgz, chop and put file into AMT trie, return the root
* `fileAMTroot``ChainID`SeedingNounce++  
* `FileAMTroot``ChainID`SeedingNounceIPFSpeer = seeding peer id

6. Put new generated states into  cbor block, levelDB.add `ChainID`ContractResultStateRoot = hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid
7. random walk until connect to a next relay
through graphrelaySync randomly request a chainPeer (get chainPeerIPFSaddr) for the future receipt state root candidate 
```
graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); 
```
8. mining by following the most difficult chain: if received futureContractResultStateRoot/ContractJSON shows a more difficult chain, then verify this chain's transactions for ONEWEEK
9. If verification succesful, levelDB_update(SafetyContractResultStateRoot, futureContractResultStateRoot), go to step (5) to get a new state prediction; Else go to step (7)
10. if network-disconnected from internet 48 hours, go to step (1).
### C. File Downloader
```
input (File root); // this is for single thread download

From Fileroot, find nouce, loop the File nounce, find contractJSON/msgTx/IPFS witness;
{ random walk on all relays

graphRelaySync(relay, chainID, chainPeerIPFSID, File, selector(field:=section 1..m))

until finish all relays or find the chainPeer
}
```
# To do
## II. TAU Chain
TAU has only function that is relay configuration and related data payment settlement.
- wormhole
relay nounce/ relaynounce = ...
* hamt_update(relayNounce, relayNounce +1)
* hamt_add(RelayNounceAddr, new relay info)
## Kademlia on relay and peers selection
## community relay data flow tracking for profit sharing. 
