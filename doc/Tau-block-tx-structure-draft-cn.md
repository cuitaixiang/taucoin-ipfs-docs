# TAU - share and preserve data
```
thread: chain and own account search; building blocks and response; attachement download relentless(tsendernounceattachlog in the future block. ); 
User experienses: data upload as TGZ include all types, data download from swarm and export to specific types eg mp4; do not in app play media to avoid illegal content; 
Profit ideas: community members can receive coin payment for hosting data or willing to do it free; TAU coins community will provide relay routers charging TAU coins and keep free TAU mining software development.
```
## Three processes exists: 
* A. Response with new futureContractReceiptStateRoot; 
* B. Collect votings from peers; 
* C. Attachment Downloader.
```
Concept: Miner is what nodes call themself, in TAU all nodes are miners predicting future; Sender is what nodes call other peers.
```
### A. When a miner receives Graphysync request for producing future state, it will process following steps: 
* get CBC(consensus by construction) current safety contract receipt state root, cid. This is the result of voting on safety. If there is no safety found, such as out of mutable range, then exit with no response, continue to allow Collecting votes happenning. 
* hamt_get(cid,TAUminerImorpheus..xNounce); // miner's nounce is the base for coinbase transaction and new contract. If fail to get nounce, it means the node is too young and exist with no respones, continue to allow Collecting votes happenning.
```
TAUminerImorpheus..x is an exmple of TAU miner address belong to imorpheus
```
* generate key 1 for the future state, hamt_update(TAUminerImorpheus..xNounce, TAUminerImorpheus..xNounce +1); 
* generate key 2 for the future state, hamp_add(contractJSON, X);

```
X = { **区块链内容** prevousStateroot-cid; version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32;miner TAU address, 20; miner nounce=tn++ e.g 11, 8, mining is treated as a tx sending to self, nounce ++;senderProfileJSON,1024,Tminer..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; }; 形成 relay log.Ta..xNounce1004RelayLog = {   } 遍历  ta..xnounce 1 .. 1004, hamt(cid, block 1234567); stateless =local 本地txOriginalJSON; {original JSON from peers nounce for wiring and message type 1&2 senderProfileJSON,1024,Ta..xProfile; {TAU:Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; opt_code, 8, 2;
current stateroot
nounce, 8;
version,8, "0x1" as default;
timestamp,4,tx expire in 12 hours;
txfee;
contractblock number = previoushash(contract number)+1;
senderProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };
thread = "other sender address + tx nounce"; if thread equal self sender nounce, it is a new thread.
msgJSON,1024;{}

Attachmentroot = hamp_put(1-10000, section of video) Hash and 
attachment Size,32; use HAMT store multimedia
 
 with sender signature}previous hamt state root,
32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte, when at same difficulty, high signature number wins.}
```
### contract execute results
#### output coinbase tx

hamt_add(statejson, new block), this is new contract hash for execution  "hash98798234..sd"

