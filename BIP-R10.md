<pre>
  BIP: R10
  Title: Drivechain using OP_COUNT_ACKS
  Author: Sergio Demian Lerner <sergio.d.lerner@gmail.com>
  Status: Draft
  Type: Standards Track
  Created: 2016-05-06
</pre>

==Abstract==

This BIP describes the new opcode OP_COUNT_ACKS that adds Bitcoin drivechain capabilities. A drivechain is practical and low-complexity extension method to add user-defined functionality executed in a secondary blockchain to Bitcoin, by means of a 2-way-peg. 

==Motivation==

Bitcoin aims to be both a decentralized settlement/payment system (the network) and a store of value (the currency). The Bitcoin network works 24/7 to keep safe billion of dollars. Therefore, there is little room for experiment and ground-breaking changes in the protocol: improving Bitcoin (the network) has been compared to repairing a plane during flight. 
This has not prevented innovation within the Bitcoin community. Nevertheless, most of the new ideas must be held forever or end up being captured by competing alt-coins. 
The Sidechain method was proposed to improve Bitcoin with low impact in its security, allowing adding functionality to Bitcoin (the currency) but isolating sidechain's potential vulnerabilities from the Bitcoin network. However, adding generic  sidechain integration to Bitcoin is complex and no generic proposal has been published. In the particular case that the sidechain is merge-mined, and there is high engagement in merge-mining, sidechains do not provide more security for the "algorithmic custody" of transferred bitcoins than basing the algorithmic-custody on a counting acknowledge tags appended by the Bitcoin miners to their blocks. This concept has been defined as a Drivechain.

We propose a soft-fork that defines a new opcode (redefining a NOP opcode) as the OP_COUNT_ACKS using the segwit script versioning system. Several opcode arguments allow the drivechain creator to specify the ack weights.
 
==Specification==

At any time there can be zero or more active ack-polls. Each ack-poll is associated with a transaction candidate. Miners that want to ack for one or more candidates embed the message tag “ACK:” followed by a serialized FULL_ACK_LIST. The grammar of FULL_ACK_LIST is the following:

<pre>
FULL_ACK_LIST: { CHAIN_ACK_LIST... }
CHAIN_ACK_LIST: { secondary_chain_id ACK_LIST }
ACK_LIST: { ACK... }
ACK: tx_hash [ tx_hash_pre ]
</pre>

In this grammar:
- { x... } is interpreted as a list of items of type x. Every item in a list has the same format, and only one format is expected for each item. Therefore, the parser can distinguish between individual items and lists of items.
- [ x ] is interpreted as an optional argument x. Optional arguments can only appear at the tail of a list.


The data is serialized in the following format:
 
- A serialized list begins with a compact Uint specifying the list length in bytes (including all list payload), followed by the serialized items. A length of zero means no items in the list. 
- An serialized item begins with a compact Uint specifying the item length in bytes, followed by the payload.

The tag and payload are stored in the coinbase field of their coinbase transactions. Therefore the first field after the "ACK:" string is a compactSize uint specifying the tag payload. To find the ACK tag, the coinbase field is scanned from start to end. Only the first occurrence of the tag "ACK:" is analysed for correctness. If the script that follows is invalid (any compact uint size greater than available space) then the whole tag is considered invalid and no data following the tag is processed. The maximum size of the payload of a ACK tag is 100 bytes (this is enforced because the maximum size of the coinbase field is 100 bytes). The secondary_chain_id cannot be repeated in a tag. If repeated, it is ignored. The same proposal cannot be ack-ed twice (identified by the chain id and the hash prefix) even if the prefixes are different but refer to the same candidate. If the same proposal is ack-ed more than once, then the additional acks are ignored. 

