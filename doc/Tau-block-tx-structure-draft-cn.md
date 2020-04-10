# TAU - Unlimited file sharing on blockchains
```
User experienses:= { 用户体验
- Core: One button: 1. create seeding blockchain 2. airdroping and seeding friends.  3. upload a file (with new chain ability)
* Data dashboard: if (download - upload) > 1G, start ads., wifi only. "seeding/uploading to increase free data"

- File/video imported to TAU will be compressed and chopped by TGZ, which includes directory, pictures and videos. Chopped file pieces will be added into AMT (Array Mapped Trie) with a `fileAMTroot` as return. Filed downloaded could be decompressed to original structure.  Files downloaded is considerred imported. Imported file can be seeded to a chain or pinned in local. 

- For each video, generate a "preview" at in the beginning of amt trie, which take a random chop of the video then show 9 pictures.  

- Chain can serve as anchor chains. TAU is first such anchor chain, providing services like relay, coins exchange pairs with TAU, genesis annoucement. Community chain can use itself as one of the anchor chains for relay annoucement. For security, community chain needs at least two anchors, another wellknow public chain like TAU and own. Otherwise, it could be blinded by secret chain. 

- All chain addresses are derivative from one private key. Nodes use IPFS peers ID for ipv4 tcp transport. (the association of TAUaddr and IPFS address is through signature using ipfs RSA private key).
}

Business model:= { 商业模式
- Tau foundation will develop TAU App and provide free public relays (TAU dev private key signed), in return for admob/mopub ads income to cover AWS data cost. Any one can config relay permission-lessly on both TAU and own chain. 
- Individual nodes will see ads to keep using data for free, the more data upload, the less ads to see. In app, show a stats of uploaded data, download data. When users getting data from community signed relay, the TAU app will counting the download. 
- Anchor chain coin price will rise when in demand. 
}

Launch steps:={ 发布步骤
- Free community creation for file sharing. TAUTest coin is an initial test by TAU dev. At this stage, TAU provide static relay service via AWS. The relay info is written in the software.
- Tau mainnet turns on for relays and exchange operation.
}
```
## Four main processes
* A. Response with predicted `ChainID`ContractResultStateRoot, which is a hamt cbor.cid. (service)
* B. Collect votings from chain peers to discover the chainid's safety state root. (single thread request, for first use, possible to run concurrency)
* C. File Downloader. (concurrent goroutine and logging download data)
* D. Reponse amt cbor.cid to file downloader request. (service and logging upload data)

## Tries
On community chain:
* chain contract result hamt trie: `ChainID`ContractResultStateRoot is the chain state hamt root; contract and results are connected in each state transition. 
```
- future state ->`ChainID`ContractJSON
- `ChainID`ContractJSON -> `ChainID`SafetyContractResultStateRoot
```
* file AMT: the root for AMT trie for chopping and storing the file.
//hamt:  hamt_node(root cbor.cid,key) -> value;  cid = hamt_node.hamt_put(cbor); flush -> put.  one key(contract, acct, nounce), put one board.  root is from newnode() or history.
//amt:   amtNode(root cbor.cid).count -> value
## Global vars, stored in database
It helps to make process internal data access efficient. 
* SafetyContractResultStateRoot[`ChainID`] cid; // this is constantly updated by voting process B(1-4). CBC 安全点
* ContractResultStateRoot[`ChainID`] cid; // after found safety, this is the new contract state, B(5-6). CBC 未来状态
* ChainList []String ; // a  list of Chains to follow/mine by users.
* FileAMTlist []cid; // a  list for imported and downloaded files trie
* PeerList [`ChainID`][index]String `peer`; // list of known IPFS peers for the chain by users.
* RelayList [`ChainID`][index]String `relayaddre`; // a list of known relays from TAU chain or community chains; initially will be hard-coded to use AWS EC2 relays.
* TXpool [`ChainID`][`TX`]String; // a list of verified txs for adding to new contract

