Charging API
PayHere Charging API lets you charge your preapproved customers programatically on demand using the encrypted tokens you retrieved from Preapproval API. If you're new to Charging API, please refer Automated Charging introduction first.

Unlike other PayHere APIs, Charging API is a RESTful API where you can directly send a HTTP Request with POST parameters to an API end point & process a payment. You will get the payment status parameters also directly in the HTTP Response for the above request. This API is secured with OAuth authentication & therefore you need first generate a pair of App ID & App Secret from your PayHere account, derive an Authorization code from them & finally retrieve an Access Token from the Authorization code in order to consume the Charging API.

Please follow the below steps.


1. Create an API Key
Sign in to your PayHere account & go to Settings > API Keys section
Click 'Create API Key' button & enter an app name & comma seperated domains to whilelist
Tick the permission 'Automated Charging API'
Click 'Save API Key' button to create the app
Once the app is created click 'View Credentials' button in front of the created record
Copy the 'App ID' & 'App Secret' values

2. Generate an Authorization code
Go to a Base64 encoding site such as https://www.base64encode.net/
Paste the App ID & App Secret you copied from Step 1 above, separated by a colon (:) mark. eg. 4OVx33RVOPg4DzdZUzq4A94D2:8n4VCj25MXp4JLDFyvsE9h4a8qgbPaZUI4JEWK4FCvop
Click 'Encode' button & copy the base64 encoded value. This is your Authorization code.

3. Retrieve an Access Token
If you don't already have a valid Access Token, you can retrieve one by consuming following API end point.

Request URL
Sandbox - https://sandbox.payhere.lk/merchant/v1/oauth/token
Live    - https://www.payhere.lk/merchant/v1/oauth/token
Header
Authorization: Basic <Authorization_code>
Content-Type: application/x-www-form-urlencoded
eg. Authorization: Basic NE9WeDMzUlZPUGc0RHpkWlV6cTRBOTREMjo4bjRWQ2oyNU1YcDRKTERGeXZzRTloNGE4cWdiUGFaVUk0SkVXSzRGQ3ZvcA==

Body
grant_type: client_credentials
The response for the above request will looks like following.

Response
{
    "access_token": "cb5c47fd-741c-489a-b69e-fd73155ca34e",
    "token_type": "bearer",
    "expires_in": 599,
    "scope": "SANDBOX"
}
Retrieve the access_token from the above response.


4. Charge your Customer
Consume the charging API with a valid access_token to charge the customer programaticaaly on demand. Use the customer_token of your customer, you retrieved from the Preapproval API in the body.

(Please note that in Live environment, this API can be consumed only from the allowed domains you white-listed when creating the Business App in Step 1. If you want to consume this from a Mobile App, write an intermediate API on your domain to initiate the charging request & integrate it with your Mobile App instead of integrating directly with the Charging API.

To enhance security in the live environment, PayHere enforces an IP-based API key whitelisting mechanism for Merchant APIs. This ensures that API requests are accepted only from pre-approved server IP addresses.

To enable access to the live Merchant APIs, please send an email to support@payhere.lk including the IP address of the server from which API requests will originate.

Once the IP address is received and verified, PayHere will whitelist it, allowing your system to successfully access the live Merchant APIs.)

Request URL
Sandbox - https://sandbox.payhere.lk/merchant/v1/payment/charge
Live    - https://www.payhere.lk/merchant/v1/payment/charge
Header
Authorization: Bearer <access_token>
Content-Type: application/json
eg. Authorization: Bearer cb5c47fd-741c-489a-b69e-fd73155ca34e

Body
{
    "type": "PAYMENT",
    "order_id": "Order12345",
    "items": "Taxi Hire 123",
    "currency": "LKR",
    "amount": 345.67,
    "customer_token": "59AFEE022CC69CA39D325E1B59130862",
    "custom_1": "custom parameter 1",
    "custom_2": null,
    "notify_url": "http://www.abc.com/hire/notify",
    "itemList": [
        {
            "name": "Hire from Colombo 1 to Colombo 4",
            "number": "HIRE_12345",
            "quantity": 1,
            "unit_amount": 300.00
        },
        {
            "name": "Tax",
            "number": "TAX_12345",
            "quantity": 1,
            "unit_amount": 45.67
        }
    ]
}
Possible type field values

PAYMENT - Payment will be captured immediately. (Default)
AUTHORIZE - Payment will be authorized(held) and will return authorization_token for capturing payment later
The custom_1, custom_2, notify_url, itemList fields are optional.

The response for the above request will look like following.

Response
{
    "status": 1,
    "msg": "Automatic payment charged successfully",
    "data": {
        "order_id": "Order12345",
        "items": "Taxi Hire 123",
        "currency": "LKR",
        "amount": 345.67,
        "custom_1": null,
        "custom_2": null,
        "payment_id": 320025021815,
        "status_code": 2,
        "status_message": "Successfully completed the test tokenized payment.",
        "md5sig": "A098FEBCC06293734641770555B4D569",
        "authorization_token": "74d7f304-7f9d-481d-b47f-6c9cad32d3d5"
    }
}
You can retrieve the status_code (2, 0, -1, -2) value from the response to check whether the payment has been failed or success.

Payment Status Codes

2 - success
0 - pending
-1 - canceled
-2 - failed
The authorization_token will have a value if the request type is set to AUTHORIZE. Otherwise, it will be null.


Please note that;

You need to follow Steps 1 & 2 only one time to generate an Authorization code for your app
You need to follow Step 3 & retrieve a new Access Token only if your current Access Token is expired
If you already have a valid Access Token, you can directly charge your customer using the same Access Token by following just the Step 4

5. Error Handling
Error messages may returned for above two API end points due to the following reasons. You need to identify them & handle them properly in your application.

Invalid Access Token

{
    "error": "invalid_token",
    "error_description": "Invalid access token: e291493a-99a5-4177-9c8b-e8cd18ee9f85"
}
Access Token Expired

{
    "error": "invalid_token",
    "error_description": "Access token expired: cb5c47fd-741c-489a-b69e-fd73155ca34e"
}
Invalid Customer Token

{
    "status": -1,
    "msg": "Invalid token",
    "data": null
}
Invalid Currency

{
    "status": -1,
    "msg": "Invalid currency",
    "data": null
}
Invalid Amount

{
    "status": -1,
    "msg": "Invalid amount",
    "data": null
}
Invalid API Authorization (eg. Consuming through a not allowed domain, ect.)

{
    "status": -2,
    "msg": "Authentication error",
    "data": null
}
Still need help? Get in touch!Last updated on 17th Dec 2025