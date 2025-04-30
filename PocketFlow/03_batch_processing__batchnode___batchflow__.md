# Chapter 3: Batch Processing (BatchNode & BatchFlow)

In [Chapter 2: Flow](02_flow_.md) we saw how to wire individual Nodes into a directed graph and execute them in sequence. Often, however, you don’t want to run a flow exactly once—you need to process a collection of inputs (e.g., a list of files, document chunks, or prompt variations) without littering your code with loops. Enter **BatchNode** and **BatchFlow**: the map‐style “batch” abstraction built on top of the core Node/Flow model.

---

## 1. Motivation & Central Use Case

Imagine you have a single large Markdown document and you need to translate it into **eight different languages** using an LLM API. A naïve approach:

```python
for lang in languages:
    result = call_llm(prompt_for(lang, text))
    save_translation(lang, result)
```

This works, but:

- You lose retry/fallback logic per item.
- You must manually collect and persist results.
- You miss out on clean observability that PocketFlow gives you.

With **BatchNode**, you simply describe the work for one item (one language) and PocketFlow will loop for you.  
With **BatchFlow**, you can wrap an entire graph (possibly multiple Nodes) and execute it many times with different parameter sets.

In this chapter you’ll learn:

1. How to write a **BatchNode** that takes a list of items and “maps” your single‐item logic across them.  
2. How to use **BatchFlow** to run a full Flow multiple times with varied parameters, aggregating results at the end.  

By the end, you’ll have a reusable translation pipeline that requires zero manual loops.

---

## 2. Key Concepts

### 2.1 BatchNode

A `BatchNode` is just like a `Node`, except its `exec` sees a **list** of items. Internally, for each element it invokes the standard retry‐aware logic of `Node.exec`, and collects all of the results into a list.

Core behavior (in `pocketflow/__init__.py`):

```python
class BatchNode(Node):
    def _exec(self, items):
        # items: list of prep_res values
        results = []
        for item in (items or []):
            # Call Node._exec (with retries/fallback) for each element
            res = super(BatchNode, self)._exec(item)
            results.append(res)
        return results
```

#### Example: Translating Text into Multiple Languages

```python
# file: translate_nodes.py
import os
from pocketflow import BatchNode
from utils import call_llm  # your LLM API wrapper

class TranslateTextNode(BatchNode):
    def prep(self, shared):
        text = shared["text"]
        langs = shared.get("languages",
            ["Chinese", "Spanish", "Japanese", "German",
             "Russian", "Portuguese", "French", "Korean"])
        # Return one (text, language) tuple per target
        return [(text, lang) for lang in langs]

    def exec(self, data_tuple):
        text, lang = data_tuple
        prompt = f"""
Translate the following markdown into {lang}, preserving all formatting:

Original:
{text}

Translated:"""
        translation = call_llm(prompt)
        print(f"[BatchNode] Translated into {lang}")
        return {"language": lang, "translation": translation}

    def post(self, shared, prep_res, exec_res_list):
        out_dir = shared.get("output_dir", "translations")
        os.makedirs(out_dir, exist_ok=True)
        for item in exec_res_list:
            fn = os.path.join(out_dir, f"README_{item['language']}.md")
            with open(fn, "w", encoding="utf-8") as f:
                f.write(item["translation"])
        print(f"[BatchNode] Saved {len(exec_res_list)} translations to '{out_dir}'")
        # No branching from here; end of graph
        return "default"
```

Explanation:

1. `prep` returns a list of `(text, language)` pairs.  
2. `BatchNode._exec` calls your `exec` for each pair, handling retries.  
3. `post` writes out all translations and signals `"default"`.  

You can wire this directly into a `Flow`:

```python
from pocketflow import Flow
from translate_nodes import TranslateTextNode

shared = {
    "text": open("README.md").read(),
    "languages": ["Spanish", "French"],
    "output_dir": "out"
}

flow = Flow(start=TranslateTextNode(max_retries=2))
flow.run(shared)
```

That’s it—no manual loops!

---

### 2.2 BatchFlow

A `BatchFlow` elevates the “batch” idea to an entire graph. Instead of a single Node, you wrap a complete `Flow` so it runs **once per parameter‐dict**.

Core stages:

1. **prep(shared)** → iterable of parameter dictionaries (one per invocation).  
2. For each `batch_params`:
   - Merge `self.params` with `batch_params`.  
   - Run the underlying Flow orchestration (`_orch(shared, merged_params)`).
3. **post(shared, prep_res, None)** → final aggregation or summary.

Core implementation (in `pocketflow/__init__.py`):

```python
class BatchFlow(Flow):
    def _run(self, shared):
        batch_params = self.prep(shared) or []
        for bp in batch_params:
            # Merge Flow‐level params with this batch's params
            merged = {**self.params, **bp}
            # Run the core orchestrator on each parameter set
            self._orch(shared, merged)
        # Final hook for aggregation; exec_res is always None here
        return self.post(shared, batch_params, None)
```

#### Example: Converting Multiple Documents

Suppose you have a conversion Node that turns one file from Markdown → HTML:

