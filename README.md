# Attestations

## Introduction

In simple terms, an `Attestator` contract is a tool for creating a network of trust by allowing people or contracts (called `attestors`) to make statements (called `attestations`) about specific facts or characteristics of other addresses (EOA or contracts). These `attestations` are stored securely and can be used to create a network of trusted `attestations` that can be used to establish the credibility and reputation of the attested parties. In addition, an `Attestator` contract allows for attestations to be made by either a person or a contract, making it an incredibly versatile tool for establishing trust in various contexts.

### Basic Example 

Let's say **Alice** wants to rent an apartment from **Bob**. However, **Bob** is hesitant to rent the apartment to her because he does not know anything about her rental history. **Alice** then uses an `Attestator` contract to ask **Evan**, _her previous landlord_, to attest to her being a responsible and reliable tenant. **Evan** attests to this using the `Attestator` contract where the `attestation` is `about` **Alice**'s address, setting a `key` as the "rental_history" and the `value` "reliable and responsible."

The attestation is then stored in the `Attestator` contract, and **Bob** can see that **Alice** has an attestation from **Evan** saying she is a good tenant. Based on this attestation, **Bob** can trust that Alice is a reliable renter and can decide to rent the apartment to her.

This system of attestations can help create a network of trust between people or contracts, where the credibility of attestors can be considered when evaluating the attested party. In this case, **Evan**'s credibility as a previous landlord helps establish **Alice**'s credibility as a reliable tenant.

## Implementation

The Attestator contract is a tool for creating a trusted network of attestations between different addresses, whether individuals or contracts. This contract allows for the secure and transparent storage of attestations by allowing attestors to make statements about specific attributes of other addresses, which are then stored in the attestations mapping.

Notice that _anyone_ can make an attestation on this contract. Nevertheless, that is not really important. It is more of a matter of _whom_ makes an attestation with the keys _we_ care about. 

```solidity=
contract Attestator {
    // Creator => About => Key => Value.
    // A mapping of mappings where the outer mapping's key is the address of the attestor
    // the middle mapping's key is the address of the attested party
    // the inner mapping's key is the attribute being attested to.
    mapping(address => mapping(address => mapping(bytes32 => bytes))) public attestations;
    
    // Function to attest a value to a specific key for a specific address
    function attest(address _about, bytes32 _key, bytes memory _val) public {
        // Sets the value for the specified key and addresses for the sender (attestor)
        attestations[msg.sender][_about][_key] = _val;
    }
}
```

One of the prominent features of the `Attestator` contract is its _simplicity_. The contract has a simple and easy-to-understand structure, with just one central mapping that stores all the attestations. This makes it easy for developers to understand and implement at minimal gas costs. Additionally, the `attest` function is straightforward, with just three parameters: the address of the attested party, the attribute to, and the value attested. This simplicity makes it easy for developers to quickly integrate the `Attestator` contract and start using it to establish a trusted network.

Another key feature of the `Attestator` contract is its _flexibility_. The contract can be used to establish trust in a wide variety of contexts, including both individuals and contracts. This makes it a powerful tool for establishing trust in decentralized applications that rely on interactions between different addresses. The contract can also be used to attest to various attributes and even the functionality of dependent contracts. This flexibility makes it easy for developers to customize the contracts that rely on an `Attestator` contract to suit their specific needs.  

Furthermore, using an `Attestator` contract overcomes the need for complex upgrade operations. To illustrate, we want to switch a child contract (Program, Round, Vote, etc) and we still want to rely on existing data and not affect our other implementations. The attestator allows that. Or, suppose we want to utilize new attestations with different keys. The attestator allows that. Or maybe we want to use Merkle trees or even zk-proofs. Attestator. Or suppose we just want to store custom data for some other purpose. Of course, the `Attestator` can do that, too. 

An `Attestator` breaks down the complexity of relationships in a network because an attestation's creator is always trustable. **Instead of focusing on restricting what can be done by who, it focuses on what can be done as permitted by who.**

In our case, the _who_ is the contract that performs the attestations (Program, Round, Vote, etc.), and the _what_ is the keys we would like to utilize. The only restriction being the keys need to be 32-bytes. The data `value` can be any length of encoded bytes only restricted by gas cost.

## How-to Gitcoin Grants

