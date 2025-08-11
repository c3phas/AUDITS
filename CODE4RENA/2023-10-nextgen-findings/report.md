<!-- vscode-markdown-toc -->

- 1. [Findings](#Findings)
- 2. [HIGH: Possible to cancel bids after the auction is over which allows a malicious user to win the Auction using the lowest bid](#HIGH:PossibletocancelbidsaftertheauctionisoverwhichallowsamalicioususertowintheAuctionusingthelowestbid)
- 3. [Vulnerability details](#Vulnerabilitydetails)
  - 3.1. [Recommendation](#Recommendation)
- 4. [MEDIUM: Funds can be locked in the contract #1562](#MEDIUM:Fundscanbelockedinthecontract1562)
  - 4.1. [Vulnerability details](#Vulnerabilitydetails-1)
  - 4.2. [Recommendation](#Recommendation-1)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## 1. <a name='Findings'></a>Findings

## 2. <a name='HIGH:PossibletocancelbidsaftertheauctionisoverwhichallowsamalicioususertowintheAuctionusingthelowestbid'></a>HIGH: Possible to cancel bids after the auction is over which allows a malicious user to win the Auction using the lowest bid

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L124-L130

## 3. <a name='Vulnerabilitydetails'></a>Vulnerability details

Possible to cancel bids after the auction is over which allows a malicious user to win the Auction using the lowest bid

To participate in the auction we call https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L57-L61

```solidity
    function participateToAuction(uint256 _tokenid) public payable {
        require(msg.value > returnHighestBid(_tokenid) && block.timestamp <= minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true);
        auctionInfoStru memory newBid = auctionInfoStru(msg.sender, msg.value, true);
        auctionInfoData[_tokenid].push(newBid);
    }
```

Say we have a user Alice ,Alice makes a bid quoting a very high amount,this would prevent majority from participating in the bid since Alice set the bid amount too high.

Due to how the require statement is used `block.timestamp <= minter.getAuctionEndTime(_tokenid)` it's possible to participate in the Auction when `blocktimestanp is equal the endtime`

The main issue arises in that, it's possible for a user to cancel their bid when the Auction ends
The `CancelBid()` function is implemented as follows https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L124-L130

```solidity
    function cancelBid(uint256 _tokenid, uint256 index) public {
        require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended");
        require(auctionInfoData[_tokenid][index].bidder == msg.sender && auctionInfoData[_tokenid][index].status == true);
        auctionInfoData[_tokenid][index].status = false;
        (bool success, ) = payable(auctionInfoData[_tokenid][index].bidder).call{value: auctionInfoData[_tokenid][index].bid}("");
        emit CancelBid(msg.sender, _tokenid, index, success, auctionInfoData[_tokenid][index].bid);
    }

```

So Alice, notices that we can cancel a bid when `block.timestamp <= minter.getAuctionEndTime(_tokenid) `which means , it's possible to cancel when `block.timestamp` is equal to the end time.

So Alice waits until the end and immediately calls the `cancelBid()` function while calling the `participateToAuction()` function also but this time with a lower bid.

Another scenario would be if we have two malicious accounts ,the first places a lower bid say 1 wei which goes through since its the first bid,the other malicious account places a very big bid preventing other bidders from bidding, When the Auction is over ,the one with the higher bid cancels their bid leaving the first one who placed the smallest bid as the winner

### 3.1. <a name='Recommendation'></a>Recommendation

We can solve it in two ways, either do not allow bids to be cancelled unless `block.timestamp > minter.getAuctionEndTime(_tokenid)` or limit participation to only allow bids when `block.timestamp <= minter.getAuctionEndTime(_tokenid)`

## 4. <a name='MEDIUM:Fundscanbelockedinthecontract1562'></a>MEDIUM: Funds can be locked in the contract #1562

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L57-L61
https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L104-L120

### 4.1. <a name='Vulnerabilitydetails-1'></a>Vulnerability details

To claim the win we call the function `claimAuction()` which has a few checks
Of main interest is the first require statement `require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);` especially the first part `block.timestamp >= minter.getAuctionEndTime(_tokenid)`
We check that the timestamp is greater or equal to when the auction Ends as claiming should only happen after the auction ends.

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L104-L120

```solidity
    function claimAuction(uint256 _tokenid) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
        require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);
        auctionClaim[_tokenid] = true;
        uint256 highestBid = returnHighestBid(_tokenid);
        address ownerOfToken = IERC721(gencore).ownerOf(_tokenid);
        address highestBidder = returnHighestBidder(_tokenid);
        for (uint256 i=0; i< auctionInfoData[_tokenid].length; i ++) {
            if (auctionInfoData[_tokenid][i].bidder == highestBidder && auctionInfoData[_tokenid][i].bid == highestBid && auctionInfoData[_tokenid][i].status == true) {
                IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);
                (bool success, ) = payable(owner()).call{value: highestBid}("");
                emit ClaimAuction(owner(), _tokenid, success, highestBid);
            } else if (auctionInfoData[_tokenid][i].status == true) {
                (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
                emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
            } else {}
        }
    }
```

However the equality operator introduces another side to this claiming process, When the timestamp is equal to the auction end time, one is allowed to claim. This would be ok if no one can place a bid when `timestamp == auction end time`. Let's look at the function used to place bids

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L57-L61

```solidity
    function participateToAuction(uint256 _tokenid) public payable {
        require(msg.value > returnHighestBid(_tokenid) && block.timestamp <= minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true);
        auctionInfoStru memory newBid = auctionInfoStru(msg.sender, msg.value, true);
        auctionInfoData[_tokenid].push(newBid);
    }
```

The require statement also has a time check,`block.timestamp <= minter.getAuctionEndTime(_tokenid)`. This means if we place a bid when `block.timestamp is equal to auction end time` it would still go through.

The problem however is, since we called `claimAuction()` the variable `auctionClaim[_tokenid]` is updated to true, which means no one else can claim. The amount sent by the bidder would then be locked

### 4.2. <a name='Recommendation-1'></a>Recommendation

We should ensure that Claiming is only done when `block.timestamp > minter.getAuctionEndTime(_tokenid)` or enforce this on the bidding process where we can ensure that one would only bid if `block.timestamp < minter.getAuctionEndTime(_tokenid)`
