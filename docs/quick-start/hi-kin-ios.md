---
id: hi-kin-ios
title: Hello World with the iOS SDK
---

As you probably expect from the name, this article provides a quick code walk-through demonstrating all the key concepts you need to create Android clients that allow your users to earn, spend, and manage Kin. The entire app is contained in a single Swift file for expediency and simplicity, we don't recommend this for real apps.

## Install the Kin SDK

In this tutorial we will use Xcode, we last tested it with Xcode 10.1.

Let’s start by installing the [Kin SDK for iOS](https://github.com/kinecosystem/kin-sdk-ios) in your iOS app.

Create a new project in Xcode and select a **Single View App**, this will create an empty project with a basic `ViewController.swift` file. This is where most of our work will be.

Add `pod 'KinSDK', '~> 0.8.0’` to your Podfile then run `pod install`. We use Cocoapods for convenience, if you are not familiar visit [Cocoapods.org](https://cocoapods.org/) and make sure you execute `pod init`.

Make sure you have the latest release of the SDK by checking the release page [github
.com/kinecosystem/kin-sdk-ios/releases](https://github.com/kinecosystem/kin-sdk-ios/releases).

Close the Xcode project and open it again. When you open Xcode choose to `open another project` and choose the `.xcworkspace` file instead of the `.xcodeproj` file. As Xcode reopens you will have your project files and on the left you'll notice a new section called `Pods` on the bottom, `KinSDK` will be listed in the Pods sub-folder.


## Environments

In this tutorial, we will use the Playground environment. The Playground environment allows to create accounts, fund them and send Kin between accounts.
The Kin Playground environment is dedicated to Kin development where you can develop and test your Kin integration.

Transition to the Production environment when your app is ready and tested. The Playground environment is meant to be as close as possible to the Production environment with the exception of funding accounts. In the production environment, an account can only be funded by receiving Kin from another account.

## Get started with the Kin SDK

### Import the SDK in your swift file

Open `ViewController.swift` and add import the Kin SDK:

```swift
import KinSDK
```

### Initialize the Kin client

Let's first create a few functions that will be helpful to connect to the Kin blockchain, manage accounts and send Kin. Below is a code snippet that will connect to the Playground environment.

In your view controller, add this function.

```swift
/**
Initializes the Kin Client with the playground environment.
*/
func initializeKinClientOnPlaygroundNetwork() -> KinClient? {
    let url = "http://horizon-testnet.kininfrastructure.com"
    guard let providerUrl = URL(string: url) else { return nil }

    do {
        let appId = try AppId("test")
        return KinClient(with: providerUrl, network: .testNet, appId: appId)
    }
    catch let error {
        print("Error \(error)")
    }
    return nil
}
```

`KinClient` expects three parameters, the first one is the URL to the Horizon servers, the second is the environment and the third is the `appId`. appId is a 4 character string, provided by Kin and associated uniquely with each app. In the test environment you can use any valid appId.

And add a call to initialize the client in `viewDidLoad()`:

```swift
let kinClient: KinClient! = initializeKinClientOnPlaygroundNetwork()
```

### Info.plist

Let’s quickly configure the `Info.plist` file to allow HTTP requests:
```swift
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>  
</dict>
```

## Manage accounts

`kinClient` can manage multiple accounts. In this tutorial, we will only be using one.

Creating an account is a two step process in Kin. The first step is to create a key pair locally and the second is to actually create the account on the public blockchain.

### Check and create an account

If an account is available from the local store, it can be loaded with `let account = kinClient.accounts.first`.

In view controller we can create a new function for this:
```swift
func getFirstAccount(kinClient: KinClient) -> KinAccount? {
    return kinClient.accounts.first
}
```

If no account is available it's easy to create a new one:
```swift
func createLocalAccount(kinClient: KinClient) -> KinAccount? {
    do {
        let account = try kinClient.addAccount()
        return account
    }
    catch let error {
        print("Error creating an account \(error)")
    }
    return nil
}
```

Once the account has been added, it will be stored locally and it can be retrieved it the next time the app runs.

### Account identification

Every account on the Kin blockchain is made of a key pair: public address and private key. Public address is often called public key. The private key is often also called private seed.
The public address is especially useful because this is what is used to identify an account publicly and send or receive Kin.
In Swift the account's public address can retrieved easily with `.publicAddress`.

```swift
account.publicAddress
```

### Delete a stored account

If you want to make sure there are no account stored locally, delete the accounts from the `KinClient`.

:warning: If the account has not been backed up previously by exporting the key pair, it will be lost and its Kin will be inaccessible.

```swift
/**
Delete the first stored account of the client.
*/
func deleteFirstAccount(kinClient: KinClient) {
    do {
        try kinClient.deleteAccount(at: 0)
        print("First stored account deleted!")
    }
    catch let error {
        print("Could not delete account \(error)")
    }
}
```

### Create the account on the blockchain

If you have just created the account locally with `kinClient.addAccount()`, you still need to create that account on
the blockchain. Only once the account is created on the public blockchain it will be possible to query its status, balance and execute transactions.

As a reminder new accounts created in the Playground environment are automatically funded by `friendbot` a service not available in production.

```swift
func createAccountOnPlaygroundBlockchain(account: KinAccount,
                                         completionHandler: @escaping (([String: Any]?) -> ())) {
    // Playground blockchain URL for account creation and funding
    let createUrlString = "http://friendbot-testnet.kininfrastructure.com?addr=\(account.publicAddress)"

    guard let createUrl = URL(string: createUrlString) else { return }

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

### Putting it together

Now that the basic functions are present it is possible to add some basic variables and calls to `viewDidLoad`. As stated before, this is not the preferred method for a production app, but good enough for our first example.

```swift
// Initialize the Kin client on the playground blockchain
let kinClient: KinClient! = initializeKinClientOnPlaygroundNetwork()

// The Kin Account we're going to create and use
var account: KinAccount! = nil

// This deletes the stored account so that we can recreate one - uncomment to delete
// deleteFirstAccount(kinClient: kinClient)

// Get any user account stored locally
if let existingAccount = getFirstAccount(kinClient: kinClient) {
    print("Current account with address \(existingAccount.publicAddress)")
    account = existingAccount
}
// or else create a new account
else if let newAccount = createLocalAccount(kinClient: kinClient) {
    print("Created account with address \(newAccount.publicAddress)")
    account = newAccount
}
```


### Get account status

An account’s status is either `.created` or `.notCreated`. If an account only exists locally after a call to `kinClient.addAccount()`, its status will still be `.notCreated`. The account will change status only after it is created on the blockchain as described earlier.

To get an account status, an asynchronous call is performed with `func status(completion: @escaping (AccountStatus?,
Error?) -> Void)`

```swift
account.status { (status: AccountStatus?, error: Error?) in
    guard let status = status else { return }
    switch status {
    case .notCreated:
        // The account has just been created locally. It needs to be added to the blockchain
        // We create that account on the playground blockchain
        ()
    case .created:
        // The account exists on the blockchain - We can send transactions (provided there are enough Kin)
        ()
    }
}
```

### Get balance

The balance gives you the number of Kin available on your account.
To get the balance, an asynchronous call is performed with `func balance(completion: @escaping BalanceCompletion)`

```swift
/**
Get the balance using the method with callback.
*/
func getBalance(forAccount account: KinAccount, completionHandler: ((Kin?) -> ())?) {
    account.balance { (balance: Kin?, error: Error?) in
        if error != nil || balance == nil {
            print("Error getting the balance")
            if let error = error { print("with error: \(error)") }
            completionHandler?(nil)
            return
        }
        completionHandler?(balance!)
    }
}
```


## Transfer Kin

Provided the account has been created and funded in the Playground environment, it is now possible to send Kin to another account. This process happens in two steps:
- `generateTransaction` builds the transaction request and returns a `TransactionEnvelope` object if successful
- `sendTransaction` sends the request and returns a `TransactionId` if successful

The following snippet generates the transaction request and then sends it.

Every transaction costs a fee to execute on the blockchain network.
Fee for individual transactions are trivial (1 Kin = 10E5 Fee).

A whitelist of pre-approved Kin apps have their fee waived. See [Send kin with a whitelist transaction](#send-kin-with-a-whitelist-transaction) for an example.


```swift
/**
Sends a transaction to the given account.
*/
func sendTransaction(fromAccount account: KinAccount,
                     toAddress address: String,
                     kinAmount kin: Kin,
                     memo: String?,
                     fee: Stroop,
                     completionHandler: ((String?) -> ())?) {
    // Get a transaction envelope object
    account.generateTransaction(to: address, kin: kin, memo: memo, fee: fee) { (envelope, error) in
        if error != nil || envelope == nil {
            print("Could not generate the transaction")
            if let error = error { print("with error: \(error)")}
            completionHandler?(nil)
            return
        }

        // Sends the transaction
        account.sendTransaction(envelope!) { (txId, error) in
            if error != nil || txId == nil {
                print("Error send transaction")
                if let error = error {
                    print("with error: \(error)")
                }
                completionHandler?(nil)
                return
            }
            print("Transaction was sent successfully for \(kin) Kins - id: \(txId!)")
            completionHandler?(txId!)
        }
    }
}
```

`generateTransaction` expects three parameters: the destination public address, the amount of Kin, a memo (can be empty) and the amount of Fee.
The `memo` parameter can be up to 21 characters and developers are free to enter any information that is useful to them, for instance to specify an order number. The appId is automatically added to the memo field.

## Send Kin with a whitelist transaction

A whitelist of pre-approved Kin apps have their fee waived. When sending
transactions for an app that is whitelisted, an additional step is
necessary to sign the transaction envelope with a whitelist service.

The steps are thus the following:

- Build the transaction request, which returns a `TransactionEnvelope` object if successful
- `WhitelistEnvelope` creates a whitelist envelope passing the generated envelope and the network Id of the client
- `signWhitelistTransaction` signs the whitelist envelope by sending it to a specific whitelist service. Another `TransactionEnvelope` object is returned if the call is successful
- Send the request, which returns a `TransactionId` if successful

```swift
/**
Sends Kin to the given public address
*/
func sendWhitelistTransaction(fromAccount account: KinAccount,
                              toAddress address: String,
                              kinAmount kin: Kin,
                              memo: String?,
                              fee: Stroop,
                              completionHandler: ((String?) -> ())?) {
    // Get a transaction envelope object
    account.generateTransaction(to: address, kin: kin, memo: memo, fee: fee) { (envelope, error) in
        if error != nil || envelope == nil {
            print("Could not generate the transaction")
            if let error = error {
                print("with error: \(error)")
            }
            completionHandler?(nil)
            return
        }

        let networkId = Network.testNet.id
        let whitelistEnvelope = WhitelistEnvelope(transactionEnvelope: envelope!, networkId: networkId)

        self.signWhitelistTransaction(whitelistServiceUrl: "WHITELIST_SERVICE_URL",
                envelope: whitelistEnvelope) { (signedEnvelope, error) in
            if error != nil || signedEnvelope == nil {
                print("Error whitelisting the envelope")
                if let error = error {
                    print("with error: \(error)")
                }
                completionHandler?(nil)
                return
            }
            // send the whitelist transaction
            account.sendTransaction(signedEnvelope!) { (txId, error) in
                if error != nil || txId == nil {
                    print("Error send whitelist transaction")
                    if let error = error {
                        print("with error: \(error)")
                    }
                    completionHandler?(nil)
                    return
                }
                print("Whitelist transaction was sent successfully for \(kin) Kins - id: \(txId!)")
                completionHandler?(txId!)
            }

        }
    }
}
```

Below is the snippet that signs the transaction. Note that the example uses `whitelistServiceUrl`, but you are required to provide that. To implement the service you will have to setup your own back-end service using the Kind SDK for Python. See [Transferring Kin to another account using whitelist service](../documentation/python-sdk#transferring-kin-to-another-account-using-whitelist-service) for more information on how to whitelist a transaction.

```swift
/**
Sign the given transaction envelope so that the transaction can be submitted with the fee waived
*/
func signWhitelistTransaction(whitelistServiceUrl: String,
                              envelope: WhitelistEnvelope,
                              completionHandler: @escaping ((TransactionEnvelope?, Error?) -> ())) {
    let whitelistingUrl = URL(string: whitelistServiceUrl)!

    var request = URLRequest(url: whitelistingUrl)
    request.addValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpMethod = "POST"
    request.httpBody = try? JSONEncoder().encode(envelope)

    let task = URLSession.shared.dataTask(with: request) { (data, response, error) in
        do {
            let envelope = try TransactionEnvelope.decodeResponse(data: data, error: error)
            completionHandler(envelope, nil)
        }
        catch {
            completionHandler(nil, error)
        }
    }
    task.resume()
}
```

## Export an account

Once an account has been added to the `KinClient`, its information is stored securely locally. If you want to use
that same account for instance in a different app, you need to export it to a JSON string, encoded with a passphrase.
 To import it later you just need the JSON and the same passphrase.

```swift
// Exports the account to a JSON string with the given passphrase. You can later import the account with the
// same passphrase and the JSON string.
let json = try! account.export(passphrase: “a-secret-passphrase-here”)
print("Exported JSON \n\(json)\n")
```

The resulting JSON looks like this:
```json
{"pkey":"GA5J72WTDFR7E2IJPAOG65V5MP3QDQCVVNWIGXXLSYVTQMY7RZAYV3EM",
"seed":"abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234567",
"salt":"aafca71bc57d1065198b88281425f22c”}
```

`pkey` is the public key of the account.

The same JSON can be used with `importAccount`, see [Importing an account](../documentation/ios-sdk#importing-an-account) for more details.

The combination of `account.export` and `KinClient.importAccount` is ideal to create a backup and restore functionality in your app or to allow users to import an existing wallet in a newly installed (or upgraded) app.

## Conclusions

This tutorial should have helped you get started with the Kin SDK for iOS. Other topics not covered here are:

- Developing your own back-end server to support your client apps with the Kin SDK for Python
- Transition to the production environment, e.g. get an `appId` for your app

## Downloads

See all the code for this tutorial in [ViewController.swift](ViewController.swift)
