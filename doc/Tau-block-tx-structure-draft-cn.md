# TAU - File sharing on blockchains

User experienses:= { 用户体验 Create blockchain/ Send coins/ Upload files
- Core: 1. Create blockchain 2. Send coins.  3. Upload files
- // auto seeding  is a collection of seeding of a file, no support to automatic seeding future files, which cause spam on network. 
   - Data dashboard: if (download - upload) > 1G, start ads. display "seeding to increase free data"
   - Default config: auto-download on, daily limit 20m, files lower than 2M, video 1%; auto-seeding tx triggered on the download results. 
- TAU provides global relay services and chain annoucement.
- All chain addresses are derivative from one private key. Nodes use IPFS peers ID for ipv4 tcp transport. (the association of TAUaddr and IPFS address is through signature using ipfs RSA private key).
- User uses relay from TAU, own chain and suzheccessed history, in the weight of 2:1:7
- User can config automatic download size file X and daily maximum Y; for files less than X will be downloaded, for video only download X size of the overall video. 
- Using the new chain annoucement info, app will survey the chains for files and display to users. when user choose to join then random become stable. stable vs random 8/2. 
}

Business model:= { 商业模式
- Tau foundation will develop TAU App and provide free public relays, in return for admob/mopub ads income to cover AWS data cost. Any one can contribute relays on both TAU and own chain. 
- Individual nodes will see ads to keep using data for free, the more data upload, the less ads. In app, show a stats of uploaded data, download data. 
- TAUT is tau-torrent, a file sharing service by tau dev
- TAU is a relay annoucement service by tau dev. 
}

## Two Tries
* State Key-value pairs are in HAMT trie: ContractResultStateRoot is the state root; contract and results are connected in each state transition. 
   * future state includes ContractNumberJSON 
   * ContractNumberJSON includes SafetyContractResultStateRoot
* file AMT: the AMT trie for storing the file.

## Six core processes
On Hamt trie.
* A. Response with predicted ContractResultStateRoot, which is a hamt cbor.cid. (service response to HamtGraphRelaySync). One instatnce per connection to prevent ddos. a call-back registerred in libp2p. 
* B. Collect votings from chain peers to discover the ChainID's safety state root. (single thread func)<br/> <br/>
On Amt trie.
* C. File Downloader. (download files and logging download data)
* D. Reponse AMT cbor.cid to file downloader request. (service response to AMTGraphRelaySync and logging upload data). One instatnce per connection to prevent ddos.  改到以chain 为服务单位<br/> <br/>
On Environment
* E. Process manager, main(); schedule above 4 processes instance existing and prevent DDOS. Process genesis. 
* F. Resource management: 1. pin everything of self created and syced blocks, 2, When safty root time pass the mutable range, unpin states root out of finality.  All seeded files will be pinned, while unseeded AMTroots are not pinned.

## Persistence variables in database
in each transition, following variables will be populated from execution and run-time. 
```
1. myChains                                      map[ChainID] config; //Chains to follow, string is for planned config info

2. myContractResultStateRoots                    map[ChainID] cbor.cid; // the new contract state
3. mySafetyContractResultStateRoots              map[ChainID] cbor.cid;
   
4. mySafetyContractResultStateRootMiners         map[ChainID] address;
5. myPreviousSafetyContractResultStateRootMiners map[ChainID] address; // Safety miner and previous safety miner; 
      if  safety miner = previous safety miner, then the miner is treated as disconnected or new, so go to voting. 

6. myPeers              map[ChainID]map[TAUaddress]config;// include IPFSAddr
7. myRelays             map[ChainID]map[RelaysMultipleAddr]config;// incllude timestampInRelaySwitchTimeUnit; timestamp is to selelct relays in the mutable ranges. 
8. myTXsPool            map[ChainID]map[hash(txjson)]TXJSON; // include timestampInRelaySwitchTimeUnit
9. myDownloadPool       map[ChainID]map[FileAMT]config;   // when file finish downloaded, remove chainID/fileAMT combo from the pool

10. myFileAMTSeeders     map[FileAMTroot]map[TAUaddress]config;// include timestampInRelaySwitchTimeUnit // IPFSaddress from myPeers[chainid][TAUaddress]
11. myFileAMTroots       map[FileAMTroot]config;// include filename ; // a  list for imported or downloaded files trie
   
12. mytotalFileAMTDownloadedData
13. mytotalFileAMTUploadedData

```
## Temporary Variables
* currentChainID
* currentAccountDB map[ChainID]map[TAUaddress]map[Total Balance | POT power | ...] value

