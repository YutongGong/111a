# Tutorial on Using the Simulation Matching Engine
In medium and high-frequency strategies, we often encounter a situation: some strategies that perform well in backtesting do not achieve expected results in actual trading. One very important reason is the underestimation of trading costs. To simulate the costs in actual trading more accurately during backtesting, we can introduce a simulated matching system. DolphinDB provides a simulated matching engine plugin, enabling us to more reasonably assess and predict the performance of strategies in actual trading and to make corresponding optimizations.

### Introduction

The main function of the simulated matching engine is to simulate the actions of placing or canceling orders by users at a certain point in time and obtain corresponding trading results. Different from the built-in stream data engine in DolphinDB Server, the simulated matching engine is provided as a plugin service, and its logical architecture is shown in the figure below.

![images/MatchEngineSimulator/DM_20230922142321_001.png](images/MatchEngineSimulator/DM_20230922142321_001.png)

The simulated matching engine takes market data (snapshot data or tick data) and user commission orders (buy or sell) as inputs. Based on the order matching rules, it simulates the matching, and the results of the orders (including partial transaction results, rejected orders, and canceled orders) are output to the trade detail output table. The unexecuted parts wait to be matched with subsequent market data for transaction or wait to be canceled.

- Supports settings for transaction ratio and latency.
- When multiple user commission orders in the same direction are matched simultaneously, they are matched and transacted following the principle of price priority and time priority.
- When the market data is tick data, the matching engine synthesizes the market order book in real-time. When the user commission order arrives, it is instantly matched with the order book to generate transaction information. The unexecuted parts, together with subsequent market commission orders and transaction orders, are matched and transacted following the principle of price priority and time priority.
- When the market data is snapshot data, when the user commission order arrives, it is instantly matched with the order book to generate transaction information. The unexecuted parts, according to the configuration of the matching engine, can choose to be matched and transacted with subsequent snapshot market data.
  - Matching mode one, match with the latest transaction price and the counterpart's market quotes according to the configured ratio.
  - Matching mode two, match and transact with the interval's transaction list and the counterpart's market quotes.
From the perspective of user commission orders, it includes limit orders (Limit Order), market orders (Market Order), and cancel orders (Cancel Order). Different combinations of order types and market data types correspond to different matching rules, as detailed in the subsequent sections.

### Table Structure

When creating a simulated matching engine, it is necessary to specify the table structure information for the input tables (market data, user commission orders) and the output tables (transaction details).

For input tables, the simulated matching engine requires certain specified field names. If the field names in the custom market data table are inconsistent with the requirements of the engine, a quotationColMap dictionary can be used for mapping; similarly, if the field names in the custom user commission order data table are inconsistent with the requirements of the engine, a userOrderColMap dictionary can be used for mapping.

For output tables, the simulated matching engine does not require specific field names.

#### Snapshot Market Data

For snapshot mode, the required columns for the market table are as follows:

| **Name**        | **Type**  | **Meaning**                 |
| --------------- | --------- | ------------------------ |
| symbol          | SYMBOL    | Stock subject                 |
| symbolSource    | SYMBOL    | Securities market: Shenzhen Stock Exchange, Shanghai Stock Exchange |
| time            | TIMESTAMP | Time                     |
| lastPrice       | DOUBLE    | Latest price                   |
| upperLimitPrice | DOUBLE    | Upper limit price              |
| lowerLimitPrice | DOUBLE    | Lower limit price                 |
| totalBidQty     | LONG      | Total bid quantity in the interval  |
| totalOfferQty   | LONG      | Total offer quantity in the interval    |
| bidPrice        | DOUBLE[]  | Bid price list           |
| bidQty          | LONG[]    | Bid quantity list             |
| offerPrice      | DOUBLE[]  | Offer price list      |
| offerQty        | LONG[]    | Offer quantity list             |
| tradePrice      | DOUBLE[]  | Interval transaction price list        |
| tradeQty        | LONG[]    |Interval transaction volume list         |

**Note**：

- In mode two, it is necessary to provide the interval transaction price list (tradePrice) and the interval transaction volume list (tradeQty) fields;
- In mode one, the interval transaction price list (tradePrice) and the interval transaction volume list (tradeQty) are not mandatory fields.

#### Tick Market Data



For tick mode, the required columns for the market table are:

| **Name**     | **Type**  | **Meaning**                                                     |
| ------------ | --------- | ------------------------------------------------------------ |
| symbol       | SYMBOL    | Stock subject                                                     |
| symbolSource | SYMBOL    | Securities market: Shenzhen Stock Exchange, Shanghai Stock Exchange                                     |
| time         | TIMESTAMP | Time                                                       |
| sourceType   | INT       | 0 for order; 1 for transaction                             |
| orderType    | INT       | order: <br> 1: market price; <br> 2: limit price; <br> 3: best price on our side; <br> 10: cancel, only for Shanghai Stock Exchange, i.e., cancellation records in order <br> transaction: <br> 0: transaction; <br> 1: cancellation, only for Shenzhen Stock Exchange, i.e., cancellation records in transaction |
| price        | DOUBLE    | Order price                                                    |
| qty          | LONG      | Order quantity                                                    |
| buyNo        | LONG      | Corresponding original data for transaction; fill buyNo for order                    |
| sellNo       | LONG      | Corresponding original data for transaction; fill sellNo for order                |
| direction    | INT       | 1 (buy) or 2 (sell)                                        |
| seqNum       | LONG      | Tick data sequence number                                         |


User order table must provide the following columns:

| **Name**  | **Type**  | **Meaning**                                                     |
| --------- | --------- | ------------------------------------------------------------ |
| symbol    | SYMBOL    | Stock subject                                                     |
| time      | TIMESTAMP | Time                                                        |
| orderType | INT       | 	Shanghai Stock Exchange: <br> 0: Market order with the best five levels of immediate cancellation of the remaining orders <br> 1: Market order with the best five levels of immediate conversion to a limit order for the remaining orders <br> 2: Market order with the best price on our side <br> 3: Market order with the best price on the opposite side <br> 5: Limit order <br> 6: Cancellation <br> Shenzhen Stock Exchange: <br> 0: Market order with the best five levels of immediate cancellation of the remaining orders <br> 1: Market order with immediate cancellation of the remaining orders <br> 2: Market order with the best price on our side <br> 3: Market order with the best price on the opposite side <br> 4: Market order with full transaction or cancellation <br> 5: Limit order <br> 6: Cancellation |
| price     | DOUBLE    | Commission order price                                                |
| qty       | LONG      | Commission order quantity                                                |
| direction | INT       | 1 (buy) or 2 (sell)                                           |
| orderID   | LONG      | User order ID, only effective when cancelling                                  |

The transaction detail output table tradeOutputTable, where the engine inserts the transaction results of the commission orders, can be defined in the following order (names can be modified, each field has its specific meaning, so the order of field types cannot be changed; note that the last three columns are only enabled when the configuration item timeDetail is set to 1):

| **Name**            | **Type**      | **Meaning**                                                     |
| ------------------- | ------------- | ------------------------------------------------------------ |
| orderID             | LONG          | User order ID that was traded                                            |
| symbol              | SYMBOL        | Stock subject                                                   |
| direction           | INT           | 1 (buy) or 2 (sell)                                        |
| sendingTime         | TIMESTAMP     | Order sending time                                           |
| limitPrice          | DOUBLE        | Commission price                                                   |
| volumeTotalOriginal | LONG          | Order commission quantity                                                 |
| tradeTime           | TIMESTAMP     | Transaction time                                                    |
| tradePrice          | DOUBLE        | Transaction price                                            |
| volumeTraded        | LONG          | Transaction volume                                                     |
| orderStatus         | LONG          | Order status, indicating whether the user order was fully transacted -1: indicates the order was rejected 0: indicates the order was partially transacted 1: indicates the order was fully transacted 2: indicates the order was canceled |
| orderReceiveTime    | NANOTIMESTAMP | Time when the order was received (system time)                                |
| insertTime          | TIMESTAMP     | Latest time of the market commission when the order was received                                   |
| startMatchTime      | NANOTIMESTAMP | Match start time                                                 |
| endMatchTime        | NANOTIMESTAMP | Match completion time                                           |



