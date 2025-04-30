# Chapter 2: Dependency Graph Representation

Welcome back! In [Chapter 1: Agent Execution Loop](01_agent_execution_loop_.md), we learned about the "engine" of LocAgent, the **Agent Execution Loop**, which manages the investigation process. We saw how the Agent (our LLM brain) decides what to do next and uses Tools to interact with the codebase.

But how does the Agent actually "see" and understand the codebase it's investigating? If the project has hundreds or thousands of files, reading every single line of code would be incredibly slow and inefficient. Imagine trying to find a specific shop in a huge city by visiting every single building! You'd want a map instead.

That's exactly what the **Dependency Graph Representation** provides for LocAgent – it's the detailed map of the software city.

## What is a Dependency Graph?

Think of a large software project like a bustling city. This city has:

*   **Districts:** These are the main folders or directories in your project.
*   **Buildings:** These represent individual code files (like `user_profile.py` or `api_handler.js`).
*   **Rooms:** Inside buildings, you have specific rooms like classes (`UserProfile`, `ApiHandler`) and offices (`save_profile_data()`, `fetch_user_info()`).

Now, how do things connect in this city?

*   One district might **contain** several buildings (a directory contains files).
*   A building **contains** many rooms (a file contains classes and functions).
*   A room inside one building might need instructions or tools from another building (File A **imports** File B).
*   An activity in one office might trigger an activity in another (Function X **calls** Function Y).
*   Sometimes, a new building's design is heavily based on an existing one (Class B **inherits** from Class A).

The **Dependency Graph** is the master map that shows all these districts, buildings, and rooms (we call them **Nodes**) and all the roads and corridors connecting them (we call these **Edges**).

*   **Nodes:** Represent code entities.
    *   `Directory`: Like a city district.
    *   `File`: Like a building.
    *   `Class`: Like a large room or suite within a building.
    *   `Function` (or `Method`): Like a specific office or workspace within a building or suite.
*   **Edges:** Represent relationships between code entities.
    *   `Contains`: Shows what's inside what (e.g., `src` directory -> `contains` -> `utils.py` file; `utils.py` file -> `contains` -> `helper_function()`).
    *   `Imports`: Shows when one file uses code from another (e.g., `main.py` -> `imports` -> `utils.py`).
    *   `Invokes` (or `Calls`): Shows when one function/method uses another (e.g., `main_function()` -> `invokes` -> `helper_function()`).
    *   `Inherits`: Shows when one class is based on another (e.g., `AdminUser` class -> `inherits` -> `BaseUser` class).

This graph gives the Agent a bird's-eye view of the entire project's structure and how different pieces connect *without* needing to read all the underlying code initially. It's a much faster way to navigate and understand the codebase.

## Why is this Graph Useful?

Imagine our bug report again: "The 'Save' button doesn't work on the profile page."

Instead of reading thousands of lines of code, the Agent can use the Dependency Graph like a GPS:

1.  **Find Landmarks:** The Agent might ask the graph: "Show me all nodes (buildings/rooms) with 'profile' or 'save' in their name." The graph quickly provides a list (e.g., `profile_view.py`, `save_handler.py`, `UserProfile` class, `handle_save()` function).
2.  **Follow Connections:** The Agent sees `handle_save()` function. It can ask the graph: "What other functions call (`invokes`) `handle_save()`?" or "What functions does `handle_save()` call?" This helps trace the flow of actions related to saving.
3.  **Understand Structure:** The Agent can see that `handle_save()` is `contained` within the `UserProfile` class, which is `contained` within `profile_view.py`. This gives context.
4.  **Targeted Reading:** Only when the Agent narrows down the possibilities using the graph does it need to actually read the code snippets associated with the most promising nodes (like the code for the `handle_save()` function).

This graph allows for efficient exploration, much like using a map to find your way in a city instead of wandering aimlessly. The process of exploring the graph is covered in detail in [Chapter 6: Graph Search & Traversal](06_graph_search___traversal_.md).

## A Simple Example Graph

Let's say we have two simple Python files:

**`calculator.py`:**
```python
# calculator.py
def add(a, b):
  return a + b

def subtract(a, b):
  return a - b
```

**`main.py`:**
```python
# main.py
import calculator # Import relationship

def run_calculation():
  result = calculator.add(5, 3) # Invokes relationship
  print(f"The result is: {result}")

run_calculation()
```

A simplified Dependency Graph for this might look like this:

```mermaid
graph TD
    subgraph Project Root (/)
        N1["/"] -- contains --> N2["calculator.py (File)"];
        N1["/"] -- contains --> N3["main.py (File)"];
    end

    subgraph calculator.py
        N2 -- contains --> F1["add() (Function)"];
        N2 -- contains --> F2["subtract() (Function)"];
    end

    subgraph main.py
        N3 -- contains --> F3["run_calculation() (Function)"];
        N3 -- imports --> N2;
    end

    F3 -- invokes --> F1;

    classDef node fill:#f9f,stroke:#333,stroke-width:2px;
    classDef function fill:#ccf,stroke:#333,stroke-width:2px;
    classDef file fill:#eee,stroke:#333,stroke-width:2px;
    classDef directory fill:#fdb,stroke:#333,stroke-width:2px;

    class N1 directory;
    class N2,N3 file;
    class F1,F2,F3 function;
```

This diagram shows:
*   The root directory (`/`) contains two files.
*   `calculator.py` contains two functions (`add`, `subtract`).
*   `main.py` contains one function (`run_calculation`).
*   `main.py` imports `calculator.py`.
*   `run_calculation()` invokes `calculator.add()`.

The Agent can query this structure to understand these relationships quickly.

## Looking Under the Hood

How is this graph built? We won't dive deep here (that's for [Chapter 3: Graph Building Process](03_graph_building_process_.md)), but essentially, LocAgent:

1.  **Scans the Code:** It looks at all the Python files in the project.
2.  **Parses the Syntax:** It uses tools (like Python's built-in `ast` module) to understand the structure of each file – identifying classes, functions, import statements, and function calls without actually running the code. This is like reading the blueprints of each building.
3.  **Builds the Graph:** It creates nodes for each identified entity (directory, file, class, function) and adds edges based on the relationships found during parsing (contains, imports, invokes, inherits).

The code that defines the types of nodes and edges we use lives in `dependency_graph/build_graph.py`. These act like the legend for our city map.

```python
# --- Simplified from dependency_graph/build_graph.py ---

# Define the types of "places" on our map
NODE_TYPE_DIRECTORY = 'directory'
NODE_TYPE_FILE = 'file'
NODE_TYPE_CLASS = 'class'
NODE_TYPE_FUNCTION = 'function'

# Define the types of "roads" connecting places
EDGE_TYPE_CONTAINS = 'contains' # e.g., File -> Function
EDGE_TYPE_INHERITS = 'inherits' # e.g., Class -> Class
EDGE_TYPE_INVOKES = 'invokes'   # e.g., Function -> Function
EDGE_TYPE_IMPORTS = 'imports'   # e.g., File -> File or File -> Class/Function
```
These simple strings define the categories used within the graph structure.

When the graph is built, operations conceptually similar to this happen (using the `networkx` library):

```python
# --- Conceptual Example ---
import networkx as nx

# Create an empty graph G = nx.MultiDiGraph()
# ... (scan code) ...

# Add a node for a file
G.add_node("main.py", type=NODE_TYPE_FILE, code="...")

# Add a node for a function inside that file
G.add_node("main.py:run_calculation", type=NODE_TYPE_FUNCTION, code="...", start_line=4, end_line=6)

# Add an edge showing the file contains the function
G.add_edge("main.py", "main.py:run_calculation", type=EDGE_TYPE_CONTAINS)

# ... (add nodes and edges for calculator.py) ...

# Add an edge showing main.py imports calculator.py
G.add_edge("main.py", "calculator.py", type=EDGE_TYPE_IMPORTS)

# Add an edge showing run_calculation invokes add
G.add_edge("main.py:run_calculation", "calculator.py:add", type=EDGE_TYPE_INVOKES)

```
This snippet illustrates how nodes (representing code entities) and edges (representing relationships) with specific types are added programmatically to build the map. Each node also stores relevant information like its type and potentially its source code snippet and line numbers.

The actual graph building logic in `build_graph.py` is more sophisticated to handle various code structures and edge cases, but this gives you the basic idea.

## Conclusion

The Dependency Graph is a fundamental data structure in LocAgent. It acts as a structured, queryable map of the entire codebase, capturing files, classes, functions, and their intricate relationships like containment, imports, calls, and inheritance. This representation allows the LLM Agent to efficiently navigate, understand dependencies, and pinpoint relevant code locations without getting lost in the vastness of the source code.

Now that we understand *what* this map is, let's learn how it's created.

Next up: [Chapter 3: Graph Building Process](03_graph_building_process_.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)