## Concept explain
- Single thread function for full chains voting and full pool download. To increase performance, run concurrency on top level. 
- **Block size is 1**, one tx included in each block. This is important to prevent complex double income issue. 
- Miner is what nodes call itself, and sender is what nodes call other peers. In TAU POT, all miners predict a future state; 
- Safety is the CBC Casper concept of the safe and agreed history by BFT group. The future contract result state is a CBC prediction. TAU uses this concept to describe the prediction and safety as well, but our scope are all peers than BFT group.
- Mutable range is defined as "one week" for now, beyond mutalble range, it is considered finality.<br/> <br/>

- HamtGraphRelaySync(relay multiaddress, remotePeerIPFS addr, chainID, cbor.cid, selector); // replaced the IPFS relay circuit. When cbor.cid is null, it means asking peer for the ContractResultStateRoot prediction on the chainID.
- AMTgraphRelaySync(relay multiaddress, remote ipfs peer, cbor.cid, selector); cid can not be null. 
- In both GraphRelaySync, it needs to test wether the target KV holding cbor.cid are already in local or not. <br/> <br/>
- Whoever provides state root, it has to include all witness data. Principle of traverse, once in a "ChainID+relay+peer" communication, we will not incur another recursive process to a new peer to get supporting evidence. If some Key-values are missing to prevent validation, just abort process to go next randomness. <br/> <br/>

- Address system: 
- TAU private key: the base for all community chain address generation;
- ChainID := `Nickname`+`blocktime` + signature(timestamp) // in stateless environment, chain config info needs to be embedded into ChainID, otherwise, might be lost. <br/> <br/>

- TX types
   * coin base, msg is the only transaction attached
   * send, msg is the contract relating to this tx
   * receive, 
   * relay, msg is the contract relating to this tx, include the relay info
   * file, msg is the contract relating to this tx, include discription of the file or future file command
   * founders claim for bootstrap a new chain <br/> <br/>
   
- relay: each chain config relay on own chain by members, TAU mainchain annouce the relay canditimestamps in the daily basis, each node config own successed relays. three of those sharing the time slots: 1:2:7  
- download: TAU always download entire myDownloadPool rather than one file. This is like IPFS on a single large file space, than torrents are file specific operation. 
- POT use power as square root the nounce. 
- Stateless for blockchain scope, statefull for address scope. TAU technology is a pure stateless in blockchain level. There is no full nodes. However, TAU implement statefull for each address data, which means each address has to store own state information. Each node will pin: states chain passed mutable range and all blocks with own address transactions; along with these info, the underline blocks will contain other peers info as well. 

- graphSync ->  RootSyncViaRelay ( relay_multiAddr, peerIPFSAddr, ChainID, root ); // root could be null
     - No need to back and forth locate cid for KV.
     - No need to check local KV availabity
     - No need to do two phase waiting on relay. 
   
## AMT are states for contract chain history. decentral and stateless. one root one cborblock.
verifcation is provide witness from mutalbe ragne to curernt all blocks. 
Contract
```
1. ContractJSON // e.g Contract8909JSON = {"version", "safetystateroot", "contract number = 8909", ...,"signature"}
```
Sender transactions: stateless wiring tx include **TWO** parts asynchorisely, spend and income.
```
2. `TAUaddress`SpendNonce ++;  // POT power = senderNounce + receiverNounce; Ta..xSpendNounce = 9++ = 10
3. `TAUaddress``SpendNonce`TotalSpend += amount;  // Ta..xSpend10TotalSpend = 9++ = 1000+100= 1100
4. `TAUaddress`PreviousSpendContractResultStateRoot . // miner will switch hamt root to verify history k-v, tx sender will monitor blockchain for stateroot confirm.  Ta..xPreviousSpendContractResultStateRoot = some cid. 

FileSeeding/RelayRegister/ChainFoundersClaim transactions are not in state key value, 
because nonce consensus is hard in shared keys. Peers has to traverse history to get those, more relay on leveldb. 
```
Receiver transactions: stateless blockchain requires adddress to claim income. Wiring Tx is two steps to full completion.
```
2. `TAUaddress`IncomeNonce ++; 
3. `TAUaddress``IncomeNonce`TotalINcome  += amount; // Income = sender's amount - transaction fee
4. `TAUaddress`PreviousIncomeContractResultStateRoot; 
5. `TAUaddress``IncomeNonce`UTXOContractResultStateRoot;  // point to a spend contract state root

Block miner: coinbase and genesis
2. same as receiver tx 2 
3. same as receiver tx 3  // form total tx fee or genesis coins issue
4. same as receiver tx 4

```
Four types of root:
- ContractJSON/safetystateroot: link to previous state
- PreviousSpendContractResultStateRoot: link to address's previous spend root
- PreviousIncomeContractResultStateRoot: link to address's previous income root
- UTXOContractResultStateRoot: line to address income associated UTXO


