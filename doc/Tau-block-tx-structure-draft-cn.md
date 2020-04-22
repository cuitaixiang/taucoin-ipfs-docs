# TAU - File sharing on blockchains
User experienses:= {
- Core: 1. Create blockchain 2. Share files //直接分享，不用上传和转存储
- Data dashboard: one place to config for all chains, this version does not differentiate chains.
   - Seeding: ON for for commercial free; OFF with commercials.  //简单化，体验好
   - Preview download: ON/OFF
   - Mining: ON/OFF
- TAU provides relay and chain genesis annoucement.
- All chain addresses are derivative from one private key. Nodes use IPFS peers ID for ipv4 tcp transport. (the association of TAUaddr and IPFS address is through signature using ipfs RSA private key).
- User uses relay from TAU, own chain records and success history, in the weight of 2:1:7. Time frame is `PruneRange`. 
- app operation mode:
   - on telcom data: show messages on followed chains, default follow TAUT and TAU. support manual download, config buttons default:
      - Preview: OFF; // download files smaller than `PreviewSize`M, video `PreviewSize` key frames. 
      - Seeding: OFF
      - Mining: OFF
   - on wifi: config buttons default:
      - Preview, Seeding, Mining: ON  // 
   - on wifi + power plug: multiple processes allowed. 
   - in the sleeping mode, random wake up between 1..WakeUpTime,  run for one minute follow the above rules.
}
Business model:= {
- Tau foundation will develop TAU App and provide free public relays, in return for admob/mopub ads income to cover relay data cost. Any one can contribute relays on both TAU and own chain. 
- Individual nodes will see ads to keep using data for free, the more data upload, the less ads. 
- TAUT is tau-torrent, a file sharing service by tau dev. 
- TAU is a relay annoucement service by tau dev. 
}
--- 
## Six core processes

* A. Response with predicted BlockRoot. 
* B. Request longest chains and votings from peers.
* C. File Downloader. 
* D. Reponse to file downloader request. <br/> <br/>
On Environment
* E. Process manager; schedule above 4 processes' instances and prevent DDOS. Process genesis block. 
* F. Resource management: TBD. 

## Persistence variables in database 
```
1. myChains             map[ChainID] config; //Chains to follow or mining, string is for planned config info
2. myBlockRoots         map[ChainID] cbor.cid; // the new contract block
3. myMutableRange       map[ChainID]string
4. myPruneRange         map[ChainID]string
5. myPeers              map[ChainID]map[TAUaddress]config;// include IPFSAddr
6. myRelays             map[ChainID]map[RelaysMultipleAddr]config;// include timestampInRelaySwitchTimeUnit; timestamp is to selelct relays in the mutable ranges. 
7. myTXsPool            map[ChainID]map[hash(txjson)]TXJSON; // include timestampInRelaySwitchTimeUnit
8. myDownloadPool       map[ChainID]map[FileAMT]config;   // when file finish downloaded, remove chainID/fileAMT combo from the pool
9. mytotalFileAMTDownloadedData
10. mytotalFileAMTUploadedData
11. myCheckPoint        map[ChainID] config
12. myNewCheckPoint     map[ChainID] config 
```

## Concept explain
- Single thread function for chains voting and pool download. To increase performance, run concurrency on top level E. 
- **Block size is fixed on 1 tx per block, 5 minutes a block **
- ChainID := `Nickname` + signature(privatekey + timestampInRelaySwitchTimeUnit) <br/> <br/>
- TX types
   * coin base, msg is the only transaction attached
   * send, msg is the contract relating to this tx
   * relay, msg is the contract relating to this tx, include the relay info
   * file, msg is the contract relating to this tx 
   * new chain annoucement tx on tau or others<br/> <br/>
 
- relay: each chain config relay on own chain by members, TAU mainchain annouce the relay canditimestamps in the daily basis, each node config own successed relays. three of those sharing the time slots: 1:2:7. Observe window is PruneRange. 
- download: TAU always download entire myDownloadPool rather than one file. This is like IPFS on a single large file space, than torrents are file specific operation. 
- POT use power as square root the nounce. 
- 投票策略设计。
   - 投票范围：当前CheckPoint**往未来**的 `MutableRange` 周期为计算票范围。这个范围会达到未来。
   - 每到一次新的range结束时，统计投票产生新的CheckPoint, 得票最高的当选, 同样票数时间最近的胜利。
      - 如果投出来的新CheckPoint, 这个root不在目前链上，表示finality失败，本节点使用新的CheckPoint，继续发现和从CheckPoint验证最长链，在找到新的最长链的情况下，检查下自己的交易以前已经上链的是否在新链上，不在新链上的放回交易池。。
      - 新节点上线，快速启动策略，随机相信一个能够覆盖到全部历史的链拿数据，设置链的（顶端-`MutableRange`）为CheckPoint，开始出块做交易，等待下次投票结果。
      - 如果投票出的新CheckPoint，root在同一链上，说明是链的正常发展，继续发现最长链，在找到新的最长链的情况下，检查下自己以前已经上链的交易是否在新链上，不在新链上的放回交易池，交易池要维护自己地址交易到PruneRange。// 新的CheckPoint root不可能早于目前CheckPoint的。
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
10. ChainID := `Nickname`+ hash(privatekey + timestampInRelaySwitchTimeUnit)
11. `Tsender`Noune;
12. `Tsender`Balance;
13. `Tgenesis/Tminer`Balance;
14. `Treceiver`Balance;
15. signature;
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
* 9 PreviewSize: 9; files smaller than `PreviewSize`MB, video download `PreviewSize`Frames
* 10 Relay distribution ratio:  2:1:7 tau/self/successHistory.
* 11 RequestTimeSpan: 5 minutes, minblocktime; ChainID+peer Repeat timespan: for same chainID+peer, the time span between repeat request. // when a chain only has few address, this prevents frequent requesting for root. 


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
9. msg; // "hello world"
10. ChainID := `Nickname`+ hash(privatekey + timestampInRelaySwitchTimeUnit)
11. `Tgenesis`Balance; // 1,000,000
12. `Tgenesis`Nonce; // 0
13. signature; //by genesis miner to derive the TAUaddress
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
1.Generate "chainID+relay+peer" combo, Pick up ONE random `chainID` in the myChains ACCORDING to myTXsPool Chain weight.
according to the global time in the base of RelaySwitchTimeUnit, H = hash (RelaySwitchTimeUnit + chain ID) 
- call func PickupRelayAndPeer(H)
  If chainID+peer is requested within RequestTimeSpan, go to (1); //不要对一个peer重复访问
