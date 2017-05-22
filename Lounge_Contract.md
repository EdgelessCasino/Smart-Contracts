/**
 * Profit sharing contract for the Edgeless Casino. 
 * author: Julia Altenried
 **/

pragma solidity ^0.4.11;

contract token {
  function transferFrom(address sender, address receiver, uint amount) returns(bool success){}
  function transfer(address receiver, uint amount) returns(bool success){}
  function balanceOf(address holder) returns(uint){}
}

contract owned {
  address public owner;
  modifier onlyOwner {
    if (msg.sender != owner)
      throw;
    _;
  }

  function owned() {
    owner = msg.sender;
  }

  function changeOwner(address newOwner) onlyOwner {
    owner = newOwner;
  }
}

contract SafeMath {
  //internals

  function safeMul(uint a, uint b) internal returns (uint) {
    uint c = a * b;
    assert(a == 0 || c / a == b);
    return c;
  }

  function safeSub(uint a, uint b) internal returns (uint) {
    assert(b <= a);
    return a - b;
  }

  function safeAdd(uint a, uint b) internal returns (uint) {
    uint c = a + b;
    assert(c>=a && c>=b);
    return c;
  }

  function assert(bool assertion) internal {
    if (!assertion) throw;
  }
}

contract profitSharing is owned, SafeMath {
  /** the balance of EDG tokens per address held by this contract **/
  mapping(address => uint) public balances;
  /** timestamp of the last payout **/
  uint public payoutTime;
  /** the total balance to be shared before the next payout **/
  uint public payoutBalance;
  /** the token balance at the last payout time **/
  uint public tokenBalance;
  /** the current token balance **/
  uint public currentTokenBalance;
  /** the timestamp of the last holder action (sending of tokens or claiming of share) **/
  mapping(address => uint) public lastAction;
  /** tells if a address is authorized to call the payout method **/
  mapping(address => bool) public authorized;
  /** the edgeless token contract **/
  token edg;
  
  /** notifies listeners that EDG tokens have been sent to the contract**/
  event storedEDG(address indexed holder, uint amount);
  /** notifies listeners that EDG tokens have been retrieved from the contract**/
  event retrievedEDG(address indexed holder, uint amount);
  /** notifies listeners that a Edgeless Casino contract payed out the profit **/
  event sharedProfit(address indexed sharer, uint amount);
  /** notifies listeners that a token holder retrieved his share **/
  event profitClaimed(address indexed holder, uint amount);
  
  
  /**
  * constructor. sets the token contract.
  **/
  function profitSharing(address tokenAddress){
    edg = token(tokenAddress);
  }
  
  /**
   * transfers the specified amount of tokens from the sender to the contract.
   * the sender needs to approve this transaction in advance.
   * @param amount the number of tokens to store in the contract
   **/
  function storeEDG(uint amount) {
    if (edg.transferFrom(msg.sender, address(this), amount)) {
      balances[msg.sender] = amount;
      lastAction[msg.sender] = now;
      currentTokenBalance=safeAdd(currentTokenBalance,amount);
      storedEDG(msg.sender, amount);
    }
  }

  /**
   * retrieves the sender's share of the profit 
   * if the sender previously transfered the tokens to the contract and did not collect his share yet.
   **/
  function claimShare() {
    if (balances[msg.sender] > 0 && payoutBalance > 0 && tokenBalance > 0) {
      if(lastAction[msg.sender] < payoutTime){
        lastAction[msg.sender] = now;
        uint share = safeMul(payoutBalance, balances[msg.sender])/tokenBalance;
        if(share > 0 && !msg.sender.send(share)) 
           lastAction[msg.sender] = 0;//only important that it's lesser than payoutTime
        else
           profitClaimed(msg.sender, share);
      }
    }
  }

  /**
   * retieves the tokens stored in the contract by the user 
   **/
  function retrieveEDG() {
    if (balances[msg.sender] > 0) {
      uint b = balances[msg.sender];
      balances[msg.sender] = 0;
      if (!edg.transfer(msg.sender, b))
        balances[msg.sender] = b; //restore balance in case of failure
      else{
        currentTokenBalance = safeSub(currentTokenBalance,b);
        retrievedEDG(msg.sender, b); //notify listeners
      }
    }
  }

  /**
   * transfers the profit to be shared between the token holders which sent their tokens to the contract earlier.
   * sets the total payout balance.
   * callable only by an authorized address 
   * (so people cannot increase the payout timestamp theirselves and collect more than they're allowed to)
   **/
  function payout() payable returns (bool success){
    if (!authorized[msg.sender]) throw;
    //if the payout phase was not previously initiated (e.g. by another sender)
    payoutTime = now;
    payoutBalance = address(this).balance;
    tokenBalance = currentTokenBalance;//ideally == edg.balanceOf(address(this)), but can be lower if tokens have been sent directly to the contract
    sharedProfit(msg.sender, msg.value);
    return true;
  }

  /**
   * authorizes an address to call the payout method
   * @param addr the address to be approved
   **/
  function approveAddress(address addr) onlyOwner {
    authorized[addr] = true;
  }

  /**
   * removes an address from the list of authorized profit sharers
   * @param addr the address to be disapproved
   **/
  function disapproveAddress(address addr) onlyOwner {
    authorized[addr] = false;
  }
  
  /**
  * exception handling in case somebody sends tokens directly to the contract and they get stuck.
  * checks if the contract holds more tokens than it should, before assigning the extra tokens to an address.
  * this way the owner can return the tokens to the user.
  * @param amount the number of tokens
  *        holder the address to assign the tokens to
  **/
  function assignTokens(uint amount, address holder) onlyOwner{
    uint sum = safeAdd(currentTokenBalance,amount);
    if(sum <= edg.balanceOf(address(this))){//there are at least amount many tokens more on the contract than there should be
      currentTokenBalance = sum;
      balances[holder] = safeAdd(balances[holder],amount);
      lastAction[holder] = now;
      storedEDG(holder, amount);
    }
  }

}
