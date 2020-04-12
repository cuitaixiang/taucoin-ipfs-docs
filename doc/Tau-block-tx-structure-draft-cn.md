# TAU - Unlimited file sharing on blockchains
```
User experienses:= { 用户体验
- Core: One button: 1. create sharing blockchain 2. airdroping to friends and follow preview.  3. upload a file (with new chain ability)
   - Data dashboard: if (download - upload) > 1G, start ads., wifi only. "seeding/uploading to increase free data"
   - For saving resources on mobile device, our implementation is single thread with lots of randomness design.

- File/video imported to TAU will be compressed and chopped by TGZ, which includes directory zip, pictures and videos. Chopped file pieces will be added into AMT (Array Mapped Trie) with a `fileAMTroot` as return. Filed downloaded could be decompressed to original structure.  Files downloaded is considerred imported. Imported file can be seeded to a chain or pinned in local. 

- For each video, since we are randomly download pieces, so we can support a `hopping player` to only play the randon pieces that is downloaded. 

- TAU provides basic communication services like relay, TAU payment, content annoucement and genesis annoucement. Community chain can use itself for relay annoucement as well.

- All chain addresses are derivative from one private key. Nodes use IPFS peers ID for ipv4 tcp transport. (the association of TAUaddr and IPFS address is through signature using ipfs RSA private key).
}

Business model:= { 商业模式
- Tau foundation will develop TAU App and provide free public relays (TAU dev private key signed), in return for admob/mopub ads income to cover AWS data cost. Any one can config relay permission-lessly on both TAU and own chain. 
- Individual nodes will see ads to keep using data for free, the more data upload, the less ads to see. In app, show a stats of uploaded data, download data. When users getting data from community signed relay, the TAU app will counting the download. 
- TAU coin price will rise when cross-chain comm in demand. 
}

Launch steps:={ 发布步骤
- Free community creation for file sharing. TAUTest coin is an initial test by TAU dev. At this stage, TAU provide static relay service via AWS. The relay info is written in the software.
- Tau mainnet turns on for relays and exchange operation.
}
```
## Four core processes
* Chain hamt trie, blockchain meta data level.
  * A. Response HAMT with predicted `ChainID`ContractResultStateRoot, which is a hamt cbor.cid. (service response to HamtGraphRelaySync)
  * B. Collect votings from chain peers to discover the chainid's safety state root. (single thread func)
* File amt trie, ipfs block store level.
  * C. File Downloader. (download files and logging download data)
  * D. Reponse AMT cbor.cid to file downloader request. (service response to AMTGraphRelaySync and logging upload data)

## Tries
On community chain:
* chain contract result hamt trie: `ChainID`ContractResultStateRoot is the chain state hamt root; contract and results are connected in each state transition. 

> future state ->`ChainID`ContractJSON

> `ChainID`ContractJSON -> `ChainID`SafetyContractResultStateRoot

* file AMT: the root for AMT trie for chopping and storing the file.
//hamt:  hamt_node(root cbor.cid,key) -> value;  cid = hamt_node.hamt_put(cbor); flush -> put.  one key(contract, acct, nounce), put one board.  root is from newnode() or history.
//amt:   amtNode(root cbor.cid).count -> value

## Node local variables - stored in database leveldb
* mySafetyContractResultStateRoot[`ChainID`] cid; // this is constantly updated by voting process B(1-4). CBC 安全点
* myContractResultStateRoot[`ChainID`] cid; // after found safety, this is the new contract state, B(5-6). CBC 未来状态
* myChainsList [index]String ; // a  list of Chains to follow/mine by users.
* myFilesAMTlist [index]cid; // a  list for imported and downloaded files trie
* myFilesAMTseedersDB[fileAMT][index]=seeders IPFS addresss, a local database recording all files seeding, the chainID is for relay timing pace. 
* myPeersList [`ChainID`][index]String `peer`; // list of known IPFS peers for the chain by users.
* myHamtRelaysList [`ChainID`][index]String `relayaddre`; // a list of known relays for different chains; initially will be hard-coded to use AWS EC2 relays.there will be RelayList[TAU][...]; Relay[community #1][...]. The real final relay list for community 1 is the combination of TAU + community #1
* myAMTRelaysList[index]; amt is chain non specific, amt is ipfs layer
* myTXsPool [`ChainID`][`TX`] = String; // a list of verified txs for adding to new contract
* myDownloadQue[index] []map{fileAMT:completion count}
* myDownloaded data
* myUploaded data