For specific interface field descriptions, see(https://gitee.com/dolphindb/DolphinDBPlugin/tree/release200.10/MatchingEngineSimulator)。

### Matching Rules

The matching rules of the simulated matching engine follow the securities competitive trading rules of the exchange (matched and transacted based on the principle of price priority and time priority). Specifically, the price priority principle means that buy orders at higher prices will be matched before buy orders at lower prices, while sell orders at lower prices will be matched before sell orders at higher prices. The time priority principle refers to, in the case of the same price, earlier orders being matched first.

The simulated matching engine simulates trading of the simulated orders according to user-configured parameters (price matching depth, latency, etc.).

Currently configurable parameters (detailed field descriptions see the plugin interface) are as follows:

| **Configuration Parameter**           | **Parameter Description**                                           | **Remarks**                                                     |
| ---------------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| dataType               | Market category: 0 (stock tick data) or 1 (market snapshot data)       | Default value 1                                                     |
| orderBookMatchingRatio | Transaction percentage with the market order book                              | Default value 1 (0-1)                                               |
| depth                  | Price matching depth for limit orders                           | Default value 10; maximum setting of 50 for tick market data;                       |
| latency                |Order latency (in milliseconds)                                 | Default value 0; for example, if the order latency configuration is T3, the latest time of market data is T1, and the time when the simulated order is sent is T2, if T3 + T2 < T1, then it is judged whether the transaction occurs at the time T1 if T3 + T2 >= T1, it will not be immediately transacted, waiting for the latest time of market data to be greater than T3 + T2 before judging whether the transaction occurs |
| outputOrderBook        | For tick market data, you can set whether to output the synthesized order book at a specified frequency     | Default value 0; 0 (do not output) or 1 (output); invalid for snapshot market data;          |
| orderBookInterval      | If you need to output the order book, the minimum interval for outputting the order book, in milliseconds | Default value 1000; invalid for snapshot market data;                               |
| matchingMode           | In snapshot mode, two modes of matching,                           | Default value 1; 1 (match orders according to mode one) or 2 (match orders according to mode two); invalid for tick market data; |
| matchingRatio          | In snapshot mode, the transaction percentage for the snapshot interval,                    |By default, equal to the transaction percentage orderBookMatchingRatio; invalid for tick market data;|
| mergeOutputs           | Whether to output to the composite output table                              | Default value 0; valid for tick market data when set to 1, invalid for snapshot market data; |
| timeDetail             | Whether to output "match start time" and "match completion time" in the transaction output table      | Default value 0; 0 (do not output) or 1 (output);                |
| cpuId                  | The ID of the CPU core to which it is bound, only binds the thread at the first appendMsg | Default value -1                   |

The matching rules for limit orders and market orders are introduced separately below.

#### Limit Order Matching Rules

Limit orders will use the corresponding matching rules based on different market categories.

- Snapshot Market

    When the market data is a snapshot, the matching steps are as follows:
    
    <img src="images/MatchEngineSimulator/DM_20230922142321_002.png" width="350"/>
    
    
    1. Limit orders are matched instantly with N levels of the market order book:
    
        - Buy orders: When the commission price >= sell one price, it is transacted sequentially according to the sell one order book. The transaction price is the corresponding order book price, and the transaction volume is the smaller of the volume of a certain order book multiplied by the order book transaction ratio and the commission volume minus the already transacted volume of the commission. If the commission is not fully transacted, it enters the pending order queuing stage. It ranks first at that price level, i.e., the volume in front of the commission is 0, then enters the pending order matching stage.
        - Sell orders: When the commission price <= buy one price, it is transacted sequentially according to the buy one order book. The transaction price is the corresponding order book price, and the transaction volume is the smaller of the volume of a certain order book multiplied by the order book transaction ratio and the commission volume minus the already transacted volume of the commission. If the commission is still not fully transacted, it enters the pending order queuing stage. It ranks first at that price level, i.e., the volume in front of the commission is 0, then enters the pending order matching stage.
    
    2. Pending Orders：
    
        - When commissioning, if the commission price falls within the own side's order book, the commission enters the pending order queuing stage at this time, ranking in front of the commission volume as the volume at the commission price level, then enters the pending order matching stage.
    
    3. Pending Order Matching Stage (for untransacted or partially untransacted parts):
    
        - Matching mode one
          - When the interval transaction percentage is set to 0, it is the same as step 1. The commission will be simulated to match with the latest market order book of the counterpart at the time of the latest market data. The transaction price is the commission order price. Until the order is fully transacted or the market closes for the day, the simulation match ends. Orders that are untransacted or partially untransacted can be canceled according to the cancellation request.
          - When the interval transaction percentage is set greater than 0, it is matched and transacted in the following mode:
            - Based on the latest price and the latest order book of the snapshot market data for matching:
              - When the latest price equals the commission price, the volume in front of the commission is first deducted (the interval transaction volume minus the volume in front of the commission), which is marked as the current volume. When the volume in front of the commission drops to 0, the user's commission order begins to transact. The transaction price is the commission price, and the transaction volume is the smaller of the user's commission volume and the current volume multiplied by the interval transaction percentage.
              - When the latest price is higher than the commission price (for buy orders, the latest price < commission price; for sell orders, the latest price > commission price), the transaction price is the commission price, and the transaction volume is the smaller of the user's commission volume and the actual transaction volume of the interval multiplied by the interval transaction percentage.
              - The remaining untransacted orders are matched and transacted with the counterpart's order book price level, with the transaction price being the commission order price.
              - Until the order is fully transacted or the market closes for the day, orders that are untransacted or partially untransacted can be canceled according to the cancellation request.
        - Matching mode two
          - Based on the interval transaction list information and the latest order book of the snapshot market data for matching (following the principle of price priority and time priority when matching with the interval transaction prices):
            - User commission buy orders are matched with the transaction list information, where transactions in the list less than or equal to the user's commission price can be matched and transacted. First, the volume in front of the user's commission is deducted (subtracting the volume in front of the commission order at the same price level and the remaining volume after the market and user commission orders), marked as the current volume. When the current volume is greater than 0, the user's commission order can be transacted. The transaction price is the commission price, and the transaction volume is the minimum of the user's remaining commission volume and the current volume.
            - User commission sell orders are matched with the transaction list information, where transactions in the list greater than or equal to the user's commission price can be matched and transacted. First, the volume in front of the user's commission is deducted (subtracting the volume in front of the commission order at the same price level and the remaining volume after the market and user commission orders), marked as the current volume. When the current volume is greater than 0, the user's commission order can be transacted. The transaction price is the commission price, and the transaction volume is the minimum of the user's remaining commission volume and the current volume.
            - The parts not transacted above are then matched and transacted with the counterpart's order book price levels, with the transaction price being the commission order price and the transaction volume being the minimum of the user's remaining commission volume and the matching volume multiplied by the transaction ratio.
            - If it is still not fully transacted, update the volume in front of the user's commission order at the same price level:
            - Until fully transacted or the market closes for the day, orders that are untransacted or partially untransacted can also be canceled according to the cancellation request.

- Tick Market

    When the market data is tick data, the matching steps are as follows:
    
    <img src="images/MatchEngineSimulator/DM_20230922142321_003.png" width="250"/>
    
    1. First, limit orders are instantly matched with N levels of the market order book (can be matched according to the configured transaction ratio)
    1. For untransacted or partially untransacted parts, the transaction is matched based on the principle of price priority and time priority.
    
        1. For subsequent market data, the user order is matched with the market order according to the principle of price priority and time priority.
        1. Orders that are untransacted or partially untransacted can also be canceled according to the cancellation request.
        1. Until fully transacted or the market closes for the day.

### Market Order Matching Rules

Market orders are matched immediately with N levels of the market order book according to the respective trading rules of the Shanghai Stock Exchange or Shenzhen Stock Exchange.

#### Shanghai Stock Exchange

The trading rules for market orders on the Shanghai Stock Exchange are as follows:

<img src="images/MatchEngineSimulator/DM_20230922142321_004.png" width="350"/>

- The best five levels of immediate transaction remaining cancellation, i.e., the commission is transacted sequentially at the most optimal five price levels of the counterpart according to the counterpart's prices. If there is an untransacted part of the commission order after the transaction, the untransacted part is automatically canceled.
- The best five levels of immediate transaction remaining conversion to limit order, i.e., the commission is transacted at the most optimal five price levels of the counterpart according to the counterpart's prices within the commission. The remaining untransacted part is converted to a limit order at the latest transaction price of the own side's market commission; if there is no transaction for the commission, it is converted to a limit order at the most optimal price of the own side's market commission; if there is no own side's commission, the commission is canceled.
- The best price on our side commission, i.e., the commission order price is the most optimal price of the own side's order book at the time of the current order book. If there is no corresponding order on the counterpart side, the commission is automatically canceled.
- The best price on the opposite side commission, i.e., the commission order price is the most optimal price of the counterpart side's order book at the time of the current order book. If there is no corresponding order on the counterpart side, the commission is automatically canceled.

#### Shenzhen Stock Exchange

The trading rules for market orders on the Shenzhen Stock Exchange are as follows:

<img src="images/MatchEngineSimulator/DM_20230922142321_005.png" width="500"/>

- Full transaction or cancellation commission, with the counterpart's price as the transaction price. If, at the time of the user's commission, the most recent order book of the market contains counterpart market commission orders that can be sequentially transacted to fully transact the order, then it is transacted in order; otherwise, the order is automatically fully canceled.
- The best five levels of immediate transaction remaining cancellation commission, with the counterpart's price as the transaction price, based on the most recent order book's counterpart's best five price levels of market commission orders at the time of the user's commission, sequentially transacted. The untransacted part is automatically canceled.
- Immediate transaction remaining cancellation commission, with the counterpart's price as the transaction price, based on the most recent order book's counterpart market commission orders at the time of the user's commission, sequentially transacted. The untransacted part is automatically canceled.
- The best price on our side commission, with the most optimal price of the own side's order book in the most recent order book as the commission price.
- The best price on the opposite side commission, with the most optimal price of the counterpart side's order book in the most recent order book as the commission price.

## Using the Simulated Matching Engine

To use this feature, please contact DolphinDB technical support to apply for a special license file for the simulated matching engine.

For the introduction to the simulated matching engine interface and its installation and usage, refer to the Simulated Matching Engine Usage Instructions on the DolphinDB official website.

### Creating and Using the Simulated Matching Engine

The use of the simulated matching engine mainly involves several steps: modifying the configuration file, defining the table structure and column name mapping for the market table and user commission order table, defining the transaction detail output table and other tables, creating the engine, and inputting data. Below, taking tick market data as an example, the use of the simulated matching engine is demonstrated.

- Configuring the simulated matching engine

    ```
    config = dict(STRING, DOUBLE);
    config["latency"] = 0;                     //用户订单时延为0
    config["orderBookMatchingRatio"] = 1;      //与订单薄匹配时的成交百分比
    config["dataType"] = 0;                    //行情类别：0表示逐笔，1表示快照
    ```

- Creating corresponding column name mapping dictionaries based on the table structure of the market table and user commission order table:

    ```
    dummyQuotationTable = table(1:0, `symbol`symbolSource`time`sourceType`orderType`price`qty`buyNo`sellNo`BSFlag`seqNum, 
                                    [STRING, STRING, TIMESTAMP,INT, INT, DOUBLE, LONG, LONG, LONG, INT, LONG])
    quotationColMap = dict( `symbol`symbolSource`time`sourceType`orderType`price`qty`buyNo`sellNo`direction`seqNum, 
                            `symbol`symbolSource`time`sourceType`orderType`price`qty`buyNo`sellNo`BSFlag`seqNum)
    
    dummyUserOrderTable = table(1:0, `symbol`time`orderType`price`qty`BSFlag`orderID,  
                                      [STRING, TIMESTAMP, INT, DOUBLE, LONG, INT, LONG])
    userOrderColMap = dict( `symbol`time`orderType`price`qty`direction`orderID, 
                            `symbol`time`orderType`price`qty`BSFlag`orderID) 
    ```

- Defining Output Tables for Transactions and Snapshots

    ```
    tradeOutputTable  = table(1:0, `OrderSysID`Symbol`Direction`SendingTime`LimitPrice`VolumeTotalOriginal`TradeTime`TradePrice`VolumeTraded`OrderStatus`orderReceiveTime, 
                                   [LONG, STRING, INT, TIMESTAMP,DOUBLE,LONG, TIMESTAMP,DOUBLE,LONG, INT, NANOTIMESTAMP])
    snapshotOutputTable  = table(1:0, `symbol`time`avgBidPrice`avgOfferPrice`totalBidQty`totalOfferQty`bidPrice`bidQty`offerPrice`offerQty`lastPrice`highPrice`lowPrice, 
                                       [STRING, TIMESTAMP,DOUBLE,DOUBLE, LONG, LONG,DOUBLE[],LONG[], DOUBLE[], LONG[], DOUBLE, DOUBLE, DOUBLE])
    ```

- Create Engine

    ```
    engine = MatchingEngineSimulator::createMatchEngine(name, exchange, config, dummyQuotationTable, quotationColMap, dummyUserOrderTable, userOrderColMap, tradeOutputTable, , snapshotOutputTable)
    ```

- Insert market trends or user orders into the engine through the appendMsg interface. The syntax is:

    ```
    appendMsg(engine, msgBody, msgId)
    Parameter explanation:
    engine: The simulated matching engine returned by createMatchEngine
    msgBody: Market data or user orders, which can be a tuple or table. Note that the schema must be consistent with dummyQuotationTable and dummyUserOrderTable.
    msgId: Message identifier, 1 for market data, 2 for user orders
    ```

    Example usage:
    
    ```
    symbol = "000001"
    //for hq order
    TYPE_ORDER = 0
    HQ_LIMIT_ORDER = 2
    
    //for user order
    LIMIT_ORDER = 5
    ORDER_SEL = 2
    ORDER_BUY = 1
    
    MatchingEngineSimulator::resetMatchEngine(engine)
    appendMsg(engine, (symbol, "XSHE", 2021.01.08 09:14:00.100, TYPE_ORDER,  HQ_LIMIT_ORDER, 7., 100, 1, 1, ORDER_BUY,1), 1)
    appendMsg(engine, (symbol, "XSHE", 2021.01.08 09:14:00.100, TYPE_ORDER,  HQ_LIMIT_ORDER, 6., 100, 2, 2, ORDER_BUY,1), 1)
    appendMsg(engine, (symbol, "XSHE", 2021.01.08 09:14:00.100, TYPE_ORDER,  HQ_LIMIT_ORDER, 5., 100, 3, 3, ORDER_BUY,1), 1)
    appendMsg(engine, (symbol, 2021.01.08 09:14:00.400, LIMIT_ORDER, 6., 100, ORDER_BUY, 1), 2)
    ```

- Viewing Unexecuted User Orders and Resetting the Simulation Matching Engine

    ```
    opentable = MatchingEngineSimulator::getOpenOrders(engine)
    MatchingEngineSimulator::resetMatchEngine(engine)
    ```

### Engine Operational Mechanisms

- An engine can support multiple stocks.
- Currently, the engine does not have a background working thread, and all interfaces are synchronous. When the interface returns, the matching process has already ended, and the output tables can be immediately checked to obtain the matching results.
- If you wish to concurrently match multiple stocks, you can create multiple simulation matching engines and insert data into these engines concurrently in the script.

## Simulating User Orders with Level 2 Snapshot Data

Based on the fields included in snapshot data, the simulation matching engine offers two different matching modes for users to choose from. Level 2 market snapshot data with a 3-second frequency includes the most recent market data and the latest transaction prices. Users can select matching mode one, which matches commission orders with the latest price and the latest order book of quick market data. When snapshot data is linked with transaction details within an interval, users can select matching mode two, which matches commission orders with the interval's transaction price list and the latest order book.

### Example

Take a moment at 2022.04.14T09:35:00.040 with the quick market order book as a reference (Table 1), with the latest transaction price being 16.34. At this time, a user commissions a sell order at a price of 16.32 for 50,000 shares. Below, we simulate the commission order's transaction using matching modes one and two, respectively.

Table1

| **bidQty** | **bidPrice** | **askPrice** | **askQty** |
| ---------- | ------------ | ------------ | ---------- |
| 10100      | 16.33        | 16.34        | 5400       |
| 22000      | 16.32        | 16.35        | 197300     |
| 18300      | 16.31        | 16.36        | 246400     |
| 113200     | 16.30        | 16.37        | 183400     |
| 3900       | 16.29        | 16.38        | 313800     |
| 12800      | 16.28        | 16.39        | 454600     |
| 16600      | 16.27        | 16.40        | 696100     |
| 17800      | 16.26        | 16.41        | 49000      |
| 39054      | 16.25        | 16.42        | 59400      |
| 4400       | 16.24        | 16.43        | 76300      |

- Matching with mode one:

    In mode one, limit orders are instantly matched with N levels of market order book data, and any unmatched portion is queued. In subsequent market data, the commission order first matches with the interval's latest price and then with the latest order book sequentially. This time, "matchingMode" is set to 1, and the interval transaction ratio (matchingRatio) is set to 10%, with other specific configurations as follows:
    
    ```
    config = dict(STRING, DOUBLE);
    /// Market type: 0 for tick, 1 for snapshot
    config["dataType"] = 1;
    /// Matching order book depth, 5 - 50
    config["depth"] = 10;
    /// Simulation delay, in milliseconds
    config["latency"] = 0;
    /// Whether to output the order book for tick data, 0: no, 1: yes
    config["outputOrderBook"] = 0;
    /// Minimum interval for outputting the order book, in milliseconds
    config["orderBookInterval"] = 1;
    /// Transaction percentage with the market order book
    config["orderBookMatchingRatio"] = 1;
    /// The two modes of matching in snapshot mode, can be set to 1 or 2
    config["matchingMode"] = 1;
    /// The transaction percentage for the snapshot interval, by default equal to orderBookMatchingRatio
    config["matchingRatio"]=0.1
    ```
    
    At time t, the order book (Table 2) shows, and the user's commission order immediately transacts with counterpart orders at prices of 16.33 and 16.32, with transaction volumes of 10100 and 22000, respectively. The remaining unexecuted quantity is 17900, queued at the first level on the sell side.
    
    ```
    appendMsg(engine, ("000001.SZ", 2022.04.15T09:55:15.000, 5, 16.32, 50000, 2, 1) ,2)
    select * from tradeOutputTable
    ```
    
    The transaction details at this moment are as follows:
    
    <img src="images/MatchEngineSimulator/DM_20230922142321_006.png" width="500"/>
    
    In the next moment, the market snapshot order book (Table 3) updates, the latest transaction price is 16.34, and the interval's total transaction volume is 25500.
    
    表2
    
    | **bidQty** | **bidPrice** | **askPrice** | **askQty** |
    | ---------- | ------------ | ------------ | ---------- |
    | 28900      | 16.33        | 16.34        | 1700       |
    | 22000      | 16.32        | 16.35        | 224800     |
    | 18300      | 16.31        | 16.36        | 241100     |
    | 113200     | 16.30        | 16.37        | 183500     |
    | 3900       | 16.29        | 16.38        | 313800     |
    | 12800      | 16.28        | 16.39        | 454600     |
    | 16600      | 16.27        | 16.40        | 696000     |
    | 17800      | 16.26        | 16.41        | 49000      |
    | 39054      | 16.25        | 16.42        | 59400      |
    | 4400       | 16.24        | 16.43        | 76300      |
    
    At this point, the user's remaining commission order matches with the interval's latest transaction price, transacting a quantity of 2500. The remaining 15400 is then transacted with the first level on the buy side, completing the user's commission order. The detailed transaction results are as follows:
    
    <img src="images/MatchEngineSimulator/DM_20230922142321_007.png" width="500"/>
    
    The above script can be found in the attachment.

- Matching with mode two:

    In mode two, limit orders are instantly matched with N levels of the order book. Unexecuted orders enter a queuing process, using the interval transaction prices for price matching, and then sequentially match with the latest moment's order book for transactions. At this time, "matchingMode" is set to 2, with other specific configurations as follows:
    
    ```
    config = dict(STRING, DOUBLE);
    //Market type: 0 for tick data, 1 for snapshot
    config["dataType"] = 1;
    ///Matching order book depth, 5 - 50
    config["depth"] = 10;
    ///Simulation delay, in milliseconds, simulating the delay from when a user order is issued to when it is processed
    config["latency"] = 0;
    ///For tick data, whether to output the order book, 0: do not output, 1: output
    config["outputOrderBook"] = 0;
    ///If the order book needs to be output, the minimum time interval for outputting the order book, in milliseconds
    config["orderBookInterval"] = 1;
    ////Transaction percentage with the market order book
    config["orderBookMatchingRatio"] = 1;
    ///In snapshot mode, the two matching modes, can be set to 1 or 2
    config["matchingMode"] = 2;
    ////In snapshot mode, the snapshot interval transaction percentage, by default, is equal to the transaction percentage orderBookMatchingRatio
    config["matchingRatio"]=0.1
    ////Whether need to output to the composite output table
    ```
    
    At time t, the order book is as shown above, with the user commissioning a sell order at a price of 16.32 for 50,000 shares. The commission order immediately transacts with counterpart orders at prices of 16.33 and 16.32, with transaction volumes of 10,100 and 22,000, respectively. The remaining unexecuted quantity is 17,900, ranking first on the sell side.
    
    ```
    appendMsg(engine, ("000001.SZ", 2022.04.15T09:55:15.000, 5, 16.32, 50000, 2, 1), 2)
    select * from tradeOutputTable
    ```
    
    The detailed transaction results are similar to those in mode one:
    
    <img src="images/MatchEngineSimulator/DM_20230922142321_010.png" width="500"/>
    
    At this stage, the user's order is positioned optimally on their side. The next moment's market snapshot order book remains as shown above, with the latest interval transaction prices being [16.34, 16.33, 16.34] and volumes [300, 1000, 24200]. The commission order is matched with the interval's transaction price list for execution. At this point, the user's commission order is completed, and the detailed transaction results are as follows:
    
    <img src="images/MatchEngineSimulator/DM_20230922142321_011.png" width="500"/>
    
    The above script is attached.

### Performance Testing

#### System Configuration

| **Name**         | **Configuration Details**                                |
| ---------------- | ------------------------------------------- |
| Operating System         | CentOS 7 64位                               |
| CPU              | Intel(R) Xeon(R) Silver 4210R CPU @ 2.40GHz |
| Memory             | 512GB                                       |
| DolphinDB        | 2.00.9.7                                    |
| Simulation Matching Engine Plugin | release200.9.3                              |

#### Testing Method

The testing uses Level-2 3s market snapshot data as the input for simulation matching, with a transaction ratio set to 0.5 and an interval transaction ratio set to 0.1. Every 3 seconds, a 1000-share order is placed for each stock. The test measures the time taken for stock quantities of 100, 500, and 2000, including time for data query from the original distributed table and calculation. The complete testing script will be provided as an attachment.

```
def strategy(mutable engine,msg,mutable contextdict,mutable logTB){ 
	/*
	 * Place a 1000-share order for each stock every 3 seconds
	 */
	try{
		temp=select last(DateTime) as time ,last(lastPx) as price from msg group by SecurityID
		codes=distinct(msg.SecurityID)
		prices=dict(temp.SecurityID,temp.price)
		for (icode in codes){
			appendMsg(engine,(icode,contextdict["now"],5,prices[icode],1000,1,1),2)
			}
		//log
		text="open_signal: "+string(contextdict["now"])+"  code : "+ 
		concat(temp.SecurityID,",")+" price:  "+string(concat(string(temp.price),","))
		logTB.append!(table(text as info))		
		}
	catch(ex){
		logTB.append!(table("error:"+string(contextdict["now"])+ex[1] as info))
	}
}



timer{
	for (idate in 2022.04.15..2022.04.15){
		contextdict["now"]=idate
		tb = select "XSHE" as symbolSource,SecurityID,DateTime,lastPrice,highestPrice,lowestPrice,deltas(TotalVolumeTrade)  as newTradeQty ,BidPrice,
		BidOrderQty,OfferPrice,OfferOrderQty from loadTable("dfs://level2_tickTrade",`tickTradeTable) where  SecurityID in universe and 
		date(DateTime)=idate and  DateTime.second() between 09:30:00:15:00:00
		context by SecurityID csort DateTime order by DateTime map  
		times=tb.DateTime
		nTimes=cumsum(groupby(count,times,times).values()[1])
		i=1
		for (i in 0..(size(nTimes)-1)){
			if(i==0){
				contextdict["now"]=times[nTimes[0]-1]
				msg=tb[0:nTimes[0]]
			}
			else{
				contextdict["now"]=times[nTimes[i]-1]
				msg=tb[nTimes[i-1]:nTimes[i]]
				}		
			if(size(msg)>0){
				savehqdataToMatchEngine(engine, msg,logTB)	
				strategy(engine,msg, contextdict, logTB)
				}
			}

		MatchingEngineSimulator::resetMatchEngine(engine)
	}
}
```



#### Testing Results

| **Number of Stocks** | **Data Entries** | **Threads** | **Time** |
| ---------- | ------------ | ---------- | -------- |
| 100        | 409084       | 1          | 6.5s     |
| 500        | 1970583      | 1          | 38.4s    |
| 2000       | 8033198      | 4          | 40s      |

The trade detail result table is shown below:


<img src="images/MatchEngineSimulator/DM_20230922142321_009.png" width="550"/>

## Simulating Orders with Level 2 Tick Data



The simulation matching engine uses commission orders and market data as inputs, simulating trades according to the trading rules of the securities exchange, based on the principles of price and time priority.

For example, at a specific moment (2022.04.14T09:35:00.040), the order book is referenced for the simulation of trades under different conditions based on price and time priority rules.

Table3

| **bidQty** | **bidPrice** | **askPrice** | **askQty** |
| ---------- | ------------ | ------------ | ---------- |
| 2000       | 15.81        | 16.45        | 2000       |
| 4000       | 15.80        | 16.65        | 4000       |
| 2000       | 15.56        | 16.67        | 8000       |
| 2000       | 15.50        | 16.80        | 2000       |
| 2000       | 15.25        | 16.85        | 2000       |
| 4000       | 15.00        | 16.90        | 2000       |
| 1000       | 14.80        | 17.10        | 4000       |
| 2000       | 14.75        | 17.15        | 4000       |
| 1281000    | 14.61        | 17.25        | 2000       |
| 2000       | 14.35        | 17.45        | 2000       |

- Selling 1000 shares at a limit price of 16.45

     At the moment 2022.04.14T09:35:00.040, the best buy price is 15.81. The commission order to sell 1000 shares at a limit price of 16.45 is higher than the best buy price of 15.81, making it unable to execute immediately. The commission order then queues at the sell side's first level after the 2000 shares at 16.45 have been fully executed.
    
    ```
    appendMsg(engine, (`000001, `XSHE, 2022.04.14T09:35:00.050, 0, 2, 16.45, 2500, 901, 901, 1, 34) ,1) 
    select * from tradeOutputTable
    ```

    The simulation matching engine can receive market data and user commissioned orders through the appendMsg (engine, msgBody, msgId) function interface. If msgId=1, it indicates market data, and if it is 2, it indicates user commissioned orders.
    
    On April 14, 2022, at 09:35:00:050, a market purchase order with a price of 16.45 and a quantity of 2500 entered. At this time, when checking the Volume Traded, it was found that the user's order had completed 500 shares, and the order status (OrderStatus) was 0, indicating incomplete transaction.
    
    If you place a purchase order with a quantity of 500 at the market price at 09:35:00:070 on April 14, 2022, you can find that the remaining orders in the user's commissioned order are being executed at this moment.

    The above script can be found in the attachment.

## Performance testing of simulated matching of user orders based on Level 2 transaction by transaction data

When using a simulation matching engine, users need to send market data to the engine in a cyclic manner. The amount of data per transaction is very large, and in order to pursue higher performance, users can write C++code to use simulation matching engines.

### Using the Simulation Matching Engine in C++ Environment

This section introduces how to write DolphinDB plugins in C++ and call the simulation matching engine interface within those plugins. The example plugin provided here is for TWAP (Time-Weighted Average Price) algorithmic trading, and the complete plugin code is available in the attachments.

Before writing the plugin, users should familiarize themselves with [DolphinDB 插件开发教程](../plugin/plugin_development_tutorial.md)o understand the basics of plugin development. For instance, in the TWAP algorithm trading plugin, the getFunctionDef method can be used to retrieve the simulation matching engine's function interface, returning a function pointer which is then called using createEngineFunc->call(heap_, args). Here's a code snippet for reference:

```
FunctionDefSP createEngineFunc = heap_->currentSession()->getFunctionDef("MatchingEngineSimulator::createMatchEngine");
vector<ConstantSP> args = {...}
auto engine = createEngineFunc->call(heap_, args);
```

### TWAP Algorithm Trading Plugin

The TWAP algorithm trading example plugin implements the following process:

1. Iterates over the input data table, which is the result of replaying bid and trade tick data, formatted as BLOB.
2. Parses the replayed market data and sends it to the simulation matching engine.
3. Executes the trading strategy when the market data's time reaches the trigger time defined by the strategy.

- TWAP Trading Strategy:

    The strategy triggers trades within the time windows of 10:00 to 11:30 and 13:00 to 14:30, at a frequency of once per minute, performing the following steps:
    
    1. Cancels any unexecuted orders from the previous minute.
    2. Places a new buy order for each stock once, with each order for 2000 shares.

#### Plugin Usage

- Loading the Plugin

    ```
    loadPlugin("<path>/MatchEngineTest.txt")
    ```
    
    The algorithm trading plugin is named MatchEngineTest, and its text file is located in the plugin's bin directory.

- Plugin Interface

    ```
    MatchEngineTest::runTestTickData(messageTable, stockList, engineConfig)
    ```
    
    The messageTable reads in replayed market data (bid and trade tick data) post-replay. The stockList contains a list of stocks for which orders are placed. The engineConfig is a dictionary containing configuration parameters for the engine.
    
    - engineConfig\[“engineName“] is the name of this test, which is also the name used when creating simulation matching engines. When creating multiple simulation matching engines, the engine name cannot be duplicated.".
    - engineConfig\[“market“]  is the exchange used, XSHG (Shanghai Stock Exchange) or XSHE.
    - The parameter configuration for "outputOrderBook" in both the engine configuration and simulation matching engine is the same, indicating whether to output the order book.

- Input and Output Description

    The input data table, messageTable, has columns named and typed as follows:
    
    ```
    colName = `msgTime`msgType`msgBody`sourceType`seqNum
    colType =  [TIMESTAMP, SYMBOL, BLOB, INT, INT]
    ```
    
    msgType values include entrust, trade, and snapshot, representing bid orders, trades, and snapshot data, respectively. The plugin internally parses BLOB-formatted data. This plugin's data source for algorithm trading uses only tick data, with the structure defined as tickSchema in the C++ code:
    
    ```
    TableSP tickSchema = Util::createTable({"symbol", "symbolSource", "TradeTime", "sourceType", "orderType", "price", "qty", "buyNo", "sellNo", "BSFlag", "seqNum"}, \
                {DT_SYMBOL, DT_SYMBOL, DT_TIMESTAMP, DT_INT, DT_INT, DT_DOUBLE, DT_INT, DT_INT, DT_INT, DT_INT, DT_INT}, 0, 0);
    ```
    
    If the trading strategy requires snapshot data, then the input data source must include snapshot data, structured as defined in the C++ code:
    
    ```
    TableSP snapshotSchema = Util::createTable({"SecurityID", "TradeTime", "LastPrice", "BidPrice", "OfferPrice", "BidOrderQty", "OfferOrderQty", "TotalValueTrade", "TotalVolumeTrade", "seqNum"}, \
                {DT_SYMBOL, DT_TIMESTAMP, DT_DOUBLE, DATA_TYPE(ARRAY_TYPE_BASE + DT_DOUBLE), DATA_TYPE(ARRAY_TYPE_BASE + DT_DOUBLE), DATA_TYPE(ARRAY_TYPE_BASE + DT_INT), \
                DATA_TYPE(ARRAY_TYPE_BASE + DT_INT), DT_DOUBLE, DT_INT, DT_INT}, 0, 0);
    ```
    
    In the plugin code, the output tables for transactions (tradeOutputTable) and market snapshots (snapshotOutputTable) are saved as shared tables, allowing users to access these tables to view the results of the simulation matching.
    
    ```
    // output
    vector<string> arg0 = {"tmp_tradeOutputTable"};
    vector<ConstantSP> arg1 = {tradeOutputTable};
    heap_->currentSession()->run(arg0, arg1);
    runScript("share tmp_tradeOutputTable as "+ testName_ + "_tradeOutputTable");
    
    arg0 = {"tmp_snapshotOutputTable"};
    arg1 = {snapshotOutputTable};
    heap_->currentSession()->run(arg0, arg1);
    runScript("share tmp_snapshotOutputTable as "+ testName_ + "_snapshotOutputTable");
    ```
    
    When executing algorithmic trading with multiple threads for stocks, each thread has a separate tradeOutputTable and snapshotOutputTable. When users view stock transactions, they need to summarize the result tables of each thread and output them.
    
    ```
    tradeResult = table(100:0, MatchEngineTest1_tradeOutputTable.schema().colDefs.name, MatchEngineTest1_tradeOutputTable.schema().colDefs.typeString)
    for (i in 1..thread_num) {
        tradeResult.append!(objByName("MatchEngineTest"+i+"_tradeOutputTable"))
    }
    // View the total transaction volume of user orders sent
    select Symbol, sum(VolumeTraded) from tradeResult where OrderStatus != 3 and OrderStatus != 2 group by Symbol order by Symbol
    ```