The Gitcoin grants protocol relies on many contracts to manage Gitcoin Grants Programs and Rounds. This includes defining voting strategies, payout strategies, and any variety of related program/round/vote/payout metadata, etc. 

In a basic implementation, we may need the following: 

- `Program` contract to establish Grant Program(s).
- `Round` contract to establish Program Round(s).
- `Vote` contract to establish Voting on projects in a particular Program's Round.
- `Payout` contract to establish Paying-out a Round to various projects based on their Votes, respectively.

In other words, we need a _network of trusted contracts and addresses_ to read and write and thus execute some logic to perform grant program operations. 

### So what could this look like? 

In short, when someone decides to create a Grant program, they deploy an `Attestator` (or define an already deployed `Attestator`), which serves as the "glue" of the subsequent program, round, vote, and payout contracts. The `Attestator` would store all relevant data meant to be used amongst the deployed contracts. 

We can implement these features and core-immutable logic using interface contracts. 
The child contracts would implement additional logic with attestations from specific contracts/addresses whose keys we care to use. 

#### `IProgram.sol`

```solidity=
abstract contract IProgram {
    
  Attestator public immutable ATTESTATOR;
  address public immutable ATTESTER;
  constructor(
    address _attester, 
    Attestator _attestator
  ) {
    ATTESTATOR = _attestator; 
    ATTESTER = _attester;
  }

  modifier onlyProgramAdmin() {
    require(
        ATTESTATOR.attestations(
            ATTESTER, msg.sender, bytes32("program.is_admin")
        ).length > 0, "Program::onlyProgramAdmin: NOT_ADMIN");
    _;
  }

  function updateProgramAttestation(
      bytes32 _key, 
      bytes memory _value
  ) public onlyProgramAdmin {
    ATTESTATOR.attest(
      { _about: address(this), _key: _key, _val: _value }
    );
  }

}
```

The given code defines an abstract interface contract called `IProgram`. This contract contains a public immutable variable called `ATTESTATOR` that holds an instance of the Attestator contract, and another public immutable variable called `ATTESTER` that holds the address of the attester. The interface also includes a constructor that sets these variables, as well as a modifier called `onlyProgramAdmin` that restricts access to the program administrator. The modifier verifies the caller's identity by checking if the attestation exists for the caller and the "program.is_admin" key. Finally, the interface includes a function called `updateProgramAttestation` that updates an attestation for the program. This function can only be called by the program administrator, as specified by the `onlyProgramAdmin` modifier. The function calls the attest function of the `Attestator` contract to update the attestation for the program.

#### `IRound.sol`

```solidity=
abstract contract IRound {

  Attestator public ATTESTATOR;
  address public ATTESTER;

  constructor(address attester, Attestator attestator) {
    ATTESTATOR = attestator;
    ATTESTER = attester;
  }

  modifier onlyRoundAdmin() {
    require(
        ATTESTATOR.attestations(
            ATTESTER, msg.sender, bytes32("round.is_admin")
        ).length > 0, "Round::onlyRoundAdmin: NOT_ADMIN");
    _;
  }

  function updateRoundAttestation(
      bytes32 _key, 
      bytes memory _value
  ) public onlyRoundAdmin {
    ATTESTATOR.attest(
      { _about: address(this), _key: _key, _val: _value }
    );
  }
  function submitApplication(bytes memory _data) external virtual;
  function submitPayout(bytes[] calldata _data) external virtual payable;
  function submitVotes(bytes[] calldata _data) external virtual payable;
    
}
```

The `IRound` contract is an abstract contract that defines the basic structure and rules for a round. It uses the `Attestator` contract to keep track of the `attestations` of the different parties involved in the round. The contract has two public variables: `ATTESTATOR` and `ATTESTER`. `ATTESTATOR` is the instance of the `Attestator` contract that is used for keeping track of attestations, `ATTESTER` is the address authorized to perform admin actions.

The contract has a modifier called `onlyRoundAdmin` that restricts certain functions to be only accessible by the `ATTESTER` address. The function `updateRoundAttestation` is a function that allows the `ATTESTER` address to update the attestation value for a specific key.

The contract is an abstract contract, so it does not have any implementation of the functions. It declares three functions that are meant to be implemented by child contracts: `submitApplication`, `submitPayout`, and `submitVotes`. These functions handle the application, payout, and voting process for the funding round. The `IRound` contract provides the structure and rules for these processes, but it is up to the child contracts to implement the specific logic.