* generate key 3a, coinbase transaction 
``` 
hamt_add(TAUminerImorpheus..xNounceTXOutputJSON =
{
opt_code, 0;// 0 - coinbase tx, 1 - means coin wiring, 2 - means data upload and message 
amount,5; // sum of transaction fees from the mining
}
```
* generate Key 4a. hamt_update(TAUminerImorpheus..xBalance,TAUminerImorpheus..xBalance + amount); // update balance
* generate Key 5a. hamt_add(TAUminerNounceAttachmentsLogJSON={ atachment1:graphsync request from VisitorTAUaddr; attachement 2...} // attachment host will charge coins for each graphsync on attachment as compensation, log is the proof, each new TAU address can get a mininum service from hosts such as 1G, it is configurable in the sendersProfile json. 
```
 hamt_add(TAUminerImorpheus..x11AttachmentsLogJSON={ "starwar:graphsync request from VisitorTAUaddr; attachement 2...} 
```

#### output Coins Wiring tx
* generate Key-3b. TAUsenderNounceTXOutputJSON = {}
``` 
hamt_add(TAUsender..xNounceTXOutputJSON =
{
opt_code, 1; // this is a wiring, very simple
}
```
* generate Key-4b. hamt_update(TAUsender..xBalance,amount); // update balance
* generate Key-5b hamt_update(TAUtxreceiver..xBalance,amount);

### - c.Message transaction
* generate Key 3c. TAUsenderNounceTXOutputJSON
``` 
hamt_add(TAUsender..xNounceTXOutputJSON =
{
opt_code, 2; // this is an attachment or message
}
```
* generate Key 4c hamt_update(TAUsender..xBalance,amount); // update balance


### produce the final futureContractReceiptStateRoot = hamt_put(cbor); // this the for returning to requestor


# background procedures for mining user, which requires wifi and power plugged, while missing wifi or plug will switch to normal user mode. TAU POT consensus is a type of CBC enhanced POT.
0. load android wake-lock and wifi-lock;  utorrent
1. random walk until connected to next relay, and keep a list of relays; random walk until connected to next miner, and keep a list of addresses with power and balance; note: combine relay and peer random walking in one step to reduce connection jam;
2. request the future state from the peer; 
get coinbase tx, from coinbase tx, get previous root, 
3. graphsync for TminerBalance, if success, hamt_get(TminerBalance and TminerNounce) traverse ONEWEEK history states from the miner, while keep accounting of the state root array; 
mining based on curent safty K and build&validate (k+1) state JSON for requested; when connection timeout from the peer, go to step(1)
4. go to step (1), until half of the know mining peers are traversed. 
5. calculate the CBC safety state k, if k it out of mutable range, that is n-ONEWEEK, then go to step (1). 
6. random walk until connect to a next relay; random walk until connect to a next miner
7. start mining by following the longest chain, and verify n-ONEWEEK to n+1, build&validate (n+1) state JSON for requested when peers timeout, go to step (6)
8. when new added mining nodes increase 33% or self-disconnected 12 hours, go to step (1).
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync. mutable range is set for one week. 
当用户访问视频时，先拿到transactionNounceLog,里面含有swarm peers，就是种子服务，然后开始下载，其实去多个ipld node下载。一个video可以被切成很多key-blocks. 

#### background procedures for non mining users on battery or 4G 
0. release android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance; note: combine relay and peer random walking to reduce connection jam;
2. request the miner peer for the future state according to CBC (correct by construction); 
get coinbase tx, from coinbase tx, get previous root, 
3. graphsync for TminerBalance, if success, hamt_get(TminerBalance and TminerNounce). traverse ONEWEEK history states from the miner and keep accounting of the root array; mining based on curent safty k to propose k+1 state with own tx uppon request; when connection timeout, go to step(1)
4. update the CBC safety state k, then go to step (1). 
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.


# B procedure for getting a block for voting. 无目的的搜索。对于某个视频
Sender is other peer request your blocks, miner is you call yourself to generate blocks.
when found new relay always add to local trie:   TminerRelay(TminerrelayTotal++)=Qm..x; in local trie hamt
when found new peers always add TAUaddress to local trie: TminerPeerTAU(TminerPeersTotal ++)=Tsender..x; 
when found new peers always add IPFS address to local trie: TminerPeerTAU"IPLD"(TminerPeerstotal)=; e.g TAUDavid123..xIPLD = Qmdavid...x

x=rand(TminerRelaysTotal), y =rand(TminerPeersTotal); 
TminerRelay(x);
TminerPeerTAU(y);TAUdavid..x
TiminerPeerIPLD(y); Qmdavid..x  -> form multiaddress := relay+tminerpeerIPFSaddr
Procedure for start request future blocks:
1. graphsync( TminerPeerIPLD(y), ? DEFAUTL_CID - 0 means the hamt cid, select(hamtkey contractJSON));  coinbase is entry with current stateroot in it.   return receipt root= execution results of contract hash. **problem** we do not know this cid at the moment, but we do now the contractJSON as key from protocol
2. hamt_get(stateroot, "previous state root"); Until mutable range.
