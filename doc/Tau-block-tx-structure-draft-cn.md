# TAU - share and preserve data
```
thread: chain and own account search; building blocks and response; attachement download relentless(tsendernounceattachlog in the future block. ); 
User experienses: data upload as TGZ include all types, data download from swarm and export to specific types eg mp4; do not in app play media to avoid illegal content; 
```
# A Procedure for building new blocks for both miner and regular.  no graphsync
### uppon request by remote peer for upcoming block.
### from B_searchinig process, get CBC safety roothash -cid. hamt_get(safetyroothash, tn:=TminerNounce (ex. TAUdavidwu1098..qNounce=10)) if fail abort(means not found own account and back to search voting).
* key 1 = tn: TAUdavidwu1098..qNounce = 10
 { **区块链内容** prevousStateroot-cid; version,8; timestamp, 4; base target, 8; cumulative difficulty,8 ; generation signature,32;miner TAU address, 20; miner nounce=tn++ e.g 11, 8, mining is treated as a tx sending to self, nounce ++;senderProfileJSON,1024,Tminer..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; }; 形成 relay log.Ta..xNounce1004RelayLog = {   } 遍历  ta..xnounce 1 .. 1004, hamt(cid, block 1234567); stateless =local 本地txOriginalJSON; {original JSON from peers nounce for wiring and message type 1&2 senderProfileJSON,1024,Ta..xProfile; {TAU:Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; opt_code, 8, 2;
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
hamt_add(statejson, new block), this is new contract hash for execution  "hash98798234..sd"
execute results ....


## key 2 = contract hash ={block content or contract}
### - a. add coinbase tx
* Key-3a. hamt_add(Tminer..xNounceTXOutputJSON ={  TAUDavidwu88..q10TXoutputJSON = ...
opt_code, 1 byte, 0; 0 - coinbase tx; amount,5;
current contract hash (key),  you can only get state root from txoutput, the block does not know own stateroot. 
}
* Key-4a. hamt_update(Tminer..xBalance, + amount);  TAUdavdiwu..qBalance = 33455
* Key-5a. TminerNounceAttachmentsLogJSON={ atachment1:VisitorTAUaddr,download %, charge coins from the request; attachement 2...}
 e.g:   TAUdavidwu..qAttachmentLOGjson = {}
hamt_put() n-> cid is the receipt state root for return.
-->receipt "future state root" for return. 

### - b.Coins Wiring
### Key-3b. senderNounceTXOutputJSON = {
opt_code, 8, 1;
current state root
};   //sendernounc = current statroot/STATEjson/txjson/sendertauaddress/nounce
### Key-5b. sender balance        | 5       	|Tsender..xBalance" | 10000 ; through senderNounce to get TXJSON, ProfielJSON
### Key-6b receiver balance      | 5     		|Treceiver..xBalance" | 10000

### - c.Message transaction
### Key-3c. senderNounceTXoutputJSON = { 
opt_code, 8, 2;
current stateroot
}
### Key-5c sender balance, 5  


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
1. graphsync( TminerPeerIPLD(y), ? DEFAUTL_CID - 0 means the hamt cid, select( coinbase tx));  coinbase is entry with current stateroot in it.   return receipt root= execution results of contract hash.
2. hamt_get(stateroot, "previous state root"); Until mutable range.