To illustrate an implementation, suppose we want only allowed voters to be able to call `sumbitVotes` in our round implementation. Our contract would simply look up a voters attestation made by a trusted source (could be the round contract itself, a different contract, or trusted address) and check if they have that permission (alternatively, this could be implemented on a Vote contract): 

```solidity=
contract RoundExample is IRound {
    // ...
    function submitVotes(bytes[] memory votes) {
        // check permission of sender defined by the 
        // `trustedAddress` attestation with the key `is_in_allow_list`
        // note: here we check the length of the byte array to not be 0
        //       we could choose to decode it and check the val itself.
        require(
            ATTESTATOR.attestations(
                trustedAddress, msg.sender, bytes32("is_on_allow_list")
            ).length > 0, "RoundExample::submitVotes: NOT_ALLOWED");
        // vote logic
        // i.e IVote(voteContract).vote(votes); 
    } 
    // ...
}
```
Now if a voter who does not have the attestation with the key `is_in_allow_list` made by `trustedAddress`, they will be met with a revert, otherwise we continue with the vote. 
Furthermore, the `trustedAddress` could be an entirely seperate contract itself, if desired; say someone wants a registration contract. Simple -- the registration contract would contain a `register` method which subseqently calls the `attest` method in the `Attestator` so the `trustedAddress` is the contract address who registered the user with the method's defined key. To extend our example even further, the `trustedAddress` could also de-register an address by setting the key's attestation to empty, or a replaced value, for any of its attestations. 

#### `IVote.sol`

```solidity=
abstract contract IVote {

  Attestator public ATTESTATOR;
  address public ATTESTER;

  constructor(address attester, Attestator attestator) {
    ATTESTATOR = attestator;
    ATTESTER = attester;
  }
  
  function vote(bytes[] calldata _votes) external virtual payable;

}
```

The `IVote` contract is an abstract contract that defines the basic structure and rules for voting in a round. It uses the `Attestator` contract to keep track of the attestations of the different parties involved in the round. The contract has two public variables: `ATTESTATOR`, and `ATTESTER`.

It declares one function that is meant to be implemented by a child contract, `vote`, where the child contract can implement any form of logic they desire, as long as it accepts an array of encoded round votes.

#### `IPayout.sol`

```solidity=
abstract contract IPayout {

  Attestator public ATTESTATOR;
  address public ATTESTER;

  constructor(address attester, Attestator attestator) {
    ATTESTATOR = attestator;
    ATTESTER = attester;
  }
  
  function payout(bytes[] calldata _data) external virtual payable;

}
```

The `IPayout` contract is an abstract contract that defines the basic structure and rules for paying out a round. It uses the `Attestator` contract to keep track of the attestations of the different parties involved in the round. The contract has two public variables: `ATTESTATOR`, and `ATTESTER`.

All subsequent logic (who can payout, what keys to look for, etc.) are defined in a child contract where any other logic can be implemented. 

The big thing to note here is all these contracts can share the same `Attestator` instance. All further implementations perform read operations to determine persmissions or write opertations to attest new data -- independent of the context logic. It is secure, specific, modular, and entirely flexible. 

## Summary

The `Attestator` contract is a tool for creating a network of trust between people or contracts by allowing them to make statements (attestations) about specific facts or characteristics of other addresses in a secure way. These attestations can be used to establish the credibility and reputation of the attested parties. The contract is simple, easy to use, and flexible, making it a powerful tool for establishing trust in various contexts. In the context of the Gitcoin grants protocol, the Attestator contract can be used as a "glue" to connect different contracts and establish a network of trusted addresses to execute grant program operations. Interface contracts such as `IProgram`, `IRound`, `IVote`, and `IPayout` can be used to define the structure and rules for different aspects of the grant program while using the Attestator contract to keep track of attestations and for use in down-the-line logic. 

If you'd like to see a basic implementation for Gitcoin, look at [this repository](https://github.com/hmrtn/gtc-attest). 
If you want to see this in action, look at Optimism's `op-nft` project [here](https://github.com/ethereum-optimism/optimism/tree/develop/packages/contracts-periphery/contracts/universal/op-nft).

-- Hans 
