# MQService

The MQService is a process running at each Superalgos Node that enables the communication between system components in general. Components can send messages between them through this service or raise events for other to listen and react to them.

This does not mean to eliminate micro-service API access but to complement it. Well known components are better to be accesed thouth their static APIs. Components like bots instances, which are not allways running might prefer to communicate themselves through this MQService.

## Messages Formats

We are going to be slowly transitioning the current infraestructure to use this messaging service. For the first set of use cases we already know a message format we will use and is the one described in this section. Expect this to be extended / improved as we move fordward.

### Trading Orders related Messages

With this format we will enable communications between the Simulation Engine, Simulation Executor, Trading Cockpit and Trading Assistant. We expect that messages flowing between these components to be in this agreed format, while messages logged into files should go through a minification process to save space. The minification process criteria is not the about taking the messages to their absolute minimun weight but to a balance where the weight is small but at the same time understandable by a human with enough context while reading them from a file.

```
{
  "id": 12345, // This is a unique Id within the system component that originated the message.
  "from": "Simulation Executor|Trading Cockpit|Simulation Engine|Trading Assistant", // --> "EX|CO|EN|AS"
  "to": "Simulation Executor|Trading Cockpit|Simulation Engine|Trading Assistant", // --> "EX|CO|EN|AS"
  "messageType": "Heart Beat|Order Authorization Request|Order Authorization Response|Order|Order Update", // --> "HBT|ARQ|ARS|ORD|UPT"
  "dateTime": 1551579300000, // This is the timestamp of the message.
  "order": {
    "id": 31231, // This is a unique Id within the system component that originated the order.
    "creator": "Simulation Engine|Human Trader", // --> "S|H"
    "dateTime": 1551579300000, // This is the datetime when the order was created. After the order travels from one system componet to the other it becomes diffrent from the message.datetime.
    "owner": "Node/Team/User",  // --> "N|T|U" Once we have our login system at a node level, here it will come this info.
    "exchange": "Poloniex",
    "market": "BTC/USDST",
    "marginEnabled": true, // true | false --> 1|0
    "type": "Market|Limit|Stop", // --> "M|L|S"
    "rate": 6368.10044495,
    "stop": 6463.62195162425,
    "takeProfit": 6463.62195162425,
    "direction": "Sell|Buy", // --> "Sell|Buy"
    "size": "floating-point number|All", // All means to use all the available balance.
    "status": "Signaled|Manual Authorized|Manual Not Authorized|Auto Authorized|Auto Not Authorized|Executing|Cancelled|Filled|Partially Filled|Discarded|Placed", // --> "SIG|MAU|MNA|AAU|ANA|EXE|CAN|FIL|PRT|DIS|PLA"
    "sizeFilled": 0.00045,
    "exitOutcome": "Stop Loss|Take Profit" // --> "SL|TP"
  }
}
```

When writing this information in files for logging or audit purposes, we will turn int into an array of fields where the order of them is relevant. The following is a record example of how the previous object would be recorded into a file.

```
let record = [
  90,
  "EN",
  "EX",
  "ORD",
  1553850096262,
  [
    1,
    "S",
    155385234234,
    "U",
    "Poloniex",
    "BTC/USDT",
    0,
    "L",
    6286.707,
    6381.007,
    0,
    "Sell",
    0,
    "SIG",
    0,
    "SL"
  ]
]
```

Heartbeat example:

```
[1,"EN","EX","HBT",1553850096045,[0,"","","","",0,"",0,0,0,"","","",0,""]]
```

### Lifecycle for the Orders to enter a Trade

An order can be created by the Simulation Engine or by a Human Trader.

It is created by the Simulation Engine as a result of processing the strategies defined at the Strategizer. In this case it's initial status is "Signaled". Once the order reaches the Simulation Executor in this initial state, it needs to be authorized at the Trading Cockpit. There the Human Trader can set it into the possible status of "Manually Authorized" or "Manually Not Authorized". It can also happen that the Human Traders have set some Autopilot instructions, in which case the orders could get into "Auto Authorized" or "Auto Not Authorized".

If it is created by a Human Trader from the Trading Cockpit the initial status is "Ordered". Once it gets into the Simulation Executor this status does not requires further authorization.

The Simulation Executor then send the order to the Trading Assistant at which point its status become "Executing". From there and depending on what happens inside the exchange the order can end up being "Cancelled", "Filled" or "Partially Filled". In the last case, and with enought time, the order could finally turn into "Filled".

There are a few situations in which the Simulation Executor might need to reject an order: for example if it forwared an order previously authorized to the Trading Assistant and it receives a new one from the Simulation Engine. It might also happen that it receives an order from the Human Trader while an order is already in "Executing" state or one that follows them. In these scenarios, the received order status is set to "Discarded".

### Lifecycle for the Orders to exit a Trade

The Simulation Executor will be managing the Stop and Take Profit levels according to the information that receives from the Simulation Engine. To do that, it places a Limit Order and a Stop Order at the Exchange through the Trading Assistant. As times goes by, it moves those orders according to the feed it is receiving from the Simulation Engine.

Once those orders are created and sent to the Trading Assistant, their status becomes "Placed". We dont expect those two to be executed inmidiatelly in most cases. From there they could turn into "Filled", "Partially Filled" or "Cancelled" depending on what happens inside the exchange.

In our current version there is no manual or automated human intervention in this situation, meaning that the Trading Cokpit is not involved.

## Usage

```
npm install @superalgos/mqservice
```

After npm install, use the following code to use the library.

```
const {
  MESSAGE_ENTITY, MESSAGE_TYPE, ORDER_CREATOR, ORDER_TYPE,
  ORDER_OWNER, ORDER_DIRECTION, ORDER_STATUS, ORDER_EXIT_OUTCOME,
  createRecord, getRecord, getExpandedRecord
} = require("@superalgos/mqservice")

let record = createRecord(90, MESSAGE_ENTITY.SimulationEngine, MESSAGE_ENTITY.SimulationExecutor,
    MESSAGE_TYPE.Order, 1553850096262, 1, ORDER_CREATOR.SimulationEngine, 155385234234, ORDER_OWNER.User,
    "Poloniex", "BTC/USDT", 0, ORDER_TYPE.Limit, 6286.707, 6381.007, 0, ORDER_DIRECTION.Sell, 0,
    ORDER_STATUS.Signaled, 0, ORDER_EXIT_OUTCOME.StopLoss)
```

### createRecord

Returns an Array that can be used to write the information to a file.

### getRecord

Receives an array identical to the one created with createRecord and converts it to a JSON object with some fields values abbreviated.

### getExpandedRecord

Receives an array identical to the one created with createRecord and converts it to a JSON object with some fields values with full lenght.


