# Chapter 4: Asynchronous Nodes & Flow (AsyncNode & AsyncFlow)

In the previous chapter, we explored [Batch Processing (BatchNode & BatchFlow)](03_batch_processing__batchnode___batchflow__.md) for map-style loops over synchronous pipelines. In modern applications, however, you often need to interleave non-blocking I/O—HTTP calls, LLM streaming, user prompts—without tying up threads. Enter **AsyncNode** and **AsyncFlow**, PocketFlow’s bridge into Python’s `asyncio` world.

---

## 1. Motivation & Central Use Case

Imagine you’re building an interactive **recipe suggestion pipeline**:

1. **Fetch** a list of recipes from a remote HTTP API.
2. **Generate** a highlighted suggestion using an LLM (streaming back tokens).
3. **Prompt** the user asynchronously to accept or reject the suggestion.
4. **Loop** until the user approves or exhausts retries.
5. **Finish** cleanly.

With synchronous Nodes, any network or I/O call would block the thread, preventing you from running multiple pipelines or handling other tasks. **AsyncNode** splits each Node’s lifecycle into `prep_async` → `exec_async` → `post_async`, with built-in retry and fallback, all scheduled by `asyncio` so your event loop stays responsive.

By the end of this chapter, you’ll be able to:

- Write `AsyncNode` classes for HTTP, LLM, and user-prompting steps.
- Wire them into an `AsyncFlow` that interleaves tasks without blocking.
- Understand how PocketFlow implements coroutine scheduling under the hood.

---

## 2. Key Concepts

### 2.1 AsyncNode Lifecycle

An **AsyncNode** mirrors `Node` but uses `async` methods:

1. `async def prep_async(self, shared: dict) -> Any`  
   Prepare inputs (e.g., extract shared parameters).

2. `async def exec_async(self, prep_res: Any) -> Any`  
   Perform the core async work (HTTP calls, streaming, etc.).

3. `async def exec_fallback_async(self, prep_res: Any, exc: Exception) -> Any`  
   Final handler on repeated failures (default: re-raise).

4. `async def post_async(self, shared: dict, prep_res: Any, exec_res: Any) -> str`  
   Write back to `shared` and return an action label.

Behind the scenes, `AsyncNode._exec` loops up to `max_retries`, awaiting `exec_async` and `asyncio.sleep(self.wait)` between retries, then calling `exec_fallback_async` if all attempts fail.

### 2.2 AsyncFlow Orchestration

**AsyncFlow** extends the synchronous `Flow` to:

- **Await** each Node’s `run_async` (or synchronous `run` for classic `Node`).
- **Clone** nodes with `copy.copy` to reset transient state (retry counters).
- **Interleave** execution across the event loop, analogous to how an `asyncio` scheduler or Trio nursery yields control.

It provides:

- `async def run_async(self, shared: dict) -> str`  
  Entry point for your async pipeline.

- `async def prep_async(self, shared)` and `async def post_async(self, shared, prep_res, exec_res)` hooks at the flow level.

### 2.3 Mixing Sync & Async

`AsyncFlow` seamlessly handles a mix of `AsyncNode` and classic `Node`:

- If a node is an `AsyncNode`, `AsyncFlow` awaits its `_run_async`.  
- Otherwise, it calls the synchronous `_run`.

This lets you reuse existing `Node` implementations in an async pipeline.

---

## 3. Building the Recipe Suggestion Pipeline

Let’s implement the use case step by step. We’ll define three `AsyncNode` subclasses plus a no-op end node, wire them into an `AsyncFlow`, then run it.

### 3.1 FetchRecipes (HTTP call)

```python
# file: recipe_nodes.py
import aiohttp
import asyncio
from pocketflow import AsyncNode

class FetchRecipes(AsyncNode):
    def __init__(self, max_retries=3, wait=1):
        super().__init__(max_retries=max_retries, wait=wait)

    async def prep_async(self, shared: dict) -> str:
        # Choose a cuisine; default to 'italian'
        return shared.get("cuisine", "italian")

    async def exec_async(self, cuisine: str) -> list:
        url = f"https://api.recipes.example.com/search?cuisine={cuisine}"
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                resp.raise_for_status()
                data = await resp.json()
        return data.get("recipes", [])

    async def post_async(self, shared: dict, cuisine: str, recipes: list) -> str:
        shared["recipes"] = recipes
        print(f"[FetchRecipes] Fetched {len(recipes)} recipes for '{cuisine}'.")
        return "suggest"
```

