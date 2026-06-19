# Architecture Doc

version : MVP

## Overview

### Presentation

The concept is to make the N8N / ComfyUI of the trading world. N8N and
ComfyUI are node based app where you can create workflow by linking the nodes
together to automate some process.
The Idea is to create an app that use the power of node based workflow to
automate trading orders and send them to the user’s Broker.
This app, called ForgeTick, sits between the user and the broker’s API. The
user will not use his logic to trade directly himself anymore, he will put his logic on a
workflow. After this, ForgeTick will execute the workflow and send the trading order
to the broker’s API. The user will not anymore execute his logic and tell the trade
order to the Broker, everything will be automated by the workflow he created using
his logic.

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

## Future-proofing decisions

This architecture doc is only for the MVP. However there are future features that constrains the architecture of this MVP. The architecture of this MVP will deliberately be designed for these future features, but these features will not be built in the MVP.

Architectural features for future features : (already needed for the future features)

- Engine and node apart
- Three layers of the execution model
- Scheduler wiring
- Registry between node class and node type

Future architectural features : (needed in the future for the future features)

- Node groups
- Trigger domains
- Message-passing node
- compensation logic

Future features :

- Non-Sequential Execution of the Workflows
- OpenClaw and LLM agent
- Simulation mode
- Custom nodes
- Large variety of nodes
- Link to N8N

Workflow example:

(The following section needs to be rewrite)
In my head I’m thinking about a workflow I would like to be able to do later on. Not right now because it’s not the point of this MVP. However I’m thinking that this workflow would be a great example to test the app and see if we are going in the right path I would like to do a workflow with a part that use LLM to analyze the mood of the market. And another part with multiple strategy already built in the workflow. With the LLM part choosing if one strategy suits the mood among the many strategies to choose from in the workflow and execute the strategy. This is an idea about a workflow adapting to the mood of the market. But each strategy probably doesn’t has the same timeframe. One strategy will work on a 1h timeframe, another one will work on 10min timeframe and a third one will work on market tick directly. And the LLM part will just loop back each time it finishes. This means multiple schedulers in the same workflow with different time and schedulers working on market tick and schedulers working on looping back directly for the LLM part. We also need to think that an LLM node can take easily 30 sec to run when there probably is a strategy that need to run entirely in less than a few seconds. So the engine need to be able to run multiple nodes while another node run in parallel for a long time. This means multiple loop in the engine because the first loop in the engine is stuck on node.execute with an LLM node. Or an even better solution is to keep the engine as it is (simple and easy) and run an engine per-scheduler in the workflow. We just need to think about an engine manager on top in the architecture.

## Vocabulary

- Trigger domains — A trigger domain is a branch of a workflow, each branch has its own scheduler.
- Engine run —  When an engine is doing a pass on a trigger domain and executing every nodes.
- Node execution — When an engine execute a specific node in a trigger domain of a workflow, "node.execute" in the engine.

## Execution model & nodes

To execute a workflow there are two distinct jobs :

- The engine, who knows how to run a graph of nodes: what order to execute them in, how to pass data along the wires. It knows nothing about what any individual node does.
- The node, who knows how to do one thing: compute an SMA, place an order, compare two numbers. It knows nothing about the graph it lives in.

It's really important to keep this two job apart, the whole architecture rests on keeping them apart.

### Layers of the execution model

The execution model is layered in three layers.

The deepest layer is the engine layer, this layer execute the nodes in a trigger domains (in a branch) of a workflow. Not in this MVP, however in the future, a workflow would be able to have multiple trigger domains, this means multiple engine per workflows. The second layer is the WorkflowRunner layer, this layer manage and run every engine in a workflow. On top of this, multiple workflows can run in parallel. The third and shallowest layer is the WorkflowManager layer, this layer manage every running workflows.

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

    async def run_engine_pass(self, graph):
        for node in topological_order(graph):
            if self.stop_requested:        # checked BETWEEN nodes
                return "interrupted"        # safe: no node was cut mid-execution
            node_inputs = collect_from_upstream(node, graph, results)
            results[node.id] = await node.execute(node_inputs)  # runs to completion
        return "completed"

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

### Node groups

