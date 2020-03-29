# TAU - blockchain cloud for unlimited data publishing.

## TAU is designed to enable mobile device. Three types of modes exists, which are statefull miner, stateless miner and normal user. 
### procedures for statefull miner, which requires wifi and power plugged to download full history, and mising wifi or plug will switch to normal user mode
0. open android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance and swarm connection history; note: combine relay and peer randomness to reduce connection jam;
2. request the miner peer for the n+1 state according to CBC (correct by construction); 
3. traverse 144 history states from the miner and keep accounting of the root array; 
mining based on curent root and build&validate (n+1) state JSON; when connection timeout, go to step(1)
4. go to step (1), until half of the know mining peers are traversed. 
5. calculate the CBC safety state k, if k it out of mutable range, that is n-144, then go to step (1). 
6. random walk until connect to a next relay; random walk until connect to a next miner
7. start mining by asking the peers longest chain, and verify k to n, build&validate (n+1) state JSON, when timeout, go to step (6)
8. as full nodes, it will verify state #1 to #n in the background. 
9. when new added mining nodes increase 33% or self-disconnected 12 hours, go to step (1).
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

### Procedures for stateless miner, which requires wifi and power plugged to download partial history, and missing wifi or plug will switch to normal user mode
0. open android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance and swarm connection history; note: combine relay and peer randomness to reduce connection jam;
2. request the miner peer for the (n+1) state according to CBC (correct by construction); 
3. traverse 144 history states from the miner and keep accounting of the root array; 
mining based on curent root and build&validate (n+1) state JSON; when connection timeout, go to step(1)
4. go to step (1), until half of the know mining peers are traversed. 
5. calculate the CBC safety state k, if k it out of mutable range, that is n-144, then go to step (1). 
6. random walk until connect to a next relay; random walk until connect to a next miner
7. start mining by asking the peers longest chain, and verify k to n, build&validate (n+1) state JSON, when timeout, go to step (6) 
8. when new added mining nodes increase 33% or self-disconnected 12 hours, go to step (1).
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

### steps for normal users on battery or 4G 
0. open android wake-lock and wifi-lock
1. random walk until connect to a next relay, and keep a list of know relays; random walk until connect to a next miner, and keep a list of know addresses with power and balance and swarm connection history; note: combine relay and peer randomness to reduce connection jam;
2. request the miner peer for the n+1 state according to CBC (correct by construction); 
3. traverse 144 history states from the miner and keep accounting of the root array; 
mining based on curent root k and build&validate (n+1) state JSON; when connection timeout, go to step(1)
4. update the CBC safety state k, then go to step (1). 
* miner always response to request of n+1 state, never initating push blocks to others. It is simple and staying in graphsync.

## State structure with entry point of "cid" + peersID, key-values are:

### 1. stateNumber, 8; stateNumber=1234567

### 2. statJSON1234567content={ 

version,8; 

timestamp, 4; 

state number, 8; 

base target, 8; 

cumulative difficulty,8 ; 

generation signature,32;

sender/miner TAU address, 20; 

sender/miner nounce, 8, mining is treated as a tx sending to self, nounce ++;

senderProfileJSON,1024,Ta..xProfile; {relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; }; 

txJSON; {original JSON from peers for wiring and message type 1&2}

previous hamt state root,

32; signature , 65:r: 32 bytes, s: 32 bytes, v: 1 byte, when at same difficulty, high signature number wins.

}

## exporting three type Transactions Nounce JSON and its vars, exported, three type of txs: 0-coinbase, 1-wiring, 2-message.
### -coinbase tx
### 3a. sender/minerNounceJSON ={

version,8, "0x1" as default;

opt_code, 8, 0x0 is coinbase;

nounce, 8;

timestamp,4,tx expire in 12 hours;

amount,5;

sender/minerProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

stateNumber, 8;

}

### 4a. sender/miner nounce	|8;
### 5a. sender/miner balance        | 5;

### -Coins Wiring
### 3b. senderNounceJSON = {

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


### 4b. sender nounce	|8 			|Ta..xNounce"	10;used as power
### 5b. sender balance        | 5       	|Tsender..xBalance" | 10000 ; through senderNounce to get TXJSON, ProfielJSON
### 6b receiver balance      | 5     		|Treceiver..xBalance" | 10000

### -Message transaction
### 3c. senderNounceJSON = {

opt_code, 8, 2;

nounce, 8;

version,8, "0x1" as default;

timestamp,4,tx expire in 12 hours;
stateNumber, 8;
txfee;

senderProfileJSON,1024,Ta..xProfile; {TAU: Ta..x; relay:relay multiaddress: {}; IPLD:Qm..x; telegram:/t/...; };

thread = "other sender address + tx nounce"; if thread equal self sender nounce, it is a new thread.
msgJSON,1024;{}
msgAttachmentSize,8;
msgAattachmentCid,32;
}


### 4c sender nounce	|8 	
### 5c sender balance        | 5     
