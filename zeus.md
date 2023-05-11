---
marp: true
title: Linking
theme: default
# class: invert
paginate: true
author: Christian Bale
math: mathjax
style: |
  .columns {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 1rem;
  }
  .big-title {
    font-size: 2.63rem;
  }
  .lll-title {
    font-size: 1.5rem;
  }

# Created by Tommy on 15.01.2023

---

<h1 class="big-title">Zeus: Analyzing Safety of Smart Contracts</h1>

#### Sukrit Kalra, Seep Goel, Mohan Dhawan and Subodh Sharma
:zap: :zap: :zap:

Tommy and Idan

---

# Introduction

- Smart contracts are programs that run on the blockchain
- They are written in high-level languages such as Solidity
- Faithful execution of a smart contract is enforced by the blockchain’s consensus protocol
- Correctness and fairness of the smart contracts is not enforced by the blockchain, and should be verified by the developer

---

# Correctness and Fairness

- Correctness means the code is accurate and complete, producing intended results without errors
- Fairness means the code adheres to the agreed upon higher-level business logic for interaction.
The code shouldn't be biased towards any party, and shouldn't allow any party to cheat

---

# Correctness and Fairness - Example

```c
while (Balance > (depositors[index].Amount * 115/100) && index<Total_Investors) {
    if(depositors[index].Amount!=0)) {
        payment = depositors[index].Amount * 115/100;
        depositors[index].EtherAddress.send(payment);
        Balance -= payment;
        Total_Paid_Out += payment;
        depositors[index].Amount=0; // Remove investor
    } break;
}
```

The contract offers a 15% payout to any investor.
Sadly, the contract has both fairness and correctness issues.

---

# Correctness and Fairness - Example

```c
while (Balance > (depositors[index].Amount * 115/100) && index<Total_Investors) {
    if(depositors[index].Amount!=0)) {
        payment = depositors[index].Amount * 115/100;
        depositors[index].EtherAddress.send(payment);
        Balance -= payment;         // --------------------------
        Total_Paid_Out += payment;  // POTENTIAL OVERFLOW! 😱😱😱
        depositors[index].Amount=0; // --------------------------
    } break;
}
```

Correctness issue: The contract has a potential overflow in the `Total_Paid_Out` variable.

---

# Correctness and Fairness - Example

```c
while (Balance > (depositors[index].Amount * 115/100) && index<Total_Investors) {
    if(depositors[index].Amount!=0)) {
        payment = depositors[index].Amount * 115/100;
        depositors[index].EtherAddress.send(payment);
        Balance -= payment;
        Total_Paid_Out += payment;
        depositors[index].Amount=0;
    } break;
}
```

Fairness issue (1): `index` is never incremented within the loop, and so the payout is made to just one investor.

---

# Correctness and Fairness - Example

```c
while (Balance > (depositors[index].Amount * 115/100) && index<Total_Investors) {
    if(depositors[index].Amount!=0)) {
        payment = depositors[index].Amount * 115/100;
        depositors[index].EtherAddress.send(payment);
        Balance -= payment;
        Total_Paid_Out += payment;
        depositors[index].Amount=0;
    } break; // <------------------------------------
}
```

Fairness issue (2): The `break` statement is inside the `while` statement, and so the loop will always break after the first iteration.
Meaning, only the first investor will get paid. (Prob. the owner)

---

# Incorrect Contracts - Reentrancy

```js
contract Wallet {
    mapping(address => uint) private userBalances;
    function withdrawBalance() {
        uint amountToWithdraw = userBalances[msg.sender];
        if (amountToWithdraw > 0) {
            msg.sender.call(userBalances[msg.sender]);
            userBalances[msg.sender] = 0;
        }
    }
    // ...
}
```

```js
contract AttackerContract {
    function () {
        Wallet wallet;
        wallet.withdrawBalance();
    }
}
```

---

# Incorrect Contracts - Unchecked Send

- Solidity allows only $2300$ gas upon a send call
- Computation-heavy fallback functiono at the receiving contract will cause the invoking send to fail
- Contracts not handling failed send calls correctly may result in the loss of Ether

---

# Incorrect Contracts - Unchecked Send

```js
if (gameHasEnded && !prizePaidOut) {
    winner.send(1000); // Send a prize to the winner
    prizePaidOut = True;
}
```

