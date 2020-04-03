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
## Three processes exists 
* A. Response with predicted ContractResultStateRoot, which is a hamt cbor.cid. 
* B. Collect votings from peers to find the safety state root for the chain. 
* C. File Downloader.

## Tries
* ChainIDContractAMTroot is the AMT trie for store chain contracts vector
* ChainIDContractResultStateRoot is the HAMT tree for the chain state;  contract and receipt are interconnected in each state transition on that chain. 
* FileAMTroot is the AMT trie for chopping and store file.

## Global Context: store in levelDB
It helps to make process internal data exchange efficient. 
```
* SafetyContractResultStateRoot; // this is constantly updated by voting process B. When most difficulty chain is found or verified after voting process.
* ContractResultStateRoot; // after found safety, this is the new contract state, which is a hamt node.cid
* RelayList is a list of known relays from Kademlia scope and TAU chain info; 
* ChainList is a list of Chains to follow/mine by users.
* PeerList[chain ID] is list of known peers for the chain by users.
```
## Concept
```
- Miner is what nodes call itself, in CBC POT all miners predicting future; Sender is what nodes call other peers. 
- Safety is the CBC concept of the safe and agreed history milestone.
- Mutable range is one week for now. 
- HamtGraphyRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replace the relay circuit, relay server will maintain connection to two peers. When cbor.cid is null, it means asking peer for the prediction chain CID, the target's future ContractResultStateRoot.
- AMTgraphRelaySync(relay multiaddress, remote ipfs peer, CID, selector); CID can not be null. 
- Address system: 
- TAU public key: the base for all address generation;
- Community chain ID: Tgenesisaddress + random number; new hamt node built with genesis state
- Community chains peer address format : chain ID + own address; 
- File operation transaction, FileRoot and nounce, the community is designed for handle file sharing, it list fileRoot nounce and related seeders in wormhole. After file published, other peers will use basic contract command to operate file, the first command is "seeding -i info".
- Principle of traverse, the relay random walk is based on Kademlia to find close distance of chain ID and relay ID, peer random walk is real random. Once in a relay+peer communication, we will not incur another recursive process to a new relay+peer to get supporting evidence, neither using witness list. if some vars are missing, just abort process to go next randomness contact. depth priority.  However for the file search and contract search, it is the width priority to do paralell download. 
```
## Wormhole - the bridge between HAMT state and AMT data set.
In each Chain stateroot, which is the contractResultStateRoot, we will setup wormhole for some future request. 
The wormholes are:
```
TsenderNounce, power
TsenderNounceContractAMTRoot, the entry for the transaction
TsenderBalance

FileAMTRootCommandNounce // for each file, this is the history of the commands. 
FileAMTRootCommandNounceContractAMTRoot

ChainIDFileNouce
ChainIDFileNouceRoot= fileAMTroot); // on one chain, file is a series. fileAMTroot is independant, even chain not existing, fileAMTroot can be referred by other chain. 
ChainIDFileNouceCommandNounce ++) // new File tx nonce=1; commenting on File nounce ++
ChainIDFileNonceCommandNouceTXContractRoot, contractAMTRoot); // everytime seeding/commenting, the nonce ++ and easy to find seeding hosts.


ChainIDcontractAMTRoot 
ChainIDcontractNumber
ChinaIDcontractAMTRootWitness

On TAU main chain, relay command
RelayNounce  - amt trie ???
RelayNounceRoot
```
# I. community chain - supports file sharing commands
genesis state gen, parameters: block size in number of txs, block time, chain nick name, coins total default is 1 million, initial peers ipfs address. // relay bootstrap is initially written in software until TAU mainet on.  
```
* ChainIDcontractAMTroot = amt_new node().
* generate ChainIDcontractAMTRoot.add({geneis}); 
* hamt_new node;
Genesis coinbase tx to get all coins
* generate hamt_update(Tminer..xBalance, 1,000,000); 
* generate Tminer..xNounce=1
* genreate Tminer..xNounceRoot := ChainIDcontractAMTRoot
* ChainIDContractReceitpStateRoot= hamt_put() 
* hamt_add(genesisStateRoot,ChainIDContractReceitpStateRoot)
* hamt_add(genesisAddress, Tminer..x)
* ChainIDContractReceitpStateRoot= hamt_put() 
```
## A One miner receives GraphSync request from a relay. 
Miner does not know which peer requesting them, because the relay shields the peers. Two types of requests: chainIDContractResultStateRoot and file.  In the random walk on relay, no recursively switching on relay, it relies on top random working, which is based on Kademlia distance. 
### A.1 for future ContractResultStateRoot
```
1. Receive the ChainID from a graphRelaySync call
2. If active on this chain, return: the future contractResultStateRoot for this chain, which is generated in B hamt_put
```
### A.2 For file relay, from a file downloader running a graphRelaySync（ relay, peer, chainID, cid, selector). 
File content trie is ***another trie*** called FileRoot in AMT/contractJSON, that is cross chain existing.

