# OpenAPIClient-php

# General Usage
This document details use of Kraken's REST API for our Spot exchange. The Spot exchange [Websockets API](https://docs.kraken.com/websockets) and the [Kraken Futures APIs](https://docs.futures.kraken.com/) are documented separately. Our REST API is organised into publicly accessible endpoints (market data, exchange status, etc.), and private authenticated endpoints (trading, funding, user data) which require requests to be signed.

Your use of the Kraken REST API is subject to the [Kraken Terms & Conditions](https://www.kraken.com/legal), [Privacy Notice](http://www.kraken.com/legal/privacy), as well as all other applicable terms and disclosures made available on [www.kraken.com](http://www.kraken.com/).

## Support
Further information and FAQs may be found on the [API section](https://support.kraken.com/hc/en-us/sections/4402371110548-API) of our support pages. If you have trouble making any standard requests that our system permits, please [send us a ticket](https://support.kraken.com/hc/en-us/requests/new?ticket_form_id=360000104043) with a full copy of the request(s) that you're attempting, including your IP address and all headers, so that we may investigate further.

## Requests, Responses and Errors

### Requests
Request payloads are [form-encoded](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST) (`Content-Type: application/x-www-form-urlencoded`), and all requests must specify a `User-Agent` in their headers.

### Responses
Responses are JSON encoded and contain one or two top-level keys (`result` and `error` for successful requests or those with warnings, or only `error` for failed or rejected requests)

#### Example Successful Response
```javascript
{
  \"error\": [],
  \"result\": {
    \"status\": \"online\",
    \"timestamp\": \"2021-03-22T17:18:03Z\"
  }
}
```
> GET https://api.kraken.com/0/public/SystemStatus

#### Example Rejection
```javascript
{
  \"error\": [
    \"EGeneral:Invalid arguments:ordertype\"
  ]
}
```

### Error Details

HTTP status codes are generally not used by our API to convey information about the state of requests -- any errors or warnings are denoted in the `error` field of the response as described above. Status codes __other__ than 200 indicate that there was an issue with the request reaching our servers.

`error` messages follow the general format `<severity><category>`:`<error msg>`[`:add'l text`]

* `severity` can be either `E` for error or `W` for warning
* `category` can be one of `General`, `Auth`, `API`, `Query`, `Order`, `Trade`, `Funding`, or `Service`
* `error msg` can be any text string that describes the reason for the error

#### Some Common Examples

| Error | Additional Info |
| -- | -- |
| EGeneral:Invalid arguments | The request payload is malformed, incorrect or ambiguous |
| EGeneral:Invalid arguments:Index unavailable |  Index pricing is unavailable for stop/profit orders on this pair |
| EService:Unavailable | The matching engine or API is offline |
| EService:Market in cancel_only mode | Request can't be made at this time (See `SystemStatus` endpoint) |
| EService:Market in post_only mode | Request can't be made at this time (See `SystemStatus` endpoint) |   
| EService:Deadline elapsed | The request timed out according to the default or specified `deadline` |
| EAPI:Invalid key | An invalid `API-Key` header was supplied (see Authentication section) |
| EAPI:Invalid signature | An invalid `API-Sign` header was supplied (see Authentication section) |
| EAPI:Invalid nonce | An invalid `nonce` was supplied (see Authentication section) |
| EGeneral:Permission denied | API key doesn't have permission to make this request |    
| EOrder:Cannot open position | User/tier is ineligible for margin trading |
| EOrder:Margin allowance exceeded | User has exceeded their margin allowance |
| EOrder:Margin level too low | Client has insufficient equity or collateral |
| EOrder:Margin position size exceeded | Client would exceed the maximum position size for this pair |
| EOrder:Insufficient margin | Exchange does not have available funds for this margin trade |
| EOrder:Insufficient funds | Client does not have the necessary funds |
| EOrder:Order minimum not met | Order size does not meet `ordermin` (See `AssetPairs` endpoint)  |
| EOrder:Cost minimum not met | Cost (price * volume) does not meet `costmin` (See `AssetPairs` endpoint) |
| EOrder:Tick size check failed | Price submitted is not a valid multiple of the pair's tick_size (See `AssetPairs` endpoint) |
| EOrder:Orders limit exceeded | (See Rate Limits section) |
| EOrder:Rate limit exceeded | (See Rate Limits section) |
| EOrder:Domain rate limit exceeded | (See Rate Limits section) |
| EOrder:Positions limit exceeded |  |
| EOrder:Unknown position |  |
| EBM:limit exceeded:CAL | Exceeded [Canadian Acquisition Limits (CAL)](https://support.kraken.com/hc/en-us/articles/15568473780628-CAD-net-purchase-limits-for-certain-cryptocurrencies-in-Canada) |
| EFunding:Max fee exceeded | Processed fee exceeds `max_fee` set in `Withdraw` request |

Additional information can be found on our [support center](https://support.kraken.com/hc/en-us/articles/360001491786-API-error-messages).



# Authentication

Authenticated requests must include both `API-Key` and `API-Sign` HTTP headers, and `nonce` in the request payload. `otp` is also required in the payload if two-factor authentication (2FA) is enabled.


## Nonce and 2FA

### `nonce`

Nonce must be an always increasing, unsigned 64-bit integer, for each request that is made with a particular API key. While a simple counter would provide a valid nonce, a more usual method of generating a valid nonce is to use e.g. a UNIX timestamp in milliseconds.

> **Note:** There is no way to reset the nonce for an API key to a lower value, so be sure to use a nonce generation method that won't produce numbers less than the previous nonce. Too many requests with invalid nonces (EAPI:Invalid nonce) can result in temporary bans. Problems can arise from requests arriving out of order due to API keys being shared across processes, or from system clock drift/recalibration. An optional \"nonce window\" can be configured to specify a tolerance between nonce values. Additional info can be found in our [support pages](https://support.kraken.com/hc/en-us/articles/360000906023-What-is-a-nonce-).

### `otp`

If two-factor authentication (2FA) is enabled for the API key and action in question, the one time password must be specified in the payload's `otp` value.


## Headers and Signature

<SecurityDefinitions />

# Rate Limits

We have various safeguards in place to protect against system abuse, order book manipulation, DDoS attacks, etc. For REST API requests, these are broadly organised into rate limits specific to the REST API, and rate limits which apply to any trading requests.

> __Note:__ For clients with performance sensitive applications, we strongly recommend the use of our Websockets API for minimising request latency and receiving real-time information, reducing or eliminating the need to frequently poll REST endpoints.

## REST API Rate Limits

### Limits

Every REST API user has a \"call counter\" which starts at `0`. Ledger/trade history calls increase the counter by `2`. All other API calls increase this counter by `1` (except AddOrder, CancelOrder which operate on a different limiter detailed further below).

| Tier         | Max API Counter | API Counter Decay |
| ------------ | --------------- | ----------------- |
| Starter      | 15              | -0.33/sec         |
| Intermediate | 20              | -0.5/sec          |
| Pro          | 20              | -1/sec            |

The user's counter is reduced every couple of seconds depending on their verification tier. Each API key's counter is separate, and if the counter exceeds the maximum value, subsequent calls using that API key would be rate limited. If the rate limits are reached, additional calls will be restricted for a few seconds (or possibly longer if calls continue to be made while the rate limits are active).

> __Note:__ Master accounts and subaccounts will share the same default trading rate limits as determined by the master account's tier. 

### Errors

* \"EAPI:Rate limit exceeded\" if the REST API counter exceeds the user's maximum.
* \"EService: Throttled: [UNIX timestamp]\" if there are too many concurrent requests. Try again after [timestamp].

Additional information can be found on our [support center](https://support.kraken.com/hc/en-us/articles/206548367-What-are-the-API-rate-limits-).

## Matching Engine Rate Limits

### Limits

Separate limits apply to the number of orders clients may have open in each pair at a time, and the speed with which they may add and cancel orders in each pair. These limits vary by the account verification tier: 

| Tier      | Max Num Orders | Max Ratecount | Ratecount Decay |
| ----------- | ----------- | ----------- | ----------- |
| Starter      | 60       | 60 | -1/sec |
| Intermediate   | 80 | 125 | -2.34/sec |
| Pro     | 225 | 180 | -3.75/sec |

### Penalties

The speed is controlled by a ratecounter for each (client, pair) which starts at zero, is incremented when penalties are applied, and decremented according to the decay rates above. A penalty is added to the ratecounter for each new order placed (`AddOrder`) or cancelled (`CancelOrder`, `CancelAll`, `CancelAllOrdersAfter`) on the pair. The cancellation penalty varies according to the lifetime of the order.

| Action       |      | <5sec  | <10sec | <15sec | <45sec | <90sec | <300s | >300s |
| ------------ | --   | -- | -- | -- | -- | -- | -- | -- |
| Add Order    | +1   |    |    |    |    |    |    |    |
| AddOrderBatch***| +(n/2)|    |    |    |    |    |    |    |
| Edit Order   |      | +6 | +5 | +4 | +2 | +1 | 0  | 0  |
| Cancel Order |      | +8 | +6 | +5 | +4 | +2 | +1 | 0 |
| CancelOrderBatch** |      | +8 | +6 | +5 | +4 | +2 | +1 | 0 |

*** n is number of orders in batch

** Rate limits penalty for CancelOrderBatch will be cumulative sum of individual orders penalty, accumulated upto max ratecount. In case, the cumulative penalty in the batch exceeds max ratecount, cancel requests in batch are still accepted.

> __Note:__ A client's exact ratecounter values can be monitored via the Websockets [openOrders](https://docs.kraken.com/websockets/#message-openOrders) feed.

### Errors

* \"EOrder:Orders limit exceeded\" if the number of open orders in a given pair is exceeded
* \"EOrder:Rate limit exceeded\" if the user's max ratecount is exceeded for a given pair

Additional information can be found on our [support center](https://support.kraken.com/hc/en-us/articles/360045239571).

# Changelog

* Oct 2023 - Added `max_fee` parameter to `Withdraw` and `minimum` field in response for `DepositMethods`.
* Sep 2023 - Added `start`, `end`, and `cursor` parameters to `DepositStatus`.
* Aug 2023 - Add Earn service documentation.
* July 2023 - Added private `BalanceEx` documentation.
* June 2023 - Added `amount` parameter to `DepositAddresses`, required for Bitcoin Lightning deposits. Added `count` parameter to `Trades`.
* May 2023 - Added `address` parameter to `Withdraw`. Added `since` parameter to `Spread`.
* Mar 2023 - Added `originators` paramater to `DepositStatus`.
* Feb 2023 - Added long and short margin position limits to `AssetPairs`.
* Feb 2023 - Specified minimum and maximum number of `txid`s and `userref`s for `CancelOrderBatch`.
* Feb 2023 - Specified maximum number of responses for `DepositStatus` and `WithdrawStatus`.
* Feb 2023 - Note to use `%2b` instead of `+` for URL encoding in `AddOrder` `starttm` and `expiretm` parameters.
* Jan 2023 - Removed requirement for `asset` in `DepositStatus` and `WithdrawalStatus`.
* Jan 2023 - Documented `reduce_only` parameter in `AddOrder` and `AddOrderBatch`.
* Jan 2023 - Added `consolidate_taker` to `TradesHistory`, `ClosedOrders`, and `QueryOrders`.
* Jan 2023 - Added Subaccounts section with `CreateSubaccount` and `AccountTransfer` endpoints.
* Jan 2023 - Added `trade_id` to private `TradesHistory`.
* Dec 2022 - Fixed `pair` parameter restriction on `TradeVolume`.
* Dec 2022 - `EditOrder` allowed on margin orders.
* Nov 2022 - Added `tick_size` and `status` parameters to `AssetPairs`, `status` and `collateral_value` to `Assets`, and `trade_id ` to public `Trades`.
* Oct 2022 - Documented `uv` (unexecuted value) field in `TradeBalance`.
* Oct 2022 - Added `costmin` trading parameter to `AssetPairs`.
* Oct 2022 - `Ticker` wildcard support - `pair` no longer required, no `pair` parameter returns tickers for all tradeable exchange assets.
* Sep 2022 - `AddOrder`/`EditOrder`/`AddOrderBatch` now support icebergs.
* July 2022 - Added support for restricting API keys to specified IP address(es)/range.
* June 2022 - Added custom self trade prevention options.
* May 2022 - New REST `AddOrderBatch` endpoint to send multiple new orders and `CancelOrderBatch` endpoint to cancel multiple open orders.
* Mar 2022 - New REST `EditOrder` endpoint to edit volume and price on open orders.
* Dec 2021 - Add REST `AddOrder` support for optional parameter `trigger` and values `last` (last traded price) and `index`.



**Note:** Most changes affecting our APIs or trading engine behaviour are currently being tracked on our [Websockets](https://docs.kraken.com/websockets/#changelog) changelog, until these documents are combined.

# Example API Clients

In order to achieve maximum performance, security and flexibility for your particular needs, we strongly encourage the implementation of this API with your own code, and to minimise reliance on third party software.

That being said, in order to aid with rapid development and prototyping, we're in the process of producing 'official' API clients in Python and Golang that will be actively maintained, coincident with the release of newer versions of both our Websockets and REST APIs. In the meantime, our Client Engagement team has compiled a number of [code snippets, examples and Postman collections](https://support.kraken.com/hc/en-us/sections/360003946512-Example-API-Code) that many find useful. 

### Third Party Software

Below are other third party API client code libraries that may be used when writing your own API client. Please keep in mind that Payward nor the third party authors are responsible for losses due to bugs or improper use of the APIs. Payward has performed an initial review of the safety of the third party code before listing them but cannot vouch for any changes added since then, or for those that may be stale. If you have concerns, please contact support.

| Language      | Link |
| ----------- | ----------- |
| C++ | [https://github.com/voidloop/krakenapi](https://github.com/voidloop/krakenapi) |
| Golang | [https://github.com/Beldur/kraken-go-api-client](https://github.com/Beldur/kraken-go-api-client) |
| NodeJS | [https://github.com/nothingisdead/npm-kraken-api](https://github.com/nothingisdead/npm-kraken-api) |
| Python 3 | [https://github.com/veox/python3-krakenex](https://github.com/veox/python3-krakenex)       |
| Python 2 | [https://github.com/veox/python2-krakenex](https://github.com/veox/python2-krakenex) |

Other 



## Installation & Usage

### Requirements

PHP 7.4 and later.
Should also work with PHP 8.0.

### Composer

To install the bindings via [Composer](https://getcomposer.org/), add the following to `composer.json`:

```json
{
  "repositories": [
    {
      "type": "vcs",
      "url": "https://github.com/GIT_USER_ID/GIT_REPO_ID.git"
    }
  ],
  "require": {
    "GIT_USER_ID/GIT_REPO_ID": "*@dev"
  }
}
```

Then run `composer install`

### Manual Installation

Download the files and include `autoload.php`:

```php
<?php
require_once('/path/to/OpenAPIClient-php/vendor/autoload.php');
```

## Getting Started

Please follow the [installation procedure](#installation--usage) and then run the following:

```php
<?php
require_once(__DIR__ . '/vendor/autoload.php');



// Configure API key authorization: API-Sign
$config = OpenAPI\Client\Configuration::getDefaultConfiguration()->setApiKey('API-Sign', 'YOUR_API_KEY');
// Uncomment below to setup prefix (e.g. Bearer) for API key, if needed
// $config = OpenAPI\Client\Configuration::getDefaultConfiguration()->setApiKeyPrefix('API-Sign', 'Bearer');

// Configure API key authorization: API-Key
$config = OpenAPI\Client\Configuration::getDefaultConfiguration()->setApiKey('API-Key', 'YOUR_API_KEY');
// Uncomment below to setup prefix (e.g. Bearer) for API key, if needed
// $config = OpenAPI\Client\Configuration::getDefaultConfiguration()->setApiKeyPrefix('API-Key', 'Bearer');


$apiInstance = new OpenAPI\Client\Api\AccountDataApi(
    // If you want use custom http client, pass your client which implements `GuzzleHttp\ClientInterface`.
    // This is optional, `GuzzleHttp\Client` will be used as default.
    new GuzzleHttp\Client(),
    $config
);
$nonce = 56; // int | Nonce used in construction of `API-Sign` header
$report = 'report_example'; // string | Type of data to export
$description = 'description_example'; // string | Description for the export
$format = 'CSV'; // string | File format to export
$fields = 'all'; // string | Comma-delimited list of fields to include  * `trades`: ordertxid, time, ordertype, price, cost, fee, vol, margin, misc, ledgers * `ledgers`: refid, time, type, aclass, asset, amount, fee, balance
$starttm = 56; // int | UNIX timestamp for report start time (default 1st of the current month)
$endtm = 56; // int | UNIX timestamp for report end time (default now)

try {
    $result = $apiInstance->addExport($nonce, $report, $description, $format, $fields, $starttm, $endtm);
    print_r($result);
} catch (Exception $e) {
    echo 'Exception when calling AccountDataApi->addExport: ', $e->getMessage(), PHP_EOL;
}

```

## API Endpoints

All URIs are relative to *https://api.kraken.com/0*

Class | Method | HTTP request | Description
------------ | ------------- | ------------- | -------------
*AccountDataApi* | [**addExport**](docs/Api/AccountDataApi.md#addexport) | **POST** /private/AddExport | Request Export Report
*AccountDataApi* | [**exportStatus**](docs/Api/AccountDataApi.md#exportstatus) | **POST** /private/ExportStatus | Get Export Report Status
*AccountDataApi* | [**getAccountBalance**](docs/Api/AccountDataApi.md#getaccountbalance) | **POST** /private/Balance | Get Account Balance
*AccountDataApi* | [**getClosedOrders**](docs/Api/AccountDataApi.md#getclosedorders) | **POST** /private/ClosedOrders | Get Closed Orders
*AccountDataApi* | [**getExtendedBalance**](docs/Api/AccountDataApi.md#getextendedbalance) | **POST** /private/BalanceEx | Get Extended Balance
*AccountDataApi* | [**getLedgers**](docs/Api/AccountDataApi.md#getledgers) | **POST** /private/Ledgers | Get Ledgers Info
*AccountDataApi* | [**getLedgersInfo**](docs/Api/AccountDataApi.md#getledgersinfo) | **POST** /private/QueryLedgers | Query Ledgers
*AccountDataApi* | [**getOpenOrders**](docs/Api/AccountDataApi.md#getopenorders) | **POST** /private/OpenOrders | Get Open Orders
*AccountDataApi* | [**getOpenPositions**](docs/Api/AccountDataApi.md#getopenpositions) | **POST** /private/OpenPositions | Get Open Positions
*AccountDataApi* | [**getOrdersInfo**](docs/Api/AccountDataApi.md#getordersinfo) | **POST** /private/QueryOrders | Query Orders Info
*AccountDataApi* | [**getTradeBalance**](docs/Api/AccountDataApi.md#gettradebalance) | **POST** /private/TradeBalance | Get Trade Balance
*AccountDataApi* | [**getTradeHistory**](docs/Api/AccountDataApi.md#gettradehistory) | **POST** /private/TradesHistory | Get Trades History
*AccountDataApi* | [**getTradeVolume**](docs/Api/AccountDataApi.md#gettradevolume) | **POST** /private/TradeVolume | Get Trade Volume
*AccountDataApi* | [**getTradesInfo**](docs/Api/AccountDataApi.md#gettradesinfo) | **POST** /private/QueryTrades | Query Trades Info
*AccountDataApi* | [**removeExport**](docs/Api/AccountDataApi.md#removeexport) | **POST** /private/RemoveExport | Delete Export Report
*AccountDataApi* | [**retrieveExport**](docs/Api/AccountDataApi.md#retrieveexport) | **POST** /private/RetrieveExport | Retrieve Data Export
*EarnApi* | [**allocateStrategy**](docs/Api/EarnApi.md#allocatestrategy) | **POST** /private/Earn/Allocate | Allocate Earn Funds
*EarnApi* | [**deallocateStrategy**](docs/Api/EarnApi.md#deallocatestrategy) | **POST** /private/Earn/Deallocate | Deallocate Earn Funds
*EarnApi* | [**getAllocateStrategyStatus**](docs/Api/EarnApi.md#getallocatestrategystatus) | **POST** /private/Earn/AllocateStatus | Get Allocation Status
*EarnApi* | [**getDeallocateStrategyStatus**](docs/Api/EarnApi.md#getdeallocatestrategystatus) | **POST** /private/Earn/DeallocateStatus | Get Deallocation Status
*EarnApi* | [**listAllocations**](docs/Api/EarnApi.md#listallocations) | **POST** /private/Earn/Allocations | List Earn Allocations
*EarnApi* | [**listStrategies**](docs/Api/EarnApi.md#liststrategies) | **POST** /private/Earn/Strategies | List Earn Strategies
*FundingApi* | [**cancelWithdrawal**](docs/Api/FundingApi.md#cancelwithdrawal) | **POST** /private/WithdrawCancel | Request Withdrawal Cancelation
*FundingApi* | [**getDepositAddresses**](docs/Api/FundingApi.md#getdepositaddresses) | **POST** /private/DepositAddresses | Get Deposit Addresses
*FundingApi* | [**getDepositMethods**](docs/Api/FundingApi.md#getdepositmethods) | **POST** /private/DepositMethods | Get Deposit Methods
*FundingApi* | [**getStatusRecentDeposits**](docs/Api/FundingApi.md#getstatusrecentdeposits) | **POST** /private/DepositStatus | Get Status of Recent Deposits
*FundingApi* | [**getStatusRecentWithdrawals**](docs/Api/FundingApi.md#getstatusrecentwithdrawals) | **POST** /private/WithdrawStatus | Get Status of Recent Withdrawals
*FundingApi* | [**getWithdrawalInformation**](docs/Api/FundingApi.md#getwithdrawalinformation) | **POST** /private/WithdrawInfo | Get Withdrawal Information
*FundingApi* | [**walletTransfer**](docs/Api/FundingApi.md#wallettransfer) | **POST** /private/WalletTransfer | Request Wallet Transfer
*FundingApi* | [**withdrawFunds**](docs/Api/FundingApi.md#withdrawfunds) | **POST** /private/Withdraw | Withdraw Funds
*MarketDataApi* | [**getAssetInfo**](docs/Api/MarketDataApi.md#getassetinfo) | **GET** /public/Assets | Get Asset Info
*MarketDataApi* | [**getOHLCData**](docs/Api/MarketDataApi.md#getohlcdata) | **GET** /public/OHLC | Get OHLC Data
*MarketDataApi* | [**getOrderBook**](docs/Api/MarketDataApi.md#getorderbook) | **GET** /public/Depth | Get Order Book
*MarketDataApi* | [**getRecentSpreads**](docs/Api/MarketDataApi.md#getrecentspreads) | **GET** /public/Spread | Get Recent Spreads
*MarketDataApi* | [**getRecentTrades**](docs/Api/MarketDataApi.md#getrecenttrades) | **GET** /public/Trades | Get Recent Trades
*MarketDataApi* | [**getServerTime**](docs/Api/MarketDataApi.md#getservertime) | **GET** /public/Time | Get Server Time
*MarketDataApi* | [**getSystemStatus**](docs/Api/MarketDataApi.md#getsystemstatus) | **GET** /public/SystemStatus | Get System Status
*MarketDataApi* | [**getTickerInformation**](docs/Api/MarketDataApi.md#gettickerinformation) | **GET** /public/Ticker | Get Ticker Information
*MarketDataApi* | [**getTradableAssetPairs**](docs/Api/MarketDataApi.md#gettradableassetpairs) | **GET** /public/AssetPairs | Get Tradable Asset Pairs
*StakingApi* | [**getStakingAssetInfo**](docs/Api/StakingApi.md#getstakingassetinfo) | **POST** /private/Staking/Assets | List of Stakeable Assets
*StakingApi* | [**getStakingPendingDeposits**](docs/Api/StakingApi.md#getstakingpendingdeposits) | **POST** /private/Staking/Pending | Get Pending Staking Transactions
*StakingApi* | [**getStakingTransactions**](docs/Api/StakingApi.md#getstakingtransactions) | **POST** /private/Staking/Transactions | List of Staking Transactions
*StakingApi* | [**stake**](docs/Api/StakingApi.md#stake) | **POST** /private/Stake | Stake Asset
*StakingApi* | [**unstake**](docs/Api/StakingApi.md#unstake) | **POST** /private/Unstake | Unstake Asset
*SubaccountsApi* | [**accountTransfer**](docs/Api/SubaccountsApi.md#accounttransfer) | **POST** /private/AccountTransfer | Account Transfer
*SubaccountsApi* | [**createSubaccount**](docs/Api/SubaccountsApi.md#createsubaccount) | **POST** /private/CreateSubaccount | Create Subaccount
*TradingApi* | [**addOrder**](docs/Api/TradingApi.md#addorder) | **POST** /private/AddOrder | Add Order
*TradingApi* | [**addOrderBatch**](docs/Api/TradingApi.md#addorderbatch) | **POST** /private/AddOrderBatch | Add Order Batch
*TradingApi* | [**cancelAllOrders**](docs/Api/TradingApi.md#cancelallorders) | **POST** /private/CancelAll | Cancel All Orders
*TradingApi* | [**cancelAllOrdersAfter**](docs/Api/TradingApi.md#cancelallordersafter) | **POST** /private/CancelAllOrdersAfter | Cancel All Orders After X
*TradingApi* | [**cancelOrder**](docs/Api/TradingApi.md#cancelorder) | **POST** /private/CancelOrder | Cancel Order
*TradingApi* | [**cancelOrderBatch**](docs/Api/TradingApi.md#cancelorderbatch) | **POST** /private/CancelOrderBatch | Cancel Order Batch
*TradingApi* | [**editOrder**](docs/Api/TradingApi.md#editorder) | **POST** /private/EditOrder | Edit Order
*WebsocketsAuthenticationApi* | [**getWebsocketsToken**](docs/Api/WebsocketsAuthenticationApi.md#getwebsocketstoken) | **POST** /private/GetWebSocketsToken | Get Websockets Token

## Models

- [AccountTransfer200Response](docs/Model/AccountTransfer200Response.md)
- [AccountTransfer200ResponseResult](docs/Model/AccountTransfer200ResponseResult.md)
- [Add2](docs/Model/Add2.md)
- [AddExport200Response](docs/Model/AddExport200Response.md)
- [AddExport200ResponseResult](docs/Model/AddExport200ResponseResult.md)
- [Address](docs/Model/Address.md)
- [Addresses2](docs/Model/Addresses2.md)
- [AddressesAmount](docs/Model/AddressesAmount.md)
- [AllocateStrategy200Response](docs/Model/AllocateStrategy200Response.md)
- [AllocateStrategyRequest](docs/Model/AllocateStrategyRequest.md)
- [AllocateStrategyRequestNonce](docs/Model/AllocateStrategyRequestNonce.md)
- [Asset](docs/Model/Asset.md)
- [AssetLock](docs/Model/AssetLock.md)
- [AssetMinimumAmount](docs/Model/AssetMinimumAmount.md)
- [AssetRewards](docs/Model/AssetRewards.md)
- [Balance2](docs/Model/Balance2.md)
- [Balance3](docs/Model/Balance3.md)
- [Balance4](docs/Model/Balance4.md)
- [Balanceex](docs/Model/Balanceex.md)
- [Balanceex2](docs/Model/Balanceex2.md)
- [Batchadd2](docs/Model/Batchadd2.md)
- [BatchaddOrdersInner](docs/Model/BatchaddOrdersInner.md)
- [BatchcancelOrdersInner](docs/Model/BatchcancelOrdersInner.md)
- [BatchcancelOrdersInnerTxid](docs/Model/BatchcancelOrdersInnerTxid.md)
- [CancelAllOrders200Response](docs/Model/CancelAllOrders200Response.md)
- [CancelAllOrders200ResponseResult](docs/Model/CancelAllOrders200ResponseResult.md)
- [CancelAllOrdersAfter200Response](docs/Model/CancelAllOrdersAfter200Response.md)
- [CancelAllOrdersAfter200ResponseResult](docs/Model/CancelAllOrdersAfter200ResponseResult.md)
- [CancelTxid](docs/Model/CancelTxid.md)
- [CancelWithdrawal200Response](docs/Model/CancelWithdrawal200Response.md)
- [Closed](docs/Model/Closed.md)
- [Closed2](docs/Model/Closed2.md)
- [ClosedOrders](docs/Model/ClosedOrders.md)
- [CreateSubaccount200Response](docs/Model/CreateSubaccount200Response.md)
- [DeallocateStrategyRequest](docs/Model/DeallocateStrategyRequest.md)
- [Deposit](docs/Model/Deposit.md)
- [Depth](docs/Model/Depth.md)
- [Edit2](docs/Model/Edit2.md)
- [EditTxid](docs/Model/EditTxid.md)
- [ExportStatus200Response](docs/Model/ExportStatus200Response.md)
- [ExportStatus200ResponseResultInner](docs/Model/ExportStatus200ResponseResultInner.md)
- [ExtendedBalance](docs/Model/ExtendedBalance.md)
- [Fees](docs/Model/Fees.md)
- [GetAllocateStrategyStatus200Response](docs/Model/GetAllocateStrategyStatus200Response.md)
- [GetAllocateStrategyStatus200ResponseResult](docs/Model/GetAllocateStrategyStatus200ResponseResult.md)
- [GetAllocateStrategyStatusRequest](docs/Model/GetAllocateStrategyStatusRequest.md)
- [GetOpenPositions200Response](docs/Model/GetOpenPositions200Response.md)
- [GetOpenPositions200ResponseResultValue](docs/Model/GetOpenPositions200ResponseResultValue.md)
- [GetStakingAssetInfo200Response](docs/Model/GetStakingAssetInfo200Response.md)
- [GetStakingPendingDeposits200Response](docs/Model/GetStakingPendingDeposits200Response.md)
- [GetSystemStatus200Response](docs/Model/GetSystemStatus200Response.md)
- [GetSystemStatus200ResponseResult](docs/Model/GetSystemStatus200ResponseResult.md)
- [GetTradableAssetPairs200Response](docs/Model/GetTradableAssetPairs200Response.md)
- [GetTradesInfo200Response](docs/Model/GetTradesInfo200Response.md)
- [GetWebsocketsToken200Response](docs/Model/GetWebsocketsToken200Response.md)
- [GetWebsocketsToken200ResponseResult](docs/Model/GetWebsocketsToken200ResponseResult.md)
- [History](docs/Model/History.md)
- [History2](docs/Model/History2.md)
- [Info](docs/Model/Info.md)
- [Info2](docs/Model/Info2.md)
- [Info3](docs/Model/Info3.md)
- [Info5](docs/Model/Info5.md)
- [Ledger](docs/Model/Ledger.md)
- [LedgersInfo](docs/Model/LedgersInfo.md)
- [ListAllocations200Response](docs/Model/ListAllocations200Response.md)
- [ListAllocations200ResponseResult](docs/Model/ListAllocations200ResponseResult.md)
- [ListAllocations200ResponseResultItemsInner](docs/Model/ListAllocations200ResponseResultItemsInner.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocated](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocated.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedBonding](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedBonding.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedBondingAllocationsInner](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedBondingAllocationsInner.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedExitQueue](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedExitQueue.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedExitQueueAllocationsInner](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedExitQueueAllocationsInner.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedPending](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedPending.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedTotal](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedTotal.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedUnbonding](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedUnbonding.md)
- [ListAllocations200ResponseResultItemsInnerAmountAllocatedUnbondingAllocationsInner](docs/Model/ListAllocations200ResponseResultItemsInnerAmountAllocatedUnbondingAllocationsInner.md)
- [ListAllocations200ResponseResultItemsInnerPayout](docs/Model/ListAllocations200ResponseResultItemsInnerPayout.md)
- [ListAllocations200ResponseResultItemsInnerPayoutAccumulatedReward](docs/Model/ListAllocations200ResponseResultItemsInnerPayoutAccumulatedReward.md)
- [ListAllocations200ResponseResultItemsInnerPayoutEstimatedReward](docs/Model/ListAllocations200ResponseResultItemsInnerPayoutEstimatedReward.md)
- [ListAllocations200ResponseResultItemsInnerTotalRewarded](docs/Model/ListAllocations200ResponseResultItemsInnerTotalRewarded.md)
- [ListAllocationsRequest](docs/Model/ListAllocationsRequest.md)
- [ListStrategies200Response](docs/Model/ListStrategies200Response.md)
- [ListStrategies200ResponseResult](docs/Model/ListStrategies200ResponseResult.md)
- [ListStrategies200ResponseResultItemsInner](docs/Model/ListStrategies200ResponseResultItemsInner.md)
- [ListStrategies200ResponseResultItemsInnerAllocationFee](docs/Model/ListStrategies200ResponseResultItemsInnerAllocationFee.md)
- [ListStrategies200ResponseResultItemsInnerAprEstimate](docs/Model/ListStrategies200ResponseResultItemsInnerAprEstimate.md)
- [ListStrategies200ResponseResultItemsInnerAutoCompound](docs/Model/ListStrategies200ResponseResultItemsInnerAutoCompound.md)
- [ListStrategies200ResponseResultItemsInnerAutoCompoundOneOf](docs/Model/ListStrategies200ResponseResultItemsInnerAutoCompoundOneOf.md)
- [ListStrategies200ResponseResultItemsInnerAutoCompoundOneOf1](docs/Model/ListStrategies200ResponseResultItemsInnerAutoCompoundOneOf1.md)
- [ListStrategies200ResponseResultItemsInnerAutoCompoundOneOf2](docs/Model/ListStrategies200ResponseResultItemsInnerAutoCompoundOneOf2.md)
- [ListStrategies200ResponseResultItemsInnerDeallocationFee](docs/Model/ListStrategies200ResponseResultItemsInnerDeallocationFee.md)
- [ListStrategies200ResponseResultItemsInnerLockType](docs/Model/ListStrategies200ResponseResultItemsInnerLockType.md)
- [ListStrategies200ResponseResultItemsInnerLockTypeOneOf](docs/Model/ListStrategies200ResponseResultItemsInnerLockTypeOneOf.md)
- [ListStrategies200ResponseResultItemsInnerLockTypeOneOf1](docs/Model/ListStrategies200ResponseResultItemsInnerLockTypeOneOf1.md)
- [ListStrategies200ResponseResultItemsInnerLockTypeOneOf2](docs/Model/ListStrategies200ResponseResultItemsInnerLockTypeOneOf2.md)
- [ListStrategies200ResponseResultItemsInnerLockTypeOneOf3](docs/Model/ListStrategies200ResponseResultItemsInnerLockTypeOneOf3.md)
- [ListStrategies200ResponseResultItemsInnerYieldSource](docs/Model/ListStrategies200ResponseResultItemsInnerYieldSource.md)
- [ListStrategies200ResponseResultItemsInnerYieldSourceOneOf](docs/Model/ListStrategies200ResponseResultItemsInnerYieldSourceOneOf.md)
- [ListStrategies200ResponseResultItemsInnerYieldSourceOneOf1](docs/Model/ListStrategies200ResponseResultItemsInnerYieldSourceOneOf1.md)
- [ListStrategiesRequest](docs/Model/ListStrategiesRequest.md)
- [Lock](docs/Model/Lock.md)
- [Method](docs/Model/Method.md)
- [Methods2](docs/Model/Methods2.md)
- [Ohlc](docs/Model/Ohlc.md)
- [OhlcResult](docs/Model/OhlcResult.md)
- [Open](docs/Model/Open.md)
- [Open2](docs/Model/Open2.md)
- [OpenOrders](docs/Model/OpenOrders.md)
- [Order](docs/Model/Order.md)
- [OrderAdded](docs/Model/OrderAdded.md)
- [OrderAddedDescr](docs/Model/OrderAddedDescr.md)
- [OrderBookEntry](docs/Model/OrderBookEntry.md)
- [OrderBookEntryAsksInnerInner](docs/Model/OrderBookEntryAsksInnerInner.md)
- [OrderBookEntryBidsInnerInner](docs/Model/OrderBookEntryBidsInnerInner.md)
- [OrderDescription](docs/Model/OrderDescription.md)
- [OrderEdited](docs/Model/OrderEdited.md)
- [OrderEditedDescr](docs/Model/OrderEditedDescr.md)
- [OrdersInner](docs/Model/OrdersInner.md)
- [OrdersInnerDescr](docs/Model/OrdersInnerDescr.md)
- [Ordertype](docs/Model/Ordertype.md)
- [Pairs](docs/Model/Pairs.md)
- [Query2](docs/Model/Query2.md)
- [Query2ResultValue](docs/Model/Query2ResultValue.md)
- [Query3](docs/Model/Query3.md)
- [Recent2](docs/Model/Recent2.md)
- [Recent4](docs/Model/Recent4.md)
- [RecentCursor](docs/Model/RecentCursor.md)
- [RemoveExport200Response](docs/Model/RemoveExport200Response.md)
- [RemoveExport200ResponseResult](docs/Model/RemoveExport200ResponseResult.md)
- [Result](docs/Model/Result.md)
- [RetrieveExport200Response](docs/Model/RetrieveExport200Response.md)
- [ServerTime](docs/Model/ServerTime.md)
- [Spread2](docs/Model/Spread2.md)
- [Spread2Result](docs/Model/Spread2Result.md)
- [SpreadInnerInner](docs/Model/SpreadInnerInner.md)
- [Stake200Response](docs/Model/Stake200Response.md)
- [Stake200ResponseResult](docs/Model/Stake200ResponseResult.md)
- [TickDataInnerInner](docs/Model/TickDataInnerInner.md)
- [Ticker](docs/Model/Ticker.md)
- [Ticker2](docs/Model/Ticker2.md)
- [Time](docs/Model/Time.md)
- [Trade2](docs/Model/Trade2.md)
- [TradeInnerInner](docs/Model/TradeInnerInner.md)
- [TradeVolume](docs/Model/TradeVolume.md)
- [Trades](docs/Model/Trades.md)
- [TradesResult](docs/Model/TradesResult.md)
- [Transaction](docs/Model/Transaction.md)
- [Volume](docs/Model/Volume.md)
- [WalletTransfer200Response](docs/Model/WalletTransfer200Response.md)
- [WalletTransfer200ResponseResult](docs/Model/WalletTransfer200ResponseResult.md)
- [Withdrawal2](docs/Model/Withdrawal2.md)
- [Withdrawal2Result](docs/Model/Withdrawal2Result.md)
- [Withdrawal3](docs/Model/Withdrawal3.md)
- [WithdrawalInfo](docs/Model/WithdrawalInfo.md)

## Authorization

Authentication schemes defined for the API:
### API-Key

- **Type**: API key
- **API key parameter name**: API-Key
- **Location**: HTTP header


### API-Sign

- **Type**: API key
- **API key parameter name**: API-Sign
- **Location**: HTTP header


## Tests

To run the tests, use:

```bash
composer install
vendor/bin/phpunit
```

## Author



## About this package

This PHP package is automatically generated by the [OpenAPI Generator](https://openapi-generator.tech) project:

- API version: `1.1.0`
- Build package: `org.openapitools.codegen.languages.PhpClientCodegen`
# kraken-php
# kraken-php
# kraken-php
# kraken-php
# kraken-php
# kraken-php
# kraken-php
# kraken-php
# kraken-php
# kraken-php