## Concept explain
```
- Single thread principle for mobile phone, we do not put wait time in thread, but only support one thread for each functions. The more chain mining, the lower speed on each chain. 
- Miner is what nodes call itself, and sender is what nodes call other peers. In TAU POT, all miners predict a future state; 
- Safety is the CBC Casper concept of the safe and agreed history by BFT group. The future contract result state is a CBC prediction. TAU uses this concept to describe the prediction and safety as well, but our scope are all peers than BFT group.
- Mutable range is defined as "one week" for now, beyond mutalble range, it is considered finality.

- HamtGraphRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replaced the IPFS relay circuit. When cbor.cid is null, it means asking peer for the `ChainID`ContractResultStateRoot prediction on the chainID.
- AMTgraphRelaySync(relay multiaddress, remote ipfs peer, cbor.cid, selector); cid can not be null. AMT is not chain specific, and it is rather relating to IPFS peers. 
- In both GraphRelaySync, it needs to test wether the target KV holding cbor.cid are already in local or not. 

- File operation transaction, FileAMTRoot creation and seeding, related nounce and seeders accessible through wormhole. 文件操作
- Principle of traverse, once in a "ChainID+relay+peer" communication, we will not incur another recursive process to a new peer to get supporting evidence. If some Key-values are missing to prevent validation, just abort process to go next randomness. This is the depth priority. 验证投票过程访问节点深度优先。 

- Address system: 
- TAU private key: the base for all community chain address generation;
- TAU Chain ID = "0"
- Community ChainID := `Tminer`+ `blocksize`+`blocktime` + randomness // chainID include genesis address, block size 5, time 5, 1 transaction per minute ; // in stateless environment, chain config info needs to be embedded into chainname, otherwise, might be lost. 
- Community chains peer address format : `chainID` + `TAU address`; 

- msg: the contract content for coin base, wiring and file tx
   * coin base, msg is the txpool attached
   * wiring, msg is the contract relating to this tx
   * file, msg is the discription of the file or file command
```
## Wormhole - HAMT Hashed keys are wormhole inito contract history. 
```
// TX oriented 
genesisAddress = `ChinaID` + TAUaddress
Wiring transactions
- `Tsender/receiver`TXnounce; //  balance and POT power for each address 总交易计数
- `Tsender/receiver`Balance
- `Tsender/receiver`TXnounce`Msg // for future command, such as "seeding all up to 1G" comment, no file attachment
File transactions
- `Tsender`FileNounce // file command counting 文件交易计数
- `Tsender`File`Nounce`FileAMTroot // when user follow a chain address, they can traverse its files through changing nounce. 
- `Tsender`File`Nounce`Msg

