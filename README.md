# WebThree

# This fork is now maintained at [devopsdao/webthree](https://github.com/devopsdao/webthree) repository.


[![pub package](https://img.shields.io/pub/v/webthree)](https://pub.dartlang.org/packages/webthree)
[![likes](https://img.shields.io/pub/likes/webthree)](https://pub.dartlang.org/packages/webthree/score)
[![points](https://img.shields.io/pub/points/webthree)](https://pub.dartlang.org/packages/webthree/score)
[![popularity](https://img.shields.io/pub/popularity/webthree)](https://pub.dartlang.org/packages/webthree/score)
[![license](https://img.shields.io/github/license/makinghappen/web3dart)](https://pub.dartlang.org/packages/web3dart)
[![stars](https://img.shields.io/github/stars/simolus3/web3dart)](https://github.com/makinghappen/web3dart/stargazers)
[![forks](https://img.shields.io/github/forks/simolus3/web3dart)](https://github.com/makinghappen/web3dart/network/members)
[![sdk version](https://badgen.net/pub/sdk-version/web3dart)](https://pub.dartlang.org/packages/web3dart)

WebThree - a web3 library for dart that allows you to interact with a local or remote ethereum node using HTTP or WebSocket. Suports custom credentials providers like WalletConnect and Metamask.

Fork of original [web3dart](https://github.com/simolus3/web3dart) 2.3.5 by simolus3, incorporating all changes from other forks.



## Features

- Connect to an Ethereum node with the rpc-api, call common methods
- Send signed Ethereum transactions
- Generate private keys, setup new Ethereum addresses
- Call functions on smart contracts and listen for contract events
- Code generation based on smart contract ABI for easier interaction

### TODO

- Encode all supported solidity types, although only (u)fixed,
  which are not commonly used, are not supported at the moment.
- tests coverage
- wallet connect example

## Usage

### Credentials and Wallets

In order to send transactions on the Ethereum network, some credentials
are required. The library supports raw private keys and v3 wallet files.

```dart
import 'dart:math'; //used for the random number generator

import 'package:webthree/webthree.dart';
// You can create Credentials from private keys
Credentials fromHex = EthPrivateKey.fromHex("c87509a[...]dc0d3");

// Or generate a new key randomly
var rng = Random.secure();
Credentials random = EthPrivateKey.createRandom(rng);

// In either way, the library can derive the public key and the address
// from a private key:
var address = await credentials.extractAddress();
print(address.hex);
```

Another way to obtain `Credentials` which the library uses to sign
transactions is the usage of a wallet file. Wallets store a private
key securely and require a password to unlock. The library has experimental
support for version 3 wallets commonly generated by other Ethereum clients:

```dart
import 'dart:io';
import 'package:webthree/webthree.dart';

String content = File("wallet.json").readAsStringSync();
Wallet wallet = Wallet.fromJson(content, "testpassword");

Credentials unlocked = wallet.privateKey;
// You can now use these credentials to sign transactions or messages
```

You can also create Wallet files with this library. To do so, you first need
the private key you want to encrypt and a desired password. Then, create
your wallet with

```dart
Wallet wallet = Wallet.createNew(credentials, "password", random);
print(wallet.toJson());
```

You can also write `wallet.toJson()` into a file which you can later open
with [MyEtherWallet](https://www.myetherwallet.com/#view-wallet-info)
(select Keystore / JSON File) or other Ethereum clients like geth.

#### Custom credentials

If you want to integrate `webthree` with other wallet providers, you can implement
`Credentials` and override the appropriate methods.

### Connecting to an RPC server

The library won't send signed transactions to miners itself. Instead,
it relies on an RPC client to do that. You can use a public RPC API like
[infura](https://infura.io/), setup your own using [geth](https://github.com/ethereum/go-ethereum/wiki/geth)
or, if you just want to test things out, use a private testnet with
[truffle](https://www.trufflesuite.com/) and [ganache](https://www.trufflesuite.com/ganache). All these options will give you
an RPC endpoint to which the library can connect.

```dart
import 'package:http/http.dart'; //You can also import the browser version
import 'package:webthree/webthree.dart';

var apiUrl = "http://localhost:7545"; //Replace with your API

var httpClient = Client();
var ethClient = Web3Client(apiUrl, httpClient);

var credentials = ethClient.credentialsFromPrivateKey("0x...");

// You can now call rpc methods. This one will query the amount of Ether you own
EtherAmount balance = ethClient.getBalance(credentials.address);
print(balance.getValueInUnit(EtherUnit.ether));
```

## Sending transactions

Of course, this library supports creating, signing and sending Ethereum
transactions:

```dart
import 'package:webthree/webthree.dart';

/// [...], you need to specify the url and your client, see example above
var ethClient = Web3Client(apiUrl, httpClient);

var credentials = ethClient.credentialsFromPrivateKey("0x...");

await client.sendTransaction(
  credentials,
  Transaction(
    to: EthereumAddress.fromHex('0xC91...3706'),
    gasPrice: EtherAmount.inWei(BigInt.one),
    maxGas: 100000,
    value: EtherAmount.fromUnitAndValue(EtherUnit.ether, 1),
  ),
);
```

Missing data, like the gas price, the sender and a transaction nonce will be
obtained from the connected node when not explicitly specified. If you only need
the signed transaction but don't intend to send it, you can use
`client.signTransaction`.

## Metamask Example

```dart
 import 'dart:convert';
 import 'dart:html';
 import 'dart:typed_data';

 conditionally import dependencies in order to support web and other platform builds from a single codebase
 import 'package:js/js.dart'
     if (dart.library.io) 'package:webthree/lib/src/browser/js-stub.dart'
     if (dart.library.js) 'package:js/js.dart';
 import 'package:webthree/browser.dart'
     if (dart.library.io) 'package:webthree/lib/src/browser/dart_wrappers_stub.dart'
     if (dart.library.js) 'package:webthree/browser.dart';
 import 'package:webthree/webthree.dart';

 @JS()
 @anonymous
 class JSrawRequestParams {
   external String get chainId;

    Must have an unnamed factory constructor with named arguments.
   external factory JSrawRequestParams({String chainId});
 }

 Future<void> main() async {
   final eth = window.ethereum;
   if (eth == null) {
     print('MetaMask is not available');
     return;
   }

   final client = Web3Client.custom(eth.asRpcService());
   final credentials = await eth.requestAccount();

   print('Using ${credentials.address}');
   print('Client is listening: ${await client.isListeningForNetwork()}');

   final message = Uint8List.fromList(utf8.encode('Hello from webthree'));
   final signature = await credentials.signPersonalMessage(message);
   print('Signature: ${base64.encode(signature)}');

   await eth.rawRequest('wallet_switchEthereumChain',
       params: [JSrawRequestParams(chainId: '0x507')]);
   final String chainIDHex = await eth.rawRequest('eth_chainId') as String;
   final chainID = int.parse(chainIDHex);
   print('chainID: $chainID');
 }

```

### Smart contracts

The library can parse the abi of a smart contract and send data to it. It can also
listen for events emitted by smart contracts. See [this file](https://github.com/makinghappen/webthree/blob/development/example/contracts.dart)
for an example.

### Dart Code Generator

By using [Dart's build system](https://github.com/dart-lang/build/), webthree can
generate Dart code to easily access smart contracts.

To use this feature, put a contract abi json somewhere into `lib/`.
The filename has to end with `.abi.json`.
Then, add a `dev_dependency` on the `build_runner` package and run

```
dart run build_runner build
```

You'll now find a `.g.dart` file containing code to interact with the contract.

#### Optional: Ignore naming suggestions for generated files

If importing contract ABIs with function names that don't follow dart's naming conventions, the dart analyzer will (by default) be unhappy about it, and show warnings.
This can be mitigated by excluding all the generated files from being analyzed.  
Note that this has the side effect of suppressing serious errors as well, should there exist any. (There shouldn't as these files are automatically generated).

Create a file named `analysis_options.yaml` in the root directory of your project:

```dart
analyzer:
  excluding: 
    - '**/*.g.dart'
```

See [Customizing static analysis](https://dart.dev/guides/language/analysis-options) for advanced options.

## Feature requests and bugs

Please file feature requests and bugs at the [issue tracker][tracker].
If you want to contribute to this library, please submit a Pull Request.

[tracker]: https://github.com/makinghappen/webthree/issues/new
