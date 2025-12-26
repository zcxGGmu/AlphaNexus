# NOFX Python/Go 混合架构改造方案

**文档版本**: v1.0
**创建时间**: 2025-12-27
**目标**: 将 NOFX 改造为 Python/Go 混合架构，支持 A 股/港股量化交易

---

## 目录

1. [架构选择理由](#1-架构选择理由)
2. [混合架构设计](#2-混合架构设计)
3. [模块分工](#3-模块分工)
4. [通信机制](#4-通信机制)
5. [Python 模块实现](#5-python-模块实现)
6. [Go 模块实现](#6-go-模块实现)
7. [部署架构](#7-部署架构)
8. [实施路线图](#8-实施路线图)

---

## 1. 架构选择理由

### 1.1 为什么选择混合架构？

| 维度 | 纯 Go | 纯 Python | **Python/Go 混合** |
|------|-------|-----------|-------------------|
| **量化库生态** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **性能** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **并发处理** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **AI/ML 支持** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Web 服务** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **开发效率** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **团队熟悉度** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 1.2 量化交易领域的 Python 优势

#### 成熟的量化库

```python
# 数据处理
import pandas as pd
import numpy as np

# 技术指标
import talib
import ta

# 回测框架
import backtrader
import zipline
import quantopian

# 机器学习
import sklearn
import tensorflow
import pytorch

# 数据源
import tushare as ts
import akshare as ak
import yfinance

# 券商 API
import futu as ft
```

#### 丰富的数据源支持

| 数据源 | Python SDK | Go SDK |
|--------|-----------|--------|
| **Tushare** | ✅ 官方 | ⚠️ 第三方 |
| **AkShare** | ✅ 官方 | ❌ 无 |
| **富途 OpenAPI** | ✅ 官方 | ✅ 官方 |
| **东方财富** | ✅ 社区 | ❌ 无 |
| **JoinQuant** | ✅ 官方 | ❌ 无 |
| **米筐 RQAlpha** | ✅ 官方 | ❌ 无 |

### 1.3 Go 在系统层的优势

- **高性能 Web 服务**：Gin/Echo 框架
- **并发处理**：Goroutine 处理多连接
- **内存效率**：相比 Python 节省资源
- **部署简单**：单一二进制文件
- **稳定性好**：类型安全，适合生产环境

---

## 2. 混合架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                   NOFX Python/Go 混合架构                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    前端层 (React + TS)                       │   │
│  │  - 策略工作室  - 回测实验室  - 辩论竞技场  - 实时仪表板        │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │ REST API / WebSocket                   │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  Go 后端服务层                               │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │   │
│  │  │ HTTP API   │  │ WebSocket  │  │ 认证授权    │             │   │
│  │  │ (Gin)      │  │ Service    │  │ (JWT)       │             │   │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘             │   │
│  │        │                │                │                    │   │
│  │  ┌─────┴────────────────┴────────────────┴──────┐           │   │
│  │  │            服务编排层 (Service Layer)          │           │   │
│  │  │                                               │           │   │
│  │  │  - 交易服务  - 数据服务  - 回测服务            │           │   │
│  │  └─────┬───────────────────────┬─────────────────┘           │   │
│  └────────┼───────────────────────┼─────────────────────────────┘   │
│           │ gRPC / HTTP            │                                │
│           ▼                        ▼                                │
│  ┌────────────────────┐  ┌─────────────────────────────────────┐  │
│  │  Python 引擎层     │  │  Go 数据层                          │  │
│  │                   │  │                                     │  │
│  │  ┌──────────────┐ │  │  ┌──────────────┐  ┌────────────┐ │  │
│  │  │ AI 决策引擎  │ │  │  │ PostgreSQL   │  │ Redis      │ │  │
│  │  │ (LLM 客户端) │ │  │  │ (时序数据)   │  │ (缓存)     │ │  │
│  │  └──────────────┘ │  │  └──────────────┘  └────────────┘ │  │
│  │                   │  │                                     │  │
│  │  ┌──────────────┐ │  │  ┌──────────────┐  ┌────────────┐ │  │
│  │  │ 回测引擎     │ │  │  │ 消息队列     │  │ 时序DB     │ │  │
│  │  │ (Backtrader) │ │  │  │ (NATS)       │  │ (InfluxDB) │ │  │
│  │  └──────────────┘ │  │  └──────────────┘  └────────────┘ │  │
│  │                   │  │                                     │  │
│  │  ┌──────────────┐ │  │  ┌──────────────┐  ┌────────────┐ │  │
│  │  │ 技术指标     │ │  │  │ 交易执行层   │  │ 订单管理   │ │  │
│  │  │ (TA-Lib)     │ │  │  │ (Go 实现)    │  │ (Go 实现)  │ │  │
│  │  └──────────────┘ │  │  └──────────────┘  └────────────┘ │  │
│  └───────────────────┘  └─────────────────────────────────────┘  │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    数据源层                                  │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ Tushare  │  │ AkShare  │  │ 富途 API │  │ 券商 API │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 职责划分

#### Go 层负责

| 模块 | 职责 | 理由 |
|------|------|------|
| **HTTP API** | RESTful API 服务 | 高性能、高并发 |
| **WebSocket** | 实时数据推送 | Goroutine 并发优势 |
| **交易执行** | 订单下单、撤单、状态查询 | 稳定性、实时性要求高 |
| **账户管理** | 资金、持仓管理 | 数据一致性 |
| **认证授权** | JWT、权限控制 | 安全性 |
| **数据存储** | PostgreSQL、Redis | 高效的数据操作 |
| **消息队列** | NATS、任务调度 | 并发控制 |

#### Python 层负责

| 模块 | 职责 | 理由 |
|------|------|------|
| **数据获取** | 市场数据、基本面数据 | Python 库丰富 |
| **技术指标** | TA-Lib、自定义指标 | pandas、numpy |
| **AI 决策** | LLM 调用、策略分析 | AI 库生态完善 |
| **回测引擎** | Backtrader、Zipline | 成熟回测框架 |
| **机器学习** | 模型训练、预测 | sklearn、pytorch |
| **数据分析** | 数据清洗、特征工程 | pandas 强大 |

### 2.3 通信机制

```
┌──────────────────────────────────────────────────────────────┐
│                   通信机制                                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Go ────── HTTP/REST ─────→ Python                          │
│  ├── 请求回测                                               │
│  ├── 获取 AI 决策                                            │
│  └── 计算技术指标                                            │
│                                                              │
│  Go ────── gRPC ───────→ Python                             │
│  ├── 高频数据调用                                           │
│  ├── 流式数据传输                                           │
│  └── 双向通信                                               │
│                                                              │
│  Go ────── Redis Pub/Sub ──→ Python                         │
│  ├── 实时行情推送                                           │
│  └── 事件通知                                               │
│                                                              │
│  Go ────── Message Queue (NATS) ─→ Python                   │
│  ├── 异步任务                                               │
│  ├── 任务调度                                               │
│  └── 负载均衡                                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 模块分工

### 3.1 目录结构

```
nofx-hybrid/
├── backend/                    # Go 后端
│   ├── api/                   # HTTP API
│   │   ├── server.go
│   │   ├── handlers/          # 请求处理器
│   │   └── middleware/        # 中间件
│   ├── services/              # 业务服务
│   │   ├── trading/           # 交易服务
│   │   ├── account/           # 账户服务
│   │   └── market/            # 市场数据服务
│   ├── models/                # 数据模型
│   ├── db/                    # 数据库操作
│   ├── python/                # Python 客户端
│   │   ├── client.go          # gRPC 客户端
│   │   └── proto/             # Protobuf 定义
│   ├── config/                # 配置
│   └── main.go
│
├── engine/                     # Python 引擎
│   ├── services/              # 服务实现
│   │   ├── decision_service.py
│   │   ├── backtest_service.py
│   │   └── indicator_service.py
│   ├── ai/                    # AI 模块
│   │   ├── llm/               # LLM 客户端
│   │   └── prompts/           # 提示词模板
│   ├── data/                  # 数据模块
│   │   ├── providers/         # 数据提供者
│   │   │   ├── tushare.py
│   │   │   ├── akshare.py
│   │   │   └── futu.py
│   │   └── processors/        # 数据处理
│   ├── indicators/            # 技术指标
│   ├── backtest/              # 回测引擎
│   │   ├── backtrader_wrapper.py
│   │   └── strategies/        # 策略定义
│   ├── models/                # ML 模型
│   ├── utils/                 # 工具函数
│   ├── grpc/                  # gRPC 服务
│   │   ├── server.py
│   │   └── protos/            # Protobuf 定义
│   └── main.py
│
├── web/                        # 前端 (React)
│   └── ... (同原 NOFX)
│
├── proto/                      # Protobuf 定义
│   ├── decision.proto
│   ├── backtest.proto
│   └── market_data.proto
│
├── scripts/                    # 脚本
│   ├── start-go.sh
│   ├── start-python.sh
│   └── build-proto.sh
│
├── docker/                     # Docker 配置
│   ├── Dockerfile.go
│   ├── Dockerfile.python
│   └── docker-compose.yml
│
└── requirements.txt            # Python 依赖
```

---

## 4. 通信机制

### 4.1 gRPC 接口定义

#### Protobuf 定义

```protobuf
// proto/services.proto
syntax = "proto3";

package nofx;

// ========== 决策服务 ==========
service DecisionService {
    rpc GetDecision(DecisionRequest) returns (DecisionResponse);
    rpc StreamDecision(stream DecisionRequest) returns (stream DecisionResponse);
}

message DecisionRequest {
    string symbol = 1;
    string market_type = 2;  // CRYPTO, ASTOCK, HKSTOCK
    MarketData market_data = 3;
    StrategyConfig strategy = 4;
    string language = 5;      // zh-CN, en-US
}

message DecisionResponse {
    string action = 1;        // BUY, SELL, HOLD
    float confidence = 2;
    string reasoning = 3;
    string chain_of_thought = 4;
    int64 timestamp = 5;
}

// ========== 回测服务 ==========
service BacktestService {
    rpc RunBacktest(BacktestRequest) returns (stream BacktestProgress);
    rpc GetBacktestResult(BacktestId) returns (BacktestResult);
}

message BacktestRequest {
    string symbol = 1;
    string market_type = 2;
    int64 start_time = 3;
    int64 end_time = 4;
    double initial_capital = 5;
    StrategyConfig strategy = 6;
}

message BacktestProgress {
    string backtest_id = 1;
    int32 progress = 2;       // 0-100
    string current_step = 3;
    EquityPoint equity_point = 4;
}

message BacktestResult {
    string backtest_id = 1;
    repeated Trade trades = 2;
    BacktestMetrics metrics = 3;
    repeated EquityPoint equity_curve = 4;
}

// ========== 技术指标服务 ==========
service IndicatorService {
    rpc CalculateIndicators(IndicatorRequest) returns (IndicatorResponse);
}

message IndicatorRequest {
    string symbol = 1;
    repeated Kline klines = 2;
    repeated string indicators = 3;  // EMA, MACD, RSI, etc.
}

message IndicatorResponse {
    map<string, double> values = 1;
}

// ========== 数据类型 ==========
message MarketData {
    repeated Kline klines = 1;
    StockFundamentals fundamentals = 2;
}

message Kline {
    int64 timestamp = 1;
    double open = 2;
    double high = 3;
    double low = 4;
    double close = 5;
    double volume = 6;
}

message StockFundamentals {
    double turnover_rate = 1;
    double market_cap = 2;
    double pe_ratio = 3;
    double pb_ratio = 4;
    double dividend_yield = 5;
}

message StrategyConfig {
    map<string, string> parameters = 1;
    repeated string indicators = 2;
    RiskControl risk_control = 3;
}

message RiskControl {
    double max_position_ratio = 1;
    double stop_loss = 2;
    double take_profit = 3;
}

message BacktestMetrics {
    double total_return = 1;
    double max_drawdown = 2;
    double sharpe_ratio = 3;
    double win_rate = 4;
    int32 total_trades = 5;
}

message Trade {
    string symbol = 1;
    string action = 2;
    double price = 3;
    double quantity = 4;
    int64 entry_time = 5;
    int64 exit_time = 6;
    double pnl = 7;
}

message EquityPoint {
    int64 timestamp = 1;
    double equity = 2;
}
```

### 4.2 Go gRPC 客户端

```go
// backend/python/client.go
package python

import (
    "context"
    "fmt"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/keepalive"
    pb "nofx/proto"
)

type PythonClient struct {
    conn   *grpc.ClientConn
    client struct {
        Decision   pb.DecisionServiceClient
        Backtest   pb.BacktestServiceClient
        Indicator  pb.IndicatorServiceClient
    }
}

func NewPythonClient(addr string) (*PythonClient, error) {
    // 配置 keep-alive
    ka := keepalive.ClientParameters{
        Time:                10 * time.Second,
        Timeout:             time.Second,
        PermitWithoutStream: true,
    }

    conn, err := grpc.Dial(addr,
        grpc.WithInsecure(),
        grpc.WithKeepaliveParams(ka),
        grpc.WithDefaultCallOptions(
            grpc.MaxCallRecvMsgSize(10*1024*1024), // 10MB
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to connect to Python: %w", err)
    }

    c := &PythonClient{conn: conn}
    c.client.Decision = pb.NewDecisionServiceClient(conn)
    c.client.Backtest = pb.NewBacktestServiceClient(conn)
    c.client.Indicator = pb.NewIndicatorServiceClient(conn)

    return c, nil
}

func (c *PythonClient) GetDecision(ctx context.Context, req *pb.DecisionRequest) (*pb.DecisionResponse, error) {
    return c.client.Decision.GetDecision(ctx, req)
}

func (c *PythonClient) RunBacktest(ctx context.Context, req *pb.BacktestRequest) (<-chan *pb.BacktestProgress, error) {
    stream, err := c.client.Backtest.RunBacktest(ctx, req)
    if err != nil {
        return nil, err
    }

    ch := make(chan *pb.BacktestProgress, 10)
    go func() {
        defer close(ch)
        for {
            resp, err := stream.Recv()
            if err != nil {
                return
            }
            ch <- resp
        }
    }()

    return ch, nil
}

func (c *PythonClient) Close() error {
    return c.conn.Close()
}
```

### 4.3 Python gRPC 服务

```python
# engine/grpc/server.py
import grpc
from concurrent import futures
import logging

from engine.services.decision_service import DecisionService
from engine.services.backtest_service import BacktestService
from engine.services.indicator_service import IndicatorService

# 导入生成的 protobuf
import proto.services_pb2 as pb
import proto.services_pb2_grpc as pb_grpc

logger = logging.getLogger(__name__)

class PythonGRPCServer:
    def __init__(self, port: int = 50051):
        self.port = port
        self.server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))

        # 注册服务
        decision_service = DecisionService()
        backtest_service = BacktestService()
        indicator_service = IndicatorService()

        pb_grpc.add_DecisionServiceServicer_to_server(
            decision_service, self.server
        )
        pb_grpc.add_BacktestServiceServicer_to_server(
            backtest_service, self.server
        )
        pb_grpc.add_IndicatorServiceServicer_to_server(
            indicator_service, self.server
        )

    def start(self):
        self.server.add_insecure_port(f'[::]:{self.port}')
        logger.info(f"Python gRPC server started on port {self.port}")
        self.server.start()
        self.server.wait_for_termination()

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    server = PythonGRPCServer()
    server.start()
```

---

## 5. Python 模块实现

### 5.1 项目结构

```
engine/
├── __init__.py
├── main.py                    # gRPC 服务入口
│
├── services/                  # 业务服务
│   ├── __init__.py
│   ├── decision_service.py    # 决策服务
│   ├── backtest_service.py    # 回测服务
│   └── indicator_service.py   # 指标服务
│
├── ai/                        # AI 模块
│   ├── __init__.py
│   ├── llm/                   # LLM 客户端
│   │   ├── __init__.py
│   │   ├── base.py            # 基类
│   │   ├── deepseek.py
│   │   ├── openai.py
│   │   └── qwen.py
│   └── prompts/               # 提示词模板
│       ├── __init__.py
│       ├── stock_prompts.py   # 股票提示词
│       └── crypto_prompts.py  # 加密货币提示词
│
├── data/                      # 数据模块
│   ├── __init__.py
│   ├── providers/             # 数据提供者
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── tushare.py         # Tushare
│   │   ├── akshare.py         # AkShare
│   │   ├── futu.py            # 富途
│   │   └── yahoo.py           # Yahoo Finance
│   └── processors/            # 数据处理
│       ├── __init__.py
│       ├── kline.py
│       └── fundamentals.py
│
├── indicators/                # 技术指标
│   ├── __init__.py
│   ├── ta_lib_wrapper.py      # TA-Lib 包装
│   └── custom.py              # 自定义指标
│
├── backtest/                  # 回测引擎
│   ├── __init__.py
│   ├── backtrader_wrapper.py  # Backtrader 包装
│   ├── strategies/            # 策略
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── ai_strategy.py     # AI 策略
│   │   └── signal_strategy.py # 信号策略
│   └── analyzers/             # 分析器
│       ├── __init__.py
│       ├── sharpe.py
│       └── drawdown.py
│
├── models/                    # ML 模型
│   ├── __init__.py
│   └── price_predictor.py
│
└── utils/                     # 工具
    ├── __init__.py
    ├── logger.py
    └── config.py
```

### 5.2 决策服务实现

```python
# engine/services/decision_service.py
import logging
from typing import Dict, Any

import grpc
from proto import services_pb2 as pb
from proto import services_pb2_grpc as pb_grpc

from engine.ai.llm import LLMFactory
from engine.ai.prompts import PromptBuilder
from engine.indicators import IndicatorCalculator
from engine.data.providers import DataProviderFactory

logger = logging.getLogger(__name__)

class DecisionService(pb_grpc.DecisionServiceServicer):
    """AI 决策服务"""

    def __init__(self):
        self.llm_factory = LLMFactory()
        self.prompt_builder = PromptBuilder()
        self.indicator_calc = IndicatorCalculator()
        self.data_factory = DataProviderFactory()

    def GetDecision(self, request, context):
        """获取 AI 决策"""
        try:
            logger.info(f"GetDecision request: {request.symbol} - {request.market_type}")

            # 1. 获取市场数据
            provider = self.data_factory.get_provider(request.market_type)
            market_data = provider.get_realtime_data(request.symbol)

            # 2. 计算技术指标
            indicators = self.indicator_calc.calculate(
                market_data['klines'],
                request.strategy.indicators
            )

            # 3. 构建 AI Prompt
            prompt = self.prompt_builder.build(
                market_type=request.market_type,
                language=request.language,
                klines=market_data['klines'],
                indicators=indicators,
                fundamentals=market_data.get('fundamentals'),
                strategy_config=request.strategy
            )

            # 4. 调用 LLM
            llm_client = self.llm_factory.get_client(request.strategy.parameters.get('ai_model', 'deepseek'))
            response = llm_client.complete(prompt)

            # 5. 解析决策
            decision = self._parse_decision(response)

            return pb.DecisionResponse(
                action=decision['action'],
                confidence=decision['confidence'],
                reasoning=decision['reasoning'],
                chain_of_thought=response,
                timestamp=int(time.time() * 1000)
            )

        except Exception as e:
            logger.error(f"Error in GetDecision: {e}", exc_info=True)
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))
            return pb.DecisionResponse()

    def _parse_decision(self, response: str) -> Dict[str, Any]:
        """解析 AI 响应"""
        # 解析 JSON 格式的决策
        import json
        try:
            decision = json.loads(response)
            return {
                'action': decision.get('action', 'HOLD'),
                'confidence': decision.get('confidence', 0.5),
                'reasoning': decision.get('reasoning', '')
            }
        except:
            # 如果无法解析 JSON，使用规则提取
            response_upper = response.upper()
            if 'BUY' in response_upper:
                action = 'BUY'
            elif 'SELL' in response_upper:
                action = 'SELL'
            else:
                action = 'HOLD'
            return {
                'action': action,
                'confidence': 0.5,
                'reasoning': response[:200]
            }
```

### 5.3 回测服务实现

```python
# engine/services/backtest_service.py
import logging
import time
from typing import Iterator

import grpc
from proto import services_pb2 as pb
from proto import services_pb2_grpc as pb_grpc

from engine.backtest.backtrader_wrapper import BacktestEngine
from engine.data.providers import DataProviderFactory

logger = logging.getLogger(__name__)

class BacktestService(pb_grpc.BacktestServiceServicer):
    """回测服务"""

    def __init__(self):
        self.data_factory = DataProviderFactory()
        self.running_backtests = {}

    def RunBacktest(self, request, context):
        """运行回测"""
        backtest_id = f"{request.symbol}_{int(time.time())}"
        logger.info(f"Starting backtest: {backtest_id}")

        try:
            # 获取历史数据
            provider = self.data_factory.get_provider(request.market_type)
            klines = provider.get_historical_klines(
                request.symbol,
                request.start_time,
                request.end_time
            )

            total_bars = len(klines)
            engine = BacktestEngine(
                initial_capital=request.initial_capital,
                strategy_config=request.strategy
            )

            # 流式发送进度
            for i, kline in enumerate(klines):
                result = engine.next(kline)

                progress = int((i + 1) / total_bars * 100)

                yield pb.BacktestProgress(
                    backtest_id=backtest_id,
                    progress=progress,
                    current_step=f"Processing {i+1}/{total_bars}",
                    equity_point=pb.EquityPoint(
                        timestamp=kline['timestamp'],
                        equity=result['equity']
                    )
                )

            # 发送最终结果
            final_result = engine.get_result()

            yield pb.BacktestProgress(
                backtest_id=backtest_id,
                progress=100,
                current_step="Completed",
                equity_point=pb.EquityPoint(
                    timestamp=klines[-1]['timestamp'],
                    equity=final_result['final_equity']
                )
            )

            # 保存结果
            self.running_backtests[backtest_id] = final_result

        except Exception as e:
            logger.error(f"Error in RunBacktest: {e}", exc_info=True)
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))

    def GetBacktestResult(self, request, context):
        """获取回测结果"""
        backtest_id = request.backtest_id

        if backtest_id not in self.running_backtests:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f"Backtest {backtest_id} not found")
            return pb.BacktestResult()

        result = self.running_backtests[backtest_id]

        # 转换交易记录
        trades = []
        for trade in result['trades']:
            trades.append(pb.Trade(
                symbol=trade['symbol'],
                action=trade['action'],
                price=trade['price'],
                quantity=trade['quantity'],
                entry_time=trade['entry_time'],
                exit_time=trade['exit_time'],
                pnl=trade['pnl']
            ))

        # 转换权益曲线
        equity_curve = []
        for point in result['equity_curve']:
            equity_curve.append(pb.EquityPoint(
                timestamp=point['timestamp'],
                equity=point['equity']
            ))

        return pb.BacktestResult(
            backtest_id=backtest_id,
            trades=trades,
            metrics=pb.BacktestMetrics(
                total_return=result['metrics']['total_return'],
                max_drawdown=result['metrics']['max_drawdown'],
                sharpe_ratio=result['metrics']['sharpe_ratio'],
                win_rate=result['metrics']['win_rate'],
                total_trades=result['metrics']['total_trades']
            ),
            equity_curve=equity_curve
        )
```

### 5.4 数据提供者实现

```python
# engine/data/providers/tushare.py
import logging
import pandas as pd
import tushare as ts

logger = logging.getLogger(__name__)

class TushareProvider:
    """Tushare 数据提供者"""

    def __init__(self, token: str):
        self.token = token
        ts.set_token(token)
        self.pro = ts.pro_api()

    def get_klines(self, symbol: str, start_date: str, end_date: str) -> pd.DataFrame:
        """获取 K 线数据"""
        df = self.pro.daily(
            ts_code=symbol,
            start_date=start_date,
            end_date=end_date
        )
        return df

    def get_stock_fundamentals(self, symbol: str) -> dict:
        """获取股票基本面数据"""
        # 日线基本面
        df_daily = self.pro.daily_basic(ts_code=symbol)

        # 公司信息
        df_company = self.pro.stock_company(ts_code=symbol)

        return {
            'turnover_rate': df_daily['turnover_rate'].iloc[0],
            'market_cap': df_daily['total_mv'].iloc[0] * 10000,  # 万元 -> 元
            'pe_ratio': df_daily['pe'].iloc[0],
            'pb_ratio': df_daily['pb'].iloc[0],
            'dividend_yield': df_daily['dv_ratio'].iloc[0],
        }

    def get_stock_list(self, market: str = 'SSE') -> pd.DataFrame:
        """获取股票列表"""
        return self.pro.stock_basic(
            exchange='',
            list_status='L'  # 仅上市股票
        )

    def get_index_constituents(self, index_code: str) -> list:
        """获取指数成分股"""
        df = self.pro.index_member(ts_code=index_code)
        return df['con_code'].tolist()
```

```python
# engine/data/providers/akshare.py
import logging
import akshare as ak

logger = logging.getLogger(__name__)

class AkShareProvider:
    """AkShare 数据提供者 (免费)"""

    def get_klines(self, symbol: str, start_date: str, end_date: str) -> pd.DataFrame:
        """获取 K 线数据"""
        df = ak.stock_zh_a_hist(
            symbol=symbol,
            period="daily",
            start_date=start_date.replace('-', ''),
            end_date=end_date.replace('-', ''),
            adjust="qfq"  # 前复权
        )
        return df

    def get_realtime_price(self, symbol: str) -> float:
        """获取实时价格"""
        df = ak.stock_zh_a_spot_em()
        row = df[df['代码'] == symbol]
        return row['最新价'].iloc[0]

    def get_stock_fundamentals(self, symbol: str) -> dict:
        """获取股票基本面"""
        df = ak.stock_individual_info_em(symbol=symbol)
        return {
            'turnover_rate': df['换手率'].iloc[0],
            'market_cap': df['总市值'].iloc[0],
            'pe_ratio': df['市盈率-动态'].iloc[0],
            'pb_ratio': df['市净率'].iloc[0],
        }
```

### 5.5 Backtrader 回测引擎

```python
# engine/backtest/backtrader_wrapper.py
import backtrader as bt
import logging
from typing import Dict, Any

logger = logging.getLogger(__name__)

class AIStrategy(bt.Strategy):
    """AI 驱动策略"""

    params = (
        ('ai_client', None),
        ('prompt_builder', None),
    )

    def __init__(self):
        self.data_close = self.datas[0].close
        self.data_volume = self.datas[0].volume

    def next(self):
        """每个 K 线调用"""
        # 只在交易时间执行
        if not self.is_trading_time():
            return

        # 获取当前数据
        current_data = self.get_current_data()

        # 构建 AI Prompt
        prompt = self.params.prompt_builder.build(
            market_type='ASTOCK',
            klines=current_data['klines'],
            indicators=current_data['indicators']
        )

        # 调用 AI
        decision = self.params.ai_client.decide(prompt)

        # 执行决策
        if decision['action'] == 'BUY' and not self.position:
            # 买入
            size = self.broker.getcash() / self.data_close[0] * 0.95
            self.buy(size=size)
        elif decision['action'] == 'SELL' and self.position:
            # 卖出
            self.close()

class BacktestEngine:
    """回测引擎"""

    def __init__(self, initial_capital: float, strategy_config: Dict[str, Any]):
        self.cerebro = bt.Cerebro()
        self.cerebro.broker.setcash(initial_capital)
        self.cerebro.broker.setcommission(commission=0.00025)  # 万分之2.5

        # 添加策略
        self.cerebro.addstrategy(
            AIStrategy,
            ai_client=strategy_config['ai_client'],
            prompt_builder=strategy_config['prompt_builder']
        )

        # 添加分析器
        self.cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
        self.cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')

    def add_data(self, data):
        """添加数据"""
        self.cerebro.adddata(data)

    def run(self):
        """运行回测"""
        results = self.cerebro.run()
        strategy = results[0]

        # 获取分析结果
        sharpe = strategy.analyzers.sharpe.get_analysis()
        drawdown = strategy.analyzers.drawdown.get_analysis()

        return {
            'final_equity': self.cerebro.broker.getvalue(),
            'metrics': {
                'sharpe_ratio': sharpe.get('sharperatio', 0),
                'max_drawdown': drawdown.get('max', {}).get('drawdown', 0),
            }
        }
```

### 5.6 AI 客户端实现

```python
# engine/ai/llm/base.py
from abc import ABC, abstractmethod
import logging

logger = logging.getLogger(__name__)

class BaseLLMClient(ABC):
    """LLM 客户端基类"""

    def __init__(self, api_key: str, base_url: str = None):
        self.api_key = api_key
        self.base_url = base_url

    @abstractmethod
    def complete(self, prompt: str, **kwargs) -> str:
        """完成聊天"""
        pass

    @abstractmethod
    def stream_complete(self, prompt: str, **kwargs):
        """流式完成"""
        pass
```

```python
# engine/ai/llm/deepseek.py
import openai
from .base import BaseLLMClient

class DeepSeekClient(BaseLLMClient):
    """DeepSeek 客户端"""

    def __init__(self, api_key: str):
        super().__init__(api_key)
        self.client = openai.OpenAI(
            api_key=api_key,
            base_url="https://api.deepseek.com"
        )

    def complete(self, prompt: str, **kwargs) -> str:
        response = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[
                {"role": "system", "content": "你是一个专业的量化交易分析助手。"},
                {"role": "user", "content": prompt}
            ],
            temperature=kwargs.get('temperature', 0.7),
            max_tokens=kwargs.get('max_tokens', 2000)
        )
        return response.choices[0].message.content

    def stream_complete(self, prompt: str, **kwargs):
        response = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            stream=True
        )
        for chunk in response:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