2. GraphRelaySync( Relay, peerID, chainID, root, selector); if err go to (1)
   myRelays[successed].add{this Relay}
3. if received contractJSON shows a difficult higher than current difficulty and blockJSON/PreviousBlockRoot/blockJSON/timestamp is passed present time, 不能在未来再次预测未来, then verify this chain's transactions from the CheckPoint;
   if verification successful go to (9); // found longest chain
   if the (current time -  block time ) is bigger than MaxBlockTime, go to (9) // no miners found, you have to make block
4. Put received data into CheckPoint voting pool. 

6. If the current time arrives at the votes counting time, accounting all the voting results for that MutableRange to get the block for new CheckPoint, 统计方法是所有的root的计权重，选最高。
   goto (1)

9. generate new block 
X = {
1. version;
2. timestampInRelaySwitchTimeUnit;  //  it is timestamp/RelaySwitchTimeUnit
3. contractNumber; // hamt_get(PreviousBlockRoot,contractJSON) / contractNumber +1;
4. ChainID := `Nickname`+ `blocktime` + hash(signature(timestampInRelaySwitchTimeUnit)) // chainID is the only information to pass down in the blockless mode.
5. PreviousBlockRoot;
6. basetarget;
7. cummulative difficulty int64; 
8. generation signature;
9. IPFSsigOn(minerAddress); //IPFS signature on `minerAddress` to proof association. Verifier decodes siganture to derive IPFSaddress QM..; 
10. msg = TXJSON; //profile info and msg {optcode, TXcode, msg with profile like telegram}; type of txs: send, receive, file, relay in TXJSON
11. `Tsender`Noune;
12. `Tsender`Balance;
13. `Tgenesis/Tminer`Balance;
14. `Treceiver`Balance;
15. signature; 
}  // finish X.

#### contract execute populate HAMT key values
G := hamt(PreviousBlockRoot);

* populate all related HAMT Key-values in the example of sending transaction. 
##### send, receive, file, relay output.

##### coinbase tx

#### finish contract execution
G.hamt_put(cbor)

* populate all database variables. 

* myblockRoots[`ChainID`]
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

to hash(myRelays[`ChainID`]  )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer // if any one of those fields are null, means the chain is very early, then use null adress move on. //信息不全就是链的早期，继续进行 

else if 1,2
to hash(myRelays[TAUchain]  )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 

else 3,4,5,6,7,8,9
to hash(myRelays[successed] )find the closest ONE relays. Randomly request ONE Peer from myPeers[`ChainID`][...]. 
ONE Chain + ONE Relay + ONE peer 
}
```
## C. File Downloader - nonconcurrency design // ipfs layer
* For saving mobile phone resources, we adopt non-concurrrency execution. The entire download pool is one big file to randomly retrieve.

```
1. Generate chain+relay+FileAMT+Seeder combo: pieces are not following random plan
      Pickup ONE random `chainID` in the myChains ACCORDING to myDownloadPool chain weight,
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

* 1. Create Blockchain: Own a blockchain with 1 million coins to build a community for video and files sharing. 
     * 一键建立区块链：默认配置5分钟区块时间，自动给名字，自动给创世币数量，提供一个change人口和create创建入口。
     * 自动起名字提供一个默认名字字典  
* 2. Share Files:
     * 1. Point to a File
     * 2. Choose Chain or Create Blockchain
     * 3. Publish

### Community 社区
- follow chain, first layer
- follow members, second layer
- member's messages & file, third layer, support import
### Files
- File does not store in IPFS repo to save space for mobile phone. 
- **File relay maybe through Http**
- share file to community chain
- pin a file, no directory at now, sort by timestamps and size
- delete a file
### Forum 论坛
- according to the following list, display files uploaded and its description. users can follow sender or blacklist them. 

# To do 
- [ ] resource management process
- [ ] graphyRelaySync: two step via relay
