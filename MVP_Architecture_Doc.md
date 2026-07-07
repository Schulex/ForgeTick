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

## Goal of this MVP Architecture Doc

## Vocabulary

- Trigger domains — A trigger domain is a branch of a workflow, each branch has its own scheduler.
- Engine run —  When an engine is doing a pass on a trigger domain and executing every nodes.
- Node execution — When an engine execute a specific node in a trigger domain of a workflow, "node.execute" in the engine.

## Future-proofing decisions

This architecture doc is only for the MVP. However there are future features that constrains the architecture of this MVP. The architecture of this MVP will deliberately be designed for these future features, but these features will not be built in the MVP.

Architectural features for future features : (already needed for the future features)

- Engine and node apart, for being able to easily add node and custom node later
- Three layers of the execution model, for non-sequential excution and trigger domains
- Scheduler wiring, is to explicitly know which scheduler owns which nodes and to clearly define each trigger domains.
- Registry between node class and node type, is to add custom node later
- The abstract BrokerAdapter interface, is the extension point for multi-broker support
- CCXT, already abstracts 100+ exchanges, non-CCXT brokers would implement the same interface differently inside.
- One user per-instance and multi instances, keep the security model sound, cash isolation and no catastrophic single honeypot

Future architectural features : (needed in the future for the future features)

- Node groups, is for node working together (entry + stop-loss or multi-leg orders)
- Trigger domains, is for the non-sequential execution
- Message-passing node, for being able to cleanly pass a message between two triger domains
- Compensation logic, is for failure/error handling and what to do when it happened
- Failure/error handling, is to activate compensation logic
- Per-broker capability differences, eventually need a capability-declaration mechanism
- WebSocket market data stream, is for market ticks trading
- Schedule at each market ticks, for market tick trading
- Schedule in a loop, for logics needing to loop (example : LLM analyst)
- Ports auto-selection, for multi-instances
- Logs rotation + compression, for managing huge amount of logs
- Non-blocking logging, QueueHandler/QueueListener, for writing huge amount of logs
- JSONL output option, for users feeding logs into external tooling

Future features :

- Non-Sequential execution of workflows
- Scheduler option to runs the workflow at each market ticks
- Scheduler option to runs the workflow in a loop
- Trigger domain
- Easily add new Brokers
- Editor showing brokers capabilities
- OpenClaw and LLM agent
- Simulation mode
- Custom nodes
- Large variety of nodes
- Link to N8N
- Third-party clients
- Authentication
- Per-stream logs retention settings

### Node groups

NOT in this MVP but in the future, some nodes with real-side effect can work together, for example a entry + stop-loss or multi-leg orders. However interruption can't happen between these nodes. The solution is to group the nodes together and make this group behave like a normal node from the outside view. But in reality summoning another engine to execute the nodes in the groupe. With the catch that this other engine doesn't respond to the stop flag. That it requires zero change to the engine. When the other engine finish to execute the nodes inside the group the « main » engine take back and continue. If an error or a fail happened in this engine it is manage the same way as an error or a fail in the main engine.

### Concurrency & Parallelism

Default: all nodes run on the single asyncio event loop. Escapes are decided per node by one question — does the call block the loop, and does it hold the GIL while blocking?

Four type of nodes :

- tiny compute (normal nodes)      SMA, EMA, comparison, logic      run directly on the loop
- async-capable I/O wait           Binance via CCXT, HTTP APIs      native await — asyncio's home turf
- blocking wait / C-level work     embedded LLM lib, GPU inference  run_in_executor + THREAD
- pure-Python heavy compute        custom math node, pure loops     run_in_executor + PROCESS

The per-node timeout guard (§ Lifecycle) applies wherever a node executes.
Generalized per-node parallelism remains excluded: shipping cost exceeds the work for fast nodes.

Latency-sensitive tick strategies (future): isolation at workflow / trigger-domain granularity — never per node. A tick workflow may run in its own process bundling its own event loop, its own WebSocket market-data stream, and its own BrokerAdapter instance, eliminating jitter from unrelated workflows. Accepted cost: status/P&L reporting crosses a process boundary back to the server. Process-isolated workflows/domains (V2/V3): manager control (stop/kill flags) and status/P&L reporting both cross the process boundary via a small control+status channel between WorkflowManager and the child process.

### North Star Workflow

