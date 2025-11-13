# PHP library for the Axepta BNP Paribas/Computop API


## Getting Started

### Requirements

- php >= 7.2

### Installation

**Option 1 -** from source:

1. Clone the repo
2. Extract the `lib/` folder, and copy it somewhere in your project, e.g. `myproject/vendor/axepta-paygate/lib/`
3. Add the following code snippet to your PHP script:
   ```php
   <?php
   require_once("<PATH_TO_LIBRARY>/lib/init.php");
   ```

**Option 2 -** using composer:

Not supported yet

## Usage

### Basic Usage

In essence, the following example illustrates how all payment scenarios are performed.
But before that, a few remarks need to be addressed.

**a. Access token**

Since this library heavily relies on the OAuth2-secured Computop API to perform operations, an access token is mandatory.

This access token is generated based on the merchant ID and its API key, both supplied by BNP Paribas.
It has to be provided with every subsequent API call.
To do so, include it in the configuration before building an operation.

**b. Handling Exceptions**

It's crucial to note that the usage of this library can throw exceptions during execution.

All custom exceptions thrown by this library derive from the base class named `AxeptaPaygateException`.
Make sure to catch and handle these exceptions gracefully in your code to ensure smooth operation and effective error management.

Here's the comprehensive example:

```php
<?php
require_once("<PATH_TO_LIBRARY>/lib/init.php");  // if using installation option 1

use AxeptaPaygate\Core\AxeptaPaygate;
use AxeptaPaygate\Core\OperationType;
use AxeptaPaygate\Core\PaymentRenderingMode;
use AxeptaPaygate\Exception\ApiCallException;

// 1. Get an access token, by providing API credentials
try {
    $tokenData = AxeptaPaygate::getAccessToken($merchantId, $apiKey);
} catch (ApiCallException $e) {
    // Handle the exception gracefully
}

// 2. Load the configuration into the library.
$configuration = [
    "api_access_token" => $tokenData['access_token'];
    // ...
 ];
AxeptaPaygate::init($configuration);

// 3. Create a payment request.
try {
    $operation = AxeptaPaygate::buildOperation(...);
} catch (AxeptaPaygateException $e) {
    // Handle the exception gracefully
}

// 4. Perform the payment request.
$request = $operation['request'];
try {
    $response = $request->call();
} catch (ApiCallException $e) {
    // Handle the exception gracefully
}
```

### Advanced Usage

<details>
<summary><b>Case 1 -</b> Provide the configuration array in a static way
(you already know the configuration keys to provide)</summary>

In this case, you already know the entirety of configuration key/value pairs that are required for a given payment scenario.
This is the most straight forward approach.

Simply create a `$configuration` array and pass it to `AxeptaPaygate::init`:

```php
<?php

$configuration = [
    "example.key.a" => "a value",
    "example.key.b" => "another value",
];
AxeptaPaygate::init($configuration);

// rest of the code
```

</details>

<details>
<summary><b>Case 2 -</b> Provide the configuration array in a dynamic way (<b>recommended</b>)</summary>

The first approach is only suitable if you need to implement a few payment scenarios. However, if you need to handle many different scenarios, you'll need to manage and pass the appropriate keys for each one, as they may vary. This can quickly become cumbersome.

To simplify this, the library provides a special function that returns the necessary configuration keys for a specific scenario. You can then iterate over this array of required keys, assign the corresponding values, and load them into the library.

```php
<?php

[$prefilledConfiguration, $missingConfigurationKeys] = AxeptaPaygate::getRequiredConfigurationKeys(...);

foreach ($missingConfigurationKeys as $key) {
    switch ($key) {
        case "example.key.a":
            $prefilledConfiguration[$key] = "a value"
            break
        case "example.key.b":
            $prefilledConfiguration[$key] = "another value"
            break
        // ...
    }
}

AxeptaPaygate::init($prefilledConfiguration);

// rest of the code
```
</details>

<details>
<summary><b>Case 3 -</b> Store and reuse the access token</summary>

For optimal practices and performance, consider storing the access token, since it has an expiration date and can be reused again.
FYI, `getAccessToken` actually returns the following object:

```
>>>> print_r(AxeptaPaygate::getAccessToken(...))

Array
(
    [access_token] => ...
    [token_type] => Bearer
    [expires_in] => 3600
)
```
</details>