NOT in this MVP but in the future, some nodes with real-side effect will work together, for example a entry + stop-loss or multi-leg orders. However interruption can't happen between nodes working together. The solution is to group the nodes together.
(The following section needs to be rewrite)
Like considering the groupe of nodes like a single node for the engine, like making the group  of nodes like a node from the point of view of the engine. But in reality summoning another engine to execute the nodes in the groupe. With the catch that this other engine has not the part « if self.stop_requested: ». Right now in the MVP we have no guard if a node run forever. The solution of this problem will be probably in the engine, so if we also use another engine to run the nodes in the groupe the solution will follow. And we can manage the fails also the same way. A fail in the other engine inside the group will be manage the same way a fail is manage in the « main » engine. Another advantage is that it requires zero change to the engine. When the other engine finish to execute the nodes inside the group the « main » engine take back and continue.

### MVP Nodes

Essential nodes needed to build a SMA crossover strategy.

1. Candle data source — fetches OHLCV candles from Binance for a
chosen symbol and timeframe
2. SMA — Simple Moving Average
3. EMA — Exponential Moving Average
4. RSI — Relative Strength Index
5. Comparison — outputs true/false based on >, <, =, ≠, or crossover
between two inputs
6. Logic — AND, OR, NOT on boolean inputs (combine multiple
conditions)
7. Scheduler / Trigger — runs the workflow every N seconds, or at
specific times
8. Order placement — sends a market or limit order to Binance
9. Chart output — displays a signal over time in the node in the GUI

## Lifecycle & control flow

### The core tension

Trading software have two requirements that fight each other:

- Stoping fast, "stopping soon" isn't acceptable
- Stoping safe, always stop in a way to know the truth

Almost every decision below is about honoring both. The resolution is: stopping is instant at the boundaries between operations, but never interrupts an operation that has a real-world side effect in flight.

### WorkflowRunner lifecycle states

The WorkflowRunner is a state machine

- IDLE — wait
- RUNNING — scheduler is ticking, engine runs the graph each tick
- STOPPING — asked to stop; finishing safely, cancelling orders
- STOPPED — fully halted, state saved
- ERRORED — something threw; treated like STOPPING but flagged for the user

The WorkflowManager tracks every WorkflowRunner's state. That collection of states is what status returns and what the Running Workflows panel renders.

### Graceful stop

The stop command or the stop button does not violently abort anything, it set a flag. this flag is read between each node, never mid-node. This insure that a node is always fully run or not started. This is to insure to NOT stop a node between a placed order and recording the placed order in the SQLite. A node is atomic by construction. shutdown() then cancels that workflow's open orders at the broker and marks STOPPED.

Stop on the three time scales :

- Between engine runs (scheduler is sleeping): kill is instant — nothing is running, just flip all flags and cancel pending orders.
- Between nodes (mid engine run): kill stops the loop before the next node starts.
- Within a node (a node is executing): the node finishes. If it's the order node with an HTTP call in flight, that call completes so you know the outcome, then the kill proceeds. (This is the asyncio.shield around the broker call — the one uninterruptible section.)

### Kill switch

The kill switch is graceful-stop applied to every runner at once.

### Two sources of truth

There are two sources of truth. The backend server own the execution state of workflows. The broker own the market state, what positions and orders actually exist.

### What stop/kill actually cancel

When the stop button, stop command, kill switch or the kill command is used. The app cancel pending/resting orders (limit orders sitting on the book, not yet filled). It does NOT liquidate positions (assets already owned).

### The full control-flow path

    CLI:  forgetick stop sma-cross
    → POST /workflows/sma-cross/stop
        → WorkflowManager.stop("sma-cross")
            → runner.stop_requested = True
            → (finishes current node) → cancels its open orders → STOPPED
        → server broadcasts new state over WebSocket
    → GUI's Running Workflows panel updates instantly; CLI status reflects it

## Persistence & recovery

(The following section needs to be write)
Clean shutdown
Unclean shutdown
reconcile market state against the broker,

## Broker layer

## API surface

## Logging

## Repo structure

I still need to add :

- asyncio.shield
- node groups as composite nodes (group = node from the engine's view, sub-engine inside, no stop checks within, inherits timeout guard and error path)
- per-node-execution timeouts as the runaway guard (MVP includes this)
- groups solve interruption, not partial failure — compensation logic deferred
- schedulers wired to source nodes, scheduler as sole start node; trigger domains with one scheduler each
- message-passing node pairs for cross-domain data, non-triggering, mailbox-latest semantics, skip-if-empty
- editor-level validation of domain rules.
