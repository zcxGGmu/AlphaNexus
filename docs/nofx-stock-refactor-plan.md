# NOFX 项目改造方案：从加密货币到 A 股/港股

**文档版本**: v1.0
**创建时间**: 2025-12-27
**目标**: 将 NOFX 从加密货币期货交易系统改造为支持 A 股和港股的 AI 交易系统

---

## 目录

1. [改造背景与目标](#1-改造背景与目标)
2. [市场差异分析](#2-市场差异分析)
3. [架构改造方案](#3-架构改造方案)
4. [核心模块改造详情](#4-核心模块改造详情)
5. [数据源与接口](#5-数据源与接口)
6. [实施路线图](#6-实施路线图)
7. [风险与挑战](#7-风险与挑战)

---

## 1. 改造背景与目标

### 1.1 当前 NOFX 架构特点

NOFX 当前是为加密货币期货设计的交易系统，核心特点：

| 特性 | 说明 |
|------|------|
| **24/7 交易** | 全天候无间断运行 |
| **双向交易** | 支持做多(Long)和做空(Short) |
| **高杠杆** | 支持 1x-125x 杠杆 |
| **期货合约** | 永续合约、交割合约 |
| **特有数据** | OI、资金费率、清算数据 |
| **即时结算** | T+0 即时成交和结算 |

### 1.2 传统证券市场特点

#### A 股市场

| 特性 | 说明 |
|------|------|
| **交易时间** | 周一至周五 9:30-11:30, 13:00-15:00 |
| **交易机制** | T+1（当日买入次日才能卖出） |
| **交易方向** | 仅做多（融资融券有限且门槛高） |
| **杠杆** | 融资融券（最高 1:1，需开通权限） |
| **最小单位** | 100 股（1 手） |
| **涨跌限制** | 主板 10%, 创业板/科创板 20% |
| **特有数据** | 换手率、市值、市盈率、股息率 |

#### 港股市场

| 特性 | 说明 |
|------|------|
| **交易时间** | 周一至周五 9:30-12:00, 13:00-16:00 |
| **交易机制** | T+0（当日可买卖） |
| **交易方向** | 仅做多（融资融券有限） |
| **杠杆** | 融资（孖展）需券商授权 |
| **最小单位** | 1 股（部分股票需整手买卖） |
| **涨跌限制** | 无涨跌停限制（但有市调机制） |
| **特有数据** | 换手率、市值、港股通资金流向 |

### 1.3 改造目标

1. **保留 NOFX 核心优势**：AI 决策引擎、多 AI 协作、回测系统
2. **适配传统证券**：支持 A 股和港股的交易规则
3. **多市场支持**：同时支持加密货币、A 股、港股
4. **最小化改动**：通过抽象层实现市场适配

---

## 2. 市场差异分析

### 2.1 核心差异对比

| 维度 | 加密货币期货 | A 股 | 港股 |
|------|-------------|------|------|
| **交易时间** | 24/7 | 4小时/天 | 5小时/天 |
| **交收机制** | T+0 | T+1 | T+0 |
| **做多/做空** | ✓✓ | ✓ (融资融券有限) | ✓ (融资有限) |
| **杠杆** | 1x-125x | 1x (融资1:1) | 1x (融资有限) |
| **涨跌限制** | 无 | ±10%/20% | 无 (市调机制) |
| **最小单位** | 灵活 | 100股 | 1股/整手 |
| **持仓量(OI)** | ✓ | ✗ | ✗ |
| **资金费率** | ✓ | ✗ | ✗ |
| **换手率** | - | ✓ | ✓ |
| **市值数据** | - | ✓ | ✓ |

### 2.2 数据差异

#### NOFX 当前使用的数据

```go
// 加密货币特有数据
type MarketData struct {
    Kline         []Kline      // K线数据 (OHLCV)
    OpenInterest  float64      // 持仓量 (OI)
    OIChange      float64      // OI 变化
    FundingRate   float64      // 资金费率
    Liquidations  []Liquidation // 清算数据
}
```

#### 传统证券需要的数据

```go
// 传统证券特有数据
type StockMarketData struct {
    Kline         []Kline      // K线数据 (OHLCV)
    TurnoverRate  float64      // 换手率
    MarketCap     float64      // 市值
    PEL          float64      // 市盈率 (PE)
    PBR          float64      // 市净率 (PB)
    DividendYield float64      // 股息率
    NorthboundFlow float64     // 港股通资金流向 (港股)
}
```

### 2.3 交易逻辑差异

#### NOFX 当前交易逻辑

```
决策引擎输出:
- LONG  (做多)
- SHORT (做空)
- CLOSE (平仓)

执行逻辑:
OpenLong()   // 开多仓
OpenShort()  // 开空仓
CloseLong()  // 平多仓
CloseShort() // 平空仓
```

#### 传统证券交易逻辑

```
决策引擎输出:
- BUY   (买入)
- SELL  (卖出)
- HOLD  (持有)

执行逻辑:
Buy()   // 买入股票
Sell()  // 卖出股票
```

---

## 3. 架构改造方案

### 3.1 新架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      改造后的 NOFX 架构                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐     ┌─────────────────┐                       │
│  │   Web Frontend  │     │   AI 模型层      │                       │
│  │   (React + TS)  │     │   - DeepSeek    │                       │
│  │                 │     │   - GPT/Claude  │                       │
│  │  - 策略工作室    │     │   - Gemini 等   │                       │
│  │  - 辩论竞技场    │     └────────┬────────┘                       │
│  │  - 回测实验室    │              │                                │
│  │  - 实时仪表板    │              ▼                                │
│  └────────┬────────┘     ┌─────────────────┐                       │
│           │              │   决策引擎       │                       │
│           │              │  /decision/     │                       │
│           │              │  /debate/       │                       │
│           │              └────────┬────────┘                       │
│           │                       │                                │
│           ▼                       ▼                                │
│  ┌─────────────────────────────────────────┐                      │
│  │            HTTP API 服务器              │                      │
│  │              (Gin + Go)                 │                      │
│  └──────────────────────┬──────────────────┘                      │
│                         │                                           │
│         ┌───────────────┼───────────────┐                         │
│         ▼               ▼               ▼                         │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                    │
│  │  市场抽象层  │ │  回测引擎   │ │  存储层     │                    │
│  │ /market/   │ │  /backtest/│ │  /store/   │                    │
│  │            │ │            │ │            │                    │
│  │ - 统一数据   │ │ - 多市场    │ │ - SQLite   │                    │
│  │   格式      │ │   回测      │ │ - 加密存储  │                    │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘                    │
│        │              │              │                            │
│        └──────────────┼──────────────┘                            │
│                       ▼                                           │
│         ┌─────────────────────────────┐                          │
│         │      交易接口抽象层          │                          │
│         │      /trader/interface.go   │                          │
│         └─────────────────────────────┘                          │
│                         │                                         │
│         ┌───────────────┼───────────────┐                        │
│         ▼               ▼               ▼                        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                   │
│  │ Crypto     │ │  A-Stock    │ │ HK-Stock   │                   │
│  │ Trader     │ │  Trader     │ │ Trader     │                   │
│  │            │ │             │ │            │                   │
│  │ - Binance  │ │ - 华泰证券  │ │ - 富途     │                   │
│  │ - Bybit    │ │ - 富途      │ │ - 老虎     │                   │
│  │ - OKX      │ │ - 老虎      │ │ - 盈透     │                   │
│  │ - Hyperliq │ │ - 盈透证券  │ │ - 雪球     │                   │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘                   │
│        │              │              │                            │
│        └──────────────┼──────────────┘                            │
│                       ▼                                           │
│         ┌─────────────────────────────┐                          │
│         │      数据提供商抽象层        │                          │
│         └─────────────────────────────┘                          │
│                         │                                         │
│         ┌───────────────┼───────────────┐                        │
│         ▼               ▼               ▼                        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                   │
│  │ Crypto     │ │  A-Stock    │ │ HK-Stock   │                   │
│  │ Provider   │ │  Provider   │ │ Provider   │                   │
│  │            │ │             │ │            │                   │
│  │ - CoinAnk  │ │ - Tushare   │ │ - AAStocks │                   │
│  │ - Exchange │ │ - AkShare   │ │ - Eastmoney│                   │
│  │   API      │ │ - Eastmoney │ │ - Yahoo     │                   │
│  └────────────┘ └────────────┘ └────────────┘                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 关键抽象层设计

#### 3.2.1 市场类型枚举

```go
package market

// MarketType 市场类型
type MarketType string

const (
    MarketCrypto    MarketType = "CRYPTO"    // 加密货币
    MarketAStock    MarketType = "ASTOCK"    // A股
    MarketHKStock   MarketType = "HKSTOCK"   // 港股
    MarketUSStock   MarketType = "USSTOCK"   // 美股 (未来扩展)
)

// MarketConfig 市场配置
type MarketConfig struct {
    Type           MarketType
    TradingHours   []TradingHour  // 交易时段
    SettlementMode string         // T+0, T+1 等
    SupportShort   bool           // 是否支持做空
    MaxLeverage    float64        // 最大杠杆
    MinUnit        float64        // 最小交易单位
    PriceLimit     float64        // 涨跌限制 (0 表示无限制)
}

// GetMarketConfig 获取市场配置
func GetMarketConfig(marketType MarketType) *MarketConfig {
    switch marketType {
    case MarketCrypto:
        return &MarketConfig{
            Type:           MarketCrypto,
            TradingHours:   []TradingHour{{"00:00", "23:59"}}, // 24/7
            SettlementMode: "T+0",
            SupportShort:   true,
            MaxLeverage:    125.0,
            MinUnit:        0.00000001, // 灵活
            PriceLimit:     0,          // 无限制
        }
    case MarketAStock:
        return &MarketConfig{
            Type: MarketAStock,
            TradingHours: []TradingHour{
                {"09:30", "11:30"},
                {"13:00", "15:00"},
            },
            SettlementMode: "T+1",
            SupportShort:   false, // 融资融券有限
            MaxLeverage:    1.0,   // 融资最多1:1
            MinUnit:        100,   // 100股
            PriceLimit:     0.10,  // 主板10%
        }
    case MarketHKStock:
        return &MarketConfig{
            Type: MarketHKStock,
            TradingHours: []TradingHour{
                {"09:30", "12:00"},
                {"13:00", "16:00"},
            },
            SettlementMode: "T+0",
            SupportShort:   false,
            MaxLeverage:    1.0,
            MinUnit:        1,     // 1股
            PriceLimit:     0,     // 无限制 (有市调机制)
        }
    }
    return nil
}
```

#### 3.2.2 统一的交易接口

```go
package trader

import "context"

// OrderType 订单类型
type OrderType string

const (
    OrderMarket  OrderType = "MARKET"  // 市价单
    OrderLimit   OrderType = "LIMIT"   // 限价单
    OrderStop    OrderType = "STOP"    // 止损单
)

// OrderSide 订单方向 (兼容做多和做空)
type OrderSide string

const (
    SideBuy  OrderSide = "BUY"   // 买入/做多
    SideSell OrderSide = "SELL"  // 卖出/做空
)

// UniversalOrderRequest 通用订单请求
type UniversalOrderRequest struct {
    Symbol     string     // 交易标的 (如 "00700.HK", "600036.SH")
    Side       OrderSide  // 方向: BUY/SELL
    OrderType  OrderType  // 订单类型: MARKET/LIMIT
    Quantity   float64    // 数量
    Price      float64    // 价格 (限价单必填)
    StopPrice  float64    // 止损价
}

// UniversalPosition 通用持仓
type UniversalPosition struct {
    Symbol        string    // 交易标的
    Side          OrderSide // 方向: LONG/SHORT
    Quantity      float64   // 持仓数量
    AvgPrice      float64   // 平均成本
    CurrentPrice  float64   // 当前价格
    UnrealizedPnL float64   // 未实现盈亏
    MarketValue   float64   // 市值
    MarketType    market.MarketType // 市场类型
}

// UniversalTrader 通用交易接口
type UniversalTrader interface {
    // 初始化
    Init(ctx context.Context, config *TraderConfig) error

    // 市场数据
    GetMarketPrice(ctx context.Context, symbol string) (float64, error)
    GetKlines(ctx context.Context, symbol string, interval string, limit int) ([]Kline, error)

    // 下单
    PlaceOrder(ctx context.Context, req *UniversalOrderRequest) (*OrderResult, error)
    CancelOrder(ctx context.Context, orderID string) error
    CancelAllOrders(ctx context.Context, symbol string) error

    // 持仓
    GetPositions(ctx context.Context) ([]UniversalPosition, error)
    ClosePosition(ctx context.Context, symbol string, quantity float64) error

    // 账户
    GetAccountInfo(ctx context.Context) (*AccountInfo, error)

    // 市场特性
    GetMarketType() market.MarketType
    ValidateOrder(ctx context.Context, req *UniversalOrderRequest) error
}
```

#### 3.2.3 统一的市场数据接口

```go
package provider

// MarketDataProvider 市场数据提供者接口
type MarketDataProvider interface {
    // 基础数据
    GetKlines(symbol string, startTime, endTime int64, interval string) ([]Kline, error)
    GetRealtimePrice(symbol string) (float64, error)

    // 市场深度
    GetOrderBook(symbol string, depth int) (*OrderBook, error)

    // 市场特有数据
    GetCryptoData(symbol string) (*CryptoSpecificData, error)   // OI, 资金费率
    GetStockData(symbol string) (*StockSpecificData, error)     // 换手率, 市值

    // 股票特有
    GetStockList(market string) ([]StockInfo, error)            // 股票列表
    GetIndexConstituents(index string) ([]string, error)        // 指数成分股
}

// CryptoSpecificData 加密货币特有数据
type CryptoSpecificData struct {
    OpenInterest  float64 // 持仓量
    OIChange      float64 // OI 变化
    FundingRate   float64 // 资金费率
    Liquidations  []Liquidation
}

// StockSpecificData 股票特有数据
type StockSpecificData struct {
    TurnoverRate  float64 // 换手率
    MarketCap     float64 // 市值
    CirculatingCap float64 // 流通市值
    PEL           float64 // 市盈率 (动)
    PETTM         float64 // 市盈率 (静)
    PBR           float64 // 市净率
    DividendYield float64 // 股息率
    TotalShare    float64 // 总股本
    FloatShare    float64 // 流通股本
    NorthboundFlow float64 // 港股通资金流向 (港股)
}
```

---

## 4. 核心模块改造详情

### 4.1 Trader 接口层改造

#### 当前接口 (仅支持加密货币)

```go
// trader/interface.go - 当前版本
type Trader interface {
    GetBalance() (map[string]interface{}, error)
    GetPositions() ([]map[string]interface{}, error)
    OpenLong(symbol string, quantity float64, leverage int) (map[string]interface{}, error)
    OpenShort(symbol string, quantity float64, leverage int) (map[string]interface{}, error)
    CloseLong(symbol string, quantity float64) (map[string]interface{}, error)
    CloseShort(symbol string, quantity float64) (map[string]interface{}, error)
    SetLeverage(symbol string, leverage int) error
    SetMarginMode(symbol string, isCrossMargin bool) error
    // ... 其他方法
}
```

#### 改造后的接口 (支持多市场)

```go
// trader/universal_interface.go - 新版本
type UniversalTrader interface {
    // 初始化
    Init(ctx context.Context, config *TraderConfig) error

    // 通用下单 (适配做多/做空, 买入/卖出)
    PlaceOrder(ctx context.Context, req *UniversalOrderRequest) (*OrderResult, error)

    // 持仓管理 (统一格式)
    GetPositions(ctx context.Context) ([]UniversalPosition, error)
    ClosePosition(ctx context.Context, symbol string, quantity float64) error

    // 账户信息
    GetAccountInfo(ctx context.Context) (*AccountInfo, error)

    // 市场数据
    GetMarketPrice(ctx context.Context, symbol string) (float64, error)

    // 市场类型
    GetMarketType() market.MarketType

    // 订单验证 (根据市场规则)
    ValidateOrder(ctx context.Context, req *UniversalOrderRequest) error
}

// TraderConfig 交易者配置
type TraderConfig struct {
    MarketType    market.MarketType  // 市场类型
    Exchange      string             // 交易所/券商
    APIKey        string             // API 密钥
    APISecret     string             // API 密钥
    Sandbox       bool               // 是否模拟盘
    Leverage      float64            // 杠杆倍数 (仅加密货币)
}
```

#### A 股 Trader 实现

```go
// trader/astock/futu_astock.go
package astock

import (
    "context"
    "github.com/futunnopen/futunnopen-go"
)

type FutuAStockTrader struct {
    client    *futunnopen.OpenQuoteAgent
    marketCfg *market.MarketConfig
}

func NewFutuAStockTrader(apiKey, apiSecret string) *FutuAStockTrader {
    return &FutuAStockTrader{
        client:    futunnopen.NewOpenQuoteAgent(apiKey, apiSecret),
        marketCfg: market.GetMarketConfig(market.MarketAStock),
    }
}

func (t *FutuAStockTrader) Init(ctx context.Context, config *trader.TraderConfig) error {
    // 初始化富途 A 股交易连接
    return t.client.Start()
}

func (t *FutuAStockTrader) PlaceOrder(ctx context.Context, req *trader.UniversalOrderRequest) (*trader.OrderResult, error) {
    // 验证订单
    if err := t.ValidateOrder(ctx, req); err != nil {
        return nil, err
    }

    // 转换为富途订单格式
    futuOrder := &futunnopen.PlaceOrderRequest{
        Code:     req.Symbol,
        TrdEnv:   futunnopen.TrdEnv_Real,     // 实盘
        TrdMarket: futunnopen.TrdMarket_CN,   // A 股市场
        Price:    req.Price,
        Qty:      int64(req.Quantity),
        OrderType: convertOrderType(req.OrderType),
        Side:     convertSide(req.Side),
    }

    // 下单
    resp, err := t.client.PlaceOrder(ctx, futuOrder)
    if err != nil {
        return nil, err
    }

    return &trader.OrderResult{
        OrderID: resp.OrderID,
        Status:  resp.Status,
        Message: resp.Message,
    }, nil
}

func (t *FutuAStockTrader) ValidateOrder(ctx context.Context, req *trader.UniversalOrderRequest) error {
    // A 股特定验证

    // 1. 检查交易时间
    if !t.isTradingTime(time.Now()) {
        return fmt.Errorf("outside A-stock trading hours")
    }

    // 2. 检查最小单位 (100股)
    if req.Quantity < 100 {
        return fmt.Errorf("minimum order quantity is 100 shares")
    }
    if int(req.Quantity)%100 != 0 {
        return fmt.Errorf("order quantity must be multiple of 100")
    }

    // 3. 检查价格限制 (涨跌停)
    currentPrice, _ := t.GetMarketPrice(ctx, req.Symbol)
    limitUp := currentPrice * 1.1   // 涨停价
    limitDown := currentPrice * 0.9 // 跌停价

    if req.OrderType == trader.OrderLimit {
        if req.Price > limitUp {
            return fmt.Errorf("price exceeds limit up: %.2f", limitUp)
        }
        if req.Price < limitDown {
            return fmt.Errorf("price below limit down: %.2f", limitDown)
        }
    }

    return nil
}

func (t *FutuAStockTrader) isTradingTime(t time.Time) bool {
    weekday := t.Weekday()
    if weekday == time.Saturday || weekday == time.Sunday {
        return false
    }

    hour := t.Hour()
    minute := t.Minute()
    timeMinutes := hour*60 + minute

    // 9:30-11:30, 13:00-15:00
    morningStart := 9*60 + 30
    morningEnd := 11*60 + 30
    afternoonStart := 13*60
    afternoonEnd := 15*60

    return (timeMinutes >= morningStart && timeMinutes <= morningEnd) ||
           (timeMinutes >= afternoonStart && timeMinutes <= afternoonEnd)
}

func (t *FutuAStockTrader) GetMarketType() market.MarketType {
    return market.MarketAStock
}
```

### 4.2 Decision 引擎改造

#### 当前问题

```go
// decision/schema.go - 当前版本
var DataDictionary = map[string]map[string]BilingualFieldDef{
    "MarketData": {
        "OI": {
            NameZH: "持仓量",
            NameEN: "Open Interest",
            Unit:   "USDT",
            DescZH: "未平仓合约的总价值",
        },
        "FundingRate": {
            NameZH: "资金费率",
            NameEN: "Funding Rate",
            Unit:   "%",
            DescZH: "永续合约资金费率",
        },
        // ... 加密货币特有字段
    },
}
```

#### 改造方案

```go
// decision/universal_schema.go - 新版本

// MarketDataSchema 市场数据 Schema (根据市场类型动态生成)
type MarketDataSchema struct {
    CommonFields   []BilingualFieldDef  // 通用字段 (K线, 成交量)
    CryptoFields   []BilingualFieldDef  // 加密货币字段 (OI, 资金费率)
    StockFields    []BilingualFieldDef  // 股票字段 (换手率, 市值, PE)
}

func GetMarketDataSchema(marketType market.MarketType) *MarketDataSchema {
    commonFields := []BilingualFieldDef{
        {
            NameZH: "K线数据",
            NameEN: "Candlestick Data",
            DescZH: "开高低收价格和成交量",
        },
        {
            NameZH: "成交量",
            NameEN: "Volume",
            Unit:   "shares",
            DescZH: "该时间段的交易量",
        },
    }

    cryptoFields := []BilingualFieldDef{
        {
            NameZH: "持仓量",
            NameEN: "Open Interest",
            Unit:   "USDT",
            DescZH: "未平仓合约的总价值",
        },
        {
            NameZH: "资金费率",
            NameEN: "Funding Rate",
            Unit:   "%",
            DescZH: "永续合约资金费率",
        },
    }

    stockFields := []BilingualFieldDef{
        {
            NameZH: "换手率",
            NameEN: "Turnover Rate",
            Unit:   "%",
            DescZH: "成交量 / 流通股本 × 100%",
        },
        {
            NameZH: "总市值",
            NameEN: "Total Market Cap",
            Unit:   "CNY/HKD",
            DescZH: "股价 × 总股本",
        },
        {
            NameZH: "市盈率 (PE)",
            NameEN: "Price to Earnings Ratio",
            Unit:   "x",
            DescZH: "股价 / 每股收益",
        },
        {
            NameZH: "市净率 (PB)",
            NameEN: "Price to Book Ratio",
            Unit:   "x",
            DescZH: "股价 / 每股净资产",
        },
        {
            NameZH: "股息率",
            NameEN: "Dividend Yield",
            Unit:   "%",
            DescZH: "年度股息 / 股价 × 100%",
        },
    }

    return &MarketDataSchema{
        CommonFields: commonFields,
        CryptoFields: cryptoFields,
        StockFields:  stockFields,
    }
}

// BuildAIPrompt 构建 AI Prompt (根据市场类型)
func BuildAIPrompt(marketType market.MarketType, lang Language, data *MarketData) string {
    var prompt string

    // 添加通用数据
    prompt += formatKlineData(data.Klines)

    // 根据市场类型添加特定数据
    switch marketType {
    case market.MarketCrypto:
        prompt += formatCryptoData(data.CryptoData)
    case market.MarketAStock, market.MarketHKStock:
        prompt += formatStockData(data.StockData)
    }

    // 添加市场规则
    prompt += getMarketRules(marketType, lang)

    return prompt
}

func getMarketRules(marketType market.MarketType, lang Language) string {
    switch marketType {
    case market.MarketCrypto:
        return getPromptCryptoRules(lang)
    case market.MarketAStock:
        return getPromptAStockRules(lang)
    case market.MarketHKStock:
        return getPromptHKStockRules(lang)
    }
    return ""
}

func getPromptAStockRules(lang Language) string {
    if lang == LangChinese {
        return `
## A 股交易规则

### 市场限制
- **交易时间**: 周一至周五 9:30-11:30, 13:00-15:00
- **交收机制**: T+1 (当日买入次日才能卖出)
- **交易方向**: 仅做多 (融资融券需特殊权限)
- **最小单位**: 100 股 (1 手)
- **涨跌限制**: 主板 ±10%, 创业板/科创板 ±20%

### A 股特有指标
- **换手率**: 反映股票活跃度, 换手率高表示交易活跃
- **市盈率 (PE)**: 估值指标, 低 PE 可能被低估
- **市净率 (PB)**: 估值指标, 低 PB 可能被低估
- **股息率**: 分红回报率, 高股息率适合稳健投资

### 决策输出
请仅输出以下决策之一:
- **BUY**: 买入股票
- **SELL**: 卖出股票
- **HOLD**: 持有不动

请勿输出 SHORT (做空) 指令, A 股市场不支持普通做空。
`
    }
    return `
## A-Stock Trading Rules

### Market Limitations
- **Trading Hours**: Mon-Fri 9:30-11:30, 13:00-15:00
- **Settlement**: T+1 (can only sell next day after buying)
- **Direction**: Long only (short selling requires special permissions)
- **Min Unit**: 100 shares (1 lot)
- **Price Limit**: ±10% for main board, ±20% for ChiNext/STAR

### A-Stock Specific Metrics
- **Turnover Rate**: Stock activity indicator
- **PE Ratio**: Valuation indicator, low PE may indicate undervaluation
- **PB Ratio**: Valuation indicator, low PB may indicate undervaluation
- **Dividend Yield**: Dividend return, high yield suitable for steady investment

### Decision Output
Please only output one of the following decisions:
- **BUY**: Buy stock
- **SELL**: Sell stock
- **HOLD**: Hold position

Do NOT output SHORT (short sell) as A-stock market doesn't support regular short selling.
`
}

func getPromptHKStockRules(lang Language) string {
    if lang == LangChinese {
        return `
## 港股交易规则

### 市场特点
- **交易时间**: 周一至周五 9:30-12:00, 13:00-16:00
- **交收机制**: T+0 (当日可买卖)
- **交易方向**: 仅做多 (融资有限)
- **最小单位**: 1 股 (部分股票需整手)
- **涨跌限制**: 无涨跌停 (但有市调机制)

### 港股特有指标
- **换手率**: 反映股票活跃度
- **市值**: 股价 × 总股本
- **港股通资金流向**: 南下资金流入流出情况

### 决策输出
请仅输出以下决策之一:
- **BUY**: 买入股票
- **SELL**: 卖出股票
- **HOLD**: 持有不动
`
    }
    return `
## Hong Kong Stock Trading Rules

### Market Features
- **Trading Hours**: Mon-Fri 9:30-12:00, 13:00-16:00
- **Settlement**: T+0 (can buy and sell on same day)
- **Direction**: Long only (financing limited)
- **Min Unit**: 1 share (some stocks require round lots)
- **Price Limit**: No limit (but has volatility control mechanism)

### HK-Stock Specific Metrics
- **Turnover Rate**: Stock activity indicator
- **Market Cap**: Share price × Total shares
- **Northbound Flow**: Southbound capital inflow/outflow

### Decision Output
Please only output one of the following:
- **BUY**: Buy stock
- **SELL**: Sell stock
- **HOLD**: Hold position
`
}
```

### 4.3 Backtest 引擎改造

#### 当前回测引擎限制

```go
// backtest/runner.go - 当前版本
type BacktestRunner struct {
    // 只支持加密货币的回测逻辑
    strategy   *Strategy
    aiModel    *AIModel
    exchange   string // Binance, Bybit, etc.
}
```

#### 改造方案

```go
// backtest/universal_runner.go - 新版本

type UniversalBacktestRunner struct {
    config     *BacktestConfig
    marketType market.MarketType
    trader     UniversalTrader
    provider   MarketDataProvider
}

type BacktestConfig struct {
    MarketType    market.MarketType  // 市场类型
    Symbol        string             // 交易标的
    StartTime     time.Time          // 回测开始时间
    EndTime       time.Time          // 回测结束时间
    InitialCapital float64           // 初始资金
    Strategy      *Strategy          // 策略配置
    AIModel       string             // AI 模型

    // 市场特定配置
    Commission    float64            // 手续费率
    Slippage      float64            // 滑点
    Leverage      float64            // 杠杆 (仅加密货币)
}

func (r *UniversalBacktestRunner) Run(ctx context.Context) (*BacktestResult, error) {
    // 根据市场类型执行不同的回测逻辑
    switch r.config.MarketType {
    case market.MarketCrypto:
        return r.runCryptoBacktest(ctx)
    case market.MarketAStock:
        return r.runAStockBacktest(ctx)
    case market.MarketHKStock:
        return r.runHKStockBacktest(ctx)
    }
}

func (r *UniversalBacktestRunner) runAStockBacktest(ctx context.Context) (*BacktestResult, error) {
    // A 股特定回测逻辑

    // 1. 获取历史数据
    klines, err := r.provider.GetKlines(
        r.config.Symbol,
        r.config.StartTime.Unix(),
        r.config.EndTime.Unix(),
        "1d",
    )
    if err != nil {
        return nil, err
    }

    // 2. 初始化模拟账户 (T+1 逻辑)
    account := &AStockSimAccount{
        Cash:         r.config.InitialCapital,
        Holdings:     make(map[string]*Holding),
        BuyQueue:     make([]*BuyOrder, 0), // 买入队列 (次日才能卖)
        Commission:   r.config.Commission,   // 通常万分之2.5
    }

    result := &BacktestResult{
        Trades:     make([]*Trade, 0),
        EquityCurve: make([]EquityPoint, 0),
    }

    // 3. 逐日回放
    for _, kline := range klines {
        // 处理买入队列 (T+1: 昨天买的今天可以卖)
        account.ProcessBuyQueue()

        // 获取市场数据
        marketData := r.buildMarketData(kline)

        // AI 决策
        decision, err := r.getAIDecision(ctx, marketData)
        if err != nil {
            continue
        }

        // 执行决策
        switch decision.Action {
        case "BUY":
            // 检查资金是否足够
            cost := kline.Close * kline.Volume
            if account.Cash >= cost {
                account.PlaceBuyOrder(kline, decision.Quantity)
            }
        case "SELL":
            // 检查是否可卖 (T+1: 必须是昨天或更早买入的)
            if account.CanSell(kline.Symbol) {
                account.Sell(kline.Symbol, kline.Close)
            }
        case "HOLD":
            // 不操作
        }

        // 更新权益
        account.UpdateEquity(kline.Close)

        // 记录权益曲线点
        result.EquityCurve = append(result.EquityCurve, EquityPoint{
            Time:   kline.StartTime,
            Equity: account.TotalEquity(),
        })
    }

    // 计算性能指标
    result.Metrics = r.calculateMetrics(result.EquityCurve, result.Trades)

    return result, nil
}

// AStockSimAccount A 股模拟账户 (支持 T+1)
type AStockSimAccount struct {
    Cash       float64                    // 可用现金
    Holdings   map[string]*Holding        // 持仓
    BuyQueue   []*BuyOrder                // 买入队列 (次日才能卖)
    Commission float64                    // 手续费率
}

type Holding struct {
    Symbol     string
    Quantity   float64
    AvgPrice   float64
    BuyDate    time.Time
}

type BuyOrder struct {
    Symbol     string
    Quantity   float64
    Price      float64
    BuyDate    time.Time
}

func (a *AStockSimAccount) PlaceBuyOrder(kline *Kline, quantity float64) {
    cost := kline.Close * quantity
    fee := cost * a.Commission

    a.Cash -= (cost + fee)

    // 加入买入队列 (次日才能卖)
    a.BuyQueue = append(a.BuyQueue, &BuyOrder{
        Symbol:   kline.Symbol,
        Quantity: quantity,
        Price:    kline.Close,
        BuyDate:  time.Unix(kline.StartTime/1000, 0),
    })
}

func (a *AStockSimAccount) ProcessBuyQueue() {
    // 将昨天买入的股票转移到持仓
    for _, order := range a.BuyQueue {
        if time.Since(order.BuyDate) >= 24*time.Hour {
            if _, exists := a.Holdings[order.Symbol]; !exists {
                a.Holdings[order.Symbol] = &Holding{
                    Symbol:   order.Symbol,
                    Quantity: order.Quantity,
                    AvgPrice: order.Price,
                    BuyDate:  order.BuyDate,
                }
            } else {
                // 加仓
                h := a.Holdings[order.Symbol]
                totalCost := h.AvgPrice*h.Quantity + order.Price*order.Quantity
                h.Quantity += order.Quantity
                h.AvgPrice = totalCost / h.Quantity
            }
        }
    }

    // 清理已处理的订单
    a.BuyQueue = filterBuyQueue(a.BuyQueue, func(o *BuyOrder) bool {
        return time.Since(o.BuyDate) < 24*time.Hour
    })
}

func (a *AStockSimAccount) CanSell(symbol string) bool {
    holding, exists := a.Holdings[symbol]
    if !exists {
        return false
    }

    // 检查是否已过 T+1
    return time.Since(holding.BuyDate) >= 24*time.Hour
}
```

---

## 5. 数据源与接口

### 5.1 A 股数据源

#### Tushare (推荐)

```go
package provider

import (
    "context"
    "github.com/tushare/pro"
)

type TushareProvider struct {
    client *pro.Client
    token  string
}

func NewTushareProvider(token string) *TushareProvider {
    return &TushareProvider{
        client: pro.NewClient(token),
        token:  token,
    }
}

func (p *TushareProvider) GetKlines(symbol string, startTime, endTime int64, interval string) ([]Kline, error) {
    // Tushare API: daily
    df, err := p.client.Daily(
        pro.TsCode(symbol),
        pro.StartDate(formatDate(startTime)),
        pro.EndDate(formatDate(endTime)),
    )
    if err != nil {
        return nil, err
    }

    klines := make([]Kline, len(df))
    for i, row := range df {
        klines[i] = Kline{
            Symbol:    symbol,
            OpenTime:  parseDate(row["trade_date"]),
            Open:      row["open"].(float64),
            High:      row["high"].(float64),
            Low:       row["low"].(float64),
            Close:     row["close"].(float64),
            Volume:    row["vol"].(float64),
        }
    }
    return klines, nil
}

func (p *TushareProvider) GetStockData(symbol string) (*StockSpecificData, error) {
    // 获取日线基本信息
    dfDaily, err := p.client.DailyBasic(pro.TsCode(symbol))
    if err != nil {
        return nil, err
    }

    // 获取公司基本信息
    dfCompany, err := p.client.Company(pro.TsCode(symbol))
    if err != nil {
        return nil, err
    }

    data := &StockSpecificData{
        TurnoverRate:  dfDaily[0]["turnover_rate"].(float64),
        MarketCap:     dfDaily[0]["total_mv"].(float64) * 10000, // 万元 -> 元
        CirculatingCap: dfDaily[0]["circ_mv"].(float64) * 10000,
        PEL:          dfDaily[0]["pe"].(float64),
        PETTM:        dfDaily[0]["pe_ttm"].(float64),
        PBR:          dfDaily[0]["pb"].(float64),
        TotalShare:   dfCompany[0]["total_share"].(float64) * 10000,
        FloatShare:   dfCompany[0]["float_share"].(float64) * 10000,
        DividendYield: dfDaily[0]["dv_ratio"].(float64),
    }

    return data, nil
}

func (p *TushareProvider) GetStockList(market string) ([]StockInfo, error) {
    // 获取股票列表
    df, err := p.client.StockBasic(
        pro.ListStatus("L"), // 仅上市股票
    )
    if err != nil {
        return nil, err
    }

    stocks := make([]StockInfo, len(df))
    for i, row := range df {
        stocks[i] = StockInfo{
            Symbol:    row["ts_code"].(string),
            Name:      row["name"].(string),
            Market:    row["market"].(string), // SMB, SZME 等
            Industry:  row["industry"].(string),
            ListDate:  row["list_date"].(string),
        }
    }
    return stocks, nil
}

func (p *TushareProvider) GetIndexConstituents(index string) ([]string, error) {
    // 获取指数成分股
    df, err := p.client.IndexMember(
        pro.TsCode(index), // 如 "000300.SH" (沪深300)
    )
    if err != nil {
        return nil, err
    }

    symbols := make([]string, len(df))
    for i, row := range df {
        symbols[i] = row["con_code"].(string)
    }
    return symbols, nil
}
```

#### AkShare (免费备选)

```go
package provider

import (
    "github.com/akshare/akshare-go"
)

type AkShareProvider struct {
    client *akshare.Client
}

func NewAkShareProvider() *AkShareProvider {
    return &AkShareProvider{
        client: akshare.NewClient(),
    }
}

func (p *AkShareProvider) GetKlines(symbol string, startTime, endTime int64, interval string) ([]Kline, error) {
    // AkShare API: stock_zh_a_hist
    df, err := p.client.StockZhAHist(
        symbol,               // 如 "000001"
        akshare.PeriodDaily,
        akshare.AdjustQFQ,    // 前复权
        formatDate(startTime),
        formatDate(endTime),
    )
    if err != nil {
        return nil, err
    }

    klines := make([]Kline, len(df))
    for i, row := range df {
        klines[i] = Kline{
            Symbol:    symbol,
            OpenTime:  parseDate(row["日期"]),
            Open:      row["开盘"].(float64),
            High:      row["最高"].(float64),
            Low:       row["最低"].(float64),
            Close:     row["收盘"].(float64),
            Volume:    row["成交量"].(float64),
        }
    }
    return klines, nil
}
```

### 5.2 港股数据源

#### AAStocks (免费)

```go
package provider

import (
    "context"
    "net/http"
    "encoding/json"
)

type AAStocksProvider struct {
    client *http.Client
}

func NewAAStocksProvider() *AAStocksProvider {
    return &AAStocksProvider{
        client: &http.Client{},
    }
}

func (p *AAStocksProvider) GetKlines(symbol string, startTime, endTime int64, interval string) ([]Kline, error) {
    // AAStocks API (需要爬虫或使用第三方封装)
    // 这里使用简化示例
    url := fmt.Sprintf("https://www.aastocks.com/api/historical-data/%s", symbol)

    resp, err := p.client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var data struct {
        Data []struct {
            Date   string  `json:"date"`
            Open   float64 `json:"open"`
            High   float64 `json:"high"`
            Low    float64 `json:"low"`
            Close  float64 `json:"close"`
            Volume float64 `json:"volume"`
        } `json:"data"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
        return nil, err
    }

    klines := make([]Kline, len(data.Data))
    for i, d := range data.Data {
        klines[i] = Kline{
            Symbol:    symbol,
            OpenTime:  parseDate(d.Date),
            Open:      d.Open,
            High:      d.High,
            Low:       d.Low,
            Close:     d.Close,
            Volume:    d.Volume,
        }
    }
    return klines, nil
}

func (p *AAStocksProvider) GetStockData(symbol string) (*StockSpecificData, error) {
    // 获取港股基本面数据
    url := fmt.Sprintf("https://www.aastocks.com/api/stock-fundamentals/%s", symbol)

    resp, err := p.client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var data struct {
        TurnoverRate  float64 `json:"turnoverRate"`
        MarketCap     float64 `json:"marketCap"`
        PEL          float64 `json:"pe"`
        PBR          float64 `json:"pb"`
        DividendYield float64 `json:"dividendYield"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
        return nil, err
    }

    return &StockSpecificData{
        TurnoverRate:  data.TurnoverRate,
        MarketCap:     data.MarketCap,
        PEL:          data.PEL,
        PBR:          data.PBR,
        DividendYield: data.DividendYield,
    }, nil
}
```

#### Yahoo Finance (备选)

```go
package provider

import (
    "github.com/go-gota/go-dataframe"
    "github.com/pyed/go-yahoo-finance"
)

type YahooFinanceProvider struct {
    client *yfinance.Client
}

func NewYahooFinanceProvider() *YahooFinanceProvider {
    return &YahooFinanceProvider{
        client: yfinance.NewClient(),
    }
}

func (p *YahooFinanceProvider) GetKlines(symbol string, startTime, endTime int64, interval string) ([]Kline, error) {
    // 港股代码格式: "0700.HK"
    yahooSymbol := convertToYahooSymbol(symbol)

    df, err := p.client.Historical(
        yahooSymbol,
        yfinance.WithStartDate(formatDate(startTime)),
        yfinance.WithEndDate(formatDate(endTime)),
        yfinance.WithInterval(interval),
    )
    if err != nil {
        return nil, err
    }

    klines := make([]Kline, df.Nrow())
    for i := 0; i < df.Nrow(); i++ {
        row := df.Slice(i, i+1)
        klines[i] = Kline{
            Symbol:    symbol,
            OpenTime:  parseTime(row.Elem(0).String()),
            Open:      row.Elem(1).Float(),
            High:      row.Elem(2).Float(),
            Low:       row.Elem(3).Float(),
            Close:     row.Elem(4).Float(),
            Volume:    row.Elem(6).Float(),
        }
    }
    return klines, nil
}
```

### 5.3 券商交易接口

#### 富途 OpenAPI (推荐)

```go
package trader

import (
    "context"
    "github.com/futunnopen/futunnopen-go"
)

type FutuTrader struct {
    client       *futunnopen.OpenQuoteAgent
    marketType   market.MarketType
    marketConfig *market.MarketConfig
}

func NewFutuTrader(apiKey, apiSecret, host string, port int, marketType market.MarketType) *FutuTrader {
    return &FutuTrader{
        client:     futunnopen.NewOpenQuoteAgent(apiKey, apiSecret),
        marketType: marketType,
        marketConfig: market.GetMarketConfig(marketType),
    }
}

func (t *FutuTrader) Init(ctx context.Context, config *TraderConfig) error {
    // 连接富途 OpenD
    err := t.client.Connect(ctx, config.Host, config.Port)
    if err != nil {
        return err
    }

    // 解锁交易
    return t.client.UnlockTrade(config.TradePassword)
}

func (t *FutuTrader) PlaceOrder(ctx context.Context, req *UniversalOrderRequest) (*OrderResult, error) {
    // 转换市场类型
    var trdMarket futunnopen.TrdMarket
    switch t.marketType {
    case market.MarketAStock:
        trdMarket = futunnopen.TrdMarket_CN
    case market.MarketHKStock:
        trdMarket = futunnopen.TrdMarket_HK
    }

    // 构建订单
    order := &futunnopen.PlaceOrderRequest{
        Code:       req.Symbol,
        TrdEnv:     futunnopen.TrdEnv_Real,
        TrdMarket:  trdMarket,
        Price:      req.Price,
        Qty:        int64(req.Quantity),
        OrderType:  convertOrderType(req.OrderType),
        Side:       convertSide(req.Side),
    }

    // 下单
    resp, err := t.client.PlaceOrder(ctx, order)
    if err != nil {
        return nil, err
    }

    return &OrderResult{
        OrderID: resp.OrderID,
        Status:  resp.Status,
    }, nil
}
```

#### 老虎证券 API (备选)

```go
package trader

import (
    "context"
    "github.com/tigeropen/sdk-go"
)

type TigerTrader struct {
    client     *tiger.Client
    marketType market.MarketType
}

func NewTigerTrader(apiKey, apiSecret string) *TigerTrader {
    return &TigerTrader{
        client: tiger.NewClient(apiKey, apiSecret),
    }
}

// ... 类似实现
```

---

## 6. 实施路线图

### 6.1 阶段划分

```
┌─────────────────────────────────────────────────────────────────────┐
│                      改造实施路线图                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1: 基础架构改造 (2-3 周)                                     │
│  ├── 定义市场类型枚举和配置                                         │
│  ├── 设计通用交易接口                                               │
│  ├── 设计通用数据提供者接口                                         │
│  └── 实现市场抽象层                                                 │
│                                                                     │
│  Phase 2: A 股支持 (3-4 周)                                         │
│  ├── 实现 A 股数据提供者 (Tushare)                                 │
│  ├── 实现 A 股交易接口 (富途)                                       │
│  ├── 改造决策引擎适配 A 股                                          │
│  └── 实现 A 股回测引擎                                              │
│                                                                     │
│  Phase 3: 港股支持 (2-3 周)                                        │
│  ├── 实现港股数据提供者                                             │
│  ├── 实现港股交易接口                                               │
│  └── 改造决策引擎适配港股                                           │
│                                                                     │
│  Phase 4: 前端改造 (2-3 周)                                         │
│  ├── 修改策略工作室支持市场选择                                     │
│  ├── 更新回测界面                                                   │
│  ├── 修改仪表板显示股票特有数据                                     │
│  └── 添加 A 股/港股特定配置项                                        │
│                                                                     │
│  Phase 5: 测试与优化 (2-3 周)                                       │
│  ├── 单元测试                                                       │
│  ├── 集成测试                                                       │
│  ├── 模拟盘测试                                                     │
│  └── 性能优化                                                       │
│                                                                     │
│  Total: 11-16 周 (约 3-4 个月)                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 详细任务清单

#### Phase 1: 基础架构改造 (Week 1-3)

| 任务 | 负责模块 | 优先级 | 预计工时 |
|------|----------|--------|----------|
| 定义 MarketType 枚举 | market/ | P0 | 4h |
| 实现 MarketConfig | market/ | P0 | 8h |
| 设计 UniversalTrader 接口 | trader/ | P0 | 16h |
| 设计 MarketDataProvider 接口 | provider/ | P0 | 12h |
| 实现市场抽象层 | market/ | P0 | 16h |
| 单元测试 | - | P0 | 8h |

#### Phase 2: A 股支持 (Week 4-7)

| 任务 | 负责模块 | 优先级 | 预计工时 |
|------|----------|--------|----------|
| 实现 TushareProvider | provider/astock/ | P0 | 16h |
| 实现富途 A 股交易接口 | trader/astock/ | P0 | 24h |
| 改造 Decision 引擎 | decision/ | P0 | 16h |
| 实现 A 股回测引擎 | backtest/ | P0 | 20h |
| A 股市场规则验证 | - | P0 | 8h |
| 单元测试 | - | P0 | 12h |

#### Phase 3: 港股支持 (Week 8-10)

| 任务 | 负责模块 | 优先级 | 预计工时 |
|------|----------|--------|----------|
| 实现 AAStocksProvider | provider/hkstock/ | P1 | 12h |
| 实现富途港股交易接口 | trader/hkstock/ | P1 | 16h |
| 改造 Decision 引擎 | decision/ | P1 | 8h |
| 港股回测引擎 | backtest/ | P1 | 12h |
| 单元测试 | - | P1 | 8h |

#### Phase 4: 前端改造 (Week 11-13)

| 任务 | 负责模块 | 优先级 | 预计工时 |
|------|----------|--------|----------|
| 修改策略工作室 | web/src/pages/ | P0 | 16h |
| 更新回测界面 | web/src/pages/ | P0 | 12h |
| 修改仪表板 | web/src/pages/ | P0 | 16h |
| 添加市场选择器 | web/src/components/ | P0 | 8h |
| 显示股票特有数据 | web/src/components/ | P0 | 12h |

#### Phase 5: 测试与优化 (Week 14-16)

| 任务 | 优先级 | 预计工时 |
|------|--------|----------|
| 单元测试覆盖 | P0 | 16h |
| 集成测试 | P0 | 16h |
| 模拟盘测试 | P0 | 24h |
| 性能优化 | P1 | 16h |
| 文档更新 | P1 | 12h |

---

## 7. 风险与挑战

### 7.1 技术风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **券商 API 限制** | 高 | 1. 选择支持 OpenAPI 的券商 (富途、老虎)<br>2. 准备多个备选方案<br>3. 实现模拟交易模式 |
| **数据源稳定性** | 高 | 1. 使用多个数据源<br>2. 实现数据缓存<br>3. 监控数据质量 |
| **T+1 交易限制** | 中 | 1. 在回测引擎中正确实现<br>2. 在 UI 中明确提示<br>3. 文档说明 |
| **API 变更** | 中 | 1. 版本化接口<br>2. 定期更新<br>3. 抽象层隔离变化 |
| **性能问题** | 中 | 1. 数据缓存<br>2. 异步处理<br>3. 连接池 |

### 7.2 业务风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **监管风险** | 高 | 1. 遵守当地法律法规<br>2. 不提供自动交易 (仅辅助决策)<br>3. 免责声明 |
| **资金风险** | 高 | 1. 强烈警告用户风险<br>2. 建议先模拟盘测试<br>3. 风控限制 |
| **券商账户** | 中 | 1. 用户自行开户<br>2. 不托管资金<br>3. 仅提供技术支持 |
| **决策质量** | 中 | 1. 多 AI 辩论<br>2. 回测验证<br>3. 风控规则 |

### 7.3 实施风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **开发周期长** | 中 | 1. 分阶段实施<br>2. 优先核心功能<br>3. 迭代交付 |
| **兼容性问题** | 中 | 1. 保持加密货币功能<br>2. 抽象层隔离<br>3. 充分测试 |
| **用户学习成本** | 低 | 1. 详细文档<br>2. 视频教程<br>3. 示例配置 |

---

## 8. 总结

### 8.1 改造价值

| 价值点 | 说明 |
|--------|------|
| **市场扩展** | 从加密货币扩展到 A 股/港股，用户群扩大 |
| **技术复用** | AI 决策引擎、回测系统等核心能力可复用 |
| **差异化竞争** | AI 驱动的股票交易系统较少，有竞争优势 |
| **学习价值** | 理解不同市场特点，提升系统能力 |

### 8.2 关键成功因素

1. **正确实现市场规则** (T+1、涨跌停等)
2. **稳定的数据源** (Tushare、富途等)
3. **可靠的券商接口** (富途 OpenAPI)
4. **AI 决策引擎适配** (股票特有数据)
5. **充分的测试** (特别是回测准确性)

### 8.3 建议

1. **优先实现 A 股**：国内用户需求大，数据源丰富
2. **使用成熟券商 API**：富途 OpenAPI 是最佳选择
3. **保持加密货币功能**：多市场支持是优势
4. **充分模拟测试**：实盘前必须模拟盘验证
5. **合规优先**：遵守法律法规，提供免责声明

---

**文档结束**

*本方案为 NOFX 项目从加密货币改造为支持 A 股/港股的技术架构方案，具体实施时需根据实际情况调整。*
