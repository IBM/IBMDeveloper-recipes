# Working with web3js API and JSON to Build Ethereum Blockchain Applications

## Learn how to build blockchain and decentralized applications in Ethereum with Solidity programming

[mattumd](https://developer.ibm.com/recipes/author/mahdiumd/)

Tags: Cloud computing, Collaboration solutions, Continuous engineering, Web development

Published on July 4, 2019 / Updated on May 17, 2021

![](images/ethereum.jpg)

### Overview

Skill Level: Beginner

This recipe shows you how to setup Web3.JS API for Ethereum blockchain applications step-by-step by building a peer-to-peer auction app. Along the way, you learn how to use JSON, Truffle and Golang for deploying and managing Ethereum smart contracts.

### Ingredients

To follow and complete this recipe, you need to have good knowledge of blockchain, Ethereum transactions, Solidity as well as basic knowledge of HTML, CSS, React.JS, Golang and Linux.

### Step-by-step

#### 1. Overview of Ethereum Blockchain Development

Ethereum is a general-purpose blockchain that is more suited to describing business logic, through advanced scripts, also known as smart contracts. Ethereum was designed with a broader vision, as a decentralized or world computer that attempts to marry the power of the blockchain, as a trust machine, with a Turing-complete contract engine. Although Ethereum borrows many ideas that were initially introduced by bitcoin, there are many divergences between the two.  

The Ethereum virtual machine and smart contracts are key elements of Ethereum, and constitute its main attraction. In Ethereum, smart contracts represent a piece of code written in a high-level language (Solidity, LLL, Viper) and stored as bytecode in the blockchain, in order to run reliably in a stack-based virtual machine (Ethereum Virtual Machine), in each node, once invoked. The interactions with smart contract functions happen through transactions on the blockchain network, with their payloads being executed in the Ethereum virtual machine, and the shared blockchain state being updated accordingly.

For those who are not familiar with blockchain technology reading [History and Evolution of Blockchain Technology from Bitcoin](https://myhsts.org/tutorial-history-and-evolution-of-blockchain-technology-from-bitcoin.php) article is strongly recommended. Also, if you wish to learn and practice Hyperledger blockchain development, visit [Comprehensive Hyperledger Training Tutorials](https://myhsts.org/tutorial-comprehensive-blockchain-hyperledger-developer-guide-for-all-professional-programmers.php) page to get the outline of our Hyperledger tutorial articles.

#### 2. Recipe Outline

We have written two sets of tutorials to explore Ethereum and Solidity programming in depth. First set covers the following nine recipes:

*   [Introduction to Ethereum Blockchain Development with DApps and Ethereum VM](https://myhsts.org/tutorial-learn-about-ethereum-blockchain-development-with-dapps-and-ethereum-vm.php)
*   [Building Auction DApp With Ethereum and Solidity Programming Language](https://myhsts.org/tutorial-learn-how-to-build-auction-dapp-with-ethereum-and-solidity-programming-language.php)
*   [Working with Ethereum Blockchain Applications through Remix IDE](https://myhsts.org/tutorial-learn-how-to-work-with-ethereum-blockchain-applications-through-remix-ide.php)
*   [Building Bidding Form in Web3js for Ethereum Auction DApp](https://myhsts.org/tutorial-learn-how-to-build-bidding-form-in-web3js-for-ethereum-auction-dapp.php)
*   Working with web3js API and JSON to Build Ethereum Blockchain Applications
*   [Deployment Environments for Managing Ethereum Smart Contracts](https://dev.to/weg2g/deployment-environments-for-managing-ethereum-smart-contracts-16ha)
*   [Work with Ethereum Private Network with Golang with Geth](https://myhsts.org/tutorial-learn-how-to-work-with-ethereum-private-network-with-golang-with-geth.php)
*   [Compiling and Deploying Ethereum Contracts Using Solidity Compiler](https://myhsts.org/tutorial-learn-how-to-compile-and-deploy-ethereum-contracts-using-solidity-compiler.php)
*   [Running Ethereum Auction DApp and Solidity Tips](https://myhsts.org/tutorial-learn-how-to-run-ethereum-auction-dapp-with-some-solidity-tips.php)

In short, you learn about how to set up and configure Ethereum and develop blockchain applications using Solidity programming language. We explore its key components, including smart contracts and Web3.JS API via an Auction Decentralized Application (DApp) step-by-step.

In second set, we will discuss more advance topics in Ethereum blockchain development and solidity while building a Tontine DApp game step-by-step. Specifically, we cover Truffle and Drizzle. For instance, we show you how a tool such as Truffle can be an assistant in building, testing, debugging, and deploying DApps. In summary, we are going to cover four main topics:

*   Exploring the Truffle suite
*   Learning Solidity's advanced features
*   Contract testing and debugging
*   Building a user interface using Drizzle

The 2nd set consists of 8 recipes as follows:

*   [Install Truffle and Setup Ganache for Compiling Ethereum Smart Contracts for Tontine DApp Game](https://myhsts.org/tutorial-learn-how-to-install-truffle-and-setup-ganache-for-compiling-ethereum-smart-contracts-for-tontine-dapp-game.php)
*   [Run Tontine Ethereum DApp Game Contract](https://myhsts.org/tutorial-learn-how-to-run-tontine-ethereum-dapp-game-contract.php)
*   [Design Tontine Ethereum DApp Game Interfaces](https://myhsts.org/tutorial-learn-how-to-design-tontine-ethereum-dapp-game-interfaces.php)
*   [Contract Interactions between Ethereum and Solidity via Tontine DApp Game](https://myhsts.org/tutorial-learn-about-blockchain-contract-interactions-between-ethereum-and-solidity-via-tontine-dapp-game.php)
*   [Work with Truffle Unit Testing in Tontine DApp Game](/recipes/tutorials/work-with-ethereum-solidity-and-truffle-unit-testing-in-tontine-dapp-game/)
*   [Debugging with Truffle and Ethereum Remix in Tontine DApp Game](https://dev.to/weg2g/debugging-with-truffle-and-ethereum-remix-in-tontine-dapp-game-20mi)
*   [Building Frontend Application for Tontine DApp Game with Drizzle](https://myhsts.org/tutorial-learn-how-to-build-frontend-blockchain-application-for-Ethereum-dapp-game-with-drizzle.php)
*   [Running and Playing Tontine Ethereum DApp Game](https://myhsts.org/tutorial-learn-how-to-run-and-play-tontine-ethereum-dapp-game.php)

**_IMPORTANT: Understanding and completing the first set of recipes are required prior to working on second set of recipes._**

#### 3. Web3.JS API Setup for Ethereum Smart Contracts

Ethereum provides us with web3.js, which is a useful API to make the web developer's life easier. This JavaScript API enables us to communicate with an Ethereum node using JSON RPC endpoints exposed on top of HTTP, WebSocket, and/or IPC transports from a web page.

In auction.js, copy and paste the following code, so we can walk through it step by step while introducing the main web3.js features:
```
var web3 = new Web3();  
web3.setProvider(new web3.providers.HttpProvider("http://localhost:8545")); var bidder = web3.eth.accounts\[0\];  
web3.eth.defaultAccount = bidder;  
var auctionContract = web3.eth.contract("Here the contract's ABI"); // ABI omitted to make the code concise

function bid() {  
var mybid = document.getElementById('value').value;

auction.bid({  
value: web3.toWei(mybid, "ether"), gas: 200000  
}, function(error, result) { if (error) {  
console.log("error is " + error); document.getElementById("biding\_status").innerHTML = "Think to  
bidding higher";  
} else {  
document.getElementById("biding\_status").innerHTML = "Successfull bid, transaction ID" + result;  
}  
});  
}

function init() { auction.auction\_end(function(error, result) {  
document.getElementById("auction\_end").innerHTML = result;  
});  
auction.highestBidder(function(error, result) { document.getElementById("HighestBidder").innerHTML = result;  
});  
auction.highestBid(function(error, result) {  
var bidEther = web3.fromWei(result, 'ether'); document.getElementById("HighestBid").innerHTML = bidEther;  
});  
auction.STATE(function(error, result) { document.getElementById("STATE").innerHTML = result;  
});  
auction.Mycar(function(error, result) { document.getElementById("car\_brand").innerHTML = result\[0\]; document.getElementById("registration\_number").innerHTML =  
result\[1\];  
});  
auction.bids(bidder, function(error, result) { var bidEther = web3.fromWei(result, 'ether');  
document.getElementById("MyBid").innerHTML = bidEther; console.log(bidder);  
});  
}

var auction\_owner = null; auction.get\_owner(function(error, result) {  
if (!error) {  
auction\_owner = result;  
if (bidder != auction\_owner) {  
$("#auction\_owner\_operations").hide();  
}  
}  
});

function cancel\_auction() { auction.cancel\_auction(function(error, result) {  
console.log(result);  
});  
}

function Destruct\_auction() { auction.destruct\_auction(function(error, result) {  
console.log(result);  
});  
}

var BidEvent = auction.BidEvent(); BidEvent.watch(function(error, result) {  
if (!error) {  
$("#eventslog").html(result.args.highestBidder + ' has bidden(' + result.args.highestBid + ' wei)');  
} else {  
console.log(error);  
}  
});

var CanceledEvent = auction.CanceledEvent(); CanceledEvent.watch(function(error, result) {  
if (!error) {  
$("#eventslog").html(result.args.message + ' at ' + result.args.time);  
}  
});

const filter = web3.eth.filter({ fromBlock: 0,  
toBlock: 'latest', address: contractAddress,  
topics: \[web3.sha3('BidEvent(address,uint256)')\]  
});

filter.get((error, result) => {  
if (!error) console.log(result);  
});
```

##### Step 1 – Talking to the blockchain

Web3Js provides the web3 object that enables us to exploit the Web3 API functions in JavaScript. Therefore, the first action to take is to instantiate a web3 object as follows:`` var web3 = new Web3();``

This object needs to be connected to an RPC provider to communicate with the blockchain. We set a local or remote web3 provider using
```
web3.setProvider(new web3.providers.HttpProvider("http://RPC\_IP:RPC\_Port"));
```
where RPC\_IP is the RPC provider's IP and RPC\_Port is its RPC port.

##### Step 2 – Interaction with the smart contract

Web3 also provides a JavaScript object, web3.eth.Contract, which represents your deployed contract. To find and interact with your newly deployed contract on the blockchain, this object needs to know the contract's address and its **application binary interface (ABI)**:
```
var auctionContract = web3.eth.contract(“your contract's ABI”); var auction = auctionContract.at(“contract address”);
```

#### 4. Application Binary Interface for Smart Contracts

For many, the Application Binary Interface or ABI is a slightly confusing concept. The ABI is essentially a JSON object containing a detailed description (using a special data-encoding scheme) of the functions and their arguments, which describes how to call them in the bytecode.

Based on the ABI, the web3.js will convert all calls into low-level calls over RPC that the EVM can understand. In other words, web3.js needs to know some information (in a predefined format) about the functions and variables, in order to generate instructions that can be interpreted by an Ethereum node (EVM), and to get you the result back in a human- readable way. The EVM doesn't know about function names or variables, it processes only the data as a sequence of bytes accordingly to predefined rules. For example, the first four bytes in a transaction payload specify the signature of the called function, hence the need for an ABI that helps any API or tool to generate a transaction with correct low-level payloads and to read returned values by the contract.

#### 5. Call versus Send Transactions for Ethereum

In our JavaScript script, we define the following functions previously called in the HTML file:

*   `init()`: Serves to read contract states and display auction details such as the current highest bid and bidder, the auction status and more
*   `bid()`: Enables participants to make a bid
*   `cancel\_auction()`: Enables the owner to cancel the auction
*   `destruct\_auction()`: Destroys the contract

Each of these functions invokes methods defined in your contract. In Ethereum, there are two ways to invoke a method present in a smart contract, whether by addressing a function call or by sending a transaction.

##### Invoking contract methods via a call

A call is a contract instance invocation that doesn't change the contract state, and includes calling view or pure functions or reading public states. The call only runs on the EVM of your local node and saves you the expensive gas as there is no need for broadcasting the transaction. In your init() function, we need exactly that, as we only have to read the contract states. We can call a method foo() on the local blockchain without having to send out a transaction using myContractInstance.foo.call(arg1,arg2...).

##### Invoking contract methods via a transaction

To change the state of the contract instance, instead of making a call, you need to send a transaction that costs you real money (gas) to validate your action.

For instance, to invoke the bid() method, you have to send a transaction to the blockchain network with the necessary arguments and fees. Just like with the call, to invoke the foo() method, you need to explicitly make the invocation using sendTransaction() as follows:  
```
myContractInstance.foo.sendTransaction(param1 \[, param2, ...\] \[, transactionObject\] \[, callback\]);  
```
Web3.js is smart enough to automatically determine the use of call or sendTransaction based on the method type indicated in the ABI, therefore you can directly use the following:
```
auction.bid({value: web3.toWei(mybid, "ether"), gas: 200000}, function(error, result){...});
```
In general, to execute a function foo(argument1, argument2), we use the following:  
```
auction.foo(argument1, argument2,{Transaction object}, fallback\_function{...});  
```
Here, auction is a web3.js contract object. In the invocation, we provide the following arguments in the following order:

*   The contract's function arguments.
*   A transaction object that represents details of the transaction that will execute the function (source, destination, amount of ether, gas, gas price, data). As the address for the sending account, we use the coinbase that is the default account that the Ethereum client will use. You can have as many accounts as you want and set one of them as a coinbase.
*   At the end, we define a callback function which will be executed once the transaction is validated.

In your bid() function, the bid amount is specified in the transaction object, while in the UI form, the bidder will specify the amount in ether, therefore we will need to convert the bid into wei by using web3.toWei(mybid, "ether").

##### Callbacks

Web3.js is designed to work with a local RPC node, and all of its functions use synchronous HTTP requests by default. If you want to make an asynchronous request, you can pass an optional callback as the last parameter to most functions. All callbacks us an error-first callback pattern, as follows:
```
web3.eth.foo(argument1,..,argumentn, function(error, result) { if (!error) {  
console.log(result);  
}  
});
```
#### 6. Reading State Variables and More

As reading the public state doesn't change any information in the blockchain, it can be performed locally using a call. For example, to read the auction's highestbid variable, you can use auction.highestBid.call();.

##### Watching events

One more important thing to take care of in your frontend is handling events. Your DApp should be able to listen for fired events by applying the function watch() to the expected event to detect changes, or get() to read a specific log. The following code shows an example of event watching:
```
var CanceledEvent = auction.CanceledEvent(); CanceledEvent.watch(function(error, result) {  
if (!error) {  
$("#eventslog").html(result.args.message + ' at ' + result.args.time);  
}  
});
```
Each time a new bid is received by the contract, the callback function will display the log.

##### Indexing events and ﬁltering

In our auction smart contract, we used the indexed argument in BidEvent() declaration for a clever reason:  
```
event BidEvent(address indexed highestBidder, uint256 highestBid);
```
When you use the indexed keyword in your event, your Ethereum node will internally index arguments to build on indexable search log, making it searchable, meaning you can easily perform a lookup by value later on.

Suppose that the event was emitted by the MyAuction contract with the arguments BidEvent(26, “bidder's address”);. In this case, a low-level EVM log with multiple entries will be generated with two topics and a data field as shown in the following browser console screenshot:

![Ethereum blockchain development with solidity](https://coding-bootcamps.com/img/external/12/ethereum-and-solidity-with-truffle-andweb3js.gif)

The topics and data fields are as follows:

**Topics:**  
The event signature: The large hex number 0d993d4f8a84158f5329713d6d13ef54e77b325040c887c8b3e5  
65cfd0cd3e21 is equal to the Keccak-256 hash of BidEvent (address, uint256)

**Data:**  
HighestBidder: The bidder's address, ABI-encoded and ighest bid value, ABI-encoded

As you know, logs tend to be much larger and sometimes longer. In web3.js, you can filter data in your contract's log by simply applying a filter as follows:  
```
var contractAddress="0x00..."  
const filter = web3.eth.filter({fromBlock: 1000000,toBlock: 'latest',address: contractAddress,topics: \[web3.sha3('BidEvent(address, uint256)')\]})  
filter.get((error, result) => { if (!error) {  
console.log(result);  
}  
});
```
The preceding filter will give you all the log entries for the BidEvent event corresponding to your filtering object, in which we define the blocks (fromBlock, toBlock) to read the log from, along with the account address and the event we are interested in. To listen for state changes that fit the previous filter and call a callback, you can use the watch method instead, as follows:
```
filter.watch(function(error, result) { if (!error) {  
console.log(result);  
}  
});
```
To stop and uninstall the filter, you can use the stopWatching method:
```
filter.stopWatching();  
```
As shown in the preceding screenshot, the log outputs are ABI-encoded, and to decode a log receipt, you can use decodeParameter available in web3Js 1.0 as follows:  
```
web3.eth.abi.decodeParameter(‘uint256', '0000000000000000000000001829d79cce6aa43d13e67216b355e81a7fffb220')
```
Alternatively, you can use an external library such as

```
https://github.com/ConsenSys/abi-decoder
```

##### Numbers in Ethereum and floating point

You might be surprised to learn that the Ethereum VM doesn't support floating point numbers. Discussing the reasons behind that choice is out of the scope of this book, so let's focus on how to deal with this limitation, in order to enable bidding in floating point numbers.

In Ethereum, we can overcome this issue by using the system used to represent different units of ether. In this system, the smallest denomination or base unit of ether is called wei (Ether=1018 wei) along with other units:
```
9 wei = 1 Shannon  
1012 wei = 1 Szabo  
1015 wei = 1 Finney
```
Although we did not set the units of any bids in the contract, the received bids will be counted in wei. This will help us to accept floating point numbers, as they will become integers after being multiplied by 1018. For instance, 1234.56789 ether will be represented as 123456789E+14 wei. While the bid is done through the auction form in ether and stored in the contract in wei, to display back the bid's value in ether, we convert from wei to ether using var value = web3.fromWei('21000000000000', ‘ether');.

##### Transaction status receipt

Since the Byzantium Fork, it's been possible to find out whether a transaction was successful. Indeed, you can check the transaction status using the following:
```
var receipt = web3.eth.getTransactionReceipt(transaction\_hash\_from\_60\_minutes\_ago);
```
It returns a set of information including a status field, which has a value of zero when a transaction has failed and 1(0x1) when the transaction has succeeded (it has been mined).

So far, we have tested our contract and our user interface is ready to interact with it. We now have to assemble the puzzle pieces. To get our DApp working, we need to deploy our contract in a test environment other than Remix's VM JavaScript that we used earlier. We explain it in details on our next recipe.

This recipe is written by Matt Zand (the CEO of [Hash Flow](https://www.hashflow.us/ "Blockchain Consulting and Development")) in collaboration with Brian Wu who is a senior blockchain consultant at [High School Technology Services](https://myhsts.org/) school in Washington DC.