```python
# file: convert_nodes.py
class MarkdownToHtmlNode(Node):
    def prep(self, shared):
        # Expect 'input_path' in shared params
        return shared["input_path"]

    def exec(self, path):
        with open(path, "r", encoding="utf-8") as f:
            md = f.read()
        # Imagine a real conversion here...
        html = markdown_to_html(md)
        return html

    def post(self, shared, prep_res, html):
        out_dir = shared["output_dir"]
        fname = os.path.basename(prep_res).replace(".md", ".html")
        with open(os.path.join(out_dir, fname), "w", encoding="utf-8") as f:
            f.write(html)
        print(f"[Flow] Converted {prep_res} → {fname}")
        return "converted"
```

You can wrap this in a `BatchFlow` that scans an input directory and converts all `.md` files:

```python
# file: convert_flow.py
import os
from pocketflow import BatchFlow
from convert_nodes import MarkdownToHtmlNode

class MultiDocConvertFlow(BatchFlow):
    def __init__(self, input_dir, output_dir):
        # The Flow we batch over: single‐step conversion
        super().__init__(start=MarkdownToHtmlNode())
        self.input_dir = input_dir
        self.output_dir = output_dir

    def prep(self, shared):
        # List all markdown files and pass each path to one Flow run
        files = [
            {"input_path": os.path.join(self.input_dir, fn)}
            for fn in os.listdir(self.input_dir)
            if fn.endswith(".md")
        ]
        # Ensure the output directory is known
        shared["output_dir"] = self.output_dir
        return files

    def post(self, shared, prep_res, exec_res):
        count = len(prep_res)
        print(f"[BatchFlow] Completed conversion of {count} files.")
        return "all_done"
```

Run it:

```python
shared = {}
flow = MultiDocConvertFlow(input_dir="docs", output_dir="html_out")
final = flow.run(shared)
# Prints conversion per file, then summary.
```

---

## 3. Internal Execution Walkthrough

Let’s trace what happens when you call `BatchFlow.run(shared)`.

```mermaid
sequenceDiagram
  participant U  as User Code
  participant BF as BatchFlow
  participant P  as BF.prep()
  participant LOOP as ForEach Batch
  participant OR as _orch()  Note right of OR: runs Flow once
  participant N  as MarkdownToHtmlNode
  participant S  as Shared State
  participant PF as BF.post()

  U->>BF: run(shared)
  BF->>P: prep(shared)
  P-->>BF: returns [ {'input_path':f1}, {'input_path':f2}, ... ]
  loop for each params dict
    BF->>LOOP: _orch(shared, merged_params)
    LOOP->>N: clone & set_params(params)
    N->>S: prep(shared)
    N->>N: exec(prep_res)
    N->>S: post(shared, exec_res)
    N-->>LOOP: returns action
    LOOP-->>BF: orchestration done
  end
  BF->>PF: post(shared, prep_res, None)
  PF-->>U: return "all_done"
```

1. **BatchFlow.prep** builds a list of parameter maps.  
2. For each map:
   - **_orch** clones the start node, sets its params, and runs the usual Flow loop over all successor Nodes.  
3. After all runs, **BatchFlow.post** gets the full list of prep results and can summarize or aggregate.  

---

## 4. Under‐the‐Hood: Code Snippets

File: **`pocketflow/__init__.py`**

```python
class BatchNode(Node):
    def _exec(self, items):
        # items: list of per‐element prep_res
        return [super(BatchNode, self)._exec(i) for i in (items or [])]
```

- Uses Python list comprehension to map each item through the retry‐wrapped `Node.exec`.  

```python
class BatchFlow(Flow):
    def _run(self, shared):
        # 1) Gather all parameter sets
        param_list = self.prep(shared) or []
        # 2) Run the core Flow for each params dict
        for bp in param_list:
            merged = {**self.params, **bp}
            self._orch(shared, merged)
        # 3) Final aggregation hook
        return self.post(shared, param_list, None)
```

- Inherits the Flow archetype: cloning nodes, handling successors, parameter propagation, retries, etc.  
- `BatchFlow` overloads only `_run`, leaving `prep` and `post` to your subclass.  

---

## 5. Analogy & Insights

- **Map in MapReduce**:  
  - **BatchNode** = your **Mapper** function applied to a list of inputs.  
  - **BatchFlow** = orchestrating your entire top‐ology once per input‐bundle, like running multiple maps in sequence.

- **Assembly‐line supervisor**:  
  - **BatchFlow.prep** = dispatch instructions to each worker (parameter sets).  
  - **_orch** = the conveyor belt running one batch through all stations (Nodes).  
  - **BatchFlow.post** = gathering final products and summarizing yield.

---

## 6. Conclusion & Next Steps

In this chapter, you learned how to:

- Use **BatchNode** to apply a single‐item `exec` across a list without manual loops.  
- Build a **BatchFlow** around an existing `Flow` to run it multiple times with different parameters.  
- Hook into `prep` and `post` for clean setup and aggregation.

These abstractions let you scale bulk tasks—LLM calls, file conversions, data transformations—while keeping retry, fallback, and observability intact.

Next up: handling asynchronous workloads and concurrency with **[Asynchronous Nodes & Flow (AsyncNode & AsyncFlow)](04_asynchronous_nodes___flow__asyncnode___asyncflow__.md)**.

---

Generated by fork of [AI Codebase Knowledge Builder](https://github.com/vegeta03/PocketFlow-Tutorial-Codebase-Knowledge.git)