## Concept explain
```
- Miner is what nodes call itself, in CBC POT all miners predicting future; Sender is what nodes call other peers. 
- Safety is the CBC concept of the safe and agreed history. The future contract result is a CBC prediction. 
- Mutable range is one week for now. 
- HamtGraphyRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replace the relay circuit. When cbor.cid is null, it means asking peer for the prediction, the target's future `ChainID`ContractResultStateRoot.
- AMTgraphRelaySync(relay multiaddress, remote ipfs peer, CID, selector); CID can not be null. 
- File operation transaction, FileAMTRoot creation and seeding, related nounce and seeders accessible through wormhole. 文件操作
- Principle of traverse, once in a relay+peer communication, we will not incur another recursive process to a new relay+peer to get supporting evidence. if some Key-value are missing to prevent validation, just abort process to go next randomness peer. This is the depth priority. 验证投票过程访问节点深度优先。 However for the fileAMTroot search, it is the width priority to do paralell download using all seeders. 文件下载访问节点广度优先

- Address system: 
- TAU private key: the base for all community chain address generation;
- Community chain ID: `Tgenesisaddress` + sign(random(1,000,000,000));
- Community chains peer address format : `chainID` + `TAU address`; 

- msg: the contract content for genesis, coin base, wiring and file tx
   * genesis, msg is the initial info such as telegram channel
   * coin base, msg is the txpool attached
   * wiring, msg is the contract relating to this tx
   * file, msg is the discription of the file or file command
```
## Wormhole - Keys in the HAMT, hashed keys are wormhole inito contract history. 
```
// TX oriented Tsender = `ChinaID` + TAUaddress
wiring tx
- `Tsender/receiver`TXnounce; //  balance and POT power for each address 总交易计数
- `Tsender/receiver`Balance
- `Tsender/receiver`TXNounce`Msg // for future command, such as "seeding all" comment, no file attachment
file tx
- `Tsender`FileNounce // file command counting 文件交易计数
- `Tsender`file`Nounce`FileAMTroot // when user follow a chain address, they can traverse its files through changing nounce. 
- `Tsender`file`Nounce`fileMsg

// entity oritened
file
- `FileAMTroot``ChainID`SeedingNounce // for each file, this is the total number of registerred seeders, first seeding is the creation.
- `FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer // the seeding peer id for the file. 
relay
- Relay`ChainID`Nouce  // recording the relays counter
- Relay`ChainID`NouceAddress // recording the relay address
```
## constants
* mutable range = 1 week
* new node time cut off = 48 hours
* transaction expirey 24 hours
## community chain
## Genesis
* with parameters: block size in number of txs, block time, chain nick name, coins total - default is 1 million,  relay bootstrap.  // initial mining peers is established through issue coins to those addresses, such as TAU-Torrent has initial addresses. 社区链创世区块
```
// build genesis block
type contractJSON struct { // define the contract strut
`ChainID`SafetyContractResultRoot = null; // genesis is built from null.
contractNumber int32; :=0
blocksize int; // default 3，区块交易数
blocktime int; // default 5 minutes， 出块时间
initial difficulty int64; // ???
chainNickname string; // hello world chain
totalCoins int64; // default 1,000,000， 币数量
`Tminer`TXnoucne:=0;
`Tminer`FileNounce:=0;
anchorChain string:=TAUgenesisID; // default is TAU mainchain ID
msg string:="telegram"; // https://t.me/taucoin for organizing the community. 社区通信
signature []byte //by genesis miner
}
// build genesis state
* X := hamt_node := <nil> new.hamt_node(); // execute once per chain, for future all is put.


* database.anchorChainRelaylist[`TAUmainnetchainID`][]={"multi address1", "multiaddress2"}; // relay bootstrap /ipv4/tcp， 初始中继配置表在软件文件里
* hamt_add(Relay`ChainID`Nouce, number of relays)
* hamt_add(Relay`ChainID`NouceAddress) // recording the relay address
* hamt_add(ChainID,`Tminer`+ sig(random(time seeds));用创世矿工的TAU私钥签署
* hamt_add(`ChainID`contractJSON, contractJSON { // root for contact AMT 
* hamt_add(`ChainID`SafetyContractResultRoot = null; // genesis is built from null.
* hamt_add(`Tminer`Balance, 1,000,000); 
* hamt_add(`Tminer`TXNounce, 0);
* hamt_add(`Tminer`TXNounceMsg,msg);
* hamt_add(`Tminer`FileNounce, 0);
* hamt_add(genesisAddress, `Tminer`); // add genesis address wormhole
* hamt_add(other KVs); // initial chain address.
* `ChainID`ContractResultStateRoot = hamt_node.hamt_put(cbor); // for responding to voting.

* database.`ChainID`SafetyContractResultStateRoot = null;
* database.`ChainID`ContractResultStateRoot=`ChainID`ContractResultStateRoot; 
* database.UserChainList.add(`ChainID`)
* database.PeerList[`ChainID`][].add(`Tminer);
* database.anchorChainRelayList [][].add({aws relays by taucoin dev});
```
## A. One miner receives GraphSync request from a relay.  
Miner does not know which peer requesting them, because the relay shields the peers. Two types of requests: "chainIDContractResultStateRoot" and `fileAMTroot`. 
### A.1 for future `ChainID`ContractResultStateRoot， 投票

-  Receive the `ChainID` from a graphRelaySync call
-  If the node follows on this chain, return database.`ChainID`contractResultStateRoot, which was generated in B process step(6).

### A.2 From a file downloader，提供文件
caller with graphRelaySync（ relay, peer, chainID, `fileAMTroot`, selector(range of the trie))
If the `fileAMTroot` exists, then return the blocks according to the range. 
If the `fileAMTroot` equal null, then return `ChainID``Tsender`file`Nounce`fileAMTroot, 最新文件


## B. Collect votings from peers: 
In the peer randome walking, no recursively switching peers inside the loop, it relies on top random working. In the process of voting, the loose coupling along time is good practise to keep the new miners learning without influcence from external. This process is for multiple chain, multiple relay and mulitple peers.  
nodes state changes: 节点工作状态微调
- on power charging turn on wake lock; charging off, turn off wake lock.
- on wifi data, start mining; wifi off, stop mining.
 
```

