# TAU - File sharing on blockchains

User experienses:= { 用户体验
- Core: 1. create sharing blockchain 2. send coins to friends.  3. seeding a file (with create new chain ability)
   - Data dashboard: if (download - upload) > 1G, start ads. "seeding to increase free data"
- File/video imported to TAU will be compressed and chopped by TGZ, which includes directory zip, pictures and videos. Chopped file pieces will be added into AMT (Array Mapped Trie) with a `fileAMTroot` as return. Filed downloaded could be decompressed to original structure.  Files downloaded is considerred imported. Only seeded file will be pinned in local. 

- For all videos, we will support a `hopping player` to only play the downloaded pieces. Video download options: download 1%, 5%, 25% or full through the count value.

- TAU provides basic annoucement service include free public relays and genesis. 
- Community chain can use own chain for relay recommendation. 

- All chain addresses are derivative from one private key. Nodes use IPFS peers ID for ipv4 tcp transport. (the association of TAUaddr and IPFS address is through signature using ipfs RSA private key).
}

Business model:= { 商业模式
- Tau foundation will develop TAU App and provide free public relays (TAU dev private key signed), in return for admob/mopub ads income to cover AWS data cost. Any one can config relay on own chain. 
- Individual nodes will see ads to keep using data for free, the more data upload, the less ads to see. In app, show a stats of uploaded data, download data. When users getting data from community signed relay, the TAU app will not counting community relay download. 
- TAU coin price will rise when cross-chain communications in demand. 
- TAUT is tau-torrent, a file sharing service by tau dev
- TAU is a relay annoucement service by tau dev. 
}

## Two Tries
* chain contract result hamt trie: ContractResultStateRoot is the chain state hamt root; contract and results are connected in each state transition. 
> future state ->ContractJSON

> ContractJSON -> SafetyContractResultStateRoot

* file AMT: the root for AMT trie for chopping and storing the file.

## Five core processes
> Hamt trie.
  * A. Response with predicted ContractResultStateRoot, which is a hamt cbor.cid. (service response to HamtGraphRelaySync). One instatnce per connection to prevent ddos. a call-back registerred in libp2p. 
  * B. Collect votings from chain peers to discover the ChainID's safety state root. (single thread func)
> Amt trie.
  * C. File Downloader. (download files and logging download data)
  * D. Reponse AMT cbor.cid to file downloader request. (service response to AMTGraphRelaySync and logging upload data). One instatnce per connection to prevent ddos.  改到以chain 为服务单位
> E. Process manager, main(); schedule above 4 processes instance existing and prevent DDOS. 
> F. Resource management: states key value on chain and before mutabale ranged will be pinned, seeded files will be pined, others are unpinned to garbage collection. 


## Operation variables in database, in each transition, following variables will be populated from wormhole kv. Wormhole kv is a state consensus, local db variables is for program to operate on these states. 
* mySafetyContractResultStateRoots              map[ChainID] cbor.cid;
* mySafetyContractResultStateRootMiners         map[ChainID] address;
* myPreviousSafetyContractResultStateRootMiners map[ChainID] address; // if current safety miner = previous safety miner, then the miner is treated as disconnected or new, so go to voting. 
* myContractResultStateRoots                    map[ChainID] cbor.cid; // after found safety, this is the new contract state
* myChains                                      map[ChainID] config; // a  list of Chains to follow/mine by users, string is for potential config
* myFileAMTroots                                map[AMTroot] filename; // a  list for imported and downloaded files trie

* myPeers      map[`ChainID`][]String; // known IPFS peers in the chain
* myTXsPool    map[`ChainID`][]String; // verified txs for adding to new state prediction

* myRelays        [] * struct {ChainID; RelayAddr; date}; // known relays for the chain; setup a chainID called "successed" with historically successful relays. Date is used for only selelct relays in the mutalble range. 
* myDownloadPool  [] * struct {ChainID; FileAMT cbor.cid; count int; isPause boolean} // a slice of struct
* mySeedersDB     [] * struct {fileAMT; ChainID; seeder}   // one file can exist on many chains.

* mytotalFileAMTDownloadedData
* mytotalFileAMTUploadedData


## Concept explain
- Single thread principle for mobile phone, we do not put wait time in thread, but only support one thread for each functions. The more chain mining, the lower speed on each chain. 
- Block size is 1, only one tx included in each block. encouraginig increase the frequency, which is reducing the block time.
- Miner is what nodes call itself, and sender is what nodes call other peers. In TAU POT, all miners predict a future state; 
- Safety is the CBC Casper concept of the safe and agreed history by BFT group. The future contract result state is a CBC prediction. TAU uses this concept to describe the prediction and safety as well, but our scope are all peers than BFT group.
- Mutable range is defined as "one week" for now, beyond mutalble range, it is considered finality.

- HamtGraphRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replaced the IPFS relay circuit. When cbor.cid is null, it means asking peer for the ContractResultStateRoot prediction on the chainID.
- AMTgraphRelaySync(relay multiaddress, remote ipfs peer, cbor.cid, selector); cid can not be null. AMT is not chain specific, and it is rather relating to IPFS peers. 
- In both GraphRelaySync, it needs to test wether the target KV holding cbor.cid are already in local or not. 

- File operation transaction, FileAMTRoot creation and seeding, related nounce and seeders accessible through wormhole. 文件操作
- Principle of traverse, once in a "ChainID+relay+peer" communication, we will not incur another recursive process to a new peer to get supporting evidence. If some Key-values are missing to prevent validation, just abort process to go next randomness. We use E. process manager to create top level concurrency. For mobile device,concurrency is hard to manage due to memory restraint.

- Address system: 
- TAU private key: the base for all community chain address generation;
- TAU Chain ID = "0"
- ChainID := `Nickname`+`blocktime` + signature(random) // in stateless environment, chain config info needs to be embedded into chainname, otherwise, might be lost. 
- Community chains peer address format : `chainID` + `TAU address`; 

- TX types and msg: the contract content for coin base, wiring and file tx
   * coin base, msg is the only transaction attached
   * wiring, msg is the contract relating to this tx
   * relay, msg is the contract relating to this tx, include the relay info
   * file, msg is the contract relating to this tx, include discription of the file or future file command
   
- relay: each chain config relay on own chain by members, TAU mainchain annouce the relay candidates in the daily basis, each node config own successed relays. three of those sharing the time slots.   
   

## "Wormhole" - HAMT Hashed keys are states inito contract chain history. 

genesisAddress = TAUaddress
// TX oriented 
Wiring transactions
- `Tsender/receiver`TXnounce; //  balance and POT power for each address 总交易计数
- `Tsender/receiver`Balance
- `Tsender/receiver`TXnounce`Msg // for future command, such as "seeding all up to 1G" comment, no file attachment
File transactions
- `Tsender`FileNounce // file command counting 文件交易计数
- `Tsender`File`Nounce`FileAMTroot // when user follow a chain address, they can traverse its files through changing nounce. 
- `Tsender`File`Nounce`Msg

// entity oritened
File seeding info in a chain
- `FileAMTroot`SeedingNounce // for each file, this is the total number of registerred seeders, first seeding is the creation.
- `FileAMTroot``Seeding`Nounce`IPFSPeer // the seeding peer id for the file. 

Relay of a chain ID 
- RelayNounce  // recording the relays counter, these relays are own chain annouced relays. The TAU relays are in the levelDb. Relay pool are the combination of TAU relays and own chain relays. 
- RelayNounceAddress // recording the relay address

## Constants
* MutableRange:  1 week
* TXExpiry: transaction expirey 24 hours
* VotingPercentage: voting cover percentage 67%
* RelaySwitchTimeUnit: relay time base, 15 seconds, which is what peers comes to their scheduled relays. 
* WakeUpTime: sleeping mode wake up random range 5 minutes
* SelfMiningTime: self mining qualify time 60 minutes. 
* GenesisDefaultCoins: default coins 1,000,000
* initial difficulty according to the BlockTime.
* block size = 1 transaction fixed
* BlockTimeDefault = default 5 minutes

## Community chain
### Genesis
* with parameters: block time, chain nick name, coins total - default is 1 million.  // initial mining peers is established through issue coins to other addresses. 社区链创世区块
```
// build genesis block
Y:= { 
ChainID := `Nickname`+ `blocktime` + hash(signature(random)) // chainID is the only information to pass down in the stateless mode.
SafetyContractResultRoot = null; // genesis is built from null.
contractNumber:=0 int32;
initial difficulty int64; // ???
totalCoins int64; // GenesisDefaultCoins， 币数量
`Tminer`TXnoucne:=0;
`Tminer`FileNounce:=0;
msg;
signature []byte //by genesis miner
}
// build genesis state
* X := hamt_node := null new.hamt_node(); // execute once per chain, for future all is put.
* stateroot.hamt_add(ChainID, `Nickname`+`blocktime` + hash.signature(random));用创世矿工的TAU私钥签署 randomness
* stateroot.hamt_add(contractJSON, Y) 
* stateroot.hamt_add(SafetyContractResultRoot = null; // genesis is built from null.
* stateroot.hamt_add(`Tminer`Balance, 1,000,000); 
* stateroot.hamt_add(`Tminer`TXNounce, 0);
* stateroot.hamt_add(`Tminer`TXNounceMsg,msg);
* stateroot.hamt_add(`Tminer`FileNounce, 0);
* stateroot.hamt_add(genesisAddress, `Tminer`); // add genesis address wormhole

* myContractResultStateRoots[`ChainID`]=hamt_node.hamt_put(cbor); // for responding to voting.
* mySafetyContractResultStateRoots[`ChainID`] = null;
* mySafetyContractResultStateRootMiners[`ChainID`] = Tminer;
* mypreviousSafetyContractResultStateRootMiner[`ChainID`] = null;
* myChains.add(`ChainID`:"")
* myPeers[`ChainID`].add(`Tminer);

