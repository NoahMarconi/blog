# Solidity ABI Encoded Return Types

[Counterfactual](https://counterfactual.com/statechannels)'s generalized state channel implementation relies on abi encoding to pass arbitrary data structures between JavaScript client applications and smart contracts. This is a useful design pattern I wanted to learn more about and incorporate into my own projects.

In this blog post I describe the basics of ABI encoding in Solidity, and explain how it can be used to read and write complex data types.

Thank you to [Liam Horne](https://medium.com/@liamhorne) for talking me through how all of this works.

## Setup

In the examples below I'll use [ethers.js](https://github.com/ethers-io/ethers.js/) client side and [ganache-cli](https://github.com/trufflesuite/ganache-cli) to deploy and interact with a smart contract. Be sure to have these packages plus [solc-js](https://github.com/ethereum/solc-js) installed.

Code snippets are included throughout and self contained example (i.e. paste into a `nodejs` console and run) is shown at the end of the post.


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

[ethers.js](https://github.com/ethers-io/ethers.js/) has built in decoding of abi encoded byte arrays returned from smart contracts; it also encodes data as bytes before passing in structs as arguments to smart contract functions.

Below is a toy example showing a smart contract implementation along with the JavaScript encode/decode steps.


## Reading Complex Data Types

Let's start with a Solidity smart contract that returns an array of `User` structs. It's a good example as we can see a variety of data types (`struct`, `enum`, `dynamic array` and `uint256`) in one return value.

```{solidity}
pragma solidity ^0.4.24;


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

  function getUsers() public constant returns (User[]) {
    return users;
  }

}
```

Within a `nodejs` console you can compile as follows: 

```
const ethers = require('ethers');
const solc = require('solc');

let contractCode = `SOLIDITY SOURCE HERE`

// Compile the above snippet.
let compiledContract = solc.compile(contractCode);

// And log any errors.
console.log(compiledContract.errors[0]);
```

Compiling the above throws a compiler error:

```
:21:43: TypeError: This type is only supported in the new experimental ABI encoder. Use "pragma experimental ABIEncoderV2;" to enable the feature.
  function getUsers() public constant returns (User[]) {
                                          ^----^
```


Let's do what the message suggests and see what happens. Adding in `pragma experimental ABIEncoderV2;` to the second line.

```
pragma solidity ^0.4.24;
pragma experimental ABIEncoderV2; // Use experimental encoding.

... snip ...
```

<<<<<<< HEAD
No error when compiling this time, instead a only warning that an experimental feature is being used.
=======
No error this time, instead only a warning that an experimental feature is being used.
>>>>>>> 9dc9b034b616b8be866d0d6688d29a50bda1d6c0

```
:2:1: Warning: Experimental features are turned on. Do not use experimental features on live deployments.
pragma experimental ABIEncoderV2; // Use experimental encoding.
^-------------------------------^
```

Next, deploy using a `ganache-cli` instance and call the getter (`getUsers()`).

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
let ABIExample;
let userArray;

sendPromise.then(res => {
    web3Provider.getTransactionReceipt(res.hash).then(res => {

        // JS contract interface.
        ABIExample = new ethers.Contract(res.contractAddress, interface, web3Provider);


        ABIExample.getUsers().then(res => {

            // Log result of calling the contract's `getUsers()` function.
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

... snip ...

  // New method to encode users before returning as bytes.
  function getUsersBytes() public constant returns (bytes) { // Change return type to bytes.

    User[] memory allUsers = users; // Important: load array into memory.

    return abi.encode(allUsers); // Encode prior to return.
  }

... snip ...
```

After recompiling and calling `getUsersBytes()` the following bytes are returned:

```
0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000006f000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000de0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000014d0000000000000000000000000000000000000000000000000000000000000000
```

We can decode those bytes using the `ethers.js` [AbiCoder](https://docs.ethers.io/ethers.js/html/api-advanced.html#abi-coder).

```
let userBytesArray = "0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000006f000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000de0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000014d0000000000000000000000000000000000000000000000000000000000000000";


let usersArrayType = ["tuple(uint256,uint8)[]"];
let decodedData    = ethers.utils.AbiCoder.defaultCoder.decode(usersArrayType, userBytesArray);

console.log(decodedData);

// Or to print nicer decodedData[0].forEach(x => console.log(`id -> ${x[0]} permission -> ${x[1]}`));

// Or if you have the abi the following would work just as well.
usersArrayType = [{"components":[{"name":"id","type":"uint256"},{"name":"permission","type":"uint8"}],"name":"","type":"tuple[]"}]
decodedData    = ethers.utils.AbiCoder.defaultCoder.decode(usersArrayType, userBytesArray);

console.log(decodedData);
```


Using `"tuple(uint256,uint8)[]"` as the type string logs some nested arrays.

```
[ // AbiCoder wraps all of the returned data.

    [ // User array returned from our toy smart contract.

        [ [Object], 2 ],  // `User` struct as an array of two objects.
                          // The first of which is a BigNumber representing `uint256 id`.
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
... snip ...

  // Add a setter to add new users.
  function set(User newUser) public {

    users.push(newUser);
  }

... snip ...
```

Recompile, execute with ` ABIExample.set({ id: 444, permission: 2 }, { gasLimit: 2e5 })`, and log the result of `ABIExample.getUsers()`.


Logging shows an updated array. We've now done a round trip writing a `struct` then reading it back within an `array of structs`.

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

For this last example we'll add a caller contract to show how to pass in `bytes` as an argument and then dispatch the call to our typed `set(User newUser)` function.

```
pragma solidity ^0.4.24;


contract Caller {

    function callExternal(address _to, bytes _data)
        public
        returns (bool success)
    {
        assembly {
            success := call(not(0), _to, 0, add(_data, 0x20), mload(_data), 0, 0)
        }
    }

}
```

Compile, then deploy this new contract.

```
// Compile.
let callerCompiledContract = solc.compile('CALLER CONTRACT CODE');

// Prepare deployment with new interface and bytecode.
let callerInterface = callerCompiledContract.contracts[':Caller'].interface;
let callerBytecode  = callerCompiledContract.contracts[':Caller'].bytecode;
deployTransaction   = ethers.Contract.getDeployTransaction('0x' + callerBytecode, callerInterface);
callerSendPromise   = signer.sendTransaction(Object.assign(deployTransaction, { gas: 6e6 }));
```


The `Caller` contract allows us to pass arbitrary bytes and call out to another smart contract. We'll use this approach to send encoded bytes from our `nodejs` console and have them received as a `User struct` in the ABIExample `set(User newUser)` function.

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

`encodedData` is the following string `'0x000000000000000000000000000000000000000000000000000000000000022b0000000000000000000000000000000000000000000000000000000000000000'`. Perfect if calling out to a fallback function, but in our case we need the function signature prepended. `ethers.js` has us covered again:


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

    // Update state with User(555, Permission.ReadOnly).
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

<<<<<<< HEAD
The combination of `ethers.js` and Solidity's `ABIEncoderV2` opens the door to two means of reading and writing complex data types: using the types directly, using or their `bytes` representation. These useful techniques are used throughout Counterfactual's generalized state channels implementation and are appropriate for a variety of use cases.

Try it out the next time your project can benefit from sending and receiving complex data types.


## Full Example

For a complete example (copy and paste into `nodejs`) see: [ABIExample Gist](https://gist.github.com/NoahMarconi/be4213f8147dea60a1ab30db86760828)
=======
The combination of `ethers.js` and Solidity's `ABIEncoderV2` opens the door to two means of reading and writing complex data types: using the types directly, or using their `bytes` representation. These useful techniques are used throughout [Counterfactual's generalized state channels implementation](https://counterfactual.com/statechannels).
>>>>>>> 9dc9b034b616b8be866d0d6688d29a50bda1d6c0


## Further Reading

For more information check out:

  - [Human-Readable Contract ABIs](https://blog.ricmoo.com/human-readable-contract-abis-in-ethers-js-141902f4d917) by Richard Moore;
  - [Solidity ABIv2: A Foray into the Experimental](https://blog.ricmoo.com/solidity-abiv2-a-foray-into-the-experimental-a6afd3d47185) by Richard Moore;
  - [Api Docs](https://docs.ethers.io/ethers.js/html/api-advanced.html#abi-coder) for ethersjs;
  - [Application Binary Interface](https://solidity.readthedocs.io/en/develop/abi-spec.html#) Solidity docs
  - [ABI Encoding Functions](https://solidity.readthedocs.io/en/develop/units-and-global-variables.html#abi-encoding-functions) Solidity docs
  - Counterfactual's use of abi encoding at [TBA]()
