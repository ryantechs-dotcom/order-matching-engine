# Order-Matching Engine (Java)

A limit order book and matching engine built from scratch in Java. It accepts limit orders and two-sided market-maker quotes, keeps each product's book sorted by price, matches crossing orders automatically, and allocates fills pro-rata across resting orders at the same price.

![Java](https://img.shields.io/badge/Java-16%2B-ED8B00?style=flat-square&logo=openjdk&logoColor=white)
![Patterns](https://img.shields.io/badge/Patterns-Flyweight%20%C2%B7%20Singleton%20%C2%B7%20DTO-555?style=flat-square)

## Highlights

- **Price-sorted book per side.** Each `ProductBookSide` is a `TreeMap<Price, List<Tradable>>`. The BUY side uses reverse ordering, so the best bid and the best ask are both `firstKey()`, an O(log n) lookup.
- **Automatic matching.** Every add triggers `tryTrade()`, which keeps trading while best bid ≥ best ask and consumes volume level by level.
- **Pro-rata fill allocation.** When incoming volume can't clear a whole price level, each resting order gets a share proportional to its remaining volume, capped so the level's total is never over-filled.
- **Market-maker quotes.** A `Quote` holds a BUY and a SELL `QuoteSide`. Submitting a new quote first removes that user's existing quotes from the book, as real exchanges do.
- **Exact money math.** `Price` stores integer cents and is immutable and `Comparable`, so no floating-point rounding errors can creep in. `PriceFactory` parses strings like `"$1,234.56"` and rejects malformed input.
- **Flyweight prices.** `PriceFactory` caches one `Price` instance per value in a `HashMap`, so thousands of orders at `$134.50` share a single object.
- **Clean boundaries.** `ProductManager` and `UserManager` are Singletons. Fills are reported to users as immutable `TradableDTO` snapshots (a Java `record`), so outside code never holds live references to orders in the book.
- **Typed failure modes.** Bad input raises `InvalidPriceException`, `InvalidOrderException`, or `InvalidDataException` rather than silently corrupting the book.

## Architecture

```
ProductManager (Singleton) ──► ProductBook  (one per symbol, e.g. WMT, TGT)
                                 ├── ProductBookSide BUY   TreeMap<Price, List<Tradable>>  (descending)
                                 ├── ProductBookSide SELL  TreeMap<Price, List<Tradable>>  (ascending)
                                 └── tryTrade()  → tradeOut() on both sides
                                                      │
Tradable (interface) ◄── Order                         ▼
                     ◄── QuoteSide ◄── Quote     UserManager (Singleton) ◄── TradableDTO (record)

PriceFactory (Flyweight) ──► Price (immutable, integer cents)
```

## Project layout

```
Trading_System/src/
├── Main.java            # scripted scenario demo (full fills, partial fills, cancels, quotes)
├── prices/              # Price, PriceFactory, InvalidPriceException
├── trade/               # Order, Quote, QuoteSide, Tradable, TradableDTO, ProductBook, ProductBookSide, BookSide
└── User/                # User, UserManager, ProductManager, InvalidDataException
```

## Run it

Requires JDK 16+ (for `record`).

```bash
git clone https://github.com/ryantechs-dotcom/Trading_System.git
cd Trading_System/Trading_System
javac -d out $(find src -name "*.java")
java -cp out Main
```

`Main` walks through numbered scenarios on WMT and TGT books: single full fills, partial fills with cancels, multiple buyers against one seller, and quote replacement. It prints the book and each user's positions after every step.

## What I'd add next

- Price-time (FIFO) priority as an alternative allocation strategy behind a `FillPolicy` interface
- An Observer-based market-data feed publishing top-of-book changes
- JUnit tests for the matching and allocation invariants (no over-fills, volume conserved)