* secondary_chain_id is blob that identifies the secondary chain. 
* tx_hash_prefix is the transaction ID (double SHA256 hash) or a prefix of the transaction ID for the transaction that is proposed (in the secondary chain) to spend the locked bitcoins.
* tx_hash_pre is the single SHA256 pre_image of the tx_hash_prefix. If the pre_image has already been shown in a previous ack (when the candidate is created), this field should be left out. This is for preventing miners easily creating proposals that match another proposal prefix.

Note that the miner that acks a transaction have "pays" an amount in fees because of the block space consumed by the proposal (e.g. approximately 70 additional bytes for a new proposal, but can be as low as a 10 bytes). However, this is low and can be ignored.
By not including an ack for a proposal for a secondary_chain_id means that proposal gets a negative ack.

Example 1 
---------

The following coinbase tags in successive blocks create 5 different proposals:
<pre>
1) ACK: {{ DRVCOIN {{0xbaa501b37267c06d8d20f316622f90a3e343e9e730771f2ce2e314b794e31853 0x101010....10}}}} 
2) ACK: {{ DRVCOIN {{0x85e7eac2862f1cbd85bc18769c75172c3fdcd899ab468b9e973d59ec620d9991 0x202020....20}}}}
3) ACK: {{ DRVCOIN {{0x84e0c0eafaa95a34c293f278ac52e45ce537bab5e752a00e6959a13ae103b65a 0x303030....30}}}}
4) ACK: {{ YCOIN   {{0x92b7eb5290d8d6e3ac79215cb4bdb07fe89629ee720be4332b3daa842b7ec80a 0x404040....40}}}}
5) ACK: {{ XCOIN   {{0xb37361a0be8af8905de1f4384e701365ece4313dd5e064d375eb2851c043ba68 0x505050....50}}}}
</pre>
(First element is chain_id, second element is transaction hash proposal and third element is transaction hash pre-image)

The following block contains the tag: 
<pre>
6) ACK: { { DRVCOIN {{0xba} {0x84e0}} } { XCOIN {{0xff}}} }
</pre>

This last tag acks for the proposal 1, for proposal 3, against the proposal 2, against the proposal 5 (because the ack does not have a candidate) and ignores the proposal 4 (for YCOIN). The ack for proposal 3 is not using the minimum possible prefix (0x84), but it is still a valid ack.

Using transaction prefixes reduces coinbase space consumption. If a miner is performing "SPV mining" and does not know the contents of the previous block coinbase field, and wants to ack for a proposal that has not been published before the last block, it should include the full ack transaction hash (32 bytes) plus the pre-image (32 bytes). However miners must not include the pre-image if they are ack-ing for a pre-existent proposal, because by doing so they are flagging that a new ack-poll may start at that point due to the sliding-window nature of the ack-ing process. Miners SHOULD also include an ack tag for a sidechain_id with an empty list of acks to announce they are ready to ack for that secondary chain, but there is no active ack-poll (e.g. ACK: {{DRVCOIN {}}}).  

There can be many active proposals for the same drivechain, and a miner can ack for many of them simultaneously adding more items to ACK_LIST. 

The opcode NOP? is redefined as OP_COUNT_ACKS. This opcode scans a number of past coinbase fields and counts acks for a certain spending transaction proposal.

The opcode has the following arguments:


* secondary_chain_id 
* ack_period (in blocks)
* liveness_period (in blocks)

Description of arguments:

* secondary_chain_id is a blob that identifies the drivechain (e.g "PrivateBitcoin" for a drivechain that implements the zCash protocol).
* ack_period represents the number of blocks after the proposal is published where acks are counted. After this number of blocks, any ack for the proposal is discarded. ack_period valid range is [1..144].
* liveness_period represents the number of blocks after the ack_period ends where the transaction specified by the proposal is valid. Liveness_period range is [100..144]. Therefore the spending transaction can be included not before than 100 blocks after the ack-ing period ends. The 100 block forced confirmation gap is required so the maturity of a transaction consuming an output using a script that contains the COUNT_ACKS opcode is equal or higher than the maturity of coinbase outputs. 

