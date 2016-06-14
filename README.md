# Sage Pay Integration PSR-7 Message REST API Library

This package provides the data models and business logic for the [Sage Pay Integration](https://test.sagepay.com/documentation/) payment gateway
(sometimes called the `Pi` or `REST` API).
It does not provide the transport mechanism (at thisstage), so you can use what you like for that,
for example Guzzle, curl or a PSR-7 library.

You can use this library as a PSR-7 message generator/consumer, or go a level down and handle all the
data through arrays - both are supported.

## Package Development

The Sage Pay Integration payment gateway is a RESTful API run by by [Sage Pay](https://sagepay.com/).
You can [apply for an account here](https://applications.sagepay.com/apply/3F7A4119-8671-464F-A091-9E59EB47B80C) (partner link).

It is very much work in progress at this very early stage, while this Sage Pay API is in beta.
However, we aim to move quickly and follow changes to the API as they are released or implemented.
The aim is for the package to be a complete model for the Sage Pay Integration API, providing all the data
objects, messages (in both directions) and as much validation as is practical.

## Want to Help?

Issues, comments, suggestions and PRs are all welcome. So far as I know, this is the first API for the
Sage Pay Integration REST API, so do get involved, as there is a lot of work to do.

Tests need to be written. I can extend tests, but have not got to the stage where I can set up a test
framework from scratch.

More examples of how to handle errors is also needed. Exceptions can be raised in many places.
Some exceptions are issues at the remote end, some fatal authentication errors, and some just relate
to validation errors on the payment form, needing the user to fix their details. Temporary tokens
expire over a period and after a small number of uses, so those all need to be caught and the
user taken back to the relevant place in the protocal without losing anything they have entered
so far (that has not expired).

## Overview; How to use

Note that this example code deals only with using the gateway from the back-end.
There is a JavaScript front-end too, with hooks to deal with expired
session keys and card tokens.

### Installation

The default development branch is `PSR-7`.

Get the development branch through composer:

    composer.phar require academe/sagepaymsg:dev-PSR-7

or the latest release:

    composer.phar require academe/sagepaymsg

Until this library has been released to packagist, include the VCS in `composer.json`:

    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/academe/SagePay-Integration.git"
        }
    ]

### Get a Session Key

The `SessionKey` message has had PSR-7 support added, and can be used like this:

~~~php
// require "guzzlehttp/guzzle:~6.0"
// This will bring in guzzle/psr7 too, which is also needed.
// require "zendframework/zend-diactoros": "~1.3" if you prefer to use that instead.
use GuzzleHttp\Client;
use GuzzleHttp\Exception\ClientException;
// or use any other client that can handle PSR-7 messages.

use Academe\SagePay\Psr7\Request;
use Academe\SagePay\Psr7\Response;
use Academe\SagePay\Psr7\Model;
use Academe\SagePay\Psr7\Factory;

// Set up auth details.
$auth = new Moadel\Auth('vendor-name', 'your-key', 'your-password');

// Also the endpoint, which is now a separate class.
// This one is the test API endpoint.
$endpoint = new Model\Endpoint(Model\Auth::MODE_TEST);

// Request object to construct the session key message.
$key_request = new Request\SessionKey($endpoint, $auth);

// HTTP client to send this message.
$client = new Client();

// You should turn HTTP error exceptions off so that this package can handle all HTTP return codes.
$client = new Client(['http_errors' => false]);

// Send the PSR-7 message. Note *everything* needed is in this message.
// The message will be generated by guzzle/psr7 or zendframework/zend-diactoros, with discovery
// on which is installed. You can explictly create the PSR-7 factory instead and pass that in
// as a third parameter when creating Request\SessionKey.

$psr7_response = $client->send($key_request->message());

// Capture the result in our local response model.
// Use the ResponseFactory to automatically choose the correct message class.
$session_key = Factory\ResponseFactory::parse($psr7_response);

// If an error is indicated, then you will be returned an ErrorCollection instead
// of the session key. Look into that to diagnose the problem.

if ($session_key->isError()) {
    // $session_key will be Response\ErrorCollection
    var_dump($session_key->first());
    exit;
}

// The result we want.
echo "Session key is: " . $session_key->getMerchantSessionKey();
~~~

### Get a Card Identifier

That example involves capturing the PSR-7 message, then sending it.
The `Request\SessionKey` class will generate the PSR-7 message using `Guzzle/Psr7`,
so long as `Guzzle/Psr7` is installed,
but you can pass in your own PSR-7 factory instead if you wish to use another library.

No other messages have been converted to use PSR-7 yet - just playing with `SessionKey`
first to explore how it can work in a simple, robust, and flexible way.

This has been extended to getting the card identifier:

~~~php
use Academe\SagePay\Psr7\Request\CardIdentifier as CardIdentifierRequest;

// $endpoint, $auth and $session_key from before.
$card_identifier_request = new CardIdentifierRequest(
    $endpoint, $auth, $session_key,
    'Fred', '4929000000006', '1220', '123'
);

// Send the PSR-7 message.
// The same error handling as shown earlier can be used.
$psr7_response = $client->send($card_identifier_request->message());

// Grab the result in the local object.
$card_identifier = Factory\ResponseFactory::parse($psr7_response);

// Again, an ErrorCollection will be returned in the event of an error.
if ($card_identifier->isError()) {
    // $session_key will be Response\ErrorCollection
    var_dump($card_identifier->first());
    exit;
}

echo "Card identifier = " . $card_identifier->getCardIdentifier();
echo "Card type = " . $card_identifier->getCardType();
~~~

### Submit a Transaction

Then a transaction can be initiated.

~~~php
use Academe\SagePay\Psr7\Money;
use Academe\SagePay\Psr7\PaymentMethod;

// We have a billing address:
$billing_address = Model\Address::fromData([
    'address1' => 'address one',
    'postalCode' => 'NE26',
    'city' => 'Whitley',
    'state' => 'AL',
    'country' => 'US',
]);

// We have customer to bill:
$customer = new Model\Person(
    'Bill Firstname',
    'Bill Lastname',
    'billing@example.com',
    '+44 191 12345678'
);

// We have an amount to bill.
$amount = Money\Amount::GBP()->withMinorUnit(999);

// We have a card to charge (we get the session key and captured the card identifier earlier).
$card = new PaymentMethod\Card($session_key, $card_identifier);

// Put it all together into a transaction.
$payment = new Request\Payment(
    $endpoint,
    $auth,
    $card,
    'MyVendorTxCode-' . rand(10000000, 99999999), // This will be your local transaction key ID.
    $amount,
    'My Purchase Description',
    $billing_address,
    $customer,
    null, // Optional shipping address
    null, // Optional shipping recipient
    [
        // Don't use 3DSecure this time.
        'Apply3DSecure' => Request\Payment::APPLY_3D_SECURE_DISABLE,
    ]

);

// Want to set as a recurring payment, e.g. a subscription?
// Future payments can be taken with the `Request\Repeat` message referencing the
// `transactionId` of this `Response\Payment`.
$payment = $payment->withRecurringIndicator(Request\Payment::RECURRING_INDICATOR_RECURRING);

// Alternatively it counld be added as an option when first creating the payment:
[... 'RecurringIndicator' => Request\Payment::RECURRING_INDICATOR_RECURRING, ...]

// Send it to Sage Pay.
$psr7_response = $client->send($payment->message());

// Assuming we got no exceptions, extract the response details.
$payment_response = Factory\ResponseFactory::parse($psr7_response);

// Again, an ErrorCollection will be returned in the event of an error.
if ($payment_response->isError()) {
    // $payment_response will be Response\ErrorCollection
    var_dump($payment_response->first());
    exit;
}

if ($payment_response->isRedirect()) {
    // If the result is "3dAuth" then we will need to send the user off to do their 3D Secure
    // authorisation (more about that process in a bit).
    // A status of "Ok" means the transaction was successful.
    // A number of validation errors can be captured and linked to specific submitted
    // fields (more about that in a bit too).
    // ...
}

echo "Final status is " . $payment_response->getStatus();

if ($payment_response->isSuccess()) {
    // Payment is successfully authorised.
}
~~~

### Fetch a Transaction Result Again

Given the TransactionId, you can fetch the transaction details.
If the transaction was successful, then it will be available immediately.
If a 3D Secure action was needed, then the 3D Secure results need to be sent
to Sage Pay before you can fetch the transaction.
Either way, this is how you do it:

~~~php
// Prepare the message.
$transaction_result = new Request\TransactionResult(
    $endpoint,
    $auth,
    $transaction_response->getTransactionId() // From earlier
);

// Send it to Sage Pay.
$response = $client->send($transaction_result->message());

// Assuming no exceptions, this gives you the payment or repeat payment record.
// But do check for errors in the same way.
$fetched_transaction = Factory\ResponseFactory::parse($response);
~~~

### Using 3D Secure

Now, if you want to use 3D Secure (and you really should) then we have a callback to deal with.

To turn on 3D Secure, use the appropriate option when sending the payment:

~~~php
    [
        // Also APPLY_3D_SECURE_USEMSPSETTING and APPLY_3D_SECURE_FORCEIGNORINGRULES
        'Apply3DSecure' => Request\Payment::APPLY_3D_SECURE_FORCE,
    ]
~~~

### 3D Secure Callback

The result of the transaction, assuming all is otherwise fine, will a `Secure3DRedirect` object.
This message will return true for `isRedirect()`.
Given this, a POST redirection is needed.

This minimal form will deomstrate how the redirect is done:

~~~php
if ($transaction_response->isRedirect()) {
    // This is the bank URL that Sage Pay wants us to send the user to.
    $url = $transaction_response->getAcsUrl();

    // This is where the bank will return the user when they are finished there.
    // It needs to be an SSL URL to avoid browser errors. That is a consequence of
    // the way the banks do the redirect back, and something we cannot control.
    $termUrl = 'https://example.com/your-3dsecure-result-handler-path/';

    // $md is optional and is usually a key to help find the transaction in storage.
    // For demo, we will just send the vendorTxCode here, but you would avoid exposing
    // that value in a real site. You could leave it unused and just store the vendorTxCode
    // in the session, since it will always only be used when the user session is available
    // (i.e. all callbacks are done through the user's browser).
    $md = $transaction_response->getTransactionId();
    $paRequestFields = $transaction_response->getPaRequestFields($termUrl, $md);

    // All these fields will normally be hidden form items and the form would auto-submit
    // using JavaScript.
    echo "<p>Do 3DSecure</p>";
    echo "<form method='post' action='$url'>";
    foreach($paRequestFields as $field_name => $field_value) {
        echo "<p>$field_name <input type='text' name='$field_name' value='$field_value' /></p>";
    }
    echo "<button type='submit' />";
    echo "</form>";

    // Obviously don't exit if you are using a framework and views.
    exit;
}
~~~

The above example does not take into account how you would show the 3D Secure form in an iframe instead
of inline. That is out of scope for this simple description.

This form will then take the user off to the 3D Secure password page. For Sage Pay testing, use the code
`password` to get a successful response when you reach the test 3D Secure form.

Now you need to handle the return from the bank. Using Diactoros (and now Guzzle) you can catch the return
message as a PSR-7 ServerRequest like this:

~~~php
use Academe\SagePay\Psr7\ServerRequest;

$psr7_ServerRequest = \Zend\Diactoros\ServerRequestFactory::fromGlobals();
// or guzzlehttp/psr7 from v1.3
$psr7_ServerRequest = \GuzzleHttp\Psr7\ServerRequest::fromGlobals();

if (ServerRequest\Secure3DAcs::isRequest($psr7_ServerRequest->getBody()))
    // Yeah, we got a 3d Secure server request coming at us. Process it here.
    $secure3d_server_request = new ServerRequest\Secure3DAcs($psr7_ServerRequest);
    ...
}
~~~