// no need to config relay, myRelays[TAU] will be populated when system starts in process E and community chains will use time slots to touch TAU relays. 

```
## A. One miner receives GraphSync request from a relay.  
Miner does not know which peer requesting them, because the relay shields the peers. Two types of requests: "ContractResultStateRoot[]" or null. 
-  Receive the `ChainID` from a graphRelaySync call
-  If `ChainID` exist in myChains, return myContractResultStateRoots[`ChainID`] and blocks according to selector; else response null

## B. Votings, chain choice and state generation
This process is for multiple chain, multiple relay and mulitple peers.  
nodes state switching: 节点工作状态微调
- on power charging: turn on wake lock; charging off: turn off wake lock.
- on wifi data: start all process A-D; wifi off: stop all process A-D and turn off wake lock.
- in the sleeping mode, random wake up between 1..WakeUpTime, if Wifi is off, then stop; else run for a cycle of all chains follow up, and check whether in power charging, yes to turn on wake lock. 

- Notify on the iterface- wifi only for data flow, keep charging to prevent sleep. the data dash board, with a button to pause everything A-D. 
```
1.Generate "chainID+relay+peer" combo, Pick up ONE random `chainID` in the myChains,
according to the global time in the base of RelaySwitchTimeUnit, H = hash (time in RelaySwitchTimeUnit base + chain ID) 