OP_COUNT_ACKS performs two stages of computation: 

1. Opcode arguments validation. Arguments are validated against pre-established bounds. if not in range, the script is aborted.
2. Miner's ack counting. 

If the verification is not aborted, the input arguments are popped from the stack and following values are pushed in the stack:

1. positive_acks
2. negative_acks


Opcode arguments validation Stage
---------------------------------

All argument conditions must be met for the execution to be able to continue. If not, then the script execution is aborted (and the script does not verify):

* secondary_chain_id blob with length in [1..20]
* ack_period (in blocks) integer in [1..max_ack_period]. max_ack_period is 144.
* liveness_period (in blocks) integer in [1..max_liveness_period]. max_liveness_period is 144.
* n-liveness_period-ack_period>=0 (there must be enough blocks for the poll) 

Miner's Ack Counting Stage
---------------------------

Let secondary_chain_id be the chain id that is being evaluated. Let A be a vector containing (key,pos_acks,neg_acks) triple, where key is a blob specifying the transaction hash and ack values are uint32. 
Let spend_tx_hash be the hash of the transaction that is used to spend the output that contains the OP_COUNT_ACKS being executed.

The following pseudo-code specifies ack-counting algorithm and returns (miner_positive_acks,miner_negative_acks):

```
1. Let poll_start :=n-liveness_period-ack_period
2. For i :=poll_start to poll_start+ack_period-1 do
2.1. C := GetCoinBaseFieldOfBlock(i)
2.2. Find tag "ACK:" in C.
2.3. if not (C contains valid "ACK:" tag and payload) then continue
2.4. Extract the FULL_ACK_LIST into L.
2.5. For j :=0 to Length(L)-1 do
2.5.1. if (L[j][0]=secondary_chain_id ) then
2.5.1.1. ack_list :=L[j][1]
2.5.1.2. new_acks :=[] // set of integers corresponding to acks indexes
2.5.1.3. For k :=0 to Length(ack_list)-1 do
2.5.1.3.1. ack :=ack_list[k]
2.5.1.3.2. tx_hash :=ack[0]
2.5.1.3.3. tx_hash_pre :=ack[1]
2.5.1.3.4. valid :=((Length(tx_hash)<=32) AND (length(tx_hash_pre)=0)) OR ((Length(tx_hash)=32) AND (length(tx_hash_pre)=32) AND (SHA256(tx_hash_pre)=tx_hash)) 
2.5.1.3.5. if not valid then
2.5.1.3.5.1. continue 
2.5.1.3.6. if not (tx_hash is a prefix of any key in A) then
2.5.1.3.6.1. if length(tx_hash_pre)=32 then
2.5.1.3.6.1.1. A.Append(tx_hash,0) // tx_hash is key, 0 is initial ack count (decremented later)
2.5.1.3.6.1.2. new_acks.add(Length(A)-1)
2.5.1.3.6.2. else
2.5.1.3.6.2.1. if not (tx_hash is a prefix of spend_tx_hash) then // an optimization to reduce new_ack size
2.5.1.3.6.2.1.1. continue
2.5.1.3.6.2.2. Let t be the first index in A such as tx_hash is a prefix of A[t].key
2.5.1.3.6.2.3. new_acks.Add(t)
2.5.1.4. For k :=0 to Length(A)-1 do
2.5.1.4.1. if (k in new_acks) then
2.5.1.4.1.1. A[k].pos_acks :=A[k].pos_acks+1
2.5.1.4.2. else
2.5.1.4.2.1. if (allow_negative_acks) then
2.5.1.4.2.1.1. A[k].neg_acks :=A[k].neg_acks+1
2.5.1.5. break
3. If there is no index t so that A[t].key = spend_tx_hash then return 0, and exit
4. Let t be the first index where A[t].key = spend_tx_hash.
5. Return (A[t].pos_acks, A[t].neg_acks)
```

