# Simple LangGraph Example

This project demonstrates a minimal LangGraph state machine using a simple `StateGraph` with three nodes.

## What this example does

The script creates a graph with:

- `node_1` as the starting node
- a conditional jump from `node_1` to either `node_2` or `node_3`
- both `node_2` and `node_3` ending the workflow

The graph state is stored in a `TypedDict` called `State`, and each node updates the `graph_state` string.

## Project files

- `simple_LG.py` — the complete example script
- `README.md` — setup and usage notes

## Python dependencies

Install the required package:

```bash
python3 -m pip install langgraph typing_extensions
```

If you are using a virtual environment, activate it first and then run the same command.

## How to run

From the project directory:

```bash
python3 simple_LG.py
```

Example output:

```text
---Node 1---
---Node 3---
Final graph state: {'graph_state': 'Hi, this is Shashi. I am sad!'}
```

The exact result may vary because the script randomly chooses between `node_2` and `node_3`.
The script also saves a diagram of the graph as `graph.png` in the current working directory.

## Key LangGraph concepts shown

### 1. Typed state

```python
class State(TypedDict):
    graph_state: str
```

This defines the data passed through the graph. In this example, the state is a string that stores the current message.

### 2. Nodes

```python
def node_1(state):
    return {"graph_state": state['graph_state'] + " I am"}
```

A node is a Python function that receives the current state and returns updated state.

### 3. Conditional routing

```python
def decide_mood(state) -> Literal["node_2", "node_3"]:
    if random.random() < 0.5:
        return "node_2"
    return "node_3"
```

This function decides which node should be executed next. In this example, it randomly chooses between the two possible branches.

### 4. Graph construction

```python
builder = StateGraph(State)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
```

`StateGraph` is the builder used to define the workflow.

### 5. Entry and exit edges

```python
builder.add_edge(START, "node_1")
builder.add_conditional_edges("node_1", decide_mood)
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)
```

This wires the graph together:

- `START` goes to `node_1`
- `node_1` chooses the next node
- `node_2` and `node_3` both lead to `END`

### 6. Invoking the graph

```python
graph = builder.compile()
result = graph.invoke({"graph_state": "Hi, this is Shashi."})
print("Final graph state:", result)
```

`graph.invoke(...)` executes the graph with the initial state and returns the final state.

## Notes

- This is a simple demonstration, not a production workflow.
- The random branch is intentional to show conditional routing in a minimal way.
- To make the behavior deterministic, replace `random.random()` with a fixed condition or explicit user input.

## Optional: create a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install langgraph typing_extensions
python3 simple_LG.py
```

## Summary

This script shows the core building blocks of a LangGraph workflow:

- state definition
- node functions
- conditional routing
- graph construction
- execution with `invoke()`
