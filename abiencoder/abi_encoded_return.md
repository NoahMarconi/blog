# Solidity ABI Encoded Return Types

Counterfactual's generalized state channel implementation relies on abi encoding to pass arbitrary data structures between JavaScript client applications and smart contracts. A useful pattern I wanted to learn more about and incorporate into my own projects.

Thank you to [Liam Horne](https://medium.com/@liamhorne) for talking me through how all of this works.

## Setup

In the examples below I'll use [ethers.js](https://github.com/ethers-io/ethers.js/) client side and [ganache-cli](https://github.com/trufflesuite/ganache-cli) to deploy and interact with a smart contract. Be sure to have these packages plus [solc-js](https://github.com/ethereum/solc-js) installed.

Follow along by pasting the by using a `nodejs` console for the JavaScript examples.


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


## Reading Complex Data Types

Lets start with a Solidty smart contract that returns an array of `User` structs. It's a good example as we can see a variety of data types (`struct`, `enum`, `dynamic array` and `uint256`) in one return value.

The following code snippets are run in a `nodejs` console to offer a self contained example. In practice use your regular deployment environment.

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
    users.push(User(111, Permission.Admin));
    users.push(User(222, Permission.Write));
    users.push(User(333, Permission.ReadOnly));
  }

  function get() public constant returns (User[]) {
    return users;
  }

}`;


// Compile the above snippet.
let compiledContract = solc.compile(contractCode);

// And log any errors.
console.log(compiledContract.errors[0]);
```



Gives us an error message:

```
:21:43: TypeError: This type is only supported in the new experimental ABI encoder. Use "pragma experimental ABIEncoderV2;" to enable the feature.
  function get() public constant returns (User[]) {
                                          ^----^
```


Let's do what the message suggests and see what happens. Adding in `pragma experimental ABIEncoderV2;`.

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
    users.push(User(111, Permission.Admin));
    users.push(User(222, Permission.Write));
    users.push(User(333, Permission.ReadOnly));
  }

  function get() public constant returns (User[]) {
    return users;
  }

}`;


// Compile again.
compiledContract = solc.compile(contractCode);

// And log any errors.
console.log(compiledContract.errors[0]);
```

No error this time, instead a only warning that an experimental feature is being used.

```
:2:1: Warning: Experimental features are turned on. Do not use experimental features on live deployments.
pragma experimental ABIEncoderV2; // Use experimental encoding.
^-------------------------------^
```

Next, deploy using a `ganache-cli` instance and call the getter (`get()`).

```
// Additional imports
const ganache      = require("ganache-cli");

// Provider / Signer setup.
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


        ABIExample.get().then(res => {

            // Log result of calling the contract's `get()` function.
            userArray = res;
            console.log(userArray);
        });

    });
});
```


Logs the following array of objects to the console.

```
[ [ BigNumber { _bn: <BN: 6f> },
    2,
    id: BigNumber { _bn: <BN: 6f> },
    permission: 2 ],
  [ BigNumber { _bn: <BN: de> },
    1,
    id: BigNumber { _bn: <BN: de> },
    permission: 1 ],
  [ BigNumber { _bn: <BN: 14d> },
    0,
    id: BigNumber { _bn: <BN: 14d> },
    permission: 0 ] ]
```

Great, we've successfully read an `array` of `struct`s which contain `enums` from a smart contract.


## Reading Encoded Bytes

Another pattern is to return abi encoded `bytes` instead of a custom data type. See the added `getBytes() returns (bytes)` its use of `abi.encode` prior to returning the `users` array:


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
    users.push(User(uint256(111), Permission.Admin));
    users.push(User(uint256(222), Permission.Write));
    users.push(User(uint256(333), Permission.ReadOnly));
  }

  function getUsers() public constant returns (User[]) {
    return users;
  }

  // New method to encode users before returning as bytes.
  function getUsersBytes() public constant returns (bytes) { // Change return type to bytes.

    User[] memory allUsers = users; // Important: load array into memory.

    return abi.encode(allUsers); // Encode prior to return.
  }

}`

// Recompile
compiledContract = solc.compile(contractCode);

// Prepare deployment overwriting interface and bytecode.
interface         = compiledContract.contracts[':ABIExample'].interface;
bytecode          = compiledContract.contracts[':ABIExample'].bytecode;
deployTransaction = ethers.Contract.getDeployTransaction('0x' + bytecode, interface);
sendPromise       = signer.sendTransaction(Object.assign(deployTransaction, { gas: 6e6 }));

// Deploy contract.
let userBytesArray;
sendPromise.then(res => {
    web3Provider.getTransactionReceipt(res.hash).then(res => {
        transactionReceipt = res;

        // New JS contract interface.
        ABIExample = new ethers.Contract(transactionReceipt.contractAddress, interface, signer);

        ABIExample.getUsersBytes().then(res => {

            // Log result of calling the contract's `getUsersBytes()` function.
            userBytesArray = res;
            console.log(userBytesArray);
        });
    });
});

```


