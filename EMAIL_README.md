# Email Processing Workflow

This guide explains the logic in [`email_LG.py`](./email_LG.py), a small
LangGraph workflow that classifies emails with a local Llama model through
Ollama. Spam is sent down a handling path; legitimate email is categorized,
used to draft a response, and presented for review.

## Requirements and running

Install the Python packages used by the email workflow:

```bash
python3 -m pip install langgraph langchain-ollama
```

Make sure Ollama is running and has a local model installed. The script uses
`llama:latest` by default. To select a different local model, set
`OLLAMA_MODEL`:

```bash
OLLAMA_MODEL=llama3.2-local:latest python3 email_LG.py
```

Then run the workflow:

```bash
python3 email_LG.py
```

## Inputs and state

Each call to `compiled_graph.invoke()` receives an initial state. The email
itself is a dictionary with three required fields:

```python
{
    "sender": "john.smith@example.com",
    "subject": "Question about your services",
    "body": (
        "Dear Mr. Hugg, I was referred to you by a colleague and I'm "
        "interested in learning more about your consulting services. "
        "Could we schedule a call next week? Best regards, John Smith"
    ),
}
```

The complete initial state also supplies the workflow fields that nodes fill
in or extend:

```python
{
    "email": email,            # sender, subject, and body
    "is_spam": None,           # set by classify_email
    "spam_reason": None,       # set for spam
    "email_category": None,    # set for legitimate email
    "email_draft": None,       # set for legitimate email
    "messages": [],            # prompts and model responses for this run
}
```

`EmailState` is a `TypedDict` that describes these shared fields. Each graph
node receives the current state and returns a dictionary containing only its
updates. LangGraph merges those updates into the state for subsequent nodes.

## Classification and routing logic

The model is initialized with `ChatOllama`, using the `OLLAMA_MODEL`
environment variable or `llama:latest`. `with_structured_output()` constrains
classification results to the `EmailClassification` Pydantic schema:

- `classification`: `SPAM` or `LEGITIMATE`
- `category`: a supported email category, or `N/A`
- `reason`: a short explanation

This avoids guessing the decision from words in free-form model prose. The
classification prompt asks the model to treat the email body as data, and
classify unsolicited bulk email, phishing, scams, or malicious content as spam.
Personal service inquiries, referrals, and requests for a call are described as
legitimate examples.

## Graph functions

| Function or API | Purpose |
| --- | --- |
| `read_email` | Logs the sender and subject. It does not change the state. |
| `classify_email` | Calls the structured local model, sets `is_spam`, and stores the classification prompt and response in `messages`. |
| `route_email` | Chooses the `spam` or `legitimate` branch based on `is_spam`. |
| `handle_spam` | Prints the spam reason and a handling message. |
| `draft_response` | Uses the local model to draft a professional reply for a legitimate email and records the interaction in `messages`. |
| `notify_mr_hugg` | Prints the email category and draft for review. |
| `StateGraph(EmailState)` | Creates a graph whose nodes share the `EmailState` structure. |
| `add_node(name, function)` | Registers a function as a graph node. |
| `add_edge(source, destination)` | Connects nodes for the fixed workflow sequence. `START` and `END` mark graph entry and exits. |
| `add_conditional_edges(...)` | Uses `route_email` after classification to select the spam or legitimate path. |
| `compile()` | Builds the executable graph from the node and edge definitions. |
| `invoke(initial_state)` | Runs the compiled graph synchronously and returns the final state. |

The workflow is wired as:

```text
START -> read_email -> classify_email
                           |             \
                      spam |              | legitimate
                           v              v
                     handle_spam     draft_response -> notify_mr_hugg
                           |              |                   |
                           v              v                   v
                          END            END                 END
```

## Example input emails

The script runs once with a legitimate inquiry:

```python
{
    "sender": "john.smith@example.com",
    "subject": "Question about your services",
    "body": "I was referred by a colleague. Could we schedule a call next week?",
}
```

It also runs with a spam example:

```python
{
    "sender": "winner@lottery-intl.com",
    "subject": "YOU HAVE WON $5,000,000!!!",
    "body": (
        "You have won our lottery. Send your bank details and a "
        "processing fee of $100 to claim your prize."
    ),
}
```

## Sample output

Model-generated wording and spam reasons can vary. A successful run follows
these paths and produces output similar to:

```text
Processing legitimate email...
Alfred is processing an email from john.smith@example.com with subject: Question about your services

Sir, you've received an email from john.smith@example.com.
Subject: Question about your services
Category: inquiry

I've prepared a draft response for your review:
--------------------------------------------------
Dear Mr. Smith,

Thank you for your interest and for the referral. I'd be happy to discuss my
consulting services. Please let me know what times work for a call next week.

Best regards,
Alfred Hugg
==================================================

Processing spam email...
Alfred is processing an email from winner@lottery-intl.com with subject: YOU HAVE WON $5,000,000!!!
Alfred has marked the email as spam. Reason: Unsolicited lottery message requesting financial information and a fee.
The email has been moved to the spam folder.
```

## Credits

This example is for learning LangGraph concepts and acknowledges Hugging Face
as an educational inspiration for the project. LangGraph is a library from
LangChain; this repository's workflow is a local example, not an official
Hugging Face product.