Workflow example to see as a north star. A market-mood workflow.

The goal is to be able to execute workflows with this type of complexity later on. Not right now because it’s not the point of this MVP. However this example workflow would be a great example to test the app and see if we are going in the right path. the goal is to be able to do a workflow with a part that use LLM to analyze the mood of the market. And another part with multiple strategy already built in the workflow. With the LLM part choosing if one strategy suits the mood among the many strategies to choose from in the workflow and execute the strategy. This is an idea about a workflow adapting to the mood of the market. But each strategy probably doesn’t has the same timeframe. One strategy will work on a 1h timeframe, another one will work on 10min timeframe and a third one will work on market tick directly. And the LLM part will just loop back each time it finishes. This means multiple schedulers in the same workflow with different time and schedulers working on market tick and schedulers working on looping back directly for the LLM part. With each scheduler having this own trigger domain. Each scheduler summoning engines when it trigger to run all the nodes in its domain. With message-passing between trigger domain using the message-passing-node.

## Component Map Diagram

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

Some node just have side effect instead of returning a something. However these nodes will looks the same shape. The order node has a side effect (calling the broker) instead of returning a number.
The "typed" part matters for the GUI: if an output port is type Number and an input port is type Candles, React Flow refuses to let you connect them. The types prevent nonsense wiring before it ever runs.
Some node need memory, each node that need memory save its internal state to SQLite, that's the self.prev_*.

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

Example of node :

    class SMANode(Node):
        type_name = "sma"

        def execute(self, inputs):
            candles = inputs["candles"]          # data from upstream
            period  = self.config["period"]      # user setting
            value   = average_of_last_closes(candles, period)
            return {"value": value}              # data for downstream

Example of node with memory :

    class CrossoverNode(Node):
        type_name = "crossover"

        def __init__(self, node_id, config):
            super().__init__(node_id, config)
            self.prev_a = None       # remembered across ticks
            self.prev_b = None

        def execute(self, inputs):
            a, b = inputs["a"], inputs["b"]
            crossed_up = (self.prev_a is not None
                        and self.prev_a <= self.prev_b
                        and a > b)
            self.prev_a, self.prev_b = a, b
            return {"crossed": crossed_up}

### MVP Nodes

Essential nodes needed to build a SMA crossover strategy.

1. Candle Data Source — fetches OHLCV candles from Binance for a
chosen symbol and timeframe
2. SMA — Simple Moving Average
3. EMA — Exponential Moving Average
4. RSI — Relative Strength Index
5. Comparison — outputs true/false based on >, <, =, ≠, or crossover
between two inputs
6. Logic — AND, OR, NOT on boolean inputs (combine multiple
conditions)
7. Time Interval Scheduler — runs the workflow every N seconds
8. Order Placement — sends a market or limit order to Binance
9. Chart Output — displays a signal over time in the node in the GUI

### Scheduler

A scheduler is the sole start node of a trigger domain: only its firing triggers an engine run of that domain, and every source node of the domain is wired to it by a trigger connection. The Message-passing node is considered as a source node from the point of view of the trigger domain below, this node need to be wired to the scheduler. One scheduler per domain.

Types:

- Time Interval Scheduler (MVP) — fixed-rate only: fires every N milliseconds/seconds/minutes/hours by the clock, independent of run duration.
- Specific Time Scheduler (future) — fires at a configured list of clock times, each entry marked recurring (daily) or once.
- Market Tick Scheduler (future) — fires per tick from a streaming market-data source. overlap policy is mandatory here (ticks outpace runs by design).
- Loop Scheduler (future) — fixed-delay: refires when the previous run returns, after config delay (default=0, milliseconds/seconds/minutes/hours). The runner knows completion trivially — it awaits the engine call and refires on return.

Activation windows (Every schedulers) : a scheduler may be configured active only between given times (e.g. market open → close). Outside its window it does not fire. A window closing mid-run does not interrupt the run (per-node atomicity, § Lifecycle) and cancels nothing — a window pauses triggering; it is not a stop. Window times are stored timezone-explicit; unused in the MVP for 24/7 crypto, designed in now.

Gate (Every schedulers) : Optional condition input (V2, future): a non-triggering boolean gate, mailbox-latest semantics, default 1/active, evaluated at fire time; gate=0 pauses the fire. For example, useful for stoping a loop. The gate can be seen as the activation windows at the level of behavior; the gate pauses the triggering; it is not a stop.