```

### 5.7 依赖文件

```txt
# requirements.txt

# 数据处理
pandas>=2.0.0
numpy>=1.24.0

# 数据源
tushare>=1.2.90
akshare>=1.11.60
efx>=0.3.0  # 富途 Python SDK

# 技术指标
TA-Lib>=0.4.28
ta>=0.11.0

# 回测框架
backtrader>=1.9.78.123

# AI/ML
openai>=1.6.0
anthropic>=0.18.0
zhipuai>=0.1.0  # 智谱 AI (Qwen)

# gRPC
grpcio>=1.60.0
grpcio-tools>=1.60.0
protobuf>=4.25.0

# 工具
python-dotenv>=1.0.0
pydantic>=2.5.0
loguru>=0.7.0

# 机器学习 (可选)
scikit-learn>=1.3.0
torch>=2.1.0
```

---

## 6. Go 模块实现

### 6.1 项目结构

```
backend/
├── main.go
├── go.mod
├── go.sum
│
├── api/                       # API 层
│   ├── server.go
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── cors.go
│   │   └── logger.go
│   └── handlers/
│       ├── trading.go         # 交易接口
│       ├── account.go         # 账户接口
│       ├── market.go          # 市场接口
│       ├── backtest.go        # 回测接口
│       └── strategy.go        # 策略接口
│
├── services/                  # 业务服务
│   ├── trading/
│   │   ├── service.go         # 交易服务
│   │   ├── executor.go        # 订单执行器
│   │   ├── position.go        # 持仓管理
│   │   └── risk.go            # 风控
│   ├── account/
│   │   └── service.go         # 账户服务
│   └── market/
│       └── service.go         # 市场数据服务
│
├── python/                    # Python 客户端
│   ├── client.go              # gRPC 客户端
│   └── proto/                 # 生成的 protobuf Go 代码
│
├── models/                    # 数据模型
│   ├── trading.go
│   ├── account.go
│   └── market.go
│
├── db/                        # 数据库
│   ├── postgres.go
│   ├── redis.go
│   └── migrations/
│
├── config/                    # 配置
│   └── config.go
│
├── trader/                    # 交易执行
│   ├── interface.go
│   ├── astock/
│   │   └── futu.go            # 富途 A 股
│   └── hkstock/
│       └── futu.go            # 富途港股
│
└── utils/                     # 工具
    ├── logger.go
    └── errors.go