The `send` call may fail, but `prizePaidOut` is set to `True` regardless.
Meaning the prize will never be paid out. :cry:

---

# Incorrect Contracts - Failed Send

- Best practices suggest executing a `throw` upon a failed `send`, in order to revert the transaction
- However, this may put contracts in risk

---

# Incorrect Contracts - Failed Send

```js
for (uint i=0; i < investors.length; i++) {
    if (investors[i].invested == min investment) {
        payout = investors[i].payout;
        if (!(investors[i].address.send(payout)))
            throw;
        investors[i] = newInvestor;
    }
}
```

- A DAO that pays dividends to its smallest investor when a new investor offers more money, and the smallest is replaced
- A wallet with a fallback function that takes more than $2300$ gas to run can invest enough to become the smallest investor
- No new investors will be able to join the DAO

---

# Incorrect Contracts - Overflow/underflow

```js
uint payout = balance/participants.length;
for (var i = 0; i < participants.length; i++)
    participants[i].send(payout);
```

- `i` is of type `uint8`, and so it will overflow after $255$ iterations
- Attacker can fill up the first $255$ slots in the array, and gain payouts at the expense of other investors

---

<h1 class="lll-title">Incorrect Contracts - Transaction State Dependence</h1>

- Contract writers can utilize transaction state variables, such as `tx.origin` and `tx.gasprice`, for managing control flow within a smart contract
- `tx.gasprice` is fixed and is published upfront - cannot be exploited :smile:
- `tx.origin` allows a contract to check the address that originally initiated the call chain

---

<h1 class="lll-title">Incorrect Contracts - Transaction State Dependence</h1>

```js
contract UserWallet {
    function transfer(address dest, uint amount) {
        if (tx.origin != owner)
            throw;
        dest.send(amount);
    }
}
```

```js
contract AttackWallet {
    function() {
        UserWallet w = UserWallet(userWalletAddr);
        w.transfer(thiefStorageAddr, msg.sender.balance);
    }
}
```

---

<h1 class="lll-title">Incorrect Contracts - Transaction State Dependence</h1>

```js
contract UserWallet {
    function transfer(address dest, uint amount) {
        if (msg.sender != owner) // FIXED!
            throw;
        dest.send(amount);
    }
}
```

```js
contract AttackWallet {
    function() {
        UserWallet w = UserWallet(userWalletAddr);
        w.transfer(thiefStorageAddr, msg.sender.balance);
    }
}
```

---

# Unfair Contracts - Absence of Logic

- Access to sensitive resources and APIs must be guarded, for instance:

- `selfdestruct`:
    - Kill a contract and send its balance to a given address
    - Should be preceded by a check that only the owner of the contract is allowed to kill it
    - Several contracts did not have this check

---

# Unfair Contracts - Incorrect Logic
```js
 while (balance > persons[payoutCursor_Id_].deposit / 100 * 115) {
    payout = persons[payoutCursor_Id_].deposit / 100 * 115;
    persons[payoutCursor_Id].EtherAddress.send(payout);
    balance -= payout;
    payoutCursor_Id_ ++;
}
```

  - Two similar variables, `payoutCursor_Id` and `payoutCursor_Id_`
  - The deposits of all investors go to the 0th participant, possibly the person who created the contract

---

# Unfair Contracts - Logically Correct but Unfair

##### Auction House Contract

<div class="columns">
<div>

```js
function placeBid(uint auctionId){
    Auction a = auctions[auctionId];
    if (a.currentBid >= msg.value)
        throw;
    uint bidIdx = a.bids.length++;
    Bid b = a.bids[bidIdx];
    b.bidder = msg.sender;
    b.amount = msg.value;
    // ...
    BidPlaced(auctionId, b.bidder, b.amount);
    return true;
}
```
</div>
<div>

- The contract does not disclose whether it is "with reserve" or not
- The seller can participate in the auction and artificially bid up the price
- The seller can withdraw the property from the auction before it is sold
</div>

---

# ZEUS
 - Takes as input a smart contract and a policy against which the smart contract must be verified
 - Performs static analysis atop the smart contract code 
 - Inserts the policy predicates as asserts
 - Converts the smart contract embedded with policy assertions to LLVM bitcode
 - Invokes its verifier to determine assertion violations

