# Solidity ABI Encoded Return Types

Counterfactual's generalized state channel implementation relies on abi encoding to pass arbitrary data structures between JavaScript client applications and smart contracts. A useful pattern I wanted to learn more about and incorporate into my own projects.

Thank you to [Liam Horne](https://medium.com/@liamhorne) for talking me through how all of this works.

## Setup

In the examples below I'll use [ethers.js](https://github.com/ethers-io/ethers.js/) client side and [ganache-cli](https://github.com/trufflesuite/ganache-cli) to deploy and interact with a smart contract. Be sure to have these packages plus [solc-js](https://github.com/ethereum/solc-js) installed.

Follow along by pasting the by using a `nodejs` console for the JavaScript examples.

Before proceeding, start up ganache with:

```
$ ganache-cli
```

## Background

Solidity's `ABIEncoderV2` allows you to encode arbitrary data types, including structs and nested arrays!

Client side, the abi string `s` can represent any solidity type, with two nuances:

  1. `tuple` is used in place of a `struct`
  2. `uint256` is used in place of an `enum`

```
s := tuple
     uint256
     bytes32
     string
     uint256[]
     ...
```

For example, a dynamic `uint256` array is represented as `s = 'uint256[]'` or a more interesting example where a `struct { uint256 id; string name; }` is represented as `s = 'tuple(uint256,string)'`.

[ethers.js](https://github.com/ethers-io/ethers.js/) has built in decoding of abi encoded byte arrays returned from smart contracts, in addition to encoding of data before passing in structs as arguments to smart contract functions.

Below is a toy example showing a smart contract implementation along with the JavaScript encode/decode steps.


## Reading Encoded Data

Lets start with a Solidty smart contract that returns an array of `User` structs. It's a good example as we can see a variety of data types (`struct`, `enum`, `dynamic array` and `uint256`) in one encoded `bytes array`.

```{js}
const ethers = require('ethers');
const solc = require('solc');


let contractCode = `pragma solidity ^0.4.24;


contract ABIExample {

  enum Permission { ReadOnly, Write, Admin }

  struct User {
    uint256 id;
    Permission permission;
  }

  User[] users;

  constructor() public {
    users.push(User(0, Permission.Admin));
    users.push(User(1, Permission.Write));
    users.push(User(2, Permission.ReadOnly));
  }

  function get() public constant returns (User[]) {
    return users;
  }

}`;


// Compile the above snippet.
let compiledContract = solc.compile(contractCode);

// And log any errors.
console.log(compiledContract.errors[0])
```



Gives us an error message:

```
:21:43: TypeError: This type is only supported in the new experimental ABI encoder. Use "pragma experimental ABIEncoderV2;" to enable the feature.
  function get() public constant returns (User[]) {
                                          ^----^
```


Let's do what the message suggests and see what happens. Adding in `pragma experimental ABIEncoderV2;` and encoding with `abi.encode(users);` before returning a `bytes` array.

```
contractCode = `pragma solidity ^0.4.24;
pragma experimental ABIEncoderV2; // Use experimental encoding.

contract ABIExample {

  enum Permission { ReadOnly, Write, Admin }

  struct User {
    uint256 id;
    Permission permission;
  }

  User[] users;

  constructor() public {
    users.push(User(0, Permission.Admin));
    users.push(User(1, Permission.Write));
    users.push(User(2, Permission.ReadOnly));
  }

  function get() public constant returns (bytes) { // Edit return type.
    return abi.encode(users); // abi encode before returning.
  }

}`;


// Compile again.
compiledContract = solc.compile(contractCode);

// And log any errors.
console.log(compiledContract.errors[0]);
```

No error this time, instead a warning that an experimental feature is being used.

```
:2:1: Warning: Experimental features are turned on. Do not use experimental features on live deployments.
pragma experimental ABIEncoderV2; // Use experimental encoding.
^-------------------------------^
```

Next, deploy onto the locally running ganache instance and call the getter (`get()`).

```
// Additional imports
const ganache      = require("ganache-cli");
const providers    = ethers.providers;
const web3Provider = new providers.Web3Provider(ganache.provider({ gasLimit: 6e6 }));
const signer       = web3Provider.getSigner();

// Set up contract deployment.
let { interface, bytecode } = compiledContract.contracts[':ABIExample'];
let deployTransaction       = ethers.Contract.getDeployTransaction('0x' + bytecode, interface);
let sendPromise             = signer.sendTransaction(Object.assign(deployTransaction, { gas: 6e6 }));

// Deploy contract.
let transactionReceipt;
let ABIExample;
let userArray;

sendPromise.then(res => {
    web3Provider.getTransactionReceipt(res.hash).then(res => {
        transactionReceipt = res;
        // JS contract interface.
        ABIExample = new ethers.Contract(transactionReceipt.contractAddress, interface, web3Provider);

        // Log result of calling the contract's `get()` function.
        ABIExample.get().then(res => {
            userArray = res;
            console.log(userArray);
        });

    });
});
```

Logs the following bytes to the console:

```
0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001
```

Now, decode those bytes using `ethersjs`'s `AbiCoder`.

```
const usersArrayType = ["tuple(uint256,uint256)[]"];
let decodedData    = ethers.utils.AbiCoder.defaultCoder.decode(usersArrayType, userArray);

console.log(decodedData);
```


This returns some nested arrays.

```
[ // AbiCoder wraps all of the returned data.

    [ // User array returned from our toy smart contract.

        [ [Object], [Object] ],  // `User` struct as an array of two BigNumber objects.
                                 // The first of which is `uint256 id`.
                                 // While the second is `Permission permission` represented
                                 // as a `uint256` as is the case for Solidity enums.


        [ [Object], [Object] ],  // `User` struct
        [ [Object], [Object] ]   // `User` struct
    ]
]
```

Awesome, we can read arbitrary data structures returned from a Solidity smart contract, including arrays, structs, and emums.


Let's try the other direction and pass in encoded data as an argument.


## Writing Encoded data

First update the smart contract with a setter `set(User newUser)`:

```
contractCode = `pragma solidity ^0.4.24;
pragma experimental ABIEncoderV2;


contract ABIExample {

  enum Permission { ReadOnly, Write, Admin }

  struct User {
    uint256 id;
    Permission permission;
  }

  User[] users;

  constructor() public {
    users.push(User(0, Permission.Admin));
    users.push(User(1, Permission.Write));
    users.push(User(2, Permission.ReadOnly));
  }

  // Add a setter to add new users.
  function set(User newUser) public {

    users.push(newUser);
  }

  function get() public constant returns (bytes) {
    return abi.encode(users);
  }

}`

// Recompile
compiledContract = solc.compile(contractCode);

// Prepare deployment with new interface and bytecode.
interface         = compiledContract.contracts[':ABIExample'].interface;
bytecode          = compiledContract.contracts[':ABIExample'].bytecode;
deployTransaction = ethers.Contract.getDeployTransaction('0x' + bytecode, interface);
sendPromise       = signer.sendTransaction(Object.assign(deployTransaction, { gas: 6e6 }));

// Deploy contract.
transactionReceipt;
sendPromise.then(res => {
    web3Provider.getTransactionReceipt(res.hash).then(res => {
        console.log(res.contractAddress);
        transactionReceipt = res;
        // New JS contract interface.
        ABIExample = new ethers.Contract(transactionReceipt.contractAddress, interface, signer);

    });
});

```

Again from a `nodejs` console encode some data with:

```
let userType = ["tuple(uint256,uint256)"];
let inputData = [ // AbiCoder wraps all of the data.

    [ // User tuple.
      111, // uint256 representing `id`.
      2,   // uint256 representing `permission`.
    ]

];

let encodedData = ethers.utils.AbiCoder.defaultCoder.encode(userType, inputData);

// Or using our contract interface:
ABIExample.interface.functions.set({ id: 111, permission: 2 });

sendPromise = ABIExample.set({ id: 111, permission: 2 }, { gasLimit: 2e6 });

sendPromise.then(function(transaction) {
    console.log(transaction);
});
```

`encodedData` is the following string `'0x000000000000000000000000000000000000000000000000000000000000006f0000000000000000000000000000000000000000000000000000000000000002'` which can be passed in as an argument to the `set` function.



## Further Reading

For more information check out:

  - [Human-Readable Contract ABIs](https://blog.ricmoo.com/human-readable-contract-abis-in-ethers-js-141902f4d917) by Richard Moore;
  - [Api Docs](https://docs.ethers.io/ethers.js/html/api-advanced.html#abi-coder) for ethersjs;
  - [Application Binary Interface](https://solidity.readthedocs.io/en/develop/abi-spec.html#) Solidity docs
  - Counterfactual's use of abi encoding at [TBA]()