```

### 6.2 HTTP API 实现

```go
// backend/api/server.go
package api

import (
    "context"
    "fmt"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"

    "nofx/python"
    "nofx/services/trading"
    "nofx/config"
)

type Server struct {
    router     *gin.Engine
    python     *python.PythonClient
    tradingSvc *trading.Service
    config     *config.Config
}

func NewServer(cfg *config.Config) (*Server, error) {
    // 初始化 Python 客户端
    pythonClient, err := python.NewPythonClient(cfg.PythonAddr)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to Python: %w", err)
    }

    // 初始化交易服务
    tradingSvc := trading.NewService(pythonClient)

    s := &Server{
        router:     gin.Default(),
        python:     pythonClient,
        tradingSvc: tradingSvc,
        config:     cfg,
    }

    s.setupRoutes()

    return s, nil
}

func (s *Server) setupRoutes() {
    api := s.router.Group("/api")
    {
        // 交易接口
        api.POST("/traders/:id/start", s.startTrader)
        api.POST("/traders/:id/stop", s.stopTrader)
        api.GET("/traders/:id/positions", s.getPositions)
        api.POST("/orders", s.placeOrder)

        // 回测接口
        api.POST("/backtest", s.runBacktest)
        api.GET("/backtest/:id", s.getBacktestResult)

        // AI 决策接口
        api.POST("/decision", s.getDecision)

        // WebSocket
        api.GET("/ws", s.handleWebSocket)
    }
}