## Constants
* 1 MutableRange:  1 week
* 2 VotingPercentage: voting cover percentage 67%
* 3 RelaySwitchTimeUnit: relay time base, 15 seconds, which is what peers comes to their scheduled relays. 
* 4 WakeUpTime: sleeping mode wake up random range 5 minutes
* 5 GenesisDefaultCoins: default coins 1,000,000
* 6 initial difficulty according to the BlockTime.
* 7 BlockTime :  5 minutes;  
* 8 MaxBlockTime: 30 minutes, when no body mining, you have to generate blocks. 
* 9 auto-download total daily limit 20mb; files smaller than 2mb, video download 1%; auto-seeding on the download files.

## Community chain
### Genesis
* with parameters: block time, chain nick name, coins total - default is 1 million.  // initial mining peers is established through issue coins to other addresses. 社区链创世区块
```
// build genesis block
Y:= { 
1. version;
2. timestampInRelaySwitchTimeUnit;  //  it is timestamp/RelaySwitchTimeUnit
3. contractNumber:=0 int32;
4. ChainID := `Nickname`+`blocktime` + hash(signature(timestampInRelaySwitchTimeUnit)) // chainID is the only information to pass down in the stateless mode.
5. SafetyContractResultRoot = null; // genesis is built from null.
6. basetarget;
7. cummulative difficulty int64; // ???
8. generation signature;
9. amount = 1,000,000; // GenesisDefaultCoins 币数量
10. IncomeNonce:=0;
11. IPFSsigOn(minerAddress); //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
12. msg; // "contact information and profile"
13. signature; //by genesis miner to derive the TAUaddress
}
// build genesis state
G := cid new.hamt_node(); // execute once per chain, for future all is put.
* populate all related HAMT Key-values.  
G.hamt_add(`TAUaddress`IncomeNonce, 0);
G.hamt_add(`TAUaddress``IncomeNonce`TotalIncome, 1,000,000); 
G.hamt_add(`TAUaddress``IncomeNonce`JSON = "0";
G.hamt_add(`TAUaddress``IncomeNonce`UTXOhistory, compress(0));
G.hamt_add(ContractNumber,"0");
G.hamt_add(Contract0JSON,Y);
G.hamt_put(cbor)
* populate all database variables. 
      * mySafetyContractResultStateRoots[`ChainID`] = null;
      * mySafetyContractResultStateRootMiners[`ChainID`] = Tminer;
      * mypreviousSafetyContractResultStateRootMiner[`ChainID`] = null;
      * ...
// no need to config relay and peers, myRelays and myPeers will be populated when system starts in process E and community chains will use time slots to touch TAU relays and peers. 

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
- in the sleeping mode, random wake up between 1..WakeUpTime, if Wifi is off, then stop; else run for a cycle of all chains follow up, and check whether in power charging, yes to turn on wake lock. <br/> <br/>

- Notify on the iterface- wifi only for data flow, keep charging to prevent sleep. the data dash board, with a button to pause everything A-D. 
```
1.Generate "chainID+relay+peer" combo, Pick up ONE random `chainID` in the myChains,
according to the global time in the base of RelaySwitchTimeUnit, H = hash (time in RelaySwitchTimeUnit base + chain ID) 

call func PickupRelayAndPeer(H)

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

call func PickupRelayAndPeer(H)