Logs the following bytes to the console:

```
0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000006f000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000de0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000014d0000000000000000000000000000000000000000000000000000000000000000
```

Now, decode those bytes using the `ethers.js` `AbiCoder`.

```
let usersArrayType = ["tuple(uint256,uint8)[]"];
let decodedData    = ethers.utils.AbiCoder.defaultCoder.decode(usersArrayType, userBytesArray);

console.log(decodedData);

// Or to print nicer decodedData[0].forEach(x => console.log(`id -> ${x[0]} permission -> ${x[1]}`));

// Or if you have the abi the following would work as well.
usersArrayType = [{"components":[{"name":"id","type":"uint256"},{"name":"permission","type":"uint8"}],"name":"","type":"tuple[]"}]
decodedData    = ethers.utils.AbiCoder.defaultCoder.decode(usersArrayType, userBytesArray);

console.log(decodedData);
```


Using `"tuple(uint256,uint8)[]"` as the type string returns some nested arrays.

```
[ // AbiCoder wraps all of the returned data.

    [ // User array returned from our toy smart contract.

        [ [Object], 2 ],  // `User` struct as an array of two BigNumber objects.
                          // The first of which is `uint256 id`.
                          // While the second is `Permission permission` represented
                          // as a `uint8`.


        [ [Object], 1 ],  // `User` struct
        [ [Object], 0 ]   // `User` struct
    ]
]
```

Awesome, we can read arbitrary data structures returned from a Solidity smart contract, including `array`, `struct`, and `enum` in their raw format or encoded as `bytes`.

Let's try the other direction and pass in these data structures as arguments.