func (s *Server) Run(addr string) error {
    return s.router.Run(addr)
}

// Handler 示例
func (s *Server) getDecision(c *gin.Context) {
    var req DecisionRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 调用 Python 服务
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    response, err := s.python.GetDecision(ctx, &pb.DecisionRequest{
        Symbol:     req.Symbol,
        MarketType: req.MarketType,
        // ...
    })
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "action":    response.Action,
        "confidence": response.Confidence,
        "reasoning": response.Reasoning,
    })
}

func (s *Server) runBacktest(c *gin.Context) {
    var req BacktestRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 调用 Python 回测服务 (流式)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
    defer cancel()

    progressCh, err := s.python.RunBacktest(ctx, &pb.BacktestRequest{
        Symbol:         req.Symbol,
        MarketType:     req.MarketType,
        StartTime:      req.StartTime,
        EndTime:        req.EndTime,
        InitialCapital: req.InitialCapital,
    })
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    // 流式发送进度
    c.SSEvent("message", "")
    for progress := range progressCh {
        c.SSEvent("progress", gin.H{
            "backtest_id": progress.BacktestId,
            "progress":    progress.Progress,
            "equity":      progress.EquityPoint,
        })
        c.Writer.Flush()
    }
}
```

### 6.3 交易服务实现

```go
// backend/services/trading/service.go
package trading