#### 
Code Execution Workflow

This section will start from the plugin entry point to discuss the overall workflow of code execution. The complete C++ code will be provided in the attachments.

- Plugin Interface Function

    Each call to runTestTickData creates a TickDataMatchEngineTest class and calls its runTestTickData method.
    
    ```
    ConstantSP runTestTickData(Heap *heap, std::vector<ConstantSP> &arguments) {
        TableSP messageStream = arguments[0];
        ConstantSP stockList = arguments[1];
        DictionarySP engineConfig = arguments[2];
    
        TickDataMatchEngineTest test(heap, engineConfig);
        test.testTickData(messageStream, stockList);
    
        return new String("done");
    }
    ```
    
    The constructor of the TickDataMatchEngineTest class is defined as follows:
    
    ```
    TickDataMatchEngineTest::TickDataMatchEngineTest(Heap * heap, ConstantSP engineConfig)
        : heap_(heap) {
        string name = engine_config->get(new String("engineName"))->getString();
        market_ = engine_config->get(new String("market"))->getString();
        if (market_ == "XSHE") {
            marketOrderType_ = static_cast<int>(UserOrderType::XSHEOppositeBestPrice);
        }
        else {
            marketOrderType_ = static_cast<int>(UserOrderType::XSHGBestFivePricesWithLimit);
        }
        bool hasSnapshotOutput = engine_config->get(new String("hasSnapshotOutput"))->getBool();
        test_name_ = name;
        //  Create simulation matching engine
        engine_ = createMatchEngine(name, market_, hasSnapshotOutput);
        ...
    }
    ```
    
    The definition of the testTickData function is as follows:
    
    ```
    bool TickDataMatchEngineTest::testTickData(TableSP messageStream, ConstantSP stockList) {
        ...
        // MsgDeserializer is a class for parsing BLOB format data, tickDeserializer is for parsing tick data
        MsgDeserializer tickDeserializer(tickSchema);
        ...
        ConstantSP nowTimestamp = messageCols[0]->get(0);
        ConstantSP lastTimestamp = nowTimestamp;
        for (int i = 0; i < messageStream->size(); ++i) {
            nowTimestamp = messageCols[0]->get(i);
            string data = messageCols[2]->getString(i);
            DataInputStream stream(data.c_str(), data.size());
            if (nowTimestamp->getLong() != lastTimestamp->getLong()) {
                context.timestamp = lastTimestamp;
                TableSP msgTable = tickDeserializer.getTable(); // Market data of the same timestamp as a batch
                saveQuatationToEngine(msgTable); // Send market data to the engine
                handleStrategy(context, msgTable); // Execute algorithm strategy, generate user orders and send them to the engine
                tickDeserializer.clear();
                lastTimestamp = nowTimestamp;
            }
            tickDeserializer.deserialize(&stream); // Parse this row of data and store it in the internal table of the parser class
        }
        ...
    }
    ```