<details>
<summary><b>Case 4 - </b>Process a oneclick payment (initial + subsequent)</summary>

<u>**Step 1 - Initial payment**</u>

It is important that you provide the parameter SaveCard during the oneclick initial payment. It informs our API paygate that you intend to reuse the payment card for further payments.

Note that the operationType specified is DIRECT_PAYMENT (not ONECLICK_PAYMENT)

Don't forget to save in your DB the payment card information that you will receive on your URLNotify, those are important informations that you will need in order to process oneclick subsequent payment.

```php

<?php>

$configuration = [
    // All your configuration parameters
    // ...
    // Special parameter for Oneclick initial payment
    'operationType' => OperationType::DIRECT_PAYMENT,
    'saveCard' => true,
];

AxeptaPaygate::init($configuration);
```

<u>**Step 2 - Subsequent payment**</u>

To process a subsequent payment you will need to specify the operationType as a oneclick payment.
Don't forget to fill in the oneclick payment card data (Those are the data you saved from the oneclick initial payment).

```php
<?php

 $configuration = [
    // All your configuration parameters
    // ...
    // Special parameter for Oneclick subsequent payment
    'operationType' => OperationType::ONE_CLICK_PAYMENT,
    // OneClick Card data
    'payment.card.card.brand' => '<brand>',
    'payment.card.card.expiryDate' => '<expiry_date>',
    'payment.card.card.number' => '<number>',
];

AxeptaPaygate::init($configuration);
```

</details>

<details>
<summary><b>Case 5 - </b>Process a subscription payment (initial + subsequent)</summary>

<u>**Step 1 - Initial payment**</u>

It is important that you provide the parameter initializeSubscription during the subscription initial payment. It informs our API paygate that you intend to reuse the payment card for further payments.

Note that the operationType specified is DIRECT_PAYMENT.

Don't forget to save in your DB the payment card information that you will receive on your URLNotify, those are important informations that you will need in order to process subscription subsequent payment.

```php

<?php>

$configuration = [
    // All your configuration parameters
    // ...
    // Special parameter for Subscription initial payment
    'operationType' => OperationType::DIRECT_PAYMENT,
    'initializeSubscription' => true,
];

AxeptaPaygate::init($configuration);
```

<u>**Step 2 - Subsequent payment**</u>

To process a subsequent payment you will need to specify the operationType as a RECURRING_PAYMENT_SUBSCRIPTION.
Don't forget to fill in the payment card data (Those are the data you saved from the subscription initial payment).

```php
<?php

 $configuration = [
    // All your configuration parameters
    // ...
    // Special parameter for subscription subsequent payment
    'operationType' => OperationType::RECURRING_PAYMENT_SUBSCRIPTION,
    // Card data
    'payment.card.card.brand' => '<brand>',
    'payment.card.card.expiryDate' => '<expiry_date>',
    'payment.card.card.number' => '<number>',
];

AxeptaPaygate::init($configuration);
```

</details>

<details>
<summary><b>Case 6 - </b> Retrieve payment details</summary>

You can consult the swagger API documentation to know more about the payment details response. (https://app.swaggerhub.com/apis-docs/Computop/Paygate_REST_API/0.3#/PaymentDetailsResponse)

```php
use AxeptaPaygate\Exception\ApiCallException;
use AxeptaPaygate\Core\AxeptaPaygate;

try {
    $paymentDetails = AxeptaPaygate::getPaymentDetails($accessToken, $paymentId);
} catch (ApiCallException $e) {
    // Handle the exception gracefully
}

```
</details>

<details>
<summary><b>Case 7 - </b> Retrieve verbose code message</summary>

```php
use AxeptaPaygate\Core\AxeptaPaygate;
use AxeptaPaygate\Exception\CodeMessageException;

try {
    // Must be 8 length long
    $code = '00000000';
    // Keep it in lowercase
    $language = 'fr';

    $codeMessage = AxeptaPaygate::getCodeMessage(
        $code,
        $language /* default:en */
    );

    /*
        $codeMessage == [
            'state' => [
                'description' => 'string',
                'message' => 'string'
            ],
            'module' => [
                'description' => 'string',
                'message' => 'string'
            ]
            'parameter' => [
                'description' => 'string',
                'message' => 'string'
            ]
        ]
    */
} catch (CodeMessageException $e) {
    // Handle the exception gracefully
}

```
</details>
