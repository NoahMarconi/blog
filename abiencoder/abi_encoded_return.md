# Solidity ABI Encoded Return Types

Counterfactual's generalized state channel implementation relies on abi encoding to pass arbitrary data structures between JavaScript client applications and smart contracts. A useful pattern I wanted to learn more about and incorporate into my own projects.

Thank you to Liam Horne for talking me through how all of this works.

## Setup

In the examples below I'll use [ethers.js](https://github.com/ethers-io/ethers.js/) client side and [EthFiddle](https://ethfiddle.com/) for smart contract code snippets.

Follow along by pasting the Solidty code in EthFiddle and by using a nodejs console for the JavaScript examples.


## Background

Solidity's `ABIEncoderV2` allows you to encode arbitrary data types, including structs and nested arrays!

Client side, the abi string `s` can represent any solidity type, with two nuances:

  - `tuple` is used in place of a `struct`
  - `uint256` is used in place of an `enum`

```
s := tuple
     uint256
     bytes32
     string
     uint256[]
     ...
```

For example, a dynamic `uint256` array is represented as `s = uint256[]` or a more interesting example where a `struct { uint256 id; string name }` is represented as `s = tuple(uint256,string)`.

[ethers.js](https://github.com/ethers-io/ethers.js/) has built in encoding and decoding to read abi encoded byte arrays returned from smart contract in addition encoding data before passing in structs as arguments to smart contract functions.

Below is a toy example showing an smart contract implementation along with the JavaScript encode/decode steps.


## Reading Encoded Data

Lets start with a Solidty smart contract that returns an array of `User` structs. It's a good example as we can see a variety of data types (`struct`, `enmum`, `dynamic array` and `uint256`) in one encoded `bytes array`.

```{sol}
pragma solidity ^0.4.24;


contract ABIExample {

  uint256 value;

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

}
```


Gives us an error message:

```
:27:43: TypeError: This type is only supported in the new experimental ABI encoder. Use "pragma experimental ABIEncoderV2;" to enable the feature.
  function get() public constant returns (User[3]) {
```


Let's do what the message suggests and see what happens. Adding in `pragma experimental ABIEncoderV2;` and encoding with `abi.encode(users);` before returning a `bytes` array.

```
pragma solidity ^0.4.24;
pragma experimental ABIEncoderV2; // Use experimental encoding.

contract ABIExample {

  uint256 value;

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

}
```


Deploying and run the getter (`get()`) returns the following bytes:

```
0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001
```

In a `nodejs` console, decode with ethers.js:

```
const ethers = require('ethers');

const usersArrayType = ["tuple(uint256,uint256)[]"];

const encodedData = "0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001";

const decodedData = ethers.utils.AbiCoder.defaultCoder.decode(usersArrayType, encodedData);
```


This returns some nested arrays.

```
[ // AbiCoder wraps all of the returned data.

    [ // User array returned from our toy smart contract.

        [ [Object], [Object] ],  // `User` struct as an array of two BigNumber objects.
                                 // The first of which is `uint256 id`.
                                 // While the second is `Permission permission` represented as a `uint256` as is the case for Solidity enums.


        [ [Object], [Object] ],  // `User` struct
        [ [Object], [Object] ]   // `User` struct
    ]
]
```

Awesome, we can read arbitrary data structures returned from a Solidity smart contract, including arrays, structs, and emums.


Let's try the other direction and pass in encoded data as an argument.


## Writing Encoded data

First update the smart contract with a setter `set`:

```
pragma solidity ^0.4.24;
pragma experimental ABIEncoderV2;


contract ABIExample {

  uint256 value;

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

}
```

Again from a `nodejs` console encode some data with:

```
const ethers = require('ethers');

const userType = ["tuple(uint256,uint256)"];
const rawData = [ // AbiCoder wraps all of the data.

    [ // User tuple.
      111, // uint256 representing `id`.
      2,   // uint256 representing `permission`.
    ]

];

const encodedData =
ethers.utils.AbiCoder.defaultCoder.encode(userType, rawData);
```

`encodedData` is the followin string `'0x000000000000000000000000000000000000000000000000000000000000006f0000000000000000000000000000000000000000000000000000000000000002'` which can be passed in as an argument to the `set` function.



## Further Reading

For more information check out:

  - [Human-Readable Contract ABIs](https://blog.ricmoo.com/human-readable-contract-abis-in-ethers-js-141902f4d917) by Richard Moore; 
  - [Api Docs](https://docs.ethers.io/ethers.js/html/api-advanced.html#abi-coder) for ethersjs;
  - [Application Binary Interface](https://solidity.readthedocs.io/en/develop/abi-spec.html#) Solidity docs.