- Inserting Market Data and User Orders into the Engine

    Saving market data and sending user orders is the same as in DolphinDB scripts. Call the appendMsg method to pass data into the simulation matching engine.
    
    ```
    appendMsgFunc = heap_->currentSession()->getFunctionDef("appendMsg");
    ...
    void TickDataMatchEngineTest::saveQuatationToEngine(ConstantSP data) {
        vector<ConstantSP> args = {engine_, data, new Int(1)};
        appendMsgFunc->call(heap_, args);
    }
    
    void TickDataMatchEngineTest::saveOrdersToEngine(ConstantSP data) {
        vector<ConstantSP> args = {engine_, data, new Int(2)};
        appendMsgFunc->call(heap_, args);
    }
    ```

- Strategy Execution

    When placing orders, a user order table needs to be constructed. As the in-memory table storage is also columnar, VectorSP type columns are first constructed, and then a table is created using these columns. For SZSE data, the market order is set to 0, and for SSE data, it is set to 1. The definition of market order types can be referred to in the plugin interface manual. After the table construction, call the saveOrdersToEngine function to send user orders to the engine.
    
    ```
    void TickDataMatchEngineTest::handleStrategy(ContextDict& context, ConstantSP msgTable) {
        // Determine the strategy trigger time
        ConstantSP timestampToTrigger = new Timestamp(tradeDate_->getInt() * (long long)86400000 + timeToTrigger_->getInt());
        if (!strategyFlag_ || timestampToTrigger->getLong() > context.timestamp->getLong()) {
            return;
        }
        // Obtain incomplete orders
        vector<ConstantSP> args = {engine_};
        TableSP openOrders = getOpenOrdersFunc->call(heap_, args);
        int openOrderSize = openOrders->size();
        
        if (openOrderSize > 0) {
            // Cancel orders
            ConstantSP symbols = openOrders->getColumn(2);
            ConstantSP times = createVectorByValue(context.timestamp, openOrderSize);
            ConstantSP ordertypes = createVectorByValue(new Int(static_cast<int>(UserOrderType::CancelOrder)), openOrderSize);
            ConstantSP prices = createVectorByValue(new Double(0.0), openOrderSize);
            ConstantSP qtys = openOrders->getColumn(5);
            ConstantSP directions = createVectorByValue(new Int(1), openOrderSize);
            ConstantSP orderIDs = openOrders->getColumn(0);
            vector<ConstantSP> openOrderList = {symbols, times, ordertypes, prices, qtys, directions, orderIDs};
            TableSP ordertable = Util::createTable({"SecurityID", "time", "orderType", "price", "qty", "direction", "orderID"}, openOrderList);
            saveOrdersToEngine(ordertable);
        }
        // Place market orders (sell at the ask price) for each stock
        int stockSize = context.stockList->size();
        ConstantSP symbols = Util::createVector(DT_SYMBOL, stockSize);
        ConstantSP times = createVectorByValue(context.timestamp, stockSize);
        ConstantSP ordertypes = createVectorByValue(new Int(marketOrderType_), stockSize);
        ConstantSP prices = createVectorByValue(new Double(100.0), stockSize);
        ConstantSP qtys = createVectorByValue(new Int(2000), stockSize);
        ConstantSP directions = createVectorByValue(new Int(1), stockSize);
        ConstantSP orderIDs = createVectorByValue(new Int(1), stockSize);
        for (int i = 0; i < stockSize; ++i) {
            symbols->set(i, context.stockList->get(i));
        }
        vector<ConstantSP> limitOrderList = {symbols, times, ordertypes, prices, qtys, directions, orderIDs};
        TableSP ordertable = Util::createTable({"SecurityID", "time", "orderType", "price", "qty", "direction", "orderID"}, limitOrderList);
        saveOrdersToEngine(ordertable);
    
        // Update strategy trigger time
        timeToTrigger_ = new Time(timeToTrigger_->getInt() + 60 * 1000);
        if (timeToTrigger_->getInt() > timeToEnd_->getInt()) {
            if (intervalIdx_ + 1 < int(timeIntervals_.size())) {
                ++intervalIdx_;
                timeToTrigger_ = timeIntervals_[intervalIdx_][0];
                timeToEnd_ = timeIntervals_[intervalIdx_][1];
            }
            else {
                // End of strategy
                strategyFlag_ = false;
            }
        }
    }
    ```