import (
    "context"
    "fmt"
    "sync"

    "google.golang.org/protobuf/types/known/timestamppb"

    "nofx/python"
    "nofx/trader"
    pb "nofx/proto"
)

type Service struct {
    python      *python.PythonClient
    traders     map[string]*TraderInstance
    tradersLock sync.RWMutex
}

type TraderInstance struct {
    ID       string
    Symbol   string
    Market   string
    Trader   trader.UniversalTrader
    Strategy *StrategyConfig
    StopCh   chan struct{}
}

func NewService(python *python.PythonClient) *Service {
    return &Service{
        python:  python,
        traders: make(map[string]*TraderInstance),
    }
}

func (s *Service) StartTrader(ctx context.Context, config *StartTraderConfig) error {
    s.tradersLock.Lock()
    defer s.tradersLock.Unlock()

    // 创建交易者实例
    t := &TraderInstance{
        ID:       config.ID,
        Symbol:   config.Symbol,
        Market:   config.Market,
        StopCh:   make(chan struct{}),
    }

    // 初始化交易接口
    switch config.Market {
    case "ASTOCK":
        t.Trader = trader.NewFutuAStock(config.BrokerConfig)
    case "HKSTOCK":
        t.Trader = trader.NewFutuHKStock(config.BrokerConfig)
    default:
        return fmt.Errorf("unsupported market: %s", config.Market)
    }

    if err := t.Trader.Init(ctx, config.BrokerConfig); err != nil {
        return err
    }

    // 启动交易循环
    go s.runTradingLoop(t)

    s.traders[t.ID] = t
    return nil
}