// entity oritened
File
- `FileAMTroot``ChainID`SeedingNounce // for each file, this is the total number of registerred seeders, first seeding is the creation.
- `FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer // the seeding peer id for the file. 
Relay
- Relay`ChainID`Nounce  // recording the relays counter, these relays are own chain annouced relays. The TAU relays are in the levelDb. Relay pool are the combination of TAU relays and own chain relays. 
- Relay`ChainID`NounceAddress // recording the relay address
```
## Constants
* mutable range = 1 week
* new node qualification time cut off = 48 hours // off line longer than 48 hours will be consider new nodes for that chain
* transaction expirey 24 hours
* voting percentage 67%
## Community chain
### Genesis
* with parameters: block size in number of txs, block time, chain nick name, coins total - default is 1 million.  // initial mining peers is established through issue coins to other addresses. 社区链创世区块
```
// build genesis block
contractJSON:= { // define the contract strut
ChainID := `Nickname`+ `blocksize`+`blocktime` + signature(random) // chainID include nickname, block size 5 and time 5, these are critical information to pass down in the stateless mode.
`ChainID`SafetyContractResultRoot = null; // genesis is built from null.
contractNumber:=0 int32;
initial difficulty int64; // ???
totalCoins int64; // default 1,000,000， 币数量
`Tminer`TXnoucne:=0;
`Tminer`FileNounce:=0;
msg;
signature []byte //by genesis miner
}
// build genesis state
* X := hamt_node := null new.hamt_node(); // execute once per chain, for future all is put.

* hamt_add(Relay`ChainID`Nouce, number of relays)
* hamt_add(Relay`ChainID`NouceAddress) // recording the relay address
* hamt_add(ChainID,`Nickname`+ `blocksize`+`blocktime` + sig(random) );用创世矿工的TAU私钥签署 randomness
* hamt_add(`ChainID`contractJSON, contractJSON) 
* hamt_add(`ChainID`SafetyContractResultRoot = null; // genesis is built from null.
* hamt_add(`Tminer`Balance, 1,000,000); 
* hamt_add(`Tminer`TXNounce, 0);
* hamt_add(`Tminer`TXNounceMsg,msg);
* hamt_add(`Tminer`FileNounce, 0);
* hamt_add(genesisAddress, ChainID+`Tminer`); // add genesis address wormhole
* hamt_add(other KVs); // initial chain address.
* `ChainID`ContractResultStateRoot = hamt_node.hamt_put(cbor); // for responding to voting.

* database.my`ChainID`SafetyContractResultStateRoot = null;
* database.my`ChainID`ContractResultStateRoot=`ChainID`ContractResultStateRoot; 
* database.myChainsList.add(`ChainID`)
* database.myPeersList[`ChainID`][index].add(`Tminer);
* database.myHamtRelaysList [`ChainID][index].add({aws relays by taucoin dev}); // each chain annouced relay recorded.{"multi address1", "multiaddress2"}; // relay bootstrap /ipv4/tcp， 初始中继配置表在软件文件里
* database.myAmtRelaysList[index]
```
## A. One miner receives GraphSync request from a relay.  
Miner does not know which peer requesting them, because the relay shields the peers. Two types of requests: "chainIDContractResultStateRoot" and `fileAMTroot`. 
-  Receive the `ChainID` from a graphRelaySync call
-  If the node follows on this chain, return database.my`ChainID`contractResultStateRoot; else response null

## B. Collect votings from peers 
In the peer randome walking, no recursively switching peers inside the loop, it relies on top random working. In the process of voting, the loose coupling along time is good practise to keep the new miners learning without influcence from external. This process is for multiple chain, multiple relay and mulitple peers.  
nodes state changes: 节点工作状态微调
- on power charging turn on wake lock; charging off, turn off wake lock.
- on wifi data, start file download and upload; wifi off, stop file operation.
- in the sleeping mode, random wake up between 1..5 minutes to run for a cycle of all chains follow up and check whether in power charging to turn on wake lock. 

- Alert to user iterface- wifi only for file up and down, keep charging to prevent sleep, along with the data dash board. 
  with a button to pause everything. 
```
1. Pickup a random `chainID` in the myChainsList[index],if the `ChainID`SafetyContractResultStateRoot/time stamp is older than 48 hours of present time, jump to step 2, means a new node; else jump to step 5, means an experinces note. 如果节点安全点时间戳在48小时前，被认为是新节点，需要重新投票。
2. according to the global time in the base of 5 minutes, hash (time + chain ID), in RelayList[`ChainID`][] find vector distance the closest relays. Through that relay, randomly request a chainPeer from database.PeerList[`ChainID`][] for the future state root voting. at this time, if the node is on the same chain, it will be a definitive match. 

graphRelaySync( Relay, peerID, chainID, null, selector(field:=`ChainID`contractJSON)); // when CID is NULL,  - 0 means the relay will request y:= `ChainID`ContractResultStateRoot from the peer via tcp
if err := !null go to (2);  // how long not_found rejection?  3tx/block, 5 minutes. block time,  

3. traverse history contract and states until mutable range.

(*)  
stateroot= y/`ChainID`contractJSON/`ChainID`SafetyContractResultStateRoot // recursive getting previous stateRoot to move into history
y = graphsyncHAMT(stateroot)
goto (*) until the mutable range or any error like connect time out; // 
goto step (2) until surveyed 2/3 of the know PeerList[`ChainID`][] or 

4. accounting the voting rule, if the safety root difficulty is less than own difficulty, then use own safetyroot update the CBC safety root: database_update(`ChainID`SafetyContractResultStateRoot, voted SAFETY), 如果投票无人参加，或者投票结果比自己的难度低，就用自己的安全root. 统计方法是所有的root的计数，选最高。

--------
5. predict new future contract. 
X = {
`ChainID`SafetyContractResultStateRoot 32; // link to current safety state node.cid, and move to generate future
contractNumber = `ChainID`SafetyContractResultStateRoot/`ChainID`contractJSON/contactNumber) +1;
version,8; 
timestamp, 4; 
base target, 8; // for POT calc
cumulative difficulty,8 ; 
generation signature,32; 
ChainIDminerAddress, 20; 
TXNounce ++, 8; // mining is treated as a tx sending to self
 `ChainIDminerAddress`IPFSsig; //IPFS signature on `ChainIDminerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
ChainIDminerOtherInfo, 128 bytes; //nick name.
msg, flexible bytes; 
= { //original tx from voting collection: wiring and file operation.
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 24 hours;
txfee;
msg 2048;
`ChainIDsenderAddress`IPFSsig; //IPFS signature on `ChainIDsenderAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
ChainIDsenderOtherInfo, 128 bytes;  // nick name.

FileAMTRoot;
tx sender signature;
// the File processing
// 1. tgz then use ipfs block standard size e.g. 250k to chop the data to m pieceis
// 2. newNode.amt(1,piece(1)); loop to newNode.hamt(m,piece(m));
// 3. FileAMTroot=AMT.put(cbor)
// 4. return FileAMTroot to contract Json. 
}
 
signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte
}  // finish X.