{if H last number is 0,1,2

to hash(myRelays[`ChainID`][date less than 3Xmutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer // if any one of those fields are null, means the chain is very early, then use null adress move on. //信息不全就是链的早期，继续进行 

else if 3,4,5
to hash(myRelays[TAUchain][date within mutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 

else 6,7,8,9
to hash(myRelays[successed][date within 9xmutablerange] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 
}

2. if the  mySafetyContractResultStateRootMiners[`ChainID`] == myPreviousSafetyContractResultStateRootMiners[`ChainID`]; 
go to step (3); // 上两次连续出块是同一个地址，就要投票。 
else go to step (7); // it is educated working chain

3. HamtGraphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); if err go to (5)
   myRelays[successed].add{this Relay}
4. traverse history for states roots collection until MutableRange.
(*)  
stateroot= y/contractJSON/SafetyContractResultStateRoot // recursive getting previous stateRoot to move into history
y = HamtGraphRelaySync(stateroot)
goto (*) until the mutable range or any error; // 

5. On the same chainID, according to the global time in the base of RelaySwitchTimeUnit, H = hash (time in RelaySwitchTimeUnit base + chain ID)

{if H last number is 0,1,2

to hash(myRelays[`ChainID`][date less than 3Xmutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer // if any one of those fields are null, means the chain is very early, then use null adress move on. //信息不全就是链的早期，继续进行 

else if 3,4,5
to hash(myRelays[TAUchain][date within mutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 

else 6,7,8,9
to hash(myRelays[successed][date within 9xmutablerange] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 
}

goto step (3) until surveyed 2/3 of myPeers[`ChainID`][...]

6. accounting the voting rule, pick up the highest weight among the roots even only one vote, then use own safetyroot update the CBC safety root: mySafetyContractResultStateRoots[`ChainID`] = voted SAFETY), 统计方法是所有的root的计权重，选最高。
goto (9)
// use  map[root string]Uint to count voting. 

7. graphRelaySync( Relay, peerID_A, chainID, null, selector(field:=contractJSON));
   if err= null, myRelays[successed].add{this Relay}
8. if ok, then verify {
if received ContractResultStateRoot/contractJSON shows a more difficult chain than SafetyContractResultStateRoot/contractJSON/`difficulty`, then verify this chain's transactions until the MutableRange. in the verify process, it needs to add all db variables, hamt and amt trie to local. for some Key value, it will need `graphRelaySync` to get data from peerID_A;
goto (9)
}
  else { 
  failed or err , if the (current time - safety state time ) is bigger than SelfMiningTime, then generate a new state on own  safety root, this will cause safety miner = previous safety miner to trigger voting, go to (9)
  }; 
       else go to step 1. 

9. generate new state 
X = {
ChainID
SafetyContractResultStateRoot; // type cbor.cid
contractNumber;  hamt_get(SafetyContractResultStateRoot,contractJSON) / contractNumber +1;
version; 
timestamp; 
base target; // for POT calc
cumulative difficulty; 
generation signature; 
`minerAddress`IPFSsig; //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
msg = { // one block support one transaction only
nounce;
version;
timestamp;
txfee;
msg; // fileAMTroot is also in msg.  msg {optcode, code}
`ChainIDsenderAddress`IPFSsig; //IPFS signature on `ChainIDsenderAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
// the File importing to AMT
// 1. tgz then use ipfs block standard size e.g. 250k to chop the data to m pieceis
// 2. newNode.amt(1,piece(1)); loop to newNode.amt(m,piece(m));
// 3. FileAMTroot=AMT.put(cbor)
// 4. return FileAMTroot to here for fileAMTroot. 
tx sender signature; // this can generate `ChainID`senderAddress
}
signature;  // this can generate `ChainID`minerAddress
}  // finish X.

* stateroot.hamt_update(contractJSON, X); 

#### contract execute results
##### output coinbase tx
* stateroot.hamt_update(`Tminer`Balance,`Tminer`Balance + amount); // update balance 
* stateroot.hamt_update(`Tminer`TXnounce,`Tminer`TXounce + 1); // for the coinbase tx nounce increase
* stateroot.hamt_add(`Tminer`TXnounceMsg, contractJSON/msg); // recording the block tx pool

##### output Coins Wiring tx, both sender and receive increase power, this is good for new users to produce contract.
Account operation
* stateroot.hamt_update(`Tsender`Balance,`Tsender`Balance - amount - txfee); 
* stateroot.hamt_update(`Ttxreceiver`Balance,`Ttxreceiver`Balance + amount);
* stateroot.hamt_update(`Tsender`TXnounce,`Tsender`TXounce + 1);
* stateroot.hamt_update(`Treceiver`TXnounce,`Treceiver`TXnounce++);
* stateroot.hamt_add(`Tsender`TXnounceMsg, msg); // when user follow tsender, can traver its files.
* stateroot.hamt_add(`Treceiver`TXnounceMsg, msg); // when user follow tsender, can traver its files.


##### output relay annoucement tx, both sender and receive increase power, this is good for new users to produce contract.
Relay annoucement operation
* stateroot.hamt_update(`Tsender`Balance,`Tsender`Balance - txfee); 
* stateroot.hamt_update(`Tsender`TXnounce,`Tsender`TXounce + 1);
* stateroot.hamt_add(`Tsender`TXnounceMsg, msg); // when user follow tsender, can traver its files.
* stateroot.hamt_update(RelayNounce , ++) 
* stateroot.hamt_add(RelayNounceAddress, msg/relay multiaddress) 
* myRelays [`ChainID`][ ].add({msg/relay multiaddress});


##### File creation and seeding transaction
File operation
* stateroot.hamt_update(`Tsender`FileNounce, `Tsender`FileNounce + 1);
* stateroot.hamt_add(`ChainID``Tsender`File`Nounce`fileAMTroot, fileAMTroot); // when user follow tsender, can traver its files.
* stateroot.hamt_add(`ChainID``Tsender`File`Nounce`fileMsg, contractJSON/tx/msg); // when user follow tsender, can traver its files.
* hamt_upate(`fileAMTroot``ChainID`SeedingNounce, `fileAMTroot``ChainID`SeedingNounce+1);
* stateroot.hamt_add  (`fileAMTroot``ChainID`Seeding`Nounce`IPFSpeer, `ChainID``Tsender`IPFSaddr) // seeding peer ipfs id, the first seeder is the creator of the file.
* mySeedersDB[fileAMT][ ].add(`ChainID``Tsender`IPFSaddr)
* For file upload to chain
      * myFileAMTroots.add(fileAMTroot)
* For file seeding from other peers
      *myDownloadPool.add(fileAMTroot)

Account operation
* stateroot.hamt_update(`Tsender`Balance,`Tsender`Balance-txfee);

#### finish contract execution
Put new generated states into  cbor block, * myContractResultStateRoots[`ChainID`]=hamt_node.hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid. 

* mypreviousSafttyContractResultStateRootMiner[`ChainID`] = mySaftyContractResultStateRootMiner[`ChainID`];
* mySafttyContractResultStateRootMiner[`ChainID`] = Tminer;  // this is for deviting go voting or not

* mySafetyContractResultStateRoots[`ChainID`] = SafetyContractResultStateRoot;
* myPeers[`ChainID`][ ].add(`Tminer);

go to step (1) to get a new ChainID state prediction
```
## C. File Downloader - nonconcurrency design // ipfs layer
* For saving mobile phone resources, we adopt non-concurrrency execution. The entire download pool is one big file to randomly retrieve.

```
1. Generate chain+relay+FileAMT+Seeder+Piece combo: 
      Pickup ONE random `chainID` in the myChains[ ],
according to the global time in the base of RelaySwitchTimeUnit, H= hash (time in RelaySwitchTimeUnit base + chain ID)

{if H last number is 0,1,2

to hash(myRelays[`ChainID`][date less than 3Xmutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer // if any one of those fields are null, means the chain is very early, then use null adress move on. //信息不全就是链的早期，继续进行 

else if 3,4,5
to hash(myRelays[TAUchain][date within mutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 

else 6,7,8,9
to hash(myRelays[successed][date within 9xmutablerange] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 
}

Randomly request ONE File from myDownloadPool[`ChainID`]. Randomly select a seeder from the * mySeedersDB[fileAMT][ChainID][seeder  ]. Randomly select a piece N from fileAMTroot.count
ONE Chain + ONE Relay + ONE FileAMT + ONE seeder peer + ONE piece. 

2. If the piece is in local, go to step (1); 
else 
- AMTgraphRelaySync(relay, chainID, `FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer, `fileAMTroot`, selector(field:=piece N))
if success, myDownloadPool[`ChainID`][`fileAMTroot`]++; until myDownloadPool[`ChainID`][`fileAMTroot`] = fileAMTroot.count; remove this fileAMT from myDownloadPool[`ChainID`]; add to myFileAMTroots
go to step (1)
}
```
## D. reponse to fileAMT request

response to AMTgraphRelaySync（ relay, peer, `ChainID`,`fileAMTroot`, selector(piece N))
If the `fileAMTroot`'s piece N exists, then return the block. else null.

## E. process manager
// registration message handling 登记, this can also happen in other main func

 * myChains.add (TAU)
 * myRelays[TAU].add{initial relays}
 * onGenesisMsg creation, default each chain is "auto-relay", means genesis miner will check tau chain and add  tau relay into own chain.  auto-relay is a local config for the chain creator. any other peer can add relay info on community chain 
 * onHamtGraphsyncMsg, 
 if hamtsync not finish, reject; else 
 A.Response HAMT with predicted ContractResultStateRoot, which is a hamt cbor.cid. (service response to HamtGraphRelaySync). One instatnce per connection to prevent ddos. 
 * on AMTGraphSyncMsg 
  if amtsync not finish, reject; else 
 D. Reponse AMT cbor.cid to file downloader request. (service response to AMTGraphRelaySync and logging upload data). One instatnce per connection to prevent ddos.  改到以chain 为服务单位
 * onMyDownloadQue. not empty and C process not in running, then launch C. File Downloader. (download files and logging download data)  // download is single process too. 

// finish registration 

#### call func B.  // B is an infinite loop; 

* if cell phone resource is enough, load more goroutine "go func B()"; waitGroup().


## App UI 界面


// 建立区块链，发币邀请，上传
1. Own a blockchain with 1 million coins to build a community for video and files sharing. 
2. Send coins to friends.  // get their video previews and 2% automatically on the followed friends.  
3. Seeding files to keep ads free. // next step upload to a new seeding chain or existing chain. 

* Data dashboard:  upload - download = free download data amount, more than 1G, wifi only. "seeding to increase data"

### Community 社区
- follow chain, first layer
- follow members, second layer
- member messages & file, third layer, support import
### Files, this is where watching the ads 文件
- import files
- share file to friend
- share file to community chain
- seeding files and unseeding
- pin a file, no directory at now, sort by dates and size
- delete a file
### Forum 论坛
- according to the following list, display files uploaded and its description. users can follow sender or blacklist them. 

### Mining and account balances on different chains. 
- coins mining config

# To do 
- [ ] TAU Chain
- [ ] file operation commands

```
File operation command:
//  rootCreate: create hamt node for the file, return the root IPLD cid and number of blocks, set rootNounce = 1
//  - cat file| rootCreate -compress -chunk size -type file/video/voice -video_preview
//  e.g  cat starwar.mp4 | rootCreate -type video -video_preview; // return root hash and size of the preview
//  rootSeeding: graphRelaySync hamt node to local, and provide turn on seeding or off, set rootNounce ++
//  - rootSeeding fileRoot -seeding on/off
```