goto step (3) until surveyed 2/3 of myPeers[`ChainID`][...]

6. accounting the voting rule, pick up the highest weight among the roots even only one vote, then use own safetyroot uptimestamp the CBC safety root: mySafetyContractResultStateRoots[`ChainID`] = voted SAFETY), 统计方法是所有的root的计权重，选最高。
goto (9)
// use  map[root string]Uint to count voting. 

7. graphRelaySync( Relay, peerID_A, chainID, null, selector(field:=contractJSON));
   if err= null, myRelays[successed].add{this Relay}
8. if ok, then verify {
if received ContractResultStateRoot/contractJSON shows a more difficult chain than SafetyContractResultStateRoot/contractJSON/`difficulty` and future root/contract/json/ safetyroot timestamp is passed clock, 不能在未来再次预测未来, then verify this chain's transactions until the MutableRange. in the verify process, it needs to add all db variables, hamt and amt trie to local. for some Key value, it will need `graphRelaySync` to get data from peerID_A;
goto (9)
}
  else { 
  failed or err , if the (current time - safety state time ) is bigger than MaxBlockTime, then generate a new state on own  safety root, this will cause safety miner = previous safety miner to trigger voting, go to (9)
  }; 
       else go to step 1. 

9. generate new state 
X = {
1. version;
2. timestampInRelaySwitchTimeUnit;  //  it is timestamp/RelaySwitchTimeUnit
3. contractNumber; // hamt_get(SafetyContractResultStateRoot,contractJSON) / contractNumber +1;
4. ChainID := `Nickname`+ `blocktime` + hash(signature(timestampInRelaySwitchTimeUnit)) // chainID is the only information to pass down in the stateless mode.
5. SafetyContractResultRoot; //  type cbor.cid
6. basetarget;
7. cummulative difficulty int64; 
8. generation signature;
9. amount = `total tx fee`; // negative value due to coinbase tx is a signed sending transaction. 
10. IncomeNonce ++;
11. IPFSsigOn(minerAddress); //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
12. msg = TXJSON; //profile info and msg {optcode, TXcode, msg with profile like telegram}; type of txs: send, receive, file, relay in TXJSON
13. signature; //by genesis miner to derive the TAUaddress
}  // finish X.

#### contract execute populate HAMT key values
G := hamt(SafetyContractResultRoot);

* populate all related HAMT Key-values in the example of sending transaction. 
##### send, receive, file, relay output.
G.hamt_add(`Tsenderaddress`SpendNonce, ++);  // POT power = senderNounce + receiverNounce
G.hamt_add(`Tsenderaddress``SpendNonce`TotalSpend, + amount);  
G.hamt_add(`Tsenderaddress``SpendNonce`JSONs, `3`);  

##### coinbase tx
G.hamt_add(`Tmineraddress`IncomeNonce, ++);
G.hamt_add(`Tmineraddress``IncomeNonce`TotalIncome, TXfee); 
G.hamt_add(`Tmineraddress``IncomeNonce`JSON = `3`;
G.hamt_add(`Tmineraddress``IncomeNonce`UTXOhistory, compress(`3`+","+`Tmineraddress``IncomeNonce`UTXOhistory));
G.hamt_add(ContractNumber,`3`);
G.hamt_add(Contract0JSON,X);

#### finish contract execution
G.hamt_put(cbor)

* populate all database variables. 

* myContractResultStateRoots[`ChainID`]
* mypreviousSafttyContractResultStateRootMiner[`ChainID`] = mySaftyContractResultStateRootMiner[`ChainID`];
* mySafttyContractResultStateRootMiner[`ChainID`] = Tminer;  // this is for deviting go voting or not
* mySafetyContractResultStateRoots[`ChainID`] = SafetyContractResultStateRoot;
* myPeers[`ChainID`][ ].add(`Tminer);
* myFileAMTSeeders[fileAMT][ ].add(`ChainID``TAUaddress`IPFSaddr)
* For file upload to chain
      * myFileAMTroots.add(fileAMTroot)
* For file seeding from other peers
      *myDownloadPool.add(fileAMTroot)
* myPeers.add(`Tminer` and ipfs address);
* myRelays.add; add download and upload, add tx pool...

go to step (1) to get a new ChainID state prediction
```
func PickupRelayAndPeer(H)
```
{if H div 10, remainder 余数 is 0

to hash(myRelays[`ChainID`][timestamp less than 3Xmutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer // if any one of those fields are null, means the chain is very early, then use null adress move on. //信息不全就是链的早期，继续进行 

else if 1,2
to hash(myRelays[TAUchain][timestamp within mutable range] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 

else 3,4,5,6,7,8,9
to hash(myRelays[successed][timestamp within 9xmutablerange] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 
}
```
## C. File Downloader - nonconcurrency design // ipfs layer
* For saving mobile phone resources, we adopt non-concurrrency execution. The entire download pool is one big file to randomly retrieve.

```
1. Generate chain+relay+FileAMT+Seeder+Piece combo: 
      Pickup ONE random `chainID` in the myChains[ ],
according to the global time in the base of RelaySwitchTimeUnit, H= hash (time in RelaySwitchTimeUnit base + chain ID)

call func PickupRelayAndPeer(H)
Randomly request ONE File from myDownloadPool[`ChainID`]. Randomly select a seeder from the * myFileAMTSeeders[fileAMT][ChainID][seeder  ]. use pieceSelection select a piece N from fileAMTroot.count
ONE Chain + ONE Relay + ONE FileAMT + ONE seeder peer + ONE piece. 

2. If the piece is in local, go to step (1); 
else 
- AMTgraphRelaySync(relay, chainID, `FileAMTroot``ChainID``Seeding`Nonce`IPFSPeer, `fileAMTroot`, selector(field:=piece N))
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
 * myRelays[TAU].add{initial relays} //populate relays. 
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
* 1. Create Blockchain: Own a blockchain with 1 million coins to build a community for video and files sharing. 
     * 一键建立区块链：默认配置5分钟区块时间，自动给名字，自动给创世币数量，提供一个change人口和create创建入口。
     * 自动起名字提供一个默认名字字典
* 2. Send Coins:  to friends.  // get their video previews and 2% automatically on the followed friends.  
* 3. Upload Files: Seeding files to keep ads free. // next step upload to a new seeding chain or existing chain. <br/> <br/>
     * 1. File import
     * 2. Choose Chain or Create Blockchain
     * 3. Publish
* Data dashboard:  upload - download = free download data amount, more than 1G, wifi only. "seeding to increase data"<br/> <br/>

### Community 社区
- follow chain, first layer
- follow members, second layer
- member messages & file, third layer, support import
### Files
- File imported to TAU will be compressed and chopped by TGZ, which includes directory zip, pictures and files. Chopped file pieces will be added into AMT (Array Mapped Trie) with a `fileAMTroot` as return. Filed downloaded could be decompressed to original structure.  Files downloaded is considerred imported. Only seeded file will be pinned in local. 
- Video will be only chopped and kept original compression format to support portion play. We will support a `hopping player` to play the downloaded pieces. 
- Apps autodownload and autoseed; autoseed is an transaction. 
   - app can auto download files and videos according to config include percentage.
   - for files only 100% download and accept uplimited for config; for videos, percentage is supported with uplimit. 
   ```
   - func pieceSelection(downloadPercentage 1%-100%, file total pieces) p is the piece number selection; 
      for (n=1; n++; (n/downloadPercentage + remainder余数(SeederTAUAddr/100))<=total pieces) 
         {  
         p= INT( remainder余数(SeederTAUAddr/100) + n/downloadPercentage )
         return p;
      }
     example: assume remainder is 3 
     total piece is 1000; download percentage 10%
     n : 1 .. 1000 * 10% = 1..100
     p = 1/0.1 .. n/0.1 = 13, 23, 33, .. 993
      
   ```
- import files
- share file to friend
- share file to community chain
- seeding files and unseeding
- pin a file, no directory at now, sort by timestamps and size
- delete a file
### Forum 论坛
- according to the following list, display files uploaded and its description. users can follow sender or blacklist them. 

### Mining and account balances on different chains. 
- coins mining config

# To do 
- [ ] file operation commands planning
- [ ] resource management process
- [ ] graphyRelaySync: two step via relay
- [ ] hamtGraphsync: multiple steps to get kv on the selector
- [ ] change ipfs block to 1m.  ipld blocksize. 
