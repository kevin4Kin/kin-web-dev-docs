---
id: hi-kin-ios
title: Hello World with the iOS SDK
---

# Hello World with the iOS SDK

This tutorial will help you get started with the Kin SDK for iOS.

If you are reading this, you are already interested in cryptocurrencies and the blockchain, and you probably know
that using the Kin ecosystem in your product is going to open new and fun ways for your users to cooperate and use
your product.
Get the full introduction about the Kin ecosystem [here](https://kinecosystem.github.io/kin-ecosystem-sdk-docs/docs/intro.html).

See all the code for this tutorial in [ViewController.swift](ViewController.swift)

## Install the Kin SDK

Let’s start by installing the [Kin SDK for iOS](https://github.com/kinecosystem/kin-sdk-ios) in your iOS app.

Add  `pod 'KinSDK', '~> 0.8.0’`  to your Podfile then run `pod install`. We use Cocoapods for convenience, if you are not familiar visit [Cocoapods.org](https://cocoapods.org/).

Make sure you have the latest release of the SDK by checking the release page [github
.com/kinecosystem/kin-sdk-ios/releases](https://github.com/kinecosystem/kin-sdk-ios/releases).

If you ran `pod install` for the first time, close the`.xcproject` and open the `.xcworkspace` project.


## Environments

In this tutorial, we are using the Playground environment. The Playground environment allows us to create accounts, fund them and send Kin between accounts.
The Kin Playground environment is dedicated to Kin development where you can develop and test your Kin integration with up to 1000 users.

Transition to the main environment when your app is ready for production. The Playground environment is meant to be as close as possible to the Production environment with the exception of funding accounts. In the main environment, an account can only be funded by receiving Kin from another account.

## Getting started with the Kin SDK

### Import the SDK in your swift file.

Add `import KinSDK` at the top of your file.

### Initialize the Kin client

The SDK client manages accounts. As we are using the playground environment, the credentials used for the initialization
are fixed.

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

And call to initialize the client:

`let kinClient: KinClient! = initializeKinClientOnPlaygroundNetwork()`

### Info.plist

Let’s quickly configure the `Info.plist` file to allow HTTP requests:
```swift
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>  
</dict>
```

## Get or create the stored account

A Kin Client can manage multiple accounts. In this tutorial, we will only be using one.

If an account is available from the local store, you can get it with
`let account = kinClient.accounts.first`

If no account is available, you create one
```swift
do {
    let account = try kinClient.addAccount()
    return account
}
catch let error {
    print("Error creating an account \(error)")
}
```
Once the account has been added, it will be stored locally and you can then retrieve it the next time your run the app. 

## Kin account identification

A Kin account is identified via its public address, retrieved with `.publicAddress`.

```swift
account.publicAddress
```

## Delete a stored account

If you want to make sure there are no account stored locally, delete the accounts from the `KinClient`.

:warning: If the account has not been backed up previously by exporting it, it will be lost and its Kin inaccessible.

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

## Create the account on the blockchain

If you have just created the account locally with `kinClient.addAccount()`, you still need to create that account on
the blockchain in order to query its status, balance or exchange transactions.

```swift
/**
Create the given stored account on the playground blockchain. When on the Playground blockchain, the account
is funded with 10000 Kins automatically.
*/
func createAccountOnPlaygroundBlockchain(account: KinAccount,
                                         completionHandler: @escaping (([String: Any]?) -> ())) {
    // Playground blockchain URL for account creation
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

## Putting it together

Your `viewDidLoad` can now look like this:

```swift
// Initialize the Kin client on the playground blockchain
let kinClient: KinClient! = initializeKinClientOnPlaygroundNetwork()

// The Kin Account we're going to create and use
var account: KinAccount! = nil

// This deletes any stored account so that we recreate one - uncomment to delete
// deleteFirstAccount(kinClient: kinClient)

// Get any stored existing user account
if let existingAccount = getFirstAccount(kinClient: kinClient) {
    print("Current account with address \(existingAccount.publicAddress)")
    account = existingAccount
}
// or else create an account with the client
else if let newAccount = createLocalAccount(kinClient: kinClient) {
    print("Created account with address \(newAccount.publicAddress)")
    account = newAccount
}
```


## Get the status

An account’s status is either `.notCreated` or `.created`. If an account only exists locally after a call to
`kinClient.addAccount()`, its status is `.notCreated`. A call to create that account on the blockchain needs to be
performed as described above.

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

## Get the balance

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


## Send Kin with a transaction

Provided the account has been created and funded on the playground blockchain environment, you can now use it to send Kin
to another account. This process happens in two steps:
- Build the transaction request, which returns a `TransactionEnvelope` object if successful
- Send the request, which returns a `TransactionId` if successful

The following snippet generate the transaction request, then send it.

Every transaction costs a fee to execute on the blockchain network.
Fee for individual transactions are trivial (1 Fee = 10<sup>-5</sup> Kin).

A whitelist of pre-approved Kin apps have their fee waived. In this case, a special 'whitelist' transactions
is sent. See [Send kins with a whitelist transaction]()


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

Besides the Kin account destination address and the amount of kin to be transferred, a `memo` parameter can also be
attached to the transaction, for instance to specify an order number.

## Send Kin with a whitelist transaction

A whitelist of pre-approved Kin apps have their fee waived. When sending
transactions for an app that is whitelisted, an additional step is
necessary to sign the transaction envelope with a whitelist service.

The steps are thus the following:

- Build the transaction request, which returns a `TransactionEnvelope` object if successful
- Create a whitelist envelope passing the generated envelope and the network Id of the client
- Sign the whitelist envelope by sending it to a specific whitelist service. Another `TransactionEnvelope` object is returned if the call is successful
- Send the request, which returns a `TransactionId` if successful

```
/**
Sends a transaction to the given account.
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

```
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
{“pkey":"GA5J72WTDFR7E2IJPAOG65V5MP3QDQCVVNWIGXXLSYVTQMY7RZAYV3EM",
"seed":"abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234567",
"salt":"aafca71bc57d1065198b88281425f22c”}
```
`pkey` is the public key of the account. 


## Conclusion

This tutorial should have helped you get started with the Kin SDK for iOS. Other topics not covered here are:

- Transition to the main production environment, e.g. get an `appId` for your app