Explanation:

- `prep_async` reads the cuisine from shared state.
- `exec_async` performs an HTTP GET using `aiohttp`.
- `post_async` stores the list in `shared["recipes"]` and returns the action `"suggest"`.

### 3.2 SuggestRecipe (LLM Streaming)

```python
# file: recipe_nodes.py
class SuggestRecipe(AsyncNode):
    def __init__(self, max_retries=2, wait=0.5):
        super().__init__(max_retries=max_retries, wait=wait)

    async def prep_async(self, shared: dict) -> dict:
        # Pass all recipes to the LLM prompt
        return {"recipes": shared["recipes"]}

    async def exec_async(self, prep_res: dict) -> dict:
        # Simulate a streaming LLM call (here we just pick the first recipe)
        await asyncio.sleep(0.2)  # pretend we're streaming tokens
        suggestion = prep_res["recipes"][0]
        return suggestion

    async def post_async(self, shared: dict, prep_res: dict, suggestion: dict) -> str:
        shared["suggestion"] = suggestion
        print(f"[SuggestRecipe] Suggested: {suggestion['title']}")
        return "approve"
```

Explanation:

- We simulate token streaming with `asyncio.sleep`.
- The first recipe is chosen as our “suggestion”.

### 3.3 GetApproval (Async User Prompt)

```python
# file: recipe_nodes.py
class GetApproval(AsyncNode):
    async def prep_async(self, shared: dict) -> dict:
        return shared["suggestion"]

    async def exec_async(self, suggestion: dict) -> bool:
        prompt = f"Do you like '{suggestion['title']}'? (y/n): "
        loop = asyncio.get_event_loop()
        # Run blocking input() in a threadpool to avoid blocking the event loop
        answer = await loop.run_in_executor(None, input, prompt)
        return answer.strip().lower() == "y"

    async def post_async(self, shared: dict, suggestion: dict, approved: bool) -> str:
        if approved:
            print("[GetApproval] User approved.")
            return "accept"
        else:
            print("[GetApproval] User rejected; retrying suggestion.")
            return "retry"
```

Explanation:

- We offload `input()` to a threadpool via `run_in_executor`.
- The node returns `"accept"` or `"retry"` to drive branching.

### 3.4 NoOp (Pipeline Terminator)

```python
# file: recipe_nodes.py
class NoOp(AsyncNode):
    async def prep_async(self, shared: dict):
        pass

    async def exec_async(self, _: None):
        return None

    async def post_async(self, shared: dict, prep_res: None, exec_res: None) -> str:
        print("[NoOp] Pipeline complete.")
        return "default"
```

### 3.5 Wiring & Running the AsyncFlow

```python
# file: recipe_flow.py
import asyncio
from pocketflow import AsyncFlow
from recipe_nodes import FetchRecipes, SuggestRecipe, GetApproval, NoOp

def create_recipe_flow() -> AsyncFlow:
    fetch   = FetchRecipes()
    suggest = SuggestRecipe()
    approve = GetApproval()
    end     = NoOp()

    # Connect transitions
    fetch   - "suggest" >> suggest
    suggest - "approve" >> approve
    approve - "retry"   >> suggest
    approve - "accept"  >> end

    # Build the flow
    flow = AsyncFlow(start=fetch)
    return flow

async def main():
    shared = {"cuisine": "thai"}   # initial shared state
    flow   = create_recipe_flow()
    final  = await flow.run_async(shared)
    print("Final action:", final)
    print("Shared state:", shared)

if __name__ == "__main__":
    asyncio.run(main())
```

What happens when you run this:

- `FetchRecipes` performs a non-blocking HTTP fetch.
- `SuggestRecipe` “streams” tokens asynchronously.
- `GetApproval` prompts the user without blocking other coroutines.
- On `"retry"`, you loop back to `SuggestRecipe`.
- On `"accept"`, you hit the `NoOp` terminator and exit.

---

## 4. Internal Orchestration Walkthrough

Below is a high-level sequence diagram of `await flow.run_async(shared)`:

```mermaid
sequenceDiagram
  participant U    as Your Code
  participant AF   as AsyncFlow
  participant F    as FetchRecipes
  participant S    as Shared State
  participant SR   as SuggestRecipe
  participant GA   as GetApproval
  participant End  as NoOp

  U->>AF: run_async(shared)
  AF->>AF: prep_async(shared)
  AF->>F:   clone & set_params
  F->>S:    prep_async(shared)
  F->>F:    exec_async(cuisine)
  F->>S:    post_async(shared,recipes)
  F-->>AF:  action="suggest"
  AF->>SR:  clone & set_params
  SR->>S:   prep_async(shared)
  SR->>SR:  exec_async(recipes)
  SR->>S:   post_async(shared,suggestion)
  SR-->>AF: action="approve"
  AF->>GA:  clone & set_params
  GA->>S:   prep_async(shared)
  GA->>GA:  exec_async(suggestion)
  GA->>S:   post_async(shared,approved)
  GA-->>AF: action="retry" or "accept"
  alt retry
    AF->>SR: clone & set_params  (loop)
  else accept
    AF->>End: clone & set_params
    End->>End: exec_async()
    End->>S:   post_async()
    End-->>AF: action="default"
  end
  AF->>AF: post_async(shared,_,last_action)
  AF-->>U:  return last_action
```

### 4.1 Core Implementation Snippets

**AsyncNode internals** (in `pocketflow/__init__.py`):

```python
class AsyncNode(Node):
    async def prep_async(self, shared):           pass
    async def exec_async(self, prep_res):         pass
    async def exec_fallback_async(self, prep_res, exc): raise exc
    async def post_async(self, shared, prep_res, exec_res): pass

    async def _exec(self, prep_res):
        for i in range(self.max_retries):
            try:
                return await self.exec_async(prep_res)
            except Exception as e:
                if i == self.max_retries - 1:
                    return await self.exec_fallback_async(prep_res, e)
                if self.wait > 0:
                    await asyncio.sleep(self.wait)

    async def _run_async(self, shared):
        p = await self.prep_async(shared)
        e = await self._exec(p)
        return await self.post_async(shared, p, e)

    async def run_async(self, shared):
        if self.successors:
            warnings.warn("Node won't run successors. Use AsyncFlow.")
        return await self._run_async(shared)
```

- **Retry logic** parallels `Node._exec`, but uses `await asyncio.sleep`.
- **`run_async`** enforces that only `AsyncFlow` will handle transitions.

**AsyncFlow orchestration**:

```python
class AsyncFlow(Flow, AsyncNode):
    async def _orch_async(self, shared, params=None):
        curr = copy.copy(self.start_node)
        p    = params or {**self.params}
        last = None

        while curr:
            curr.set_params(p)
            if isinstance(curr, AsyncNode):
                last = await curr._run_async(shared)
            else:
                last = curr._run(shared)
            curr = copy.copy(self.get_next_node(curr, last))
        return last

    async def _run_async(self, shared):
        prep_res = await self.prep_async(shared)
        exec_res = await self._orch_async(shared)
        return await self.post_async(shared, prep_res, exec_res)

    async def run_async(self, shared):
        return await self._run_async(shared)
```

- Clones each node to reset retry counters.
- Checks node type and awaits appropriately.
- Transitions based on `successors[action]`.

---

## 5. Analogies & Insights

- **AsyncWorkstation on a Conveyor**  
  Each **AsyncNode** is like a robotic station that fetches parts (prep), works on them (exec), and then tags the product (post)—all without stopping the conveyor. The **AsyncFlow** is the factory controller that triggers each station in sequence but lets other factories run when one station is waiting on I/O.

- **Event-Loop Packet Router**  
  Shared state is the packet. Each Node tags the packet with a “route” (the action). The AsyncFlow router reads tags, forwards packets, and yields control whenever a Node awaits, allowing maximum overlap.

---

## 6. Conclusion & Next Steps

In this chapter you learned how to:

- Define **AsyncNode** subclasses with `prep_async` → `exec_async` → `post_async`.  
- Handle retry and fallback in asynchronous contexts.  
- Orchestrate an entire graph of async (and sync) nodes using **AsyncFlow**.  
- Keep your event loop responsive when performing HTTP calls, LLM streaming, or user interaction.

Next up, we’ll combine batching with asynchronous execution in [Parallel Batch Processing (AsyncParallelBatchNode & AsyncParallelBatchFlow)](05_parallel_batch_processing__asyncparallelbatchnode___asyncparallelbatchflow__.md). Enjoy non-blocking pipelines!

---

Generated by fork of [AI Codebase Knowledge Builder](https://github.com/vegeta03/PocketFlow-Tutorial-Codebase-Knowledge.git)