func (s *Service) runTradingLoop(t *TraderInstance) {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // 检查交易时间
            if !t.Trader.IsTradingTime() {
                continue
            }

            // 获取 AI 决策
            decision, err := s.getAIDecision(t)
            if err != nil {
                log.Printf("Error getting AI decision: %v", err)
                continue
            }

            // 执行决策
            s.executeDecision(t, decision)

        case <-t.StopCh:
            return
        }
    }
}

func (s *Service) getAIDecision(t *TraderInstance) (*pb.DecisionResponse, error) {
    // 获取市场数据
    marketData, err := t.Trader.GetMarketData(context.Background(), t.Symbol)
    if err != nil {
        return nil, err
    }

    // 调用 Python 决策服务
    return s.python.GetDecision(context.Background(), &pb.DecisionRequest{
        Symbol:     t.Symbol,
        MarketType: t.Market,
        MarketData: marketData,
        Strategy:   t.Strategy,
    })
}

func (s *Service) executeDecision(t *TraderInstance, decision *pb.DecisionResponse) error {
    ctx := context.Background()

    switch decision.Action {
    case "BUY":
        // 检查是否已持仓
        positions, _ := t.Trader.GetPositions(ctx)
        if len(positions) > 0 {
            return nil // 已持仓
        }

        // 计算买入数量
        account, _ := t.Trader.GetAccountInfo(ctx)
        price, _ := t.Trader.GetMarketPrice(ctx, t.Symbol)
        quantity := int64((account.AvailableCash * 0.95) / price)

        // 确保是 100 的整数倍 (A 股)
        if t.Market == "ASTOCK" {
            quantity = (quantity / 100) * 100
        }

        // 下单
        _, err := t.Trader.PlaceOrder(ctx, &trader.UniversalOrderRequest{
            Symbol:   t.Symbol,
            Side:     trader.SideBuy,
            OrderType: trader.OrderMarket,
            Quantity: float64(quantity),
        })
        return err

    case "SELL":
        // 平仓
        return t.Trader.ClosePosition(ctx, t.Symbol, 0)

    case "HOLD":
        return nil
    }

    return nil
}
```

---

## 7. 部署架构

### 7.1 Docker Compose 配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Go 后端
  backend:
    build:
      context: .
      dockerfile: docker/Dockerfile.go
    ports:
      - "8080:8080"
    environment:
      - PYTHON_ADDR=python:50051
      - DATABASE_URL=postgresql://user:pass@postgres:5432/nofx
      - REDIS_ADDR=redis:6379
    depends_on:
      - python
      - postgres
      - redis
    restart: unless-stopped

  # Python 引擎
  python:
    build:
      context: .
      dockerfile: docker/Dockerfile.python
    ports:
      - "50051:50051"  # gRPC
    environment:
      - TUSHARE_TOKEN=${TUSHARE_TOKEN}
      - FUTU_HOST=${FUTU_HOST}
      - FUTU_PORT=${FUTU_PORT}
    volumes:
      - ./engine:/app/engine
    restart: unless-stopped

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=nofx
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  # Redis
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  # 前端
  frontend:
    build:
      context: ./web
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 7.2 Go Dockerfile

```dockerfile
# docker/Dockerfile.go
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 安装依赖
COPY go.mod go.sum ./
RUN go mod download