### Performance Testing

#### System Configuration

| **Name**         | **Configuration Details**                                |
| ---------------- | ------------------------------------------- |
|  Operating System        | CentOS 7 64位                               |
| CPU              | Intel(R) Xeon(R) Silver 4210R CPU @ 2.40GHz |
| Memory             | 512GB                                       |
| DolphinDB        | 2.00.100                                    |
| Simulation Matching Engine Plugin | release200.10                               |

#### Testing Method

This test uses a multi-threading approach, dividing the market data according to the stock code modulus among different threads. The entire testing process is divided into market data replay and simulation matching. We first use the `replayDS` and `replay` methods to read and replay market data from the distributed table, then pass the replayed data into the algorithmic trading plugin.

1. **Market Data Replay**

    -  First, set the exchange and obtain the stock codes needed for placing orders. When `market` is set to "sz", `symbols` stores all stock codes in the SZSE trade table. When set to "sh", `symbols` stores all stock codes in the SSE trade table.
        ```
        trade_dfs = loadTable("dfs://TSDB_tradeAndentrust", "trade")
        market = "sz"  // market = "sz" or "sh"
        if (market == "sz") {
            market_name = "XSHE"
            symbols = exec SecurityID from trade_dfs where SecurityID like "0%" or SecurityID like "3%" group by SecurityID
        }
        else {
            market_name = "XSHG"
            symbols = exec SecurityID from trade_dfs where SecurityID like "6%" group by SecurityID
        }
        ```

    - Distribute stock codes among different threads by modulus operation.
    
        ```
        def genSymbolListWithHash(symbolTotal, thread_num) {
            symbol_list = []
            for (i in 0:thread_num) {
                symbol_list.append!([]$STRING)
            }
            for (symb in symbolTotal) {
                idx = int(symb) % thread_num
                tmp_list = symbol_list[idx]
                tmp_list.append!(symb)
                symbol_list[idx] = tmp_list
            }
            return symbol_list
        }
        
        thread_num = 20 // number of threads
        symbol_list = genSymbolListWithHash(symbols, thread_num)
        ```

    - Replay the `entrust` and `trade` tables, which represent order and trade tick data, respectively, into a single table sorted by time. When there are multiple records at the same time, they can be sorted by a specified field. Since these tables are distributed tables, the `replayDS` method is used in conjunction with the `replay` method.
    
        ```
        entrust_dfs = loadTable("dfs://TSDB_tradeAndentrust", "entrust")
        trade_dfs = loadTable("dfs://TSDB_tradeAndentrust", "trade")
        
        def replayBySymbol(market, marketName, symbolList, entrust, trade, i) {
            if (market == "sz") {
                ds1 = replayDS(sqlObj=<select SecurityID as symbol, marketName as symbolSource, TradeTime, 0 as sourceType, 
                    iif(OrderType in ["50"], 2, iif(OrderType in ["49"], 1, 3)) as orderType, Price as price, OrderQty as qty, int(ApplSeqNum) as buyNo, int(ApplSeqNum) as sellNo,
                    int(string(char(string(side)))) as BSFlag, int(SeqNo) as seqNum from entrust
                    where Market = market and date(TradeTime)==2022.04.14 and SecurityID in symbolList>, dateColumn = "TradeTime", timeColumn = "TradeTime")
                ds2 = replayDS(sqlObj=<select SecurityID as symbol, marketName as symbolSource, TradeTime, 1 as sourceType, 
                    iif(BidApplSeqNum==0|| OfferApplSeqNum==0,1,0) as orderType, TradePrice as price, int(tradeQty as qty), int(BidApplSeqNum) as buyNo, int(OfferApplSeqNum) as sellNo,
                    0 as BSFlag, int(ApplSeqNum) as seqNum from trade
                    where Market = market and date(TradeTime)==2022.04.14 and SecurityID in symbolList>, dateColumn = "TradeTime", timeColumn = "TradeTime")
            }
            else {
                ds1 = replayDS(sqlObj=<select SecurityID as symbol, marketName as symbolSource, TradeTime, 0 as sourceType, 
                    iif(OrderType == "A", 2, 10) as orderType, Price as price, OrderQty as qty, int(ApplSeqNum) as buyNo, int(ApplSeqNum) as sellNo,
                    iif(Side == "B", 1, 2) as BSFlag, int(SeqNo) as seqNum from entrust
                    where Market = market and date(TradeTime)==2022.04.14 and SecurityID in symbolList>, dateColumn = "TradeTime", timeColumn = "TradeTime")
                ds2 = replayDS(sqlObj=<select SecurityID as symbol, marketName as symbolSource, TradeTime, 1 as sourceType, 
                    0 as orderType, TradePrice as price, int(tradeQty as qty), int(BidApplSeqNum) as buyNo, int(OfferApplSeqNum) as sellNo,
                    0 as BSFlag, int(TradeIndex) as seqNum from trade
                    where Market = market and date(TradeTime)==2022.04.14 and SecurityID in symbolList>, dateColumn = "TradeTime", timeColumn = "TradeTime")
            }
        
            inputDict  = dict(["entrust", "trade"], [ds1, ds2])
        
            colName = `msgTime`msgType`msgBody`sourceType`seqNum
            colType =  [TIMESTAMP, SYMBOL, BLOB, INT, INT]
        
            messageTemp = table(100:0, colName, colType)
            share(messageTemp, "MatchEngineTest" + i)
            
            // When the market is on the Shenzhen Stock Exchange, data at the same time needs to be sorted in the order of first commissioning and then transaction by transaction
            // The sourceType of each order is 0, and the sourceType of each transaction is 1, so it can be sorted according to the sourceType field
            if (market == "sz") {
                replay(inputDict, "MatchEngineTest" + i,`TradeTime,`TradeTime,,,1,`sourceType`seqNum)
            }
            // The Shanghai Stock Exchange needs to sort transactions in order of first transaction by transaction and then commission by transaction
            // The data in the Shanghai Stock Exchange is strictly sorted by seqNum, so it can be directly sorted by seqNum
            else {
                replay(inputDict, "MatchEngineTest" + i,`TradeTime,`TradeTime,,,1,`seqNum)
            }
        }
        
        // Measure the total time taken for market data replay
        timer {
            job_list = [] // Stores all submitted job ids
            for (i in 1..thread_num) {
                job_list.append!(submitJob("TestJob" + i, "", replayBySymbol, market, market_name, symbol_list[i-1], entrust_dfs, trade_dfs, i))
            }
            // Block the current thread until the job is completed and then return
            for (i in 0 : thread_num) {
                getJobReturn(job_list[i], true)
            }
        }
        
        // Store the replay results as disk tables for subsequent testing
        db = database("<path>/" + market + "_messages_" + thread_num + "_part")
        for (i in 1..thread_num) {
             saveTable(db, objByName("MatchEngineTest" + i), "MatchEngineTest" + i)
        }
        ```

    - After replaying market data by stock code, we have `thread_num` shared memory tables, named "MatchEngineTest" + number. To facilitate subsequent testing, users can choose to save the replay results. Since CSV files do not support BLOB format data, users can store these data tables as disk tables.

1. **Simulation Matching**

    - First, load the simulation matching engine plugin and the algorithmic trading plugin.
    
        ```
        loadPlugin("<path>/PluginMatchingEngineSimulator.txt")
        loadPlugin("<path>/MatchEngineTest.txt")
        ```

    - Prepare data and set parameters, where the number of threads should match the one used during market data replay. Note that the algorithm testing plugin expects market data fields to be `msgTime`msgType`msgBody`sourceType`seqNum`, which are the results of replaying the `entrust` and `trade` market tables into one table.
    
        ```
        thread_num = 20
        market = "sz"
        market_name = "XSHE"
        
        messagesList = [] // Store the market data table for each thread
        symbolList = []  // Store the stock code for each thread
        message_num = 0
        for (i in 1..thread_num) {
            // Read the replayed market data table from the disk table and obtain the stock codes in the table
            messages = select * from loadTable("<path>/" + market + "_messages_" + thread_num + "_part", "MatchEngineTest"+i)
            symbols = exec distinct left(msgBody, 6) from messages
            message_num += messages.size()
            messagesList.append!(messages)
            symbolList.append!(symbols)
        }
        ```

    - DoMatchEngine Test represents the tasks that a single thread needs to execute. The tasks mainly include setting the engine configuration parameters that need to be passed in and calling the algorithm testing plugin interface. The specific execution process inside the algorithm testing plugin has been introduced in the previous section. Note that the engine configuration dictionary here needs to be created inside the function, as the way engine configuration is passed into the plugin interface is a reference, not a copy of engine configuration. If the engine configuration is created outside the function, modifying it by one thread during multi-threaded execution will affect the other thread.
    
        ```
        def doMatchEngineTest(messageData, symbolList, engineName, marketName, hasSnapshotOutput) {
            engine_config = dict(string, any)
            engine_config["engineName"] = engineName
            engine_config["market"] = marketName
            engine_config["hasSnapshotOutput"] = hasSnapshotOutput
            MatchEngineTest::runTestTickData(messageData, symbolList, engine_config)
        }
        
        ClearAllMatchEngine()
        
        // Calculate the total time required for simulation matching
        timer {
            test_job_list = [] // Stores the IDs of all test jobs
            hasSnapshotOutput = false
        
            for (i in 1..thread_num) {
                messages = objByName("MatchEngineTest" + i)
                test_job_list.append!(submitJob("EngineTestJob" + i, "", doMatchEngineTest, 
                                    messages, symbol_list[i-1], "MatchEngineTest" + i, 
                                    market_name, hasSnapshotOutput))
            }
            // Wait for all assignments to finish before returning
            for (i in 0 : thread_num) {
                getJobReturn(test_job_list[i], true)
            }
        }
        ```

    - After all multi-threaded tasks are completed, it is necessary to merge the algorithm transaction results (tradeOutputTable and snapshotOutputTable) of each thread, and store the merged results in the tradeResult and snapshotResult tables, respectively.
    
        ```
        tradeResult = table(100:0, MatchEngineTest1_tradeOutputTable.schema().colDefs.name, MatchEngineTest1_tradeOutputTable.schema().colDefs.typeString)
        snapshotResult = table(100:0, MatchEngineTest1_snapshotOutputTable.schema().colDefs.name, MatchEngineTest1_snapshotOutputTable.schema().colDefs.typeString)
        for (i in 1..thread_num) {
            tradeResult.append!(objByName("MatchEngineTest"+i+"_tradeOutputTable"))
            snapshotResult.append!(objByName("MatchEngineTest"+i+"_snapshotOutputTable"))
        }
        
        // After merging the results into the tradeResult table, delete the tradeOutputTable and snapshotOutputTable tables for each thread
        for (i in 1..thread_num) {
            try {
                undef("MatchEngineTest" + i + "_tradeOutputTable", SHARED)
                undef("MatchEngineTest" + i + "_snapshotOutputTable", SHARED)
            } catch(ex) {print(ex)}
        }
        ```

    - Query the tradeResult table to view the total transaction quantity of each stock.
    
        ```
        select Symbol, sum(VolumeTraded) from tradeResult where OrderStatus !=2 and OrderStatus != 3 group by Symbol order by Symbol
        ```

#### Test Results

The time taken for market data replay includes the time for reading and replaying the market data, while the time taken for simulation matching includes the time for parsing BLOB data and the calculation time of the simulation matching engine. Both durations are calculated using a timer.

Below are the matching results for SZSE tick data and SSE tick data under different test configurations. The test script uses a hashing method to distribute stock codes among different threads by modulus operation. Due to the varying amounts of market data associated with each stock, this might result in different execution times across threads. Efforts are made to ensure the market data volume is similar across all threads to reduce the overall test duration.

<table border="1">
  <tr>
    <th>数据</th>
    <th>股票数量（股）</th>
    <th>行情数据大小（条）</th>
    <th>线程数（个）</th>
    <th>行情回放耗时（秒）</th>
    <th>模拟撮合耗时（秒）</th>
  </tr>
  <tr>
    <td rowspan="4">深交所逐笔数据</td>
    <td rowspan="3">2611</td>
    <td rowspan="3">130,291,485</td>
    <td>2</td>
    <td>1min30s</td>
    <td>2min5s</td>
  </tr>
  <tr>
    <td>5</td>
    <td>47s</td>
    <td>1min3s</td>
  </tr>
  <tr>
    <td>10</td>
    <td>30s</td>
    <td>43s</td>
  </tr>
    <td>500</td>
    <td>37,218,476</td>
    <td>1</td>
    <td>45s</td>
    <td>1min2s</td>
  <tr>
    <td rowspan="4">上交所逐笔数据</td>
    <td rowspan="3">2069</td>
    <td rowspan="3">84,095,369</td>
    <td>2</td>
    <td>1min10s</td>
    <td>1min24s</td>
  </tr>
  <tr>
    <td>5</td>
    <td>32s</td>
    <td>45s</td>
  </tr>
  <tr>
    <td>10</td>
    <td>22s</td>
    <td>33s</td>
  </tr>
  <tr>
    <td>500</td>
    <td>30,531,252</td>
    <td>1</td>
    <td>34s</td>
    <td>55s</td>
  </tr>
</table>

The test results show that the duration for simulation matching is not directly proportional to the number of threads. This is because simulation matching is a computation-intensive task, and increasing the number of threads beyond a certain point introduces additional resource competition and scheduling overhead, adversely affecting the overall execution time.

## Summary

DolphinDB provides a Simulation Matching Engine Plugin that allows for the simulation of order matching based on snapshot and tick market data. It supports settings for transaction ratios and delays. When multiple user orders in the same direction are matched simultaneously, they are executed following the principles of price priority and time priority. This feature is useful for simulating actual trades in high-frequency strategy backtesting, offering significant value for high-frequency trading strategies. The Simulation Matching Engine Plugin is developed in C++ and, combined with DolphinDB's distributed data querying capabilities, significantly reduces the overall duration of high-frequency strategy backtesting.

## Common Issues

1. **addTrade Fail / cancelTrade Fail**

    ```
    <ERROR> :addTrade Fail symbol = 002871 seq 6517875 buyOrderNo = 6484446 sellOrderNo = 6517874 qty = 100 Can't find order.
    <ERROR> :cancelTrade Fail symbol 000776 seq 204618 orderNo 127188 qty = 300 Can't find order.
    ```
    
    **Explanation**：The error occurs when tick trade data cannot find the corresponding tick order data during market snapshot synthesis, indicating incorrect market data input. Possible reasons include:
    
    - **Incorrect order of market data**：For SZSE, tick order data should precede tick trade data when timestamps match; for SSE, tick order data should follow tick trade data. Users should check the order of buy, sell, and trade data entries based on the error messages. Note, sourceType 0 indicates an order, and sourceType 1 indicates a trade.
    - **Missing market data**：Users may have only input data for a specific time range, e.g., selecting data after 10:00. In reality, there may be orders before 10:00 that traded after 10:00, leading to this error. Users should ensure the completeness of buy, sell, and trade data.
    - **Incorrect exchange selection when creating the engine**：When using the MatchingEngineSimulator::createMatchEngine interface to create an engine, the exchange parameter indicates the selected exchange. "XSHE" represents SZSE, and "XSHG" represents SSE. The order sequences for orders and trades differ between exchanges, so this error might occur if the wrong exchange was selected

1. **Hq Order Time Can not reduce**

    ```
    <ERROR> :...appendMsg(..) => Hq Order Time Can not reduce!
    ```
    
    **Explanation**The latest internal market trend time of the engine is greater than the input market trend. The following are possible reasons:
    
    - **Market data not sorted by time**：Market data must be sorted in chronological order.
    - **The data in the engine is not cleared**：The engine still retains a portion of the data, and the latest market time of this data is greater than the input market.
    - **Entered user orders greater than the latest market time**：When entering a user order, if the user's order time is greater than the latest market time, the current version of the engine will update the latest market time to the user's order time. If the subsequent market data time is less than the user's order time, it will cause the subsequent market data input to report this error message.

1. **the time of this userOrder is Less than LastOrderTime**
    
    ```
    <WARNING>:the time of this userOrder is Less than LastOrderTime, it will be set to LastOrderTime
    ```
    
    **Explanation: This log is a WARNING, not an ERROR. It indicates that the input time of the user order is earlier than the latest market time. Possible reasons include:
    
    - **Incorrect setting/forgotten setting of the user order's time column.**
    - **Asynchronous input of market data and generation of user orders.**, which might result in situations where the algorithm strategy generates user orders more slowly. This is considered normal behavior

## Attachment

- [Complete script files](data/MatchEngineSimulator/mesuc.rar)
- [ C++ code files](script/MatchingEngineSimulator/MatchEngineTest.zip) 