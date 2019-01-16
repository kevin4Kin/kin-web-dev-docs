---
id: ios-sdk
title: iOS SDK
---
#  Kin SDK for iOS

With the Kin SDK for iOS you can give your users fun ways to earn and spend Kin in your app, and help us build a whole new digital world.

Kin SDK for iOS is implemented as a library that can be incorporated into your code. If you’re just getting started with Kin ecosystem we suggest you spend a few minutes reading this [introduction](https://kinecosystem.github.io/kin-ecosystem-sdk-docs/docs/intro.html).

## Installation

### CocoaPods

Add the following to your `Podfile`.
```
pod 'KinSDK', '~> 0.8.0’
```

See the latest releases at [github.com/kinecosystem/kin-sdk-ios/releases](https://github.com/kinecosystem/kin-sdk-ios/releases)

The main repository is at [github.com/kinecosystem/kin-sdk-ios](https://github.com/kinecosystem/kin-sdk-ios)

### Sub-project

1. Clone this repo (as a submodule or in a different directory, it's up to you).
```
git clone --recursive https://github.com/kinecosystem/kin-sdk-ios
```
2. Drag `KinSDK.xcodeproj` as a subproject.
3. In your main `.xcodeproj` file, select the desired target(s).
4. Go to **Build Phases**, expand Target Dependencies, and add `KinSDK`.
5. In Swift, `import KinSDK` and you are good to go! (We haven't yet tested Objective-C.)


## API Overview

Adding Kin features to your app means using the SDK to:

- Access the Kin blockchain
- Manage Kin accounts
- Execute transactions against Kin accounts

The two main classes of the SDK are `KinClient` and `KinAccount`.

### KinClient

iOS apps that allow users to earn, spend, and manage Kin are considered clients in the Kin architecture. The following statement creates a `KinClient` object which includes methods to manage accounts on a Kin blockchain.

A `KinClient` object is initialized for a specific Kin environment and network and it manages `KinAccount` objects for that environment.

```swift
KinClient(with: URL, network: Network, appId: AppId)
```

- `with` The URL of the Kin blockchain
- `network` You declare which Kin blockchain network you want to work with using the pre-defined enum value `Network.mainNet` or `Network.playground`.
- `appId` must be a 4-character string which identifies your application. It must contain only digits and upper and/or lower case letters.

For instance, to initialize a Kin Client to use the Playground network.
```swift
let url = "http://horizon-testnet.kininfrastructure.com"
guard let providerUrl = URL(string: url) else {
    return nil
}
do {
    let appId = try AppId("test")
    let kinClient = KinClient(with: providerUrl, network: .testNet, appId: appId)
} catch let error {
    print("Error \(error)")
}
```


### Account Management

With a `KinClient` object, it is possible to add a new account, delete or import an account and access the list of accounts.

```swift
var accounts: KinAccounts

func addAccount() throws -> KinAccount

func deleteAccount(at index: Int) throws

func importAccount(_ jsonString: String, passphrase: String) throws -> KinAccount

```

#### Accessing accounts

The list of Kin accounts of a KinClient are available via its attribute `accounts`.

```swift
var accounts: KinAccounts
```

To get the first account: `let account = kinClient.accounts.first`

or `let account = kinClient.accounts[0]`

To print the public address of each stored account:

```swift
kinClient.accounts.forEach { account in
    print("--> \(account?.publicAddress)")
}
```

#### Creating an account

Once the `KinClient` object is initialized, you need at least one `KinAccount` object to use the features from the Kin ecosystem.
Every account created or imported with the `KinClient` is also stored securely locally by `KinClient`.

To create an account:

```swift
do {
    let account = try kinClient.addAccount()
}
catch let error {
    print("Error creating an account \(error)")
}
```

#### Deleting an account

Deleting an account means removing the account data stored locally.

:warning: If the account has not been backed up previously by exporting it, it will be lost and its kins inaccessible.

Deleting the first account
```swift
do {
    try kinClient.deleteAccount(at: 0)
}
catch let error {
    print("Could not delete account \(error)")
}
```

#### Importing an account

Importing the account adds the account to the KinClient's list of managed accounts.

```swift
let json = "{\"pkey\":\"GBKN6ATMTFQOKDIJOUUP6G7A7GFAQ6XHJBV3HJ5QAQH3NCUQNXISH3AR\"," +
        "\"seed\":\"61381366f4af2c57c55e2c23411e26d5a85eae18a9e1c91e01fa7e9967f3d2b9e0f8a412c9147d7abe1529adcaef21a84ebc266da0a86b0f6a9adf2b3007652811ceaa4156834620\",\"salt\":\"a663ec77c54bb2c9efdffabb5685cda9\"}"
do {
    try kinClient.importAccount(json, passphrase: "a-secret-passphrase-here")
}
catch let error {
    print("Error importing the account \(error)")
}
```

### Kin account creation on the blockchain

If you have just created the account locally with, you still need to create that account on the blockchain in order to query its status, balance or exchange transactions.

This creates the given account on the playground blockchain by accessing a specific URL.

```swift
/**
Create the given stored account on the playground blockchain.
*/
func createPlaygroundAccountOnBlockchain(account: KinAccount, completionHandler: @escaping (([String: Any]?) -> ())) {
    // Playground blockchain URL for account creation
    let createUrlString = "http://friendbot-testnet.kininfrastructure.com?addr=\(account.publicAddress)"

    guard let createUrl = URL(string: createUrlString) else {
        return
    }
    let request = URLRequest(url: createUrl)
    let task = URLSession.shared.dataTask(with: request) { (data: Data?, response: URLResponse?, error: Error?) in
        if let error = error {
            print("Account creation on playground blockchain failed with error: \(error)")
            completionHandler(nil)
            return
        }
        guard let data = data,
              let json = try? JSONSerialization.jsonObject(with: data, options: []),
              let result = json as? [String: Any] else {
            print("Account creation on playground blockchain failed with no parsable JSON")
            completionHandler(nil)
            return
        }
        // check if there's a bad status
        guard result["status"] == nil else {
            print("Error status \(result)")
            completionHandler(nil)
            return
        }
        print("Account creation on playground blockchain was successful with response data: \(result)")
        completionHandler(result)
    }

    task.resume()
}
```

## Using a Kin account

### Kin account identification

A Kin account is identified via its public address, retrieved with `.publicAddress`.

```swift
var publicAddress: String
```

Before an account can be used on the configured network, it must be funded with the KIN currency. This step must be performed by a service, and is outside the scope of this SDK.

### Kin account status

The current account status on the blockchain is queried with `status`.

```swift
func status(completion: @escaping (AccountStatus?, Error?) -> Void)
```

### Kin balance

To retrieve the account's current balance in Kin:

```swift
func balance(completion: @escaping BalanceCompletion)
```

- `completion` callback method called with the `Kin`, `Error`

### Transactions: sending Kins to another account

To transfer Kin to another account, you need the public address of the account to which you want to transfer Kin.

By default, your user will need to spend **Fee** to transfer Kin or process any other blockchain transaction.
Fee for individual transactions are trivial (1 Fee = 10<sup>-5</sup> Kin).
Some apps can be added to the Kin whitelist, a set of pre-approved apps whose users will not be charged Fee to execute transactions.
If your app is whitelisted then refer to [TODO]()

#### Send Kins with a transaction (not whitelisted)

Transactions are executed on the Kin blockchain in a two-step process:

- **Build** the transaction which includes the calculation of the transaction hash. The transaction hash is used as an ID and is necessary to query the status of the transaction.
- **Send** the transaction for its execution on the blockchain.

##### Build the transaction

Build the transaction with:

```swift
func generateTransaction(to recipient: String,
                             kin: Kin,
                             memo: String?,
                             fee: Stroop,
                             completion: @escaping GenerateTransactionCompletion)
```

- `recipient` is the recipient's public address.
- `kin` is the amount of Kin to be sent.
- `memo` is an optional string, up-to 28 bytes in length, included on the transaction record. A typical usage is to include an order number..
- `fee` The fee in `Stroop`s used if the transaction is not whitelisted.
- `completion` callback method called with the `TransactionEnvelope` and `Error`.

##### Send the transaction

```swift
func sendTransaction(_ transactionEnvelope: TransactionEnvelope,
                       completion: @escaping SendTransactionCompletion)
```

- `transactionEnvelope`: The `TransactionEnvelope` object to send.
- `completion`: A completion callback method with the `TransactionId` or `Error`.

#### Send Kins with a whitelist transaction (Fee waived)

Transactions are executed on the Kin blockchain in a two-step process:

- **Build** the transaction which includes the calculation of the transaction hash. The transaction hash is used as an ID and is necessary to query the status of the transaction.
- Create a `WhitelistEnvelope`
- **Send** the `WhitelistEnvelope` to a whitelist service which will sign and return an new `TransactionEnvelope`.
- **Send** the transaction for its execution on the blockchain.

Pass the returned `TransactionEnvelope` to the `WhitelistEnvelope`.

```swift
init(transactionEnvelope: TransactionEnvelope, networkId: Network.Id)
```

The `WhitelistEnvelope` should be passed to a server for signing. The server response should be a  `TransactionEnvelope` with a second signature, which can then be sent.

```swift
func sendTransaction(_ transactionEnvelope: TransactionEnvelope,
                       completion: @escaping SendTransactionCompletion)
```

## Miscellaneous

### Asynchronous programming styles

Several asynchronous methods of `KinClient` and `KinAccount` come in two styles:

- **Callback parameter**: a completion handler is passed as a parameter to the method.
- **Promise**: a `Promise` object is returned by the method. The `Promise` class comes with the Kin Util library which is included in the SDK. Promises are a way to simplify asynchronous programming and make asynchronous method calls composable.

`KinAccount`'s status using the completion parameter:
```swift
func status(completion: @escaping (AccountStatus?, Error?) -> Void)
```

`KinAccount`'s status using promises:

```swift
func status() -> Promise<AccountStatus>
```

A more complete example:

```swift
account.status()
    .then { (status: AccountStatus) in
        print("Account's status is: \(status)")
    }
    .error { (error: Error) in
        print("Error getting the account's status: \(error)")
    }
```

### `KinAccount` watcher objects

To be notified of changes on a `KinAccount`, you can also use the watcher objects. The watcher object will emit an event whenever a change occurred.

They are available for:

- Account creation notification
```swift
func watchCreation() throws -> Promise<Void>
```
- Balance change notifications
```swift
func watchBalance(_ balance: Kin?) throws -> BalanceWatch
```
- Payment notifications
```swift
func watchPayments(cursor: String?) throws -> PaymentWatch
```

### Error handling

Kin SDK wraps errors in an operation-specific error for each method of `KinAccount`.
The underlying error is the actual cause of failure.

### Common errors

`StellarError.missingAccount`: The account does not exist on the Stellar network.
You must create the account by issuing an operation with `KinAccount.publicAddress` as the destination.
This is done using an app-specific service, and is outside the scope of this SDK.

## Contributing

Please review our [CONTRIBUTING.md](CONTRIBUTING.md) guide before opening issues and pull requests.

## License

This repository is licensed under the [MIT license](LICENSE.md).