The gate and the activation windows never causes execution — the "only schedulers trigger" invariant holds. However, when the activation windows open or when the gate passes at one, this is what happened to each scheduler :

- The Time Interval Scheduler will start again by firing a trigger
- The Specific Time Scheduler will wait until its next configured fire, if he doesn't have any configured fire left the workflow will be marked as COMPLETED
- The Market Tick Scheduler will trigger normally at the next market tick.
- The Loop Scheduler will start again by firing a trigger

Workaround pattern (MVP): daily-at-a-time firing ≈ activation window of one minute + interval larger than the window (e.g. window 18:00–18:01, interval 5 min → one run per day at 18:00).

Time is always stored and expressed in timezone-explicit.

A scheduler whose entries are exhausted triggers workflow COMPLETED (§Lifecycle).

MVP overlap policy: a fire arriving while the domain's previous run is in progress is skipped with a warning — one domain never runs twice concurrently.

Overlap policy (V2/V3, per-scheduler config) :

- Skip (default)
- Bounded pipeline
- Bounded queue

The bounded queue is size-limited and drop-oldest with warning — unbounded queues are excluded : memory growth under load, and oldest queued runs execute against stale market data. The default bound is N=10.

The bounded pipeline is size-limited and drop-newest with warning — drop oldest is excluded : it's wasteful to cut the engine at 95% and even worse than wasteful it's DANGEROUS if the run has already placed an entry order and hasn't reached the stop-loss node yet. And even a graceful abort doesn't free a slot instantly (STOPPING takes time), so it wouldn't even help the incoming fire. The default bound is N=10.

"Stop on overlap" is deliberately not offered: shedding load must not become an outage at the busiest moment.

Overlapping runs of the same domain touch the same stateful node instances. Newest runs will also require the node state of the previous run. If there are 4 engines on one trigger domain, each run depends on the previous for updated nodes states. To avoid any node state problem when running in pipeline, engine must NEVER overtake each other and always run down the trigger domain using the same node order.

### Registry

WORKFLOW JSON (data, on disk)            NODES FOLDER (code, on disk)
┌───────────────────────────┐          ┌──────────────────────────┐
│ {                         │          │ nodes/                   │
│   "nodes": [              │          │   candle_source.py       │
│     {                     │          │   sma.py   ◄──────┐      │
│       "id": "n1",         │          │   ema.py          │      │
│       "type": "sma", ─────┼───┐      │   rsi.py          │      │
│       "config": {         │   │      │   order.py        │      │
│         "period": 20      │   │      │   ...             │      │
│       }                   │   │      └───────────────────┼──────┘
│     }                     │   │                          │
│   ]                       │   │   each file registers    │
│ }                         │   │   its class by name      │
└───────────────────────────┘   │                          │
                                │                          │
                                ▼                          │
                      ┌─────────────────────┐              │
                      │   NODE REGISTRY     │              │
                      │   (a dictionary)    │              │
                      │                     │              │
                      │  "candle_source"→…  │              │
                      │  "sma"  ────────────┼──────────────┘
                      │  "ema"  → EMANode   │
                      │  "rsi"  → RSINode   │
                      │  "order"→ OrderNode │
                      └──────────┬──────────┘
                                 │  "look up 'sma' → get SMANode
                                 │   → build it with period=20"
                                 ▼
                      ┌─────────────────────┐
                      │      ENGINE         │
                      │  builds & runs the  │
                      │  node instances     │
                      └─────────────────────┘

    # engine.py
    NODE_REGISTRY = {}

    def register(node_class):           # a decorator
        NODE_REGISTRY[node_class.type_name] = node_class
        return node_class


    # nodes/sma.py
    from engine import Node, register

    @register                            # this line puts SMANode in the registry
    class SMANode(Node):
        type_name = "sma"
        def execute(self, inputs):
            ...


    # loading a workflow
    def build_node(node_json):
        cls = NODE_REGISTRY[node_json["type"]]      # "sma" → SMANode
        return cls(node_json["id"], node_json["config"])

### Asyncio

Most of what ForgeTick does is waiting: waiting for the broker to answer an API call, waiting for the scheduler's next fire, ... A synchronous program would freeze during each wait. ForgeTick will use Asyncio.