# 复制源码
COPY . .

# 构建
RUN CGO_ENABLED=0 GOOS=linux go build -o nofx-backend ./backend

# 运行
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/nofx-backend .
COPY --from=builder /app/config ./config

EXPOSE 8080

CMD ["./nofx-backend"]
```

### 7.3 Python Dockerfile

```dockerfile
# docker/Dockerfile.python
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    build-essential \
    ta-lib \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制源码
COPY engine/ ./engine/

# 编译 Protobuf
COPY proto/ ./proto/
RUN python -m grpc_tools.protoc \
    --proto_path=./proto \
    --python_out=./engine \
    --grpc_python_out=./engine \
    ./proto/services.proto

EXPOSE 50051

CMD ["python", "-m", "engine.grpc.server"]
```

---

## 8. 实施路线图

### 8.1 阶段划分

```
Phase 1: 基础架构搭建 (Week 1-2)
├── Protobuf 接口定义
├── Go 项目脚手架
├── Python 项目脚手架
├── Docker 环境配置
└── gRPC 通信测试

Phase 2: Python 引擎开发 (Week 3-6)
├── 数据提供者实现
│   ├── Tushare
│   ├── AkShare
│   └── 富途 API
├── 技术指标计算
├── AI 决策引擎
└── Backtrader 回测引擎

