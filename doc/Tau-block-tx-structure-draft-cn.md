# TAU - "undeletable and unlimited data sharing autonomous community"

## A. TAU is designed to enable mobile phone forming community cloud. Two types of operation mode exist. They are miner and regular. 
#### procedures for miner user, which requires wifi and power plugged, while missing wifi or plug will switch to normal user mode. TAU POT consensus is a type of CBC enhanced POT.
0. load android wake-lock and wifi-lock;
1. random walk until connected to next relay, and keep a list of relays; random walk until connected to next miner, and keep a list of addresses with power and balance; note: combine relay and peer random walking in one step to reduce connection jam;
2. request the (n+1) state from the peer; 
3. traverse ONEWEEK history states from the miner, while keep accounting of the state root array; 
mining based on curent safty K and build&validate (k+1) state JSON for requested; when connection timeout from the peer, go to step(1)
4. go to step (1), until half of the know mining peers are traversed. 
5. calculate the CBC safety state k, if k it out of mutable range, that is n-ONEWEEK, then go to step (1). 
6. random walk until connect to a next relay; random walk until connect to a next miner
7. start mining by following the longest chain, and verify n-ONEWEEK to n+1, build&validate (n+1) state JSON for requested when peers timeout, go to step (6)
8. when new added mining nodes increase 33% or self-disconnected 12 hours, go to step (1).
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync. mutable range is set for one week. 
当用户访问视频时，先拿到transactionNounceLog,里面含有swarm peers，就是种子服务，然后开始下载，其实去多个ipld node下载。一个video可以被切成很多key-blocks. 

#### procedures for regular users on battery or 4G 
0. release android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance; note: combine relay and peer random walking to reduce connection jam;
2. request the miner peer for the n+1 state according to CBC (correct by construction); 
3. traverse ONEWEEK history states from the miner and keep accounting of the root array; mining based on curent safty k to propose k+1 state with own tx uppon request; when connection timeout, go to step(1)
4. update the CBC safety state k, then go to step (1). 
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

## B. State structure with entry point of "cid" + peersID, key-values are:

### Key-1. stateNumber, 8; stateNumber=1234567

### Key-2. stateJSON1234567content={ 区块链内容

version,8; 

timestamp, 4; 

state number, 8; 

base target, 8; 

cumulative difficulty,8 ; 

generation signature,32;

sender/miner TAU address, 20; 

sender/miner nounce, 8, mining is treated as a tx sending to self, nounce ++;

senderProfileJSON,1024,Ta..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; }; 形成 relay log. 
Ta..xNounce1004RelayLog = {   } 遍历  ta..xnounce 1 .. 1004, hamt(cid, block 1234567); stateless =local 本地

txOriginalJSON; {original JSON from peers for wiring and message type 1&2  with sender signature}

previous hamt state root,

32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte, when at same difficulty, high signature number wins.

}

### Key-3 three type Transactions: 0-coinbase, 1-wiring, 2-message.
### - a.coinbase tx
### Key-3a. sender/minerNounceJSON ={

version,8, "0x1" as default;

opt_code, 8, 0x0 is coinbase;

nounce, 8;

timestamp,4,tx expire in 12 hours;

amount,5;

sender/minerProfileJSON,1024,Ta..xProfile; {TAU: Tsender..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

stateNumber, 8;

}

### Key-4a. sender/miner nounce;
### Key-5a. sender/miner balance;
### Key-6a. sender/miner nounceLogJSON; logging the peer connections include request and service

### - b.Coins Wiring
### Key-3b. senderNounceJSON = {

  opt_code, 8, 1;

nounce, 8;

receiver TAU address,20, type 2;

version,8, "0x1" as default;

timestamp,4,tx expire in 12 hours;

amount,5, type 1, 2;

stateNumber, 8;

txfee;

senderProfileJSON,1024,Ta..xProfile; {TAU:Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; 
};


### Key-4b. sender nounce	|8 			|Tsender..xNounce"	10;used as power
### Key-5b. sender balance        | 5       	|Tsender..xBalance" | 10000 ; through senderNounce to get TXJSON, ProfielJSON
### Key-6b receiver balance      | 5     		|Treceiver..xBalance" | 10000

### - c.Message transaction
### Key-3c. senderNounceJSON = {  t1234 _ 1000  898 log. 

opt_code, 8, 2;

nounce, 8;

version,8, "0x1" as default;

timestamp,4,tx expire in 12 hours;

stateNumber, 8;

txfee;

senderProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

thread = "other sender address + tx nounce"; if thread equal self sender nounce, it is a new thread.

msgJSON,1024;{}

AttachmentHash and Size,32; use HAMT store multimedia, the result blocks number

}


### Key-4c sender nounce, 8 	
### Key-5c sender balance, 5    
### Key-6c senderNounceAttachment(1 .. senderNounceAttachmentSize) = {cbor}