#### asyncio.shield

asyncio.shield is a really important feature. asyncio.shield ensures that even a forced cancellation can't interrupt critical call mid-flight (Example : order node for the broker call).

  order node running
    await shield(place_order()) ─► request sent ─► reply received ─► recorded  ✓
                                   ▲                                      │
                          TIMEOUT or SHUTDOWN fires here                  │
                                   │                                      │
                          held back until the call returns ───────────────┘
                                   │
                          THEN the cancellation proceeds
              Outcome always known.  ✓

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
- COMPLETED — the workflow's scheduler(s) have no future fires (once-only times exhausted; permanently-closed gate)
- ERRORED — something threw; treated like STOPPING but flagged for the user

The WorkflowManager tracks every WorkflowRunner's state. That collection of states is what status returns and what the Running Workflows panel renders.
Unlike STOPPED, completion cancels nothing: resting orders remain at the broker, positions untouched, the runner is released. Stop = abort and make safe (cancel open orders); Complete = natural end of triggering (leave market state as-is). Transition RUNNING → COMPLETED. Not reachable in the MVP (the Interval Scheduler never completes); state reserved and designed now. Orphaned resting orders after completion are owned by the broker and covered by reconciliation.

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

### Infinite node runtime

To protect against a node running forever is to assign to every node a configurable time budget, a maximal allowed runtime. This is to insure that there are no runaway. If the execution time of the node is greater than this maximal allowed runtime the engine stop the workflow.

### Validation of domain rules

The validation of domain rules are on the editor level

### The full control-flow path

    CLI:  forgetick stop sma-cross
    → POST /workflows/sma-cross/stop
        → WorkflowManager.stop("sma-cross")
            → runner.stop_requested = True
            → (finishes current node) → cancels its open orders → STOPPED
        → server broadcasts new state over WebSocket
    → GUI's Running Workflows panel updates instantly; CLI status reflects it

## Persistence & recovery

The app need to save the definition of the workflows and the running states.

### JSON

To be able to easily share workflows, the definition of workflows are save in a JSON.

### SQLite

SQLite save the states of running workflows. SQLite is the database where the running states are saved.

### Shutdown

Case 1 : Clean Shutdown
This can append if Mathias intentionally stops the app. All of these must come
back : workflow definitions, the current state of each running workflow node, open
positions, and pending orders, recent log. When everything is back the app must
choose between resume the workflows with correct states or stop properly the
workflow and cancels all open orders via the broker. This decision depends on the
downtime and the workflows. If a workflow trade on an hour timeframe and the
downtime is only one minute the app need to resume the workflow and continue.
However if a workflow trade on a minute timeframe and the downtime is 15 minutes
the app need to stop properly the workflow and cancels all open orders via the
broker.
Case 2 : Unclean Shutdown
This can append for multiple reason for example : crash, forced reboot of the
computer, power outage… On restart the app do not blindly resume ! The state on
disk might be stale, prices have moved, conditions have changed. Instead, the
engine should detect “the last shutdown was unclean” and present Mathias a
choice: cancel everything and start fresh, or resume and reconcile market state
against the broker.

In both case the app need to always reconcile market state against the broker. The broker is the only source of truth for the market state. Normaly after a clean shutdown this step is not useful, it's just always great to check and make sure. However after an unclean shutdown if the resume option is choosed it's mandatory to reconcile market state against the broker.

## Broker layer

        ENGINE / NODES
              │
              │  speaks only the brokerAdapter
              ▼
     ┌─────────────────────┐
     │   BrokerAdapter     │   ◄── abstract: defines WHAT operations exist
     │   (interface)       │
     └─────────┬───────────┘
               │ 
               ▼
     ┌─────────────────────┐
     │   BinanceAdapter    │   ◄── concrete: defines HOW, using CCXT
     │   (uses CCXT)       │
     └─────────┬───────────┘
               ▼
            CCXT library
               ▼
           Binance API

For this MVP there is only one adapter: BinanceAdapter, wrapping CCXT.

### The adapter interface

The engine talks to an abstract BrokerAdapter, not to CCXT directly. The adapter defines a small set of operations the engine needs:

BrokerAdapter (abstract interface)
├── fetch_candles(symbol, timeframe, limit)   → list of OHLCV candles
├── place_order(symbol, side, type, amount)   → order result (id, status)
├── cancel_order(order_id, symbol)            → cancellation result
├── fetch_open_orders(symbol)                 → list of resting orders
└── fetch_positions()                         → current holdings

This small set of operations match the order node capability in this MVP (Market and limit orders only). Spot trading only (no margin, no derivatives)

This boundary exists so adding a second broker later (Kraken, Interactive Brokers) means writing a new class — KrakenAdapter — that implements the same five operations. The engine, the nodes, and the workflows are untouched, because they only ever knew the abstract interface. "Add Kraken" becomes "write one new file and register it," exactly like the node registry pattern. The broker the user wants becomes a configuration choice, not a code change.
The engine depends on an interface, never on a concrete vendor.

### Reconciliation

The broker owns market state. The broker layer is how reconciliation happens. On unclean restart, the recovery logic calls fetch_open_orders() and fetch_positions() to ask the broker what actually exists, then compares against ForgeTick's last-known state in SQLite. Where they disagree, the broker wins, and the user is prompted before anything resumes. The adapter is the only component that can answer "what is actually true at the exchange right now."

### Credentials

The uer's API Keys live in a local configuration file on the user's own machine, read by the adapter at startup. They are never transmitted to ForgeTick and never logged. This is the local-first property. The user's own machine talks directly to the user's own broker account.

### Default Broker

To determine which broker is used for transactions in a workflow, a three-level default broker model is needed.
The three-level default broker model :

- Connection settings level — the global default broker
- Workflow level — can override the global default for this particular workflow
- Order node level — a dropdown that defaults to a default-broker option, which resolves to "whatever the workflow says," but can be pinned to a specific broker when needed.

## API surface

### Purpose

The API is the contract between the clients (Browser GUI, CLI, and any future third-party client) and the backend server. Because the server is the single source of truth, every action a client takes and every update it receives flows through this API. Clients hold no authority of their own: they send requests and render what the server reports. The API is treated as a public contract.

### Two channels, two jobs

    CLIENTS                                     SERVER
    ┌──────────────┐                      ┌──────────────────┐
    │ Browser GUI  │   ── HTTP request ──►│   commands &     │
    │     CLI      │◄── HTTP response ──  │   queries        │
    │  (3rd party) │                      │                  │
    │              │◄═══ WebSocket ═══════│   live state     │
    │              │     (push)           │   broadcasts     │
    └──────────────┘                      └──────────────────┘

- HTTP — for actions and questions: "run this workflow," "what's running?" One request up, one response back. How clients cause things and pull state.
- WebSocket — for live updates: the server pushes state changes to all connected clients the instant they happen, unprompted. How clients stay in sync without polling.

### Endpoint conventions

- All routes are versioned under /api/v1/. This lets a future /api/v2/ run alongside v1, so existing clients don't break when the API evolves.
- Reads use GET; any operation that changes server state uses POST, with the action named in the path (/run, /stop, /kill, /save). One uniform rule, no special cases: every route reads as resource/action.

### HTTP endpoints (MVP)

POST   /api/v1/workflows/{name}/run    → start a workflow               (CLI: run,    GUI: run button)
POST   /api/v1/workflows/{name}/stop   → stop one workflow + its orders (CLI: stop,   GUI: stop button)
POST   /api/v1/workflows/{name}/save   → write a workflow's JSON        (GUI: save)
GET    /api/v1/workflows/{name}/load   → fetch a workflow's JSON        (GUI: render in editor)
GET    /api/v1/workflows/running       → status of running workflows    (CLI: status, GUI: Running panel)
GET    /api/v1/workflows/list          → list available workflow files  (CLI: list,   GUI: file browser)
POST   /api/v1/kill                    → stop all + cancel all orders   (CLI: kill,   GUI: kill switch)

applogs and tradelogs are deliberately not endpoints — logs stream from files, not through the API. open/close are not endpoints either — they are GUI-local editor actions ("which workflow is on my canvas"), state the server doesn't track. The endpoint list is exactly the operations that act on server-owned state.

### The filesystem principle behind load and save

A browser runs in a security sandbox: it cannot read or write the local filesystem — only the Python server can.

- load — the browser needs a workflow's content to draw the nodes on the canvas, but cannot read the file itself, so it asks the server, which reads the file and returns the JSON. (Distinct from run, where the server reads the file and executes it server-side, sending no content to anyone.)
- save — the browser holds the graph the user drew but cannot write it to disk, so it sends the JSON to the server, which writes the file.