* hamt_update(`ChainID`contractJSON, X); 

#### contract execute results
##### output coinbase tx
* hamt_update(`Tminer`Balance,`Tminer`Balance + amount); // update balance 
* hamt_update(`Tminer`TXnounce,`Tminer`TXounce + 1); // for the coinbase tx nounce increase
* hamt_add(`Tminer`TXnounceMsg, `ChainID`contractJSON/msg); // recording the block tx pool
##### output Coins Wiring tx, both sender and receive increase power, this is good for new users to produce contract.
Account operation
* hamt_update(`Tsender`Balance,`Tsender`Balance - amount - txfee); 
* hamt_update(`Ttxreceiver`Balance,`Ttxreceiver`Balance + amount);
* hamt_update(`Tsender`TXnounce,`Tsender`TXounce + 1);
* hamt_update(`Treceiver`TXnounce,`Treceiver`TXnounce++);
* hamt_add(`Tsender`TXnounceMsg, msg); // when user follow tsender, can traver its files.
* hamt_add(`Treceiver`TXnounceMsg, msg); // when user follow tsender, can traver its files.
##### File creation and seeding transaction
File operation
* hamt_update(`Tsender`FileNounce, `Tsender`FileNounce + 1);
* hamt_add(`ChainID``Tsender`File`Nounce`fileAMTroot, fileAMTroot); // when user follow tsender, can traver its files.
* hamt_add(`ChainID``Tsender`File`Nounce`fileMsg, contractJSON/tx/msg); // when user follow tsender, can traver its files.
* hamt_upate(`fileAMTroot``ChainID`SeedingNounce, `fileAMTroot``ChainID`SeedingNounce+1);
* hamt_add  (`fileAMTroot``ChainID`Seeding`Nounce`IPFSpeer, `ChainID``Tsender`IPFSaddr) // seeding peer ipfs id, the first seeder is the creator of the file.
* myFilesAMTseedersDB[fileAMT][index].add(`ChainID``Tsender`IPFSaddr)
* myFilesAMTlist[].add(  amt.node = AMTgraphRelaySync(relay, peerID, fileAMTroot, selector))
Account operation
* hamt_update(`Tsender`Balance,`Tsender`Balance-txfee);
6. Put new generated states into  cbor block, database.add `ChainID`ContractResultStateRoot = hamt_node.hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid


---
7. random walk until connect to a next chainID relay
randomly request a PeerList[`ChainID`][] for the future receipt state 

graphRelaySync( Relay, peerID, chainID, null, selector(field:=`ChainID`contractJSON)); 

8. mining by following the most difficult chain: if received `ChainID`ContractResultStateRoot/`ChainID`contractJSON shows a more difficult chain than `ChainID`SafetyContractResultStateRoot/`ChainID`contractJSON/`difficulty`, then verify this chain's transactions for ONEWEEK range. in the verify process, it needs to add all db variables, hamt and amt trie. 
    for some Key value, it will need graphRelaySync to get data from the new root supplying peer.
 if not be able to find a more difficult chain than current "difficulty" for 3 x blockTime, then assume verifiation successful, to generate a new state on own last block, reflexing 3x time, then next miners will be in lower difficulty.  

9. If verification succesful, database_update(`ChainID`SafetyContractResultStateRoot, `ChainID`ContractResultStateRoot), go to step (1) to get a new ChainID state prediction; Else go to step (7)
```
## C. File Downloader - nonconcurrency design // ipfs layer
For saving mobile phone resources, we adopt non-concurrrency execution. Starting from a to-be downloaded myDownloadQue[] map.
```
Do a range on map myDownloadque, key, value. inputFileAMTroot = key 
random pickup a piece from fileAMTroot.count. random pick a peer myFilesAMTseedersDB[fileAMT][index]
{ 
- generate request a randam piece N for graphsync, since graphsync can identify the local exisitnce, if duplicated in local, it will ask another random block id. do not do specific cut. 
- according to the global time in the base of 5 minutes, hash (time + chain ID), in RelayList[`ChainID`][] find vector distance closest relays. Through that relay, randomly request a Peer from  myFilesAMTseedersDB[fileAMT][index].  
- graphRelaySync(relay, chainID, `FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer, `fileAMTroot`, selector(field:=section N))

until finish all relays or find the chainPeer
}
```
## D. reponse to fileAMT request - root cannot be null // ipfs layer
the process with connect to closest relay using hash(timestamp + ipfs address) to relay distance, when time stamp switch, so that other peers can find it using time consensus. c/d process is not chain specific, it is all on the ipfs layer.
response to AMTgraphRelaySync（ relay, peer, `fileAMTroot`, selector(range of the trie))
If the `fileAMTroot` exists, then return the blocks according to the range. 

## App UI 界面
leading function
* One big button, uppon open: 1. create seeding blockchain 2. airdrop or seeding friends.  3. upload a file
```
// 建立区块链，发币邀请，上传
1. Own a blockchain with 1 million coins to build a community for video and files sharing. 
2. Send coins to friends.  // get their video previews and 2% automatically on the followed friends.  
3. Seeding files to keep ads free. // next step upload to a new seeding chain or existing chain. 
```
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
- [] TAU Chain
- [] file operation commands

```
File operation command:
//  rootCreate: create hamt node for the file, return the root IPLD cid and number of blocks, set rootNounce = 1
//  - cat file| rootCreate -compress -chunk size -type file/video/voice -video_preview
//  e.g  cat starwar.mp4 | rootCreate -type video -video_preview; // return root hash and size of the preview
//  rootSeeding: graphRelaySync hamt node to local, and provide turn on seeding or off, set rootNounce ++
//  - rootSeeding fileRoot -seeding on/off
```
