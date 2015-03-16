# Game Closure DevKit Plugin : Billing

The billing plugin supports in-app purchases from the Google Play Store on
Android, and from the Apple App Store on iOS, through one simple unified API.

# IMPORTANT

This branch disables automatic item consumption from getPurchases. This is
catastrophic on android if your game has any purchases that are consumable -
(if you miss a purchase, you will be given it over and over, every time you
 load the app). DO NOT USE THIS WITHOUT UNDERSTANDING THE RAMIFICATIONS.
*YOU HAVE BEEN WARNED.*



### iOS Setup

For iOS you should ensure that your game's `manifest.json` has the correct "bundleID", "appleID", and "version" fields.

The store item Product IDs must be prefixed with your bundleID (as in "com.gameshop.sword"), but you should refer to the item as "sword" in your JavaScript code.

If any of your in-app purchases are managed instead of consumable then you will need to make additional changes.  To be accepted on the iOS app store you must have a [Restore Purchases] button.  See the `Restoring Purchases` section below for details.

After building your game, you will need to turn on the IAP entitlement.  This can be done by selecting your project, choosing the "Capabilities" tab, and turning on the In-App Purchase entitlement.  You will be prompted to log in to your development team.


### Android - Google Play Store Setup

For the play store you should ensure your game is published (it can be published
in the alpha or beta channels and not show up on the app store) and includes in
app purchases. You'll need to add test email addresses to your play store
account if you want to use test transactions instead of real transactions.

Your in app purchase names should match the item names you use in your
javascript code.

NOTE - SEE WARNING ABOVE ABOUT AUTO-CONSUME.



## Handling Purchases

In the JavaScript code for your game, you should write some code to handle
in-app purchases.  After reading the previous state of the purchasable items
from `localStorage`, your code should set up a `billing.onPurchase` handler:

~~~
// Initialize the coin counter
var coinCount = localStorage.getItem("coinCount") || 0;

var purchaseHandlers = {
	"fiveCoins": function() {
		// Update the visual coin counter here.
		coinCount += 5;
		localStorage.setItem("coinCount", coinCount);
		// Pop-up award modal here.
	}
};

function handlePurchase(item) {
	var handler = purchaseHandlers[item];
	if (typeof handler === "function") {
		handler();
	}
};

billing.onPurchase = handlePurchase;
~~~

The callback you set for `billing.onPurchase` is only called on successful
purchases.  It will be called once for each purchase that should be credited to
the player.  Purchases will be queued up until the callback is set, and then
they will all be delivered, one at a time.

After a player successfully purchases an item, it is a good idea to store it in
offline local storage to persist between runs of the game.  This can be done
with the normal HTML5 localStorage API as shown above.

Consumable purchases must be tracked by your own application.  Managed purchases
can be tracked by the App Store, but will require you to implement a Restore
Purchases button in your app.  If you are tracking managed purchases in your
local storage data, be aware that the `billing.onPurchase` callback will likely
be called with that item again while restoring purchases (and on load for
android), so you will need to avoid double-crediting the player.


## Handling Purchase Failures

When purchases fail, the failure may be handled with the `billing.onFailure` callback:

~~~
function handleFailure(reason, item) {
	if (reason !== "cancel") {
		// Market is unavailable - User should turn off Airplane mode or find reception.
	}

	// Else: Item purchase canceled - No need to present a dialog in response.
}

billing.onFailure = handleFailure;
~~~

Handling these failures is *optional*.

One way to respond is to pop up a modal dialog that says "please check that
Airplane mode is disabled and try again later."  It may also be interesting to
do some analytics on how often users cancel purchases or fail to make purchases.

## Checking for Market Availability

Purchases can fail to go through due to network failures or market unavailability.  You can verify that the market is available by checking `billing.isMarketAvailable` before displaying your in-app store.  You can also subscribe to a "MarketAvailable" event (see event documentation below).

~~~
// In response to player clicking In-App Store button:

if (!billing.isMarketAvailable) {
	// Market is unavailable - User should turn off Airplane mode or find reception.
}
~~~

Checking for availability is entirely optional.

## Requesting Purchases

All purchases are handled as consumables.  For this reason, it is up to you to make sure that players do not purchase ie. character unlocks two times as the billing plugin cannot differentiate those types of one-time upgrade -style purchases from consumable currency -style purchases.

When you request a purchase, a system modal will pop up that the user will interact with and may cancel.  Purchases may also fail for other reasons such as network outages.

Kicking off a new purchase is done with the `billing.purchase` function:

~~~
// In response to player clicking the "5 coin purchase" button:

billing.purchase("fiveCoins");
~~~

## Disabling Purchases

The purchase callback may happen at any time, even during gameplay.  So it is a good idea to disable the callback when it is inopportune by setting it to null.  When you want to receive the callback events, just set it back to the handler and any queued events will be delivered as shown in this example code:

~~~
// When player enters game and should not be disturbed by purchase callbacks:
function DisablePurchaseEvents() {
	billing.onPurchase = null;
	billing.onFailure = null;
}

// And when they return to the menu system:
function EnablePurchaseEvents() {
	billing.onPurchase = handlePurchase; // see definitions in examples above
	billing.onFailure = handleFailure;
}
~~~

## Restoring Purchases

In order to ship an app with in-app purchases other than "Consumable" on the iOS
App Store, you are required to include a [Restore Purchases] button, which must
query the App Store for past purchases made from the same Apple ID and restore
them in the game.  The way to implement this button is by using the
`billing.restore` function.

Note that on iOS you do not need to do this if all of your purchases are
consumable.

On iOS, this functions by calling the [StoreKit
`restoreCompletedTransactions`](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentQueue_Class/index.html#//apple_ref/occ/instm/SKPaymentQueue/restoreCompletedTransactions)
function on ios, then fires the `billing.onPurchase` callback for each old
purchase. On android, this does nothing.

~~~
billing.restore(function(err) {
	if (err) {
		logger.log("Unable to restore purchases:", err);
	} else {
		logger.log("Finished restoring purchases!");
	}
});
~~~

Your `billing.onPurchase` callback will receive all of the old items while
restoring.

Finally, the provided callback will be called, letting you know when the
restoration completes, or if the restoration failed and why.

If an in-game button press triggers `billing.restore` then the button should be
disabled until the result comes back to your callback.


## Localizing Purchases

The billing plugin can query the store for localized information about your
purchases so that you can display formatted, region-specific labels and prices
to your users. You can request the localized purchase information by sending a
list of purchase ids to `billing.getLocalizedPurchases` and adding a listener
for the `PurchasesLocalized` event.

NOTE: on Android, the `PurchasesLocalized` event will only be emitted in
response to a `getLocalizedPurchases` request, with just the requested
item ids. On iOS, however, `PurchasesLocalized` will be emitting every time
`getLocalizedPurchases` is called AND every time a purchase is made for
an item that has not yet been localized or purchased and will include
every purchase id localized or purchased that session. Ensure your handler
for `PurchasesLocalized` correctly handles all of the above cases.

The `PurchasesLocalized` event payload includes a `purchases` dictionary
in the following format:
```
purchases: {
    store_id_for_item: {
        title: 'localized title for this item',
        description: 'localized description for this item',
        displayPrice: 'localized price for item, including currency symbol'
    },
    ...
}
```

Example Localization Handler and Request:
~~~
// listen for localization events
billing.on("PurchasesLocalized", function (data) {
  logger.log("billing.PurchasesLocalized", data);
  var itemIds = Object.keys(data.purchases);
  for (var i = 0; i < itemIds.length; i++) {
    var itemId = itemIds[i];
    var item = data.purchases[itemId];
    logger.log(
        'Localized: ',
        itemId,
        item.displayPrice,
        item.title,
        item.description
    );
  }
});

// send the localization request
billing.getLocalizedPurchases(['item1', 'item2']);
~~~

# billing object

## Events

### "MarketAvailable"

This event fires whenever market availability changes.  It is safe to ignore these events.

~~~
billing.on('MarketAvailable', function (available) {
	if (available) {
	} else {
	}
});
~~~


### "PurchasesLocalized"

This event fires after purchases have been localized. On iOS, this includes
every localized/purchased item and also fires any time a purchase is
attempted with an item that has not yet been localized or purchased. On android,
this is only fired in response to a `getLocalizedPurchases` request and only
includes the items specifically requested. See the Localizing Purchases section
for more info and example usage.


Read the [event system documentation](http://docs.gameclosure.com/api/event.html)
for other ways to handle these events.

## Members:

### billing.isMarketAvailable

+ `boolean` ---True when market is available.

The market can become unreachable when network service is interrupted or if
the mobile device enters Airplane mode.

It is safe to disregard this flag.

~~~
if (billing.isMarketAvailable) {
  logger.log("~~~ MARKET IS AVAILABLE");
} else {
  logger.log("~~~ MARKET IS NOT AVAILABLE");
}
~~~

### billing.onPurchase (itemName, transactionInfo)

+ `callback {function}` ---Set to your callback function.
  * itemName - the name of the item that should be credited to the player
  * transactionInfo - an object with the following fields:
    * signature - the transaction signature for the given store
    * purchaseData - (google play store only) json encoded string with the
      purchase data for the transaction

Called whenever a purchase completes.  This may also be called for a purchase
that was outstanding from a previous session that had not yet been credited to
the player.

The callback function should not pop up the purchase success dialog while they
are playing.  Setting the `billing.onPurchase` callback to **null** when
purchases should not interrupt gameplay is recommended.

~~~
billing.onPurchase = function(itemName) {
	logger.log("~~~ PURCHASED:", itemName);
});
~~~

## Handling Purchase Validation

The onPurchase function is called with the itemName (matching the store id) and
a `transactionInfo` object with `signature` and `purchaseData` fields
which can be used to validate purchases against the stores using an external
server or third party service.

On iOS, `signature` is a base 64 encoded string from the
[transactionReceipt](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/index.html#//apple_ref/occ/instp/SKPaymentTransaction/transactionReceipt)
NSData (deprecated in iOS 7 but still remains the standard for most analytics
platforms and is the only way to support older devices). On iOS,
`purchaseData` is unused.

On Android, `signature` is
the value in the `INAPP_DATA_SIGNATURE` string on the purchase activity data.
Android also includes the json encoded `purchaseData` payload from the store.

More info in [the
docs](http://developer.android.com/google/play/billing/billing_integrate.html#Purchase).

Here is an example from the demoBilling app which then passes the `signature`
and `purchaseData` on to the amplitude module for validation.

~~~
  var ITEMS = {
    'testpurchase10': {price: .99, quantity: 1},
    'testpurchase50': {price: 1.99, quantity: 1}
  };

  billing.onPurchase = function onPurchase (itemName, transactionInfo) {
    console.log("Purchase Successful! Item: " + itemName);

    // if no transactionInfo, use empty object
    transactionInfo = transactionInfo || {};

    // send to amplitude for tracking and validation
    var item = ITEMS[itemName];
    amplitude.trackRevenue(
      itemName,
      item.price,
      item.quantity,
      transactionInfo.signature,
      transactionInfo.purchaseData
    );
  };
~~~


### billing.onFailure (reason, itemName)

+ `callback {function}` ---Set to your callback function.
			The first argument will be the reason for the failure.
			The second argument will be the name of the item that was requested.  Sometimes the name will be `null`.

Unlike the success callback, failures are not queued up for delivery.  When failures are not handled they are not reported.

The `itemName` argument to the callback is not reliable.  Sometimes it will be `null`.

Handling failure events is optional.

Common failure values:

+ "cancel" : User canceled the purchase or item was unavailable.
+ "service" : Not connected to the Market.  Try again later.
+ Other Reasons : Was not able to make purchase request for some other reason.

## Methods:

### billing.purchase (itemName, [simulate])

Parameters
:    `itemName {string}` ---The item name string.
:    `[simulate {string}]` ---Optional simulation mode: `undefined` means disabled. "simulate" means simulate a successful purchase.  Any other value will be used as a simulated failure string.

Returns
:    `void`

Initiate the purchase of an item by its name.

The purchase may fail if the player clicks to deny the purchase, or if the network is unavailable, among other reasons.  If the purchase fails, the `billing.onFailure` handler will be called.  Handling failures is optional.

If the purchase succeeds, then the `billing.onPurchase` callback you set will be called.  This callback should be where you credit the user for the purchase.

IN THIS BRANCH ONLY you can specify items to NOT be auto-consumed from the store
(so you can implement managed items on android) by passing an object instead of
a string for the itemName in the format `{sku: itemName, consume: false}`.


~~~
billing.purchase("fiveCoins");
~~~

### billing.restore ([callback])

Parameters
:    `[callback {function}]` ---Optional callback.

Returns
:    `void`

Initiate restoring old purchases.  These will only restore "managed" purchases
set up for your application that are tracked by the app store servers.
Consumable purchases will be lost if local storage is wiped for any reason.

This functions by calling the [StoreKit
`restoreCompletedTransactions`](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentQueue_Class/index.html#//apple_ref/occ/instm/SKPaymentQueue/restoreCompletedTransactions)
function on ios, then fires the `billing.onPurchase` callback for each old
purchase.

When restoration completes, the optional callback provided to `billing.restore` will be invoked.


~~~
billing.restore(function(err) {
	if (err) {
		logger.log("Unable to restore purchases:", err);
	} else {
		logger.log("Finished restoring purchases!");
	}
});
~~~

See the guide section above on `Restoring Purchases` for more information.

##### Simulation Mode

To test purchases, pass a second parameter to the
purchase method ("simulate" for success; "cancel", "refund", or "unavailable"
for failure).

On android, this will overwrite the item sku to use the test items the play
store provides, which will simulate the full purchase loop, including
accessing the store (this will fail if you do not have access to the store, just
like a real purchase).

On iOS, simulated purchases will simply return immediately.

~~~
billing.purchase("fiveCoins", "simulate"); // Simulates success
billing.purchase("fiveCoins", "cancel"); // Simulates failure "cancel"
billing.purchase("fiveCoins", "refund"); // Simulates failure "refunded"
billing.purchase("fiveCoins", "unavailable"); // Simulates failure "unavailable"
~~~

Simulation mode does not support the `billing.restore` method.