### WebSocket: the sync mechanism

    Mathias types `forgetick run sma-cross` in his terminal
            │
            ▼
    POST /api/v1/workflows/sma-cross/run     ── HTTP, from the CLI
            │
            ▼
    server starts the runner, state → RUNNING
            │
            ▼
    server broadcasts {workflow: sma-cross, state: RUNNING, ...}
            │
            ├──═══ WebSocket ═══►  Browser GUI: Running panel shows it appear
            └──═══ WebSocket ═══►  any other connected client updates too

Any change to server state is broadcast to every connected client immediately.

The CLI caused the change over HTTP; the GUI learned of it over WebSocket, without polling and without knowing the CLI exists. No client holds state; each renders the server's last broadcast. A client can disconnect and reconnect at any time and get the true picture by re-querying GET /workflows/running and resubscribing — a crashed browser or dropped SSH session loses nothing, because it held nothing.

MVP broadcast payloads: workflow state transitions (IDLE/RUNNING/STOPPING/STOPPED/ERRORED) and per-workflow status (runtime, trade count, P&L), keeping the Running panel and status live.

### Self-documentation

FastAPI auto-generates an interactive OpenAPI (Swagger) specification from the endpoint code, served at /docs. This is both the developer reference and the contract any third-party client author reads to build against ForgeTick.

### API MVP scope

- The seven HTTP endpoints above; one WebSocket stream broadcasting state + status to all clients.
- No authentication: single-user local app, the user is whoever is at the machine.
- Binds to localhost by default on a fixed port; the user may expose it on their LAN at their own choice (the "run on a home server" scenario).

### LAN, "home server" scenario

Binds to localhost by default on a fixed port; the user may expose it on their LAN at their own choice. This option can be change in the settings. This is if someone run ForgeTick on a local server / home server. In this case the connection will be from another computer via SSH terminal or via the broswer like usual.

### Per-user instance

One user per-instance and multi instances, NOT one instance with multiple user ! Rationale: (1) crash isolation — one user's faulty workflow cannot take down others or cost them money; (2) security — broker API keys are never pooled into one database, avoiding a catastrophic single honeypot. Per-instance keep the security model sound.

### Ports

MVP has one instance on a fixed default port, the port is 18181 because it's memorable and avoid conflict with famous ports. V2 will have dynamic ports for supporting multi instances. For V2, on startup the server writes its port (and a PID) to a small runtime file; the CLI reads that file to find the instance. With multiple instances, each writes its own entry.
Not now, not for this MVP, but a future choice would be how a V2 instance choose its ports. Two choices :

- Scan upward from a base
- Bind to port 0

### Third-party clients

Third-party clients are a native consequence of the server-as-source-of-truth design: any program speaking the versioned HTTP+WebSocket contract is a valid client, indistinguishable from the built-in GUI/CLI. Preserved (not built) by three cheap disciplines already adopted — API versioning, treating the API as a public contract, and the free OpenAPI docs. No formal plugin system, client SDK, or client-auth in the MVP; the clean versioned API is the whole enabler.

### LAN exposure

Lan exposure on the network, by default the server is only on the machine — 127.0.0.1. An option in the settings can exposed the server to all network — 0.0.0.0. The app need to restart to enable this option. When this setting is change from 127.0.0.1 to 0.0.0.0, a warning should pop at the screen and another warning need to be next to the option. When ForgeTick starts in exposed mode, it should print a warning line in the launch terminal ("⚠ listening on 0.0.0.0 — anyone on your network can control this instance; never expose to the internet"). This option to exposed ForgeTick is for the headless use for the "home server" scenario. This option justify the authentication feature in V2.

### Authentication

Authentication is introduced in V2, so only the clients with the right identity can control the server. Useful in the "home server" scenario with the server expose on the local network. The endpoints shapes are unchanged by this; instances simply gains an identity guard in front.

## Logging

Logging answers three different questions, for three different readers. ForgeTick therefore keeps three separate log streams, each with its own folder, retention, and durability policy:

- tradelogs — trade events only: orders placed, fills, cancellations, rejections. The "what happened with money" stream — the audit trail.
- workflowlogs — workflow execution events: engine runs, node executions, state transitions, scheduler fires. The "what is my strategy doing" stream.
- applogs — application events: server lifecycle, configuration, API activity, errors of the software itself. The "is the software healthy" stream.

### Event routing rule

Every event has exactly one home, classified by subject, not severity. A node crash is a workflow event (workflowlogs, level ERROR). A failed order placement is a trade event (tradelogs). A port-binding failure is an app event (applogs). The same event is never written to two streams.

### Line format

One event per line, fixed field order, human-readable and grep/parse-friendly (key=value for structured fields):

    logs/trades/2026-06-10.log:
    2026-06-10T14:23:05.123Z INFO workflow=sma-cross node=order-1 event=order_placed side=buy amount=0.01 symbol=BTC/USDT type=market order_id=8837123 status=filled price=42150.30

    logs/workflows/2026-06-10.log:
    2026-06-10T14:23:05.081Z INFO workflow=sma-cross event=engine_run_start trigger=scheduler
    2026-06-10T14:23:05.119Z INFO workflow=sma-cross node=sma-fast event=node_executed duration_ms=2
    2026-06-10T14:23:05.124Z INFO workflow=sma-cross event=engine_run_end result=completed duration_ms=43

    logs/app/2026-06-10.log:
    2026-06-10T14:20:11.002Z INFO component=server event=startup version=0.1.0 host=127.0.0.1 port=18181
    2026-06-10T14:20:11.480Z ERROR component=broker event=connection_failed broker=binance error="timeout after 10s"

Timestamps are ISO 8601 with milliseconds, UTC. Levels: INFO / WARN / ERROR.

### Files, rotation, retention

- One folder per stream: logs/trades/, logs/workflows/, logs/app/. One file per day, named by date.
- Retention (MVP): applogs and workflowlogs are deleted after two weeks. tradelogs are never auto-deleted — trade history is tiny (bytes per trade) and is the user's audit trail.
- The CLI commands applogs, workflowlogs, tradelogs each stream their file live (equivalent of tail -f), Ctrl-C to exit. Because streams are plain files, every Unix tool (tail, grep, awk) works on them directly — the CLI commands are a convenience, not a gatekeeper.
- The launch terminal streams a merged live view of all three, each line prefixed for filtering:

    [APP]   2026-06-10T14:20:11.002Z INFO component=server event=startup version=0.1.0 host=127.0.0.1 port=18181
    [WORKFLOW]  2026-06-10T14:23:05.081Z INFO workflow=sma-cross event=engine_run_start trigger=scheduler
    [TRADE] 2026-06-10T14:23:05.123Z INFO workflow=sma-cross node=order-1 event=order_placed side=buy amount=0.01 symbol=BTC/USDT type=market order_id=8837123 status=filled price=42150.30

### Durability policy (per stream)

Buffering follows each stream's tolerance for losing its final moments in a crash:

- tradelogs: flushed to disk immediately, per event. If the process dies right after an order, the line must already be on disk. Volume is tiny; immediate flushing is free.
- workflowlogs: default buffering, any ERROR record immediately flushes the entire workflowlogs's buffer and applogs's buffer (context lines + error).
- applogs: default buffering, any ERROR record immediately flushes the entire workflowlogs's buffer and applogs's buffer (context lines + error).

Residual loss window applies only to deaths the process cannot detect (power loss, SIGKILL), where no error record exists to trigger on.

### Logs MVP scope

- Three streams, daily files, the merged prefixed launch-terminal view, three CLI streaming commands.
- Two-week purge for app/workflow logs; NO purge for tradelogs.
- Standard Python logging with default buffering per the policy above; no custom buffer machinery.

### Logging Future-proofing

- Per-stream retention settings (V2): user-configurable keep-durations per stream, replacing the fixed two-week rule.
- Rotation + compression (V2): gzip files older than one day (text logs compress ~10–20×), then purge per retention policy. Required before the market-tick trigger ships — tick workflows can emit gigabytes/day of workflowlogs uncompressed.
- Non-blocking logging (V2, with tick trigger): QueueHandler/QueueListener so file I/O never blocks the event loop; flush intervals ~0.5 s.
- JSONL output option (V2+): a config switch to emit JSON-lines instead of key=value for users feeding logs into external tooling. The field structure above is already designed to survive that switch unchanged.

## Repo structure








