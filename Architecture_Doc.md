# Architecture Doc

version : MVP

## Overview

### Component Map Diagram

    CLIENTS  (stateless — just windows onto the server)
    ┌──────────────┐              ┌──────────────┐
    │ Browser GUI  │              │     CLI      │
    │ React Flow   │              │  python cmd  │
    └──────┬───────┘              └──────┬───────┘
           │  HTTP + WebSocket           │  HTTP
           └──────────────┬──────────────┘
                          ▼
        ┌────────────────────────────────┐
        │        BACKEND SERVER          │  ◄── Server owns 
        │           (FastAPI)            │    execution state
        │  HTTP API  +  WebSocket (live) │
        └────────────────┬───────────────┘
                         ▼
        ┌────────────────────────────────┐
        │        EXECUTION ENGINE        │
        │   loads & runs node graphs     │
        └───┬──────────┬─────────────┬───┘
            ▼          ▼             ▼
      ┌─────────┐ ┌──────────┐ ┌──────────┐
      │  NODES  │ │ BROKER   │ │ PERSIST. │
      │ 9 types │ │ ADAPTER  │ │ SQLite + │
      │ classes │ │ (CCXT)   │ │ JSON     │
      └─────────┘ └────┬─────┘ └──────────┘
                       ▼
                   Binance API ◄── Broker owns market state

    (all engine activity → LOGGING → logs/app + logs/trades)

The only two sources of truth are the server for the execution state and the broker for the market state. Everything else is a client or a service. Nothing else owns state.

## Execution model & nodes

To execute a workflow there are two distinct jobs :

- The engine, who knows how to run a graph of nodes: what order to execute them in, how to pass data along the wires. It knows nothing about what any individual node does.
- The node, who knows how to do one thing: compute an SMA, place an order, compare two numbers. It knows nothing about the graph it lives in.

It's really important to keep this two job apart, the whole architecture rests on keeping them apart.

### Layers of the execution model

The execution model is layered in three layers.

The deepest layer is the engine layer, this layer execute the nodes in a trigger domains (in a branch) of a workflows. A workflow can have multiple trigger domains, this means multiple engine per workflows. The second layer is the WorkflowRunner layer, this layer manage and run every engine in a workflow. On top of this, multiple workflows can run in parallel. The third and shallowest layer is the WorkflowManager layer, this layer manage every running workflows.

    ┌───────────────────────────────────────────┐
    │ WorkflowManager                           │
    └───────────┬────────────────────┬──────────┘
                ▼                    ▼
    ┌───────────────────────┐       ...
    │ WorkflowRunner        │
    └─────┬───────────┬─────┘
          ▼           ▼
    ┌───────────┐    ...
    │ Engine    │
    └───────────┘

#### The engine loop

    def run_one_tick(graph, results={}):
        for node in topological_order(graph):          # upstream first
            node_inputs = collect_from_upstream(node, graph, results)
            results[node.id] = node.execute(node_inputs)
        return results

### Node anatomy

- A type name — "sma", "candle_source", "order". Used to identify it in the JSON and to look up its code.
- Config — the user's settings from the GUI. For an SMA node, { "period": 20 }. For a candle source, { "symbol": "BTC/USDT", "timeframe": "1h" }.
- Input ports — named, typed sockets where data arrives. SMA has one input: candles.
- Output ports — named, typed sockets where results leave. SMA has one output: value.
- An execute method — takes the inputs, returns the outputs.

Some node will just have side effect instead of returning a something. However these nodes will looks the same shape. The order node has a side effect (calling the broker) instead of returning a number.
The "typed" part matters for the GUI: if an output port is type Number and an input port is type Candles, React Flow refuses to let you connect them. The types prevent nonsense wiring before it ever runs.

#### The base interface, in plain Python

Every node inherits from one base class:

    class Node:
        type_name = "base"          # each node overrides this

        def __init__(self, node_id, config):
            self.id = node_id
            self.config = config     # the user's GUI settings

        def execute(self, inputs: dict) -> dict:
            # inputs:  {port_name: value}  coming from upstream nodes
            # returns: {port_name: value}  going to downstream nodes
            raise NotImplementedError

A concrete node is small, example of node :

    class SMANode(Node):
        type_name = "sma"

        def execute(self, inputs):
            candles = inputs["candles"]          # data from upstream
            period  = self.config["period"]      # user setting
            value   = average_of_last_closes(candles, period)
            return {"value": value}              # data for downstream



## Lifecycle & control flow

## Persistence & recovery

## Broker layer

## API surface

## Logging

## Repo structure

## Future-proofing decisions
