# Oracle

## Intent

Gain access to data stored outside of the blockchain.

## Motivation
Every computation on the Ethereum blockchain has to be validated by every participating node in the network. It would not be practical to allow for any kind of external network requests from a contract, since every node would have to make that request on their own to verify its result. Not only would this lead to excessive network usage, which smaller sites could possibly not handle, also a change in the requested information would break the consensus algorithm. Contracts on the blockchain itself are therefore not able to communicate with the outside world, meaning they cannot pull information from sources like the internet. But for many contracts, information about external events is necessary to fulfill their core purpose, especially since the industry is looking for more complex use cases. There are already contracts that rely on information about the result of a sport event, the price of a currency or the status of a flight. One of the first solutions to overcome this limitation was [Orisi](https://github.com/orisi/wiki/wiki/Orisi-White-Paper), a service published in 2014, with the aim of being an intermediary between the Bitcoin blockchain and the outside world. Since then several services for different blockchains with similar concepts have emerged under the term oracle. The oracle acts as an agent living on the blockchain and providing information in the form of responses to queries.

An important point when handling data in the context of a blockchain is the notion of trust. As there is no central authority, trust has to be built from concepts like immutability and a working consensus algorithm. When relying on externally introduced information it is necessary to find a way to build up trust for that information as well. This concept is known as [the oracle problem](https://blog.chain.link/what-is-the-blockchain-oracle-problem/#:~:text=The%20oracle%20problem%20revolves%20around,computer%20with%20no%20Internet%20connection), and one of the most common ways of building up this trust is to obtain data from multiple sources using a decentralized oracle network.

## Applicability

Use the Oracle pattern when
* you rely on information that can not be provided from within the blockchain.

## Participants & Collaborations

The oracle pattern consists of three entities: the contract requesting information, the oracle(s) and the data source(s). The process begins with a contract requesting information, which they cannot retrieve from within the blockchain. Therefore a transaction is sent to an oracle contract, which lives on the blockchain as well. This transaction contains a request that the contract wishes to be fulfilled. A good example is a contract requesting the current price of Ether.

The oracle is essentially a program running off-chain that acts as a bridge between the outside world and the blockchain. An oracle then picks up this request from the on-chain oracle contract, and then forwards the request to the agreed data source. Because the data source is off-chain, this communication is not conducted via blockchain transactions, but through other forms of digital communication, usually via HTTP requests.

When the request reaches the data source, it is processed and the reply is sent back to the oracle. From there on the oracle then sends the response back to the initial calling contract via the on-chain oracle contract. Once this is complete, the initial calling contract can then perform any logic required using the obtained result.

## Implementation

[Chainlink](http://chain.link/) oracles have become the preferred oracle solution for smart contracts looking to obtain external data, thanks to their quick and easy implementation, as well as their flexibility & scalability. In this section we'll focus on the implentation of the pattern in the requesting contracts, and leave all of the off-chain logic and processing to a chainlink oracle. Another example of an oracle service is [Town Crier](http://www.town-crier.org/), which also works with trusted compute hardware.

To make a simple API request, the requesting contract simply needs to import and implement the ChainlinkClient contract, set a few contructor variables, and then provide a method for requesting data, and one to receive data. The contructor sets the following values:
1. Sets the LINK token contract to the address of the LINK token for the given network. 
2. Sets the oracle contract address to send the request to 
3. Sets the jobId to a job running on a Chainlink oracle on the given network to process the request
4. Sets the required fee to be send in LINK to the oracle to fulfill the request

More information on each value can be found in the [Chainlink documentation](https://docs.chain.link/docs/make-a-http-get-request#config)

The next part of the code is the requesting function, in this case `requestVolumeData()`. This function does the following:
1. Builds up a request to send to a Chainlink oracle, containing the URL, JSON path to traverse to obtain the result, which method to call in this contract when it has a result, and tells the node to convert the result to account for decimals.
2. Sends the request to the given oracle contract.

Once the oracle has retrieved a response, it will call the specified function in our calling contract, passing the result. In this case it's the `fulfill` function, which receives the result and stores it in the smart contract in the volume variable.

## Sample Code

In this example below we'll obtain the current volume of the ETH/USD pair on the cryptocompare API using a Chainlink oracle running on the Ethereum Kovan testnet. Before you can make a request by calling the `requestVolumeData` function, you need to [fund your contract with some LINK](https://docs.chain.link/docs/fund-your-contract)

```Solidity
// This code has not been professionally audited, therefore I cannot make any promises about
// safety or correctness. Use at own risk.
pragma solidity ^0.6.0;

import "https://raw.githubusercontent.com/smartcontractkit/chainlink/develop/evm-contracts/src/v0.6/ChainlinkClient.sol";

contract APIConsumer is ChainlinkClient {
  
    uint256 public volume;
    
    address private oracle;
    bytes32 private jobId;
    uint256 private fee;
    
    /**
     * Network: Kovan
     * Chainlink - 0x2f90A6D021db21e1B2A077c5a37B3C7E75D15b7e
     * Chainlink - 29fa9aa13bf1468788b7cc4a500a45b8
     * Fee: 0.1 LINK
     */
    constructor() public {
        setPublicChainlinkToken();
        oracle = 0x2f90A6D021db21e1B2A077c5a37B3C7E75D15b7e;
        jobId = "29fa9aa13bf1468788b7cc4a500a45b8";
        fee = 0.1 * 10 ** 18; // 0.1 LINK
    }
    
    /**
     * Create a Chainlink request to retrieve API response, find the target
     * data, then multiply by 1000000000000000000 (to remove decimal places from data).
     */
    function requestVolumeData() public returns (bytes32 requestId) 
    {
        Chainlink.Request memory request = buildChainlinkRequest(jobId, address(this), this.fulfill.selector);
        
        // Set the URL to perform the GET request on
        request.add("get", "https://min-api.cryptocompare.com/data/pricemultifull?fsyms=ETH&tsyms=USD");
        
        // Set the path to find the desired data in the API response, where the response format is:
        // {"RAW":
        //      {"ETH":
        //          {"USD":
        //              {
        //                  ...,
        //                  "VOLUME24HOUR": xxx.xxx,
        //                  ...
        //              }
        //          }
        //      }
        //  }
        request.add("path", "RAW.ETH.USD.VOLUME24HOUR");
        
        // Multiply the result by 1000000000000000000 to remove decimals
        int timesAmount = 10**18;
        request.addInt("times", timesAmount);
        
        // Sends the request
        return sendChainlinkRequestTo(oracle, request, fee);
    }
    
    /**
     * Receive the response in the form of uint256
     */ 
    function fulfill(bytes32 _requestId, uint256 _volume) public recordChainlinkFulfillment(_requestId)
    {
        volume = _volume;
    }
    
    /**
     * Withdraw LINK from this contract
     * 
     * NOTE: DO NOT USE THIS IN PRODUCTION AS IT CAN BE CALLED BY ANY ADDRESS.
     * THIS IS PURELY FOR EXAMPLE PURPOSES ONLY.
     */
    function withdrawLink() external {
        LinkTokenInterface linkToken = LinkTokenInterface(chainlinkTokenAddress());
        require(linkToken.transfer(msg.sender, linkToken.balanceOf(address(this))), "Unable to transfer");
    }
}
```

## Consequences

The most important consequences of applying the oracle pattern is gaining access to data otherwise not being available on the blockchain, and therefore allowing business models and use cases with whole new functionality. Besides providing arbitrary data from the web, oracles can be used to automatically trigger functionality in other systems, such as payment procesors or enterprise systems. It is also often used for generating random numbers, a difficult task, as described in the [Randomness pattern](./randomness.md). From a developer standpoint it is fairly easy to implement the oracle pattern, especially when using one of the already existing services such as [Chainlink](http://chain.link). Another benefit of using an existing solution is the fact that these solutions are heavily audited, reducing the risk of errors.

A negative consequence of the usage of oracles is the introduction of an additional point of failure. The contract creator as well as the users interacting with the contract rely heavily on the information provided by oracles. Oracles or their data sources have reported wrong data in the past and it is likely that there will be errors in the future again. 

Another negative consequence is the trust that has to be put into both, oracles and data sources. In an environment that strives towards decentralization, relying on a single external entity seems contradictory. This issue can be mitigated by using multiple oracle nodes and multiple data sources, thereby extending the security and tamper proof properties out to the oracle layer. The results could then be compared and evaluated or aggregated This is how [Chainlink Price Feeds](https://data.chain.link/) are implemented, and have proven to be a battle tested secure way of bringing price data on-chain.

Another method of mitigating trust from the oracles is beeing used at Oraclize: [TLSNotary proofs](https://tlsnotary.org/). Using TLSNotary, Oraclize can prove that they visited the specified website at a certain time and indeed recieved the provided result. While this would not prevent Oraclize from querying random numbers until they get the desired result, it is trustworthy in case the requested data does not fluctuate over small periods of time.

## Known Uses
 
Usage of the oracle pattern can be observed in a variety of contracts on the blockchain. An example using Chainlink Oracles is decentralied finance protocol [Aave](https://github.com/aave), which uses Chainlink Price Feeds to obtain external price data from multiple sources and multiple oracles. 

Another example of an oracle implementation is [Etherisc](https://github.com/etherisc/flightDelay-legacy), which uses Chainlink Oracles to obtain external flight data which is then used as inputs into Flight Delay insurance contracts.

[**< Back**](https://fravoll.github.io/solidity-patterns/)
