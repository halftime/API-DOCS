# A repository for documentation of the CoinMetro Crypto Exchange

## REST API

The POSTman documentation of the REST API can be found @ https://documenter.getpostman.com/view/3653795/SVfWN6KS

## WebSockets

The WebSockets can be accessed at

`wss://api.coinmetro.com/open/ws` (demo)\
`wss://api.coinmetro.com/ws` (production)

The connection supports two arguments in the query string

`pairs`: Comma separated list of pairs for which book update subscription is requested\
`token`: JWT token generated through one of the login paths. For long-lived tokens, use `token={deviceId}:{token}`

Both parameters are optional.

Example query string:

`wss://api.coinmetro.com/open/ws?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImU3Njk5OTNjIiwiZXhwIjoxNTg2MjgzNzEyNDI1LCJpZCI6IjVlOGNiZGI4OWM5NzQ5MjNjOGNhYWNhZSIsImlwIjoiMTQxLjguNDcuMTIzIiwiaWF0IjoxNTg2MjgxOTEyfQ.XA7tZGwmfRClzlm7SyB9fDfQl-SFKoPnjisOPMtY0sE&pairs=BTCEUR,LTCEUR`

The WebSockets emit the following messages:

#### Order Status Message
It's only received on *authenticated* websockets (i.e. the connection string includes a valid `token` query parameter).

Follows the same format as returned by REST API endpoints
```
{ 
  orderStatus: { 
    orderType: 'market' | 'limit',
    buyingCurrency: string,
    sellingCurrency: string,
    sellingQty: number,
    userID: string,
    orderID: string,
    timeInForce: number, // 1=GTC,2=GTD,2=IOC,3=IOC,4=FOK
    boughtQty: number,
    soldQty: number,
    creationTime: number,
    seqNumber: number,
    firstFillTime: number, // Millisecond since Epoch
    lastFillTime: number, // Millisecond since Epoch
    fills: { 
      seqNumber: number, // Millisecond since Epoch
      timestamp: number, // Millisecond since Epoch
      qty: number,
      price: number,
      side: 'buy' | 'sell'  
    }[],
    completionTime: number, // Millisecond since Epoch
    takerQty: number 
  }
}
```

#### Wallet Update Message
It's only received on *authenticated* websockets (i.e. the connection string includes a valid `token` query parameter).

Follows the same format as the `walletHistory` entries in the REST API, additionally the **walletId**, **currency**, **label** and the most current **balance** are indicated
```
{ 
  walletUpdate:{ 
    walletId: string,
    currency: string,
    label: string,
    userId: string,
    description: string,
    amount: number,
    JSONdata: { 
      // Contains data that is specific to the transaction, 
      // e.g. the references for deposit/withdraw operations, 
      // fees for orders, etc
      price: string, // ex. '6730.17 BTC/EUR',
      fees: number,
      notes: string, 
    },
    timestamp: string, // ISODate
    balance: number
  } 
}
```

#### Book Update message
It's only received for *subscribed* pairs (i.e. the connection string includes a valid `pairs` query parameter).

Every update includes a CRC32 `checksum` of the book so that it can be verified that the client and the server are in sync. 
The string to be checksummed is generated by concatenating the sorted list of all non-null asks and the list of all non-null bids. 
The sort is in ascending lexicographical order.

```
CRC32(
      Object.keys(book.ask).filter(p => book.ask[p]).sort().map(p => `${p}${book.ask[p]}`).join("") +
      Object.keys(book.bid).filter(p => book.bid[p]).sort().map(p => `${p}${book.bid[p]}`).join("")
    )
```

For example, if to `book` object is constituted as follows:
```
{
	bid: {
		"21.01": 100
		"100.01": 20
	}
	ask: {
		"120.3": 20
		"150.2": 1
	}
}
```
The string to checksum will be

**120.3**20**150.2**1**100.01**20**21.01**100

`120.320150.21100.012021.01100`

With a final checksum of `2145038063`

```
{
  bookUpdate: {
    pair: string,
    seqNumber": number,
    ask: {
      [string]: number
      // The keys are price leves, and the values are quantity deltas in the base currency, example
      // 6730.18: -0.15891774,
      // 6721.88: -0.34108226
    },
    bid: {
      [string]: number
      // The keys are price leves, and the values are quantity deltas in the base currency, example
      // 6730.18: -0.15891774,
      // 6721.88: -0.34108226
    }
    checksum: number
   }
}
```

#### Tick message
It's received for all pairs, always. Every tick also includes the most current **ask** and **bid**.

```
{
  tick: {
    pair: string,
    price: number,
    qty: number,
    timestamp: number, // Milliseconds since Epoch
    seqNum: number,
    ask: number,
    bid: number
  }
}
```

## Examples

The folder /examples contains some basic JS examples showing the capabilities of the API.

*dumb-bot.js*: A bot that sells BTC every few seconds and prints some execution data. When BTC balance goes below a certain threshold, it buys BTC back.  
Demonstrates basic authentication, market order creation, balance polling, fill polling.  
  
*boring-bot.js*: A bot that every few seconds posts 2 limit orders, 2% above and 2% below current price, then prints the book highlighting its own orders.  
Demonstrates price polling, limit order creation, order status polling, order cancellation, basic book polling.

*ws-bot.js*: A bot that connects to the WebSocket, sends and order, then outputs the updates received on the WebSocket.