## Writing Complex Data Types

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
    users.push(User(uint256(111), Permission.Admin));
    users.push(User(uint256(222), Permission.Write));
    users.push(User(uint256(333), Permission.ReadOnly));
  }

  function getUsers() public constant returns (User[]) {
    return users;
  }

  function getUsersBytes() public constant returns (bytes) {
    User[] memory allUsers = users;
    return abi.encode(allUsers);
  }

  // Add a setter to add new users.
  function set(User newUser) public {

    users.push(newUser);
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
sendPromise.then(res => {
    web3Provider.getTransactionReceipt(res.hash).then(res => {
        transactionReceipt = res;

        // New JS contract interface.
        ABIExample = new ethers.Contract(transactionReceipt.contractAddress, interface, signer);

        // Update state with User(444, Permission.Admin).
        ABIExample.set({ id: 444, permission: 2 }, { gasLimit: 2e5 }).then(function(transaction) {

            // Get the users array again to see the new `User` persisted.
            ABIExample.getUsers().then(res => {

                // Log the result.
                userArray = res;
                console.log(userArray);
            });
        });
    });
});
```

Logs an updated array. We've done a round trip writing a `struct` then reading it back as an `array of structs`.

```
[ [ BigNumber { _bn: <BN: 6f> },
    2,
    id: BigNumber { _bn: <BN: 6f> },
    permission: 2 ],
  [ BigNumber { _bn: <BN: de> },
    1,
    id: BigNumber { _bn: <BN: de> },
    permission: 1 ],
  [ BigNumber { _bn: <BN: 14d> },
    0,
    id: BigNumber { _bn: <BN: 14d> },
    permission: 0 ],
  [ BigNumber { _bn: <BN: 1bc> },
    2,
    id: BigNumber { _bn: <BN: 1bc> },
    permission: 2 ] ]
```

## Writing Encoded Bytes

For this last example we'll add a caller contract to show how

```
let callerContractCode = `pragma solidity ^0.4.24;


contract Caller {

    function callExternal(address _to, bytes _data)
        public
        returns (bool success)
    {
        assembly {
            success := call(not(0), _to, 0, add(_data, 0x20), mload(_data), 0, 0)
        }
    }

}`

// Compile.
let callerCompiledContract = solc.compile(callerContractCode);

// Prepare deployment with new interface and bytecode.
let callerInterface = callerCompiledContract.contracts[':Caller'].interface;
let callerBytecode  = callerCompiledContract.contracts[':Caller'].bytecode;
deployTransaction   = ethers.Contract.getDeployTransaction('0x' + callerBytecode, callerInterface);
callerSendPromise   = signer.sendTransaction(Object.assign(deployTransaction, { gas: 6e6 }));
```


The above allows us to pass arbitrary bytes and call out to another smart contract. We'll use this `Caller` contract to send encoded bytes from our nodejs console and have them received as a `User struct` in the ABIExample set method.

```
let userType = ["tuple(uint256,uint8)"];
let inputData = [ // AbiCoder wraps all of the data.

    [ // User tuple.
      555, // uint256 representing `id`.
      0,   // uint256 representing `permission`.
    ]

];

let encodedData = ethers.utils.AbiCoder.defaultCoder.encode(userType, inputData);

```

`encodedData` is the following string `'0x000000000000000000000000000000000000000000000000000000000000022b0000000000000000000000000000000000000000000000000000000000000000'`. Perfect if calling out to a fallback function but in our case we need the function signature prepended. `ethers.js` has us covered again:


```
// Or using our contract interface:
encodedData = ABIExample.interface.functions.set({ id: 555, permission: 0 }).data;
```

Finally, let's send this data to our Caller contract, which in turn relays it to ABIExample where the data is treated a `User struct`.

```
let Caller;

callerSendPromise.then(res => {
  web3Provider.getTransactionReceipt(res.hash).then(res => {
    transactionReceipt = res;

    // New JS contract interface.
    Caller = new ethers.Contract(transactionReceipt.contractAddress, callerInterface, signer);

    // Update state with User(111, Permission.Admin).
    Caller.callExternal(ABIExample.address, encodedData, { gasLimit: 2e5 }).then(res => {

        // Get the users array again to see the new `User` persisted.
        ABIExample.getUsers().then(res => {

            // Log result.
            userArray = res;
            console.log(userArray);
        });
    });
  });
});
```

Logs the `users` array showing the relayed new `User` has been recorded!

```
[ [ BigNumber { _bn: <BN: 6f> },
    2,
    id: BigNumber { _bn: <BN: 6f> },
    permission: 2 ],
  [ BigNumber { _bn: <BN: de> },
    1,
    id: BigNumber { _bn: <BN: de> },
    permission: 1 ],
  [ BigNumber { _bn: <BN: 14d> },
    0,
    id: BigNumber { _bn: <BN: 14d> },
    permission: 0 ],
  [ BigNumber { _bn: <BN: 1bc> },
    2,
    id: BigNumber { _bn: <BN: 1bc> },
    permission: 2 ],
  [ BigNumber { _bn: <BN: 22b> },
    0,
    id: BigNumber { _bn: <BN: 22b> },
    permission: 0 ] ]
```



## Conclusion

The combination of `ethers.js` and Solidity's `ABIEncoderV2` opens the door to two means of reading and writing complex data types: using the types directly, using or their `bytes` representation. These useful techniques are used throughout Counterfactual's generalized state channels implementation.


## Further Reading

For more information check out:

  - [Human-Readable Contract ABIs](https://blog.ricmoo.com/human-readable-contract-abis-in-ethers-js-141902f4d917) by Richard Moore;
  - [Solidity ABIv2: A Foray into the Experimental](https://blog.ricmoo.com/solidity-abiv2-a-foray-into-the-experimental-a6afd3d47185) by Richard Moore;
  - [Api Docs](https://docs.ethers.io/ethers.js/html/api-advanced.html#abi-coder) for ethersjs;
  - [Application Binary Interface](https://solidity.readthedocs.io/en/develop/abi-spec.html#) Solidity docs
  - [ABI Encoding Functions](https://solidity.readthedocs.io/en/develop/units-and-global-variables.html#abi-encoding-functions) Solidity docs
  - Counterfactual's use of abi encoding at [TBA]()