---
# Formalizing Solidity Semantics
- Abstract language that captures relevant constructs of Solidity programs
- A program consists of a sequence of contract declarations.
- Each contract is abstractly viewed as a sequence of one or more method
definitions

---

# An abstract language modeling Solidity

$$
\begin{array}{l}
P\ ::=\ C^{*}\\
C::=\mathbf{contract} \ @Id\{\ \mathbf{global} \ v\ :\ T;\ \mathbf{function} @Id( l\ :\ T) \ \{S\})^{*}\}\\
S::=( l\ :\ T@Id)^{*} \ \mid l:=e\mid S;S\\
\mid \mathbf{if} \ e\ \mathbf{then} \ S\ else\ S\\
\mid \mathbf{goto} \ l\\
\mid \mathbf{havoc} \ l\ :\ T\mid \mathbf{assert} \ e\mid \mathbf{assume} \ e\\
\mid x:=\mathbf{post} \ \mathbf{function} @\mathbf{Id} \ ( l\ :\ T)\\
\mid \mathbf{return} \ e\mid \mathbf{throw} \mid \mathbf{selfdestruct}
\end{array}
$$

---

# An abstract language modeling Solidity

$$
\begin{array}{l}
P\ ::=\ C^{*}\\
\textcolor[rgb]{0.81,0.81,0.81}{C::=\mathbf{contract} \ @Id\{\ \mathbf{global} \ v\ :\ T;\ \mathbf{function} @Id( l\ :\ T) \ \{S\})^{*}\}}\\
\textcolor[rgb]{0.81,0.81,0.81}{S::=( l\ :\ T@Id)^{*} \ \mid l:=e\mid S;S}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{if} \ e\ \mathbf{then} \ S\ else\ S}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{goto} \ l}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{havoc} \ l\ :\ T\mid \mathbf{assert} \ e\mid \mathbf{assume} \ e}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid x:=\mathbf{post} \ \mathbf{function} @\mathbf{Id} \ ( l\ :\ T)}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{return} \ e\mid \mathbf{throw} \mid \mathbf{selfdestruct}}
\end{array}
$$

 - A program consists of a sequence of contract declarations


---

# An abstract language modeling Solidity

$$
\begin{array}{l}
\textcolor[rgb]{0.81,0.81,0.81}{P\ ::=\ C^{*}}\\
C::=\mathbf{contract} \ @Id\{\ \mathbf{global} \ v\ :\ T;\ \mathbf{function} @Id( l\ :\ T) \ \{S\})^{*}\}\\
\textcolor[rgb]{0.81,0.81,0.81}{S::=( l\ :\ T@Id)^{*} \ \mid l:=e\mid S;S}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{if} \ e\ \mathbf{then} \ S\ else\ S}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{goto} \ l}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{havoc} \ l\ :\ T\mid \mathbf{assert} \ e\mid \mathbf{assume} \ e}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid x:=\mathbf{post} \ \mathbf{function} @\mathbf{Id} \ ( l\ :\ T)}\\
\textcolor[rgb]{0.81,0.81,0.81}{\mid \mathbf{return} \ e\mid \mathbf{throw} \mid \mathbf{selfdestruct}}
\end{array}
$$

 - Each contract is abstractly viewed as a sequence of one or more method definitions.
 - Persistent storage private to a contract, denoted by the keyword global

---

# An abstract language modeling Solidity

$$
\begin{array}{l}
\textcolor[rgb]{0.81,0.81,0.81}{P\ ::=\ C^{*}}\\
\textcolor[rgb]{0.81,0.81,0.81}{C::=\mathbf{contract} \ @Id\{\ \mathbf{global} \ v\ :\ T;\ \mathbf{function} @Id( l\ :\ T) \ \{S\})^{*}\}}\\
S::=( l\ :\ T@Id)^{*} \ \mid l:=e\mid S;S\\
\mid \mathbf{if} \ e\ \mathbf{then} \ S\ else\ S\\
\mid \mathbf{goto} \ l\\
\mid \mathbf{havoc} \ l\ :\ T\mid \mathbf{assert} \ e\mid \mathbf{assume} \ e\\
\mid x:=\mathbf{post} \ \mathbf{function} @\mathbf{Id} \ ( l\ :\ T)\\
\mid \mathbf{return} \ e\mid \mathbf{throw} \mid \mathbf{selfdestruct}
\end{array}
$$


