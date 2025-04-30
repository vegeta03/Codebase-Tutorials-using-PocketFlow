# Chapter 9: Chapter Writing BatchNode

In the previous chapter, we determined the optimal teaching sequence for our abstractions in [Chapter 8: Chapter Sequencing Node](08_chapter_sequencing_node_.md). Now we turn that ordered list into **fully authored Markdown chapters**—one per abstraction—using the `WriteChapters` BatchNode. This node behaves like a stateful map: it walks the sequence of concepts, composes context‐rich prompts, asks the LLM to generate beginner-friendly content, accumulates summaries for narrative flow, and emits a list of Markdown strings ready to write to disk.

---

## 9.1 Motivation: Why a BatchNode for Chapter Writing?

Central use case:  
> Given  
>
> - An ordered list of abstraction indices (`shared["chapter_order"]`),  
> - Metadata for each abstraction (`name`, `description`, `files`),  
> - The full tutorial structure (all chapter titles/links),  
> - A rolling summary of previous chapters,  
>
> produce **one** self-contained, linked Markdown chapter per concept, complete with:
>
> - Clear headings  
> - Beginner-friendly explanations  
> - Simplified code snippets  
> - Mermaid sequence diagrams  
> - “Next” & “Previous” transition links  

Without a BatchNode, you’d write a monolithic loop that loses track of state between iterations. By subclassing `BatchNode`, we get:

- **Automatic iteration** over prepared items  
- **Instance state** (`self.chapters_written_so_far`) to carry prior summaries  
- **Isolation** of per-chapter logic in `exec(item)`  
- **Post-processing hook** to collate all results into `shared["chapters"]`

---

## 9.2 Core Responsibilities of `WriteChapters`

1. **`prep(shared)`**  
   - Read `chapter_order`, `abstractions`, `files`, and `project_name`  
   - Build a list of chapter-metadata items, each containing:
     - Chapter number & filename  
     - Abstraction name & description  
     - Relevant file snippets  
     - Full chapter listing (for navigation links)  
     - References to previous/next chapters  
2. **`exec(item)`**  
   - Compose a detailed LLM prompt (using `item` + accumulated `self.chapters_written_so_far`)  
   - Call `call_llm(prompt)` to generate Markdown  
   - Prepend or adjust the chapter heading if needed  
   - Append the generated content to `self.chapters_written_so_far`  
   - Return the Markdown string  
3. **`post(shared, prep_res, results)`**  
   - Collect the list of all chapter Markdown strings  
   - Write them into `shared["chapters"]` for the final combiner node  

---

## 9.3 Simplified Code Example

```python
from pocketflow import BatchNode
from utils.call_llm import call_llm

class WriteChapters(BatchNode):
    def prep(self, shared):
        items = []
        order        = shared["chapter_order"]
        abstractions = shared["abstractions"]
        files        = shared["files"]
        project      = shared["project_name"]

        # Initialize per‐instance storage for summaries
        self.chapters_written_so_far = []

        # Build full chapter listing for nav
        listing = []
        filenames = {}
        for idx, abs_idx in enumerate(order, start=1):
            name     = abstractions[abs_idx]["name"]
            safe     = "".join(c if c.isalnum() else "_" for c in name).lower()
            fname    = f"{idx:02d}_{safe}.md"
            listing.append(f"{idx}. [{name}]({fname})")
            filenames[abs_idx] = fname

        full_listing = "\n".join(listing)

        # Prepare each chapter item
        for idx, abs_idx in enumerate(order, start=1):
            data = abstractions[abs_idx]
            files_map = {f"{i} # {p}": c for i,(p,c) in enumerate(files) if i in data["files"]}
            prev_link = None
            next_link = None
            if idx > 1:
                prev = order[idx-2]
                prev_link = filenames[prev]
            if idx < len(order):
                nxt = order[idx]
                next_link = filenames[nxt]

            items.append({
                "chapter_num": idx,
                "name": data["name"],
                "description": data["description"],
                "files_map": files_map,
                "full_listing": full_listing,
                "prev_link": prev_link,
                "next_link": next_link,
                "project": project
            })
        return items

    def exec(self, item):
        # Build prompt using previous summaries
        prev_summary = "\n---\n".join(self.chapters_written_so_far) or "This is the first chapter."
        prompt = f"""
# Chapter {item['chapter_num']}: {item['name']}

Context from previous chapters:
{prev_summary}

Complete Tutorial Structure:
{item['full_listing']}

Concept Details:
- **Name:** {item['name']}
- **Description:** {item['description']}

Relevant Code Snippets:
""" + "\n\n".join(f"```python\n{code}\n```" for code in item["files_map"].values()) + """

Write a beginner-friendly Markdown chapter. Include:
- Transition from previous chapter (use [Previous]({item['prev_link']}) if exists)
- Motivating use case
- Simplified code blocks (<20 lines)
- A minimal Mermaid sequence diagram
- Internal implementation walkthrough
- Link to next: [Next]({item['next_link']})

Provide *only* the Markdown content.
"""
        md = call_llm(prompt)
        # Ensure correct heading
        if not md.startswith(f"# Chapter {item['chapter_num']}"):
            md = f"# Chapter {item['chapter_num']}: {item['name']}\n\n" + md
        self.chapters_written_so_far.append(md)
        return md

    def post(self, shared, prep_res, results):
        shared["chapters"] = results
```

---

## 9.4 Runtime Sequence: Step-by-Step

```mermaid
sequenceDiagram
    participant Flow as TutorialFlow
    participant BC as WriteChapters (BatchNode)
    participant Prep as prep(shared)
    participant Loop as For Each Chapter
    participant Exec as exec(item)
    participant LLM as LLM Interface
    participant State as chapters_written_so_far
    participant Post as post(shared)

    Flow->>BC: pre(shared)
    BC-->>Flow: items[]
    Loop->>BC: exec(item)
    BC->>LLM: call_llm(prompt)
    LLM-->>BC: chapter_md
    BC->>State: append chapter_md
    BC-->>Flow: chapter_md
    alt more chapters
        Loop->>BC: exec(next_item)
        … repeat …
    end
    Flow->>BC: post(shared, results)
    BC-->>Flow: shared["chapters"]
```

---

## 9.5 Analogy: Stateful Map for Document Generation

- **BatchNode** is like a `map()` over chapters, but with **memory**: it carries `chapters_written_so_far` to provide context for smooth narrative flow.
- Each `exec(item)` is a mini-document generator, consuming:
  - Current abstraction details  
  - Full table of contents  
  - Running summary of previous content  
- Then it **emits** a ready-to-write Markdown string, and the pipeline collects them in order.

---

## 9.6 Conclusion & Next Steps

In this chapter, we’ve:

- Explored how `WriteChapters` iterates over abstractions with `prep/exec/post`  
- Shown simplified code to compose per-chapter prompts and accumulate state  
- Visualized the internal sequence with a Mermaid diagram  
- Framed the node as a stateful map that enriches context incrementally  

Up next is **Chapter 10: Tutorial Combiner Node**, where we’ll take the generated Markdown strings and write them—plus an index—to disk, producing your complete tutorial site.

---

Generated by fork of [AI Codebase Knowledge Builder](https://github.com/vegeta03/PocketFlow-Tutorial-Codebase-Knowledge.git)