The sliding-window nature of this ack-ing system means that miners must not stop ack-ing positively or negatively when a negative or positive threshold is reached, if new proposals are included (new proposals are identified by a non-empty tx_hash_pre). 
Step 2.5.1.3.6.2.1 is an optimization optimization step. If included, then acks for spend_tx_hash will be counted correctly, but acks for other candidates will not.

 
Ack de-serialization pseudo-code
---------------------------------

```
ReadElement(string source) -> (string result,string tail)

1. (len,rest) :=ReadCompactUint(source); // if invalid CompactUInt, raise exception
2. if (len>length(rest)) then raise exception
3. result :=copy(rest,0,len) // from index 0, len bytes
4. tail :=copy(rest,len,Length(result)-len)


ReadAcks(string s) -> (ACK_LIST)
 
1. Let s be the coinbase field from the end of the "ACK:" string to the end of the field
2. (r,t) :=ReadElement(s); 
3. i :=0
4. while(Length(s)>0) do
5.1. (Li,r) :=ReadElement(r); // Li = { chainID { ... } }
5.2. (chainId,t) :=ReadElement(Li) // t = { {v1 p1} {v2 p2}... }
5.3. CHAIN_ACK_LIST :=new List()
5.4. L.Append(CHAIN_ACK_LIST)
5.5. CHAIN_ACK_LIST.Append(chainId)
5.6. ACK_LIST  :=new List()
5.7. CHAIN_ACK_LIST.Append(ACK_LIST)
5.8. (x,y) :=ReadElement(t) // x = {v1 p2} , {v2 p2} ...
5.9. If (Length(y)>0) then raise exception
5.3. j :=0
5.4. While (Length(y)>0)
5.4.1. (a,y) :=ReadElement(y) // v = {v1 p2} , y = {v2 p2} ...
5.4.2. ACK :=new List()
5.4.3. ACK_LIST.Append(ACK)
5.4.4. (a0,a) :=ReadElement(a)
5.4.5. ACK.Append(a0)
5.4.6. if (Length(a)>0) then
5.4.6.1. (a1,a) :=ReadElement(a)
5.4.6.2. ACK.Append(a1)
5.4.6.3. if (length(a)>0) then raise exception
5.4.7. inc(j)
5.5. inc(i)
```

Examples
--------

<pre>
Let the blockchain have only the following tags (line format: block-number,  tag):
101, ACK: {{ XCOIN {{0xbaa501b37267c06d8d20f316622f90a3e343e9e730771f2ce2e314b794e31853 0x101010....10}}}} 
102, ACK: {{ XCOIN {{0xba}} }} // 2nd positive vote
... block 103..200 having the same tag as block 102..
201, ACK: {{ XCOIN {{ }} }}  // negative vote
... block 201..225 having the same tag as block 201..
</pre>

Let the script be executed on a transaction included in block 370. Then the poll starts at block 82 and ends at block 225 (included).  

``` 
scriptSig: 
	58434f494e	("XCOIN")
	144
	144
	OP_COUNT_ACKS    // Results in a stack that contains: 100 25
```

```
scriptSig:
	OP_2DUP	   // duplicate ack counts
	OP_GREATERTHAN   // more positive than negative acks ?
	OP_VERIFY	   // abort if not
	OP_SUB		   // compute positive minus negative, push result into stack
	72		   // number of diff acks required	
	OP_GREATERTHAN   // More than 50% positive difference, put 1 on stack, else put 0	
```	


==Rationale==

The benefit of COUNT_ACKS is that it allows to bootstrap a merged-mining cryptocurrency from having no merge-mining engagement to a high merge-mining engagement filling the missing security with notary acks.  When a high merge-mining engagement is reched, the notaries can cese to provide acks or they can continue to contribute to the security of the drivechain, depending on the design choices of the secondary chain creators. Initially, if there is no merge-mining engagement, the notaries provide acks to release the locked funds, but the scriptPub can be parametrized such that a threshold of signatures is required to release them, so that still no individual notary has control. There is no point in allowing the notaries to ack against (nack) a proposal. This is because the 51% of the miners can censor negative acks, so security is not increased. 

The opcode and miner's ack-ing algorithm was designed such that acks in the coinbase field can never invalidate a block. This prevents attacks against pools and also reduces the risk for miners not implementing the soft-fork. 
Also there is no ack counting happening when a new block is added to the blockchain. This minimizes any overhead in best chain management and allows testing the new opcode much more easily by building a fake blockchain and then executing different scripts with different opcode arguments. 
The liveness period and ack period limits reduces the depth to which the COUNT_ACKS opcode retrieves coinbase fields. This has two benefits: first sets a bound to the running time of the opcode and second it allows blocks older than the compound limit to be forget. 

==Backwards Compatibility==

Transactions containing the COUNT_ACKS opcode are non-standard to old implementations, which will (typically) not relay them nor include them in blocks.

Old implementations will validate that COUNT_ACKS as OP_NOP. This implies this BIP represents a soft-fork.
Miners that do not soft-fork and do not include non-standard transactions are not affected, since no consensus rule is added to the block header or coinbase validation.

This BIP will be deployed using a standard majority threshold (as used in previous soft-forks) and will use the version bits to mark miners acks (as defined in BIPn).

If a majority of hashing power does not support the new validation rules, then roll-out will be postponed (or rejected if it becomes clear that a majority will never be achieved).

==Broadcasting a transaction containing OP_ACK_COUNT in scriptSigs==

Transactions containing the OP_ACK_COUNT in scriptSigs are non-standard and should not be broadcast. Only miners should be able to include these transactions in blocks. 

==Security ==

The security parameters of a specific secondary chain are defined by the secondary chain designers. Any user wishing to lock bitcoins to move them to the secondary chain should use the parameter set specified by the secondary chain designers. 
If bitcoins are locked for a secondary chain using a different set of parameters, there is no guarantee the secondary chain will accept the transfer and use the locked output in the future to unlock bitcoins. Those bitcoins therefore can be locked forever. Therefore transactions for transfers to secondary chains should be automated by applications and not crafted by hand. 

There COUNT_ACKS opcode can not be used as a vector to perform a denial-of-service by exhausting CPU nor exhausting memory since:

- The maximum number of blocks processed is 144 (max_ack_period)
- The maximum number of different candidates that can be created per block is 1 (100 bytes/66 bytes).
- The maximum number of acks that can be included per block is 50 (100 bytes/2 bytes).
- The maximum number of different acks that can be created in total is 144.
- The size consumed in memory by each stored candidate is approximately 50 bytes.
- The maximum memory consumed by the ack counting algorithm is 7.2K bytes.
- The maximum number of loop iterations (hash prefix comparisons) while counting acks is 144.
 
In comparison, a script can use up to 400*10K bytes of stack, totalling 4M bytes.

==Computational Cost==

The cost of the COUNT_ACKS opcode in terms of sigops is set to be 1. The rationale of this selection is the following: Performing the ack counting requires fetching at most 144 recent generation transactions. The maximum amount of information that has to be fetched from memory is 7.2K bytes. Assuming the most recent 288 coinbase fields can be kept cached in-memory (either by the OS or by the application), then coinbase fetching CPU cost is comparable to the cost of a single signature check. Assuming no cache, and a 10 msec per coinbase fetching time, the fetching cost becomes prohibitive, unless transaction verification is amortized by parallelization. Therefore, the Bitcoin implementation should cache the coinbase fields of generation transactions. 

==Forwards Compatibility ==

==Reference Implementation==

<in progress> 

==See Also==


* M-of-N Multisignature Transactions [[bip-0011.mediawiki|BIP 11]]