# TAU - File sharing on blockchains

User experienses:= {
- Core: 1. Create blockchain 2. Upload files
- Data dashboard
   - if (download - upload) > 1G, start ads. display "seeding for ads free download"
   - Auto download, Auto Seeding, Mining status. This is applying to all chains. 
- TAU provides global relay services and chain annoucement.
- All chain addresses are derivative from one private key. Nodes use IPFS peers ID for ipv4 tcp transport. (the association of TAUaddr and IPFS address is through signature using ipfs RSA private key)i think .
- User uses relay from TAU, own chain records and suzheccessed history, in the weight of 2:1:7
- auto seeding is off chain function, downloader will randomless picking up peers recorded on chain. 

- app operation mode:
   - on telcom data: Watching both random and followed chains(2:8). support manual download, config buttons default:
      - manual download: ON
      - manual seeding: ON
      - auto download file: OFF
      - auto seeding: OFF
      - mining: OFF
   - on wifi: config buttons default:
      - manual download: ON
      - manual seeding: ON
      - auto-download: ON, autdownload files smaller than 10M, 9 key frames. 
      - auto-seeding: ON.
      - mining: ON
   - on wifi + power plug: multiple processes allowed. 
   - in the sleeping mode, random wake up between 1..WakeUpTime,  run for one minute follow the above rules. <br/> <br/>
}

Business model:= {
- Tau foundation will develop TAU App and provide free public relays, in return for admob/mopub ads income to cover relay data cost. Any one can contribute relays on both TAU and own chain. 
- Individual nodes will see ads to keep using data for free, the more data upload, the less ads. 
- TAUT is tau-torrent, a file sharing service by tau dev. 
- TAU is a relay annoucement service by tau dev. 
}
--- 

## Six core processes

* A. Response with predicted blockRoot. One instance to prevent ddos. 
* B. Collect votings from peers. (single thread func)<br/> <br/>
* C. File Downloader. (download files and logging download data)
* D. Reponse to file downloader request. (logging upload data). One instance to prevent ddos. <br/> <br/>
On Environment
* E. Process manager; schedule above 4 processes instance existing and prevent DDOS. Process genesis. 
* F. Resource management: TBD. 

## Persistence variables in database 
```
1. myChains             map[ChainID] config; //Chains to follow or mining, string is for planned config info
2. myblockRoots         map[ChainID] cbor.cid; // the new contract block
3. myMutableRange       map[ChainID]string
4. myPruneRange         map[ChainID]string
5. myPeers              map[ChainID]map[TAUaddress]config;// include IPFSAddr
6. myRelays             map[ChainID]map[RelaysMultipleAddr]config;// incllude timestampInRelaySwitchTimeUnit; timestamp is to selelct relays in the mutable ranges. 
7. myTXsPool            map[ChainID]map[hash(txjson)]TXJSON; // include timestampInRelaySwitchTimeUnit
8. myDownloadPool       map[ChainID]map[FileAMT]config;   // when file finish downloaded, remove chainID/fileAMT combo from the pool
9. mytotalFileAMTDownloadedData
10. mytotalFileAMTUploadedData
11. myCheckPoint        map[ChainID] config
12. myNewCheckPoint     map[ChainID] config 
```

## Concept explain
- Single thread function for chains voting and pool download. To increase performance, run concurrency on top level E. 
- **Block size is 1 tx per block, 5 minutes a block **
- TAU private key: the base for all community chain address generation;
- ChainID := `Nickname` + signature(privatekey + timestampInRelaySwitchTimeUnit) <br/> <br/>

- TX types
   * coin base, msg is the only transaction attached
   * send, msg is the contract relating to this tx
   * relay, msg is the contract relating to this tx, include the relay info
   * file, msg is the contract relating to this tx 
   * new chain annoucement tx on tau or others<br/> <br/>
   
- relay: each chain config relay on own chain by members, TAU mainchain annouce the relay canditimestamps in the daily basis, each node config own successed relays. three of those sharing the time slots: 1:2:7  
- download: TAU always download entire myDownloadPool rather than one file. This is like IPFS on a single large file space, than torrents are file specific operation. 
- POT use power as square root the nounce. 
-   
- 投票策略设计。
   - 投票范围：当前CheckPoint往后的 `MutableRange` 周期为计算票范围。这个范围会达到未来。
   - 每到一次新的range结束时，统计投票产生新的CheckPoint, 得票最高的当选, 同样票数时间最近的胜利。
      - 如果投出来的新CheckPoint, 时间早于上次CheckPoint，表示finality失败，本节点使用新的CheckPoint，继续发现最长链，在找到新的最长链的情况下，检查下自己的交易以前已经上链的是否在新链上，不在新链上的放回交易池。。
      - 新节点上线，快速启动策略，随机相信一个链拿到数据，设置新链的（顶端-`MutableRange`）为CheckPoint，开始出块做交易，等待下次投票结果。下次大概率checkpoint会早于现在的checkpoint，所以第二天会重组账号信息。
      - 如果投票出的新CheckPoint，晚于上次CheckPoint而且在同一链上，说明是链的正常发展，继续发现最长链，在找到新的最长链的情况下，检查下自己的交易以前已经上链的是否在新链上，不在新链上的放回交易池。
   - 存储建议：1. CheckPoint 前的放在levelDb. 2. CheckPoint 内的放在hamt, 每天凌晨清除hamt block。
   - 获得新root，如果是目前longest chain开始进入验证流程，如果不是进入计票流程. 

## IPLD stores blockchain
```
blockJSON  = { 
1. version;
2. timestampInRelaySwitchTimeUnit;  //  it is timestamp/RelaySwitchTimeUnit
3. BlockNumber;
4. PreviousBlockRoot; // genesis is built from null. cid.  node.link.
5. basetarget;
6. cummulative difficulty;
7. generation signature;
8. IPFSsigOn(minerAddress); //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
9. msg; // One Tx
// CRITICAL block, mostly fungible blocks, KV embedded to cover Mutable range roll back. When roll back, update memory for follow variables. 
10. ChainID := `Nickname`+ hash(privatekey + timestampInRelaySwitchTimeUnit)
11. `Tsender`Noune;
12. `Tsender`Balance;
13. `Tgenesis/Tminer`Balance;
14. `Treceiver`Balance;
 
15. signature; //by genesis miner to derive the TAUaddress
}
// FileSeeding/RelayRegister/ChainFoundersClaim transactions results are not in critical block key value. 
```

## Constants
* 1 MutableRange:  3 DAYS
* 2 PruneRange: 6 Months
* 3 RelaySwitchTimeUnit: relay time base, 15 seconds, which is what peers comes to their scheduled relays. 
* 4 WakeUpTime: sleeping mode wake up random range 10 minutes
* 5 GenesisCoins: default coins 1,000,000. Integer, no decimals. 
* 6 GenesisCummulativeDifficulty:  according to the BlockTime.
* 7 MinBlockTime:  5 minutes;  this is fixed block time. do not let user choose as for now.
* 8 MaxBlockTime: 30 minutes, when no body mining, you have to generate blocks. 
* 9 AutoDownloadSize: 9; files smaller than `AutoDownloadSize`MB, video download `AutoDownloadSize`Frames

* 11 Relay distribution ratio:  2:1:7 tau/self/successHistory.
* 12 RequestTimeSpan: 5 minutes, minblocktime; ChainID+peer Repeat timespan: for same chainID+peer, the time span between repeat request. // when a chain only has few address, this prevents frequent requesting for root. 


## Community chain
### Genesis
* with parameters: nick name. 
```
// build genesis block
blockJSON  = { 
1. version;
2. timestampInRelaySwitchTimeUnit;  //  it is timestamp/RelaySwitchTimeUnit
3. BlockNumber:=0 int32;
4. PreviousBlockRoot = null; // genesis is built from null.
5. basetarget;
6. cummulative difficulty int64; // ???
7. generation signature;
8. IPFSsigOn(minerAddress); //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
9. msg; // One Tx
10. ChainID := `Nickname`+ hash(privatekey + timestampInRelaySwitchTimeUnit)
11. signature; //by genesis miner to derive the TAUaddress
}

// no need to config relay and peers, myRelays and myPeers will be populated when system starts in process E and community chains will use time slots to touch TAU relays and peers. 

```
## A. One miner receives GraphSync request from a relay.  
Miner does not know which peer requesting them, because the relay shields the peers. Two types of requests: "BlockRoot[]" or null. 
-  Receive the `ChainID` from a graphRelaySync call
-  If `ChainID` exist in myChains, return myblockRoots[`ChainID`] and blocks according to selector; else response null

## B. Votings, chain choice and block generation
This process is for multiple chain, multiple relay and mulitple peers.  
```
1.Generate "chainID+relay+peer" combo, Pick up ONE random `chainID` in the myChains,
according to the global time in the base of RelaySwitchTimeUnit, H = hash (RelaySwitchTimeUnit + chain ID) 
- call func PickupRelayAndPeer(H)
- if the  RelaySwitchTimeUnit is odd number go voting; 
go to step (3); // 时间戳单数就投票 
else go to step (7); // verify longest chain.

3. GraphRelaySync( Relay, peerID, chainID, null, selector(field:=contractJSON)); if err go to (6)
   myRelays[successed].add{this Relay}
4. Collect MutableRange ago, full day roots. 

6. At midnight 0:00Am, accounting all the voting results for that day to get immutable block for tomorrow, 统计方法是所有的root的计权重，选最高。
goto (1)

7. if mutablerange is null, go to (1)
graphRelaySync( Relay, peerID_A, chainID, null, selector(field:=contractJSON));
   if err= null, myRelays[successed].add{this Relay}
8. if ok, then verify {
if received contractJSON shows a more difficult and future root/contract/json/ root timestamp is passed clock, 不能在未来再次预测未来, then verify this chain's transactions from the MutableRange, ;
goto (9)
}
  else { 
  failed or err , if the (current time -  block time ) is bigger than MaxBlockTime, then generate a new block on own root, this will cause  miner = previous  miner to trigger voting, go to (9)
  }; 
       else go to step 1. 

9. generate new block 
X = {
1. version;
2. timestampInRelaySwitchTimeUnit;  //  it is timestamp/RelaySwitchTimeUnit
3. contractNumber; // hamt_get(PreviousBlockRoot,contractJSON) / contractNumber +1;
4. ChainID := `Nickname`+ `blocktime` + hash(signature(timestampInRelaySwitchTimeUnit)) // chainID is the only information to pass down in the blockless mode.
5. PreviousBlockRoot; //  type cbor.cid; if mutable range is null, PreviousBlockRoot = null, means new block is just carrying transaction. 
6. basetarget;
7. cummulative difficulty int64; 
8. generation signature;
9. IPFSsigOn(minerAddress); //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
10. msg = TXJSON; //profile info and msg {optcode, TXcode, msg with profile like telegram}; type of txs: send, receive, file, relay in TXJSON
11. signature; //by genesis miner to derive the TAUaddress
}  // finish X.

#### contract execute populate HAMT key values
G := hamt(PreviousBlockRoot);

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

* myblockRoots[`ChainID`]
* mypreviousSafttyblockRootMiner[`ChainID`] = mySaftyblockRootMiner[`ChainID`];
* mySafttyblockRootMiner[`ChainID`] = Tminer;  // this is for deviting go voting or not
* myPreviousBlockRoots[`ChainID`] = PreviousBlockRoot;
* myPeers[`ChainID`][ ].add(`Tminer);
* myFileAMTSeeders[fileAMT][ ].add(`ChainID``TAUaddress`IPFSaddr)
* For file upload to chain
      * myFileAMTroots.add(fileAMTroot)
* For file seeding from other peers
      *myDownloadPool.add(fileAMTroot)
* myPeers.add(`Tminer` and ipfs address);
* myRelays.add; add download and upload, add tx pool...

go to step (1) to get a new ChainID block prediction
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
1. Generate chain+relay+FileAMT+Seeder combo: pieces are not following random plan
      Pickup ONE random `chainID` in the myChains[ ],
according to the global time in the base of RelaySwitchTimeUnit, H= hash (time in RelaySwitchTimeUnit base + chain ID)

call func PickupRelayAndPeer(H)
Randomly request ONE File from myDownloadPool[`ChainID`]. Randomly select a seeder from the * myFileAMTSeeders[fileAMT][ChainID][seeder  ]. use  select a piece N from fileAMTroot.count. // piece selection is not random. 
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
 * onGenesisMsg creation
 * onHamtGraphsyncMsg, 
 if hamtsync not finish, reject; else 
 A.Response HAMT with predicted blockRoot, which is a hamt cbor.cid. (service response to HamtGraphRelaySync). One instatnce per connection to prevent ddos. 
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
- Video will be only chopped and kept original compression format to support portion play. 
- Apps autodownload and autoseed; autoseed is not a tx. 
   - app can auto download files and videos according to config include percentage.
   - for files only 100% download and accept uplimited for config; for videos, percentage is supported with uplimit. 
- import files
- share file to friend
- share file to community chain
- seeding files and unseeding
- pin a file, no directory at now, sort by timestamps and size
- delete a file
### Forum 论坛
- according to the following list, display files uploaded and its description. users can follow sender or blacklist them. 

# To do 
- [ ] file operation commands planning
- [ ] resource management process
- [ ] graphyRelaySync: two step via relay
- [ ] hamtGraphsync: multiple steps to get kv on the selector