### B. Collect votings from peers: this process has two modes: miner and non-miner: 
In the peer randome walking, no recursively switching peers inside the loop, it relies on top random working. In the process of voting, the loose coupling along time is good practise to keep the new miners learning without influcence from external. This process is for multiple chain, multiple relay and mulitple peers.  
#### B1. non-mining users, which are on battery power or telecom data service. 

0. release android wake-lock and wifi-lock
```
code section H:
1. random walk to next followed chain id; random walk until connect to a next relay using Kademlia selection on TAU chain info.  
2. through relay, randomly request a chainPeer for the future receipt state root candidate according to CBC (correct by construction); 

graphRelaySync( Relay, peerID, chainID, null, selector(field:=contractAMTRoot)); 
// when CID is NULL,  - 0 means the relay will request ChainIDContractResultStateRoot from the peer via tcp

3. traverse history contract and states until mutable range.

stateroot = ChainContractResultStateRoot
(*) 
graphsyncAMT(contractAMTroot) 
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
senderProfileJSON,1024,Ta..xProfile; { TAU: Ta..x; relay:relay multiaddress: {}; telegram:/t/...; IPFS signature on TAU to proof its association. // verifier can decode siganture to get public key then hash to ipfs address-QM...; };

file command; if command is "create", it is a new File. otherwise, it is command, it is a seeding -l FileRoot/Nounce. // in app, we provide options for pause seeding or delete seeding file - i FileDescJSON,1024;//{ "file tx has to have File upload"}, no support for indepandent nick name tx, these info is sent along other tx. 
FileAMTRoot;
FileAMTCount,32; 
tx sender signature;
// regarding the File processing
// 1. tgz then use ipfs block standard size e.g. 250k to chop the data to m pieceis
// 2. newNode.amt(1,piece(1)); loop to newNode.hamt(m,piece(m));
// 3. FileAMTroot=AMT_flush_put()
// 4. return FileAMTroot and Count to contract Json. 
}
32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte
}
```
* generate contractAMTRoot = AMT_add(X); 
* generate contractNumber= AMT_get_count(SafetyContractResultStateRoot/contractAMTRoot number) +1
* generate contractAMTRootWitness, hamp_add( contractAMTrootWitness, {all the ipfs address in the contract} ); // for seeking ipfs peers for find contract. 

#### contract execute results
##### output coinbase tx
* generate key 2a, coinbase transaction hamt_update(Tminer..xBalance,Tminer..xBalance + amount); // update balance 
* generate TminerImopheus..xNounce++; for the sender side
* genreate TminerImppheus..xNounceRoot := contractAMTRoot
* generate TminerImopheus..xNounce++; for the receiving side
* genreate TminerImppheus..xNounceRoot := contractAMTRoot
##### output Coins Wiring tx, both sender and receive increase power, this is good for new users to produce contract.
* generate Key 2b. hamt_update(Tsender..xBalance,Tsender..xBalance - amount-txfee); 
* generate Key 5b, hamt_update(Ttxreceiver..xBalance,Ttxreceiver..xBalance + amount);
* generate Tsender..xNounce++
* genreate Tsender..xNounceRoot := contractAMTRoot
* generate Treceiver..xNounce++
* genreate Treceiver..xNounceRoot := contractAMTRoot
##### File operation transaction
```
File operation command in contractJSON
rootCreate: create hamt node for the file, return the root IPLD cid and number of blocks, set rootNounce = 1
- cat file| rootCreate -compress -chunk size -type file/video/voice -video_preview
e.g  cat starwar.mp4 | rootCreate -type video -video_preview; // return root hash and size of the preview
rootSeeding: graphRelaySync hamt node to local, and provide turn on seeding or off, set rootNounce ++
- rootSeeding fileRoot -seeding on/off
```
* generate Key 2c, hamt_update(Tsender..xBalance,Tsender..xBalance-txfee); 
* generate Tsender..xNounce++
* genreate Tsender..xNounceRoot := contractAMTRoot

* hamt_add(ChainIDFileNouce, fileNouce++);
* hamt_add(ChainIDFileNouceRoot, fileAMTroot); // on one chain, file is a series. fileAMTroot is independant, even chain not existing, fileAMTroot can be referred by other chain. 
* generate key 3c, hamt_update(ChainIDFileNouceCommandNounce ++) // new File tx nonce=1; commenting on File nounce ++
* generate key 4c, hamt_add(ChainIDFileNonceCommandNouceTXContractRoot, contractAMTRoot); // everytime seeding/commenting, the nonce ++ and easy to find seeding hosts.

6. Put the all new generated states into  cbor block, generate ContractResultStateRoot = hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid
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

# II. TAU Chain
TAU has only function that is relay configuration and related data payment settlement.
- wormhole
relay nounce/ relaynounce = ...
* hamt_update(relayNounce, relayNounce +1)
* hamt_add(RelayNounceAddr, new relay info)