Phase 3: Go 后端开发 (Week 7-9)
├── HTTP API 服务
├── 交易执行层
├── 账户管理
└── WebSocket 推送

Phase 4: 集成测试 (Week 10-11)
├── 单元测试
├── 集成测试
├── 模拟盘测试
└── 性能测试

Phase 5: 前端适配 (Week 12-13)
├── API 适配
├── 回测界面更新
└── 仪表板更新

Total: 12-13 周 (约 3 个月)
```

### 8.2 技术选型总结

| 层级 | 技术 | 理由 |
|------|------|------|
| **前端** | React + TypeScript | 保持原 NOFX 技术栈 |
| **后端** | Go + Gin | 高性能 API 服务 |
| **引擎** | Python | 量化库生态丰富 |
| **通信** | gRPC | 高效的跨语言通信 |
| **数据库** | PostgreSQL | 时序数据存储 |
| **缓存** | Redis | 实时数据缓存 |
| **消息队列** | NATS | 轻量级消息队列 |

---

## 9. 总结

### 9.1 Python/Go 混合架构优势

1. **各取所长**
   - Go: 高性能、稳定、部署简单
   - Python: 丰富库、开发效率高

2. **量化生态**
   - Python: pandas、numpy、backtrader、TA-Lib
   - 成熟的数据源 SDK

3. **可扩展性**
   - 微服务架构
   - 独立部署和扩展

4. **开发效率**
   - 算法用 Python 快速开发
   - 系统用 Go 保证稳定性

### 9.2 关键成功因素

1. **gRPC 接口设计**: 清晰的服务边界
2. **性能监控**: Go 和 Python 之间的性能瓶颈
3. **错误处理**: 跨语言的错误传递
4. **数据一致性**: 缓存和数据库的同步
5. **部署运维**: 容器化和编排

### 9.3 下一步行动

1. 确认技术选型
2. 搭建开发环境
3. 定义 Protobuf 接口
4. 实现 MVP
5. 逐步迭代完善

---

**文档结束**

*本方案为 NOFX Python/Go 混合架构改造方案，结合了 Go 的高性能和 Python 的量化生态优势。*
