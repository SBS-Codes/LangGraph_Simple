# LangGraph API Templates

This reference collects reusable templates for the LangGraph APIs used in
[`simple_LG.py`](./simple_LG.py) and [`email_LG.py`](./email_LG.py). The
examples use Python functions as graph nodes and a `TypedDict` as the shared
state.

## Imports

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
```

`TypedDict` describes the keys and value types in the graph state. `StateGraph`
builds the workflow. `START` and `END` are special graph boundary markers.

## Define the graph state

```python
class State(TypedDict):
    question: str
    answer: str
```

State fields are available to each node. A node usually returns a dictionary
containing only the fields it changes; LangGraph merges those updates into the
shared state.

## Define a node

```python
def answer_question(state: State) -> dict:
    question = state["question"]
    answer = f"You asked: {question}"
    return {"answer": answer}
```

A node is a Python function registered with the graph. It receives the current
state and returns its updates. A node that only performs a side effect, such as
logging, can return an empty dictionary:

```python
def log_question(state: State) -> dict:
    print(f"Received: {state['question']}")
    return {}
```

## Create a graph and add nodes

```python
builder = StateGraph(State)
builder.add_node("log_question", log_question)
builder.add_node("answer_question", answer_question)
```

`StateGraph(State)` creates a graph builder tied to the state schema.
`add_node(name, function)` registers a node under a name. The name is used
when connecting graph edges.

## Add a direct edge

```python
builder.add_edge(START, "log_question")
builder.add_edge("log_question", "answer_question")
builder.add_edge("answer_question", END)
```

`add_edge(source, destination)` connects one node to the next. Use `START` for
the graph entry and `END` for a terminal node. A graph may have multiple paths
that each end at `END`.

## Add conditional routing

Use a routing function to choose a destination based on the current state:

```python
def choose_path(state: State) -> Literal["answer_question", "finish"]:
    if state["question"]:
        return "answer_question"
    return "finish"
```

Register the possible routes with `add_conditional_edges`:

```python
builder.add_conditional_edges(
    "log_question",
    choose_path,
    {
        "answer_question": "answer_question",
        "finish": END,
    },
)
```

The routing function's return value is a route key. The optional mapping
connects each key to a graph node (or `END`). Without a mapping, the function
can return a node name directly. Every possible returned route should lead to a
valid destination.

The email workflow uses this pattern to choose between spam handling and
response drafting:

```python
def route_email(state: EmailState) -> str:
    return "spam" if state["is_spam"] else "legitimate"

email_graph.add_conditional_edges(
    "classify_email",
    route_email,
    {
        "spam": "handle_spam",
        "legitimate": "draft_response",
    },
)
```

## Compile and invoke a graph

```python
graph = builder.compile()

result = graph.invoke({
    "question": "What is LangGraph?",
    "answer": "",
})

print(result)
```

`compile()` validates and builds the graph into a runnable workflow.
`invoke(initial_state)` executes it synchronously and returns the final state.
The initial state should include the required state fields that are not
otherwise given defaults.

## Complete minimal template

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    message: str


def add_greeting(state: State) -> dict:
    return {"message": f"Hello, {state['message']}!"}


builder = StateGraph(State)
builder.add_node("add_greeting", add_greeting)
builder.add_edge(START, "add_greeting")
builder.add_edge("add_greeting", END)

graph = builder.compile()
result = graph.invoke({"message": "LangGraph"})
print(result)
```

Expected output:

```text
{'message': 'Hello, LangGraph!'}
```

## Draw a graph diagram

The simple example writes a Mermaid-rendered graph image to a PNG file:

```python
with open("graph.png", "wb") as image_file:
    image_file.write(graph.get_graph().draw_mermaid_png())
```

`get_graph()` returns a drawable representation of the compiled graph, and
`draw_mermaid_png()` renders it as PNG bytes. Rendering may require the
environment's graph-rendering dependencies.