~~~php
if (ServerRequest\Secure3DAcs::isRequest($_POST)) {
    $secure3d_server_request = ServerRequest\Secure3DAcs::fromData($_POST);
    ...
}
~~~

Both will work fine, but it's just about what works best for your framework and application.

Handling the 3D Secure result involves two steps:

1. Passing the result to Sage Pay to get the final transaction state.
2. If successful, fetching the final transaction from Sage Pay.

~~~php
    $request = new Request\Secure3D(
        $endpoint,
        $auth,
        $secure3d_server_request,
        // Include the transaction ID.
        // For this demo we sent that as `MD` data rather than storing it in the session.
        // The transaction ID will generally be in the session; putting it in MD is lazy
        // and exposes it to the end user, so don't do this!
        $secure3d_server_request->getMD()
    );

    // Send to Sage Pay and get the final 3D Secure result.
    $response = $client->send($request->message());
    $secure3d_response = Factory\ResponseFactory::parse($response);

    // This will be the result. We are looking for `Authenticated` or similar.
    echo $secure3d_response->getStatus();
~~~

### Final Transaction After 3D Secure

Assuming 3D Secure passed, then get the transaction. However - *do not get it too soon*.
The test instance of Sage Pay has a slight delay between getting the 3D Secure result and
being able to fetch the transaction.
It is safer just to sleep for one second at this time, which is an arbitrary period but
seems to work for now.

~~~php
    // A fix for the need to do this is in hand at Sage Pay.
    sleep(1);

    // Get fetch the transaction with full details.
    $transaction_result = new Request\TransactionResult(
        $endpoint,
        $auth,
        // transaction ID would normally be in the session, as described above.
        $secure3d_server_request->getMD()
    );

    // Send the request for the transaction to Sage Pay
    $response = $client->send($transaction_result->message());

    // We should have the payment, repeat payment, or an error collection.
    $transaction_fetch = Factory\ResponseFactory::parse($response);

    // We should now have the final results.
    echo json_encode($transaction_fetch);
~~~