1. for a random `chainID` in the ChainList[],if the `ChainID`SafetyContractResultStateRoot/time stamp is older than 48 hours of present time, jump to step 2, means a new node; else jump to step 5, means an experinces note. 如果节点安全点时间戳在48小时前，被认为是新节点，需要重新投票。
2. random walk connect to a next relay in the RelayList[`ChainID`][] using Kademlia selection. through relay, randomly request a chainPeer from database.PeerList[`ChainID`][] for the future state root voting.  

graphRelaySync( Relay, peerID, chainID, null, selector(field:=`ChainID`contractJSON)); // when CID is NULL,  - 0 means the relay will request y:= `ChainID`ContractResultStateRoot from the peer via tcp

3. traverse history contract and states until mutable range.

(*)  
stateroot= y/`ChainID`contractJSON/`ChainID`SafetyContractResultStateRoot // recursive getting previous stateRoot to move into history
y = graphsyncHAMT(stateroot)
goto (*) until the mutable range or any error like connect time out; // 
goto step (2) until surveyed half of the know PeerList[`ChainID`][] or 

4. accounting the voting rule, if the safety root difficulty is less than own difficulty, then use own safetyroot update the CBC safety root: database_update(`ChainID`SafetyContractResultStateRoot, voted SAFETY), 如果投票无人参加，或者投票结果比自己的难度低，就用自己的安全root.

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
// 3. FileAMTroot=AMT_flush_put()
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
Account operation
* hamt_update(`Tsender`Balance,`Tsender`Balance-txfee);
6. Put new generated states into  cbor block, database.add `ChainID`ContractResultStateRoot = hamt_node.hamt_put(cbor); // this is the  return to requestor for future state prediction, it is a block.cid


---
7. random walk until connect to a next relay
randomly request a PeerList[`ChainID`][] for the future receipt state 

graphRelaySync( Relay, peerID, chainID, null, selector(field:=`ChainID`contractJSON)); 

8. mining by following the most difficult chain: if received `ChainID`ContractResultStateRoot/`ChainID`contractJSON shows a more difficult chain than `ChainID`SafetyContractResultStateRoot/`ChainID`contractJSON/`difficulty`, then verify this chain's transactions for ONEWEEK range. 

 if not be able to find a more difficult chain than current "difficulty" for 3 x blockTime, then assume verifiation successful, to generate a new state on own last block, reflexing 3x time, then next miners will be in lower difficulty.  

9. If verification succesful, database_update(`ChainID`SafetyContractResultStateRoot, `ChainID`ContractResultStateRoot), go to step (5) to get a new state prediction; Else go to step (7)
10. if network-disconnected from internet 48 hours, go to step (1).
```
## C. File Downloader
```
input (`fileAMTroot`); //this root can not be null.

From `fileAMTroot`, 广度优先遍历 the `FileAMTroot``ChainID`SeedingNounce;
{ 
random walk on RelayList[`ChainID`][] to find `FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer

graphRelaySync(relay, chainID, `FileAMTroot``ChainID``Seeding`Nounce`IPFSPeer, `fileAMTroot`, selector(field:=section 1..m))

until finish all relays or find the chainPeer
}
```
## D. reponse to fileAMT request
this is part of graphsync. 

## App UI 界面
leading function
* One big button, uppon open: 1. create seeding blockchain 2. airdrop or seeding friends.  3. upload a file
```
// 建立区块链，发币邀请，上传
1. Own a blockchain with 1 million coins to build a community for video and files sharing. 
2. Send coins to friends and seeding all their latest uploads upto 1G. 可以配置。 or seeding the previews only. 
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

# To do "anchor chain"
community chains needs place to get relay info and exchange coins. TAU main chain is an anchor chain. 
## TAU Chain
TAU has services on relay, exchange coins. .
* wormhole
- relay nounce/ relaynounce = ...
- hamt_update(relayNounce, relayNounce +1)
- hamt_add(RelayNounceAddr, new relay info)
## Kademlia on relay and peers selection
## community relay annoucement on community chain. 
## file operation commands
```
File operation command:
//  rootCreate: create hamt node for the file, return the root IPLD cid and number of blocks, set rootNounce = 1
//  - cat file| rootCreate -compress -chunk size -type file/video/voice -video_preview
//  e.g  cat starwar.mp4 | rootCreate -type video -video_preview; // return root hash and size of the preview
//  rootSeeding: graphRelaySync hamt node to local, and provide turn on seeding or off, set rootNounce ++
//  - rootSeeding fileRoot -seeding on/off
```
