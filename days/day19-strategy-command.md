# Day 19: Strategy & Command

[← Back to Study Plan](../lld-study-plan.md) | [← Day 18](day18-observer-pattern.md)

> **Time**: ~1.5-2 hours
> **Goal**: Two related behavioral patterns. **Strategy** swaps an algorithm at runtime — encapsulate "how" so the host code can change behavior without changing structure. **Command** captures a request as an object — encapsulate "what was requested" so it can be queued, logged, and **undone**. Build a small text editor whose actions (Type, Delete) are commands on an undo/redo stack, with text-rendering as a Strategy.

---
---

# PART 1: STRATEGY — RUNTIME-SWAPPABLE ALGORITHMS

---
---

<br>

<h2 style="color: #2980B9;">📘 19.1 The Core Idea</h2>

A Strategy is an interchangeable **algorithm**. The host class delegates a step to a strategy object, and you can swap the object at runtime to change behavior — no inheritance from the host required.

```
Without Strategy:                     With Strategy:
  class Sorter {                       class Sorter {
    void sort(vector<int>& v) {          ICompare& m_cmp;
      // hard-coded ascending             void sort(vector<int>& v) {
      std::sort(v.begin(), v.end());        std::sort(v.begin(), v.end(),
    }                                          [&](int a, int b){ return m_cmp(a,b); });
  };                                       }
                                         };
  → behavior fixed at compile time     → behavior chosen at construction or call site
```

<br>

#### When to reach for it

| Use case | Why Strategy fits |
|----------|-------------------|
| **Sorting / comparison** | Plug in different orderings |
| **Compression** | gzip vs zstd vs none, picked at runtime |
| **Pricing rules** | Discount strategies vary by region/promo |
| **Path finding** | Dijkstra vs A* vs BFS, same map |
| **Validation** | Different validators per input class |

The host class names "what step needs doing"; the strategy answers "how".

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 19.2 Implementation Variants</h2>

<br>

#### Variant A: Interface-based (classic OOP)

```cpp
class ICompressor {
public:
    virtual ~ICompressor() = default;
    virtual std::string compress(const std::string& in) = 0;
};

class GzipCompressor  : public ICompressor { /* ... */ };
class ZstdCompressor  : public ICompressor { /* ... */ };
class NoneCompressor  : public ICompressor {
public:
    std::string compress(const std::string& in) override { return in; }
};

class Archiver {
    std::unique_ptr<ICompressor> m_strategy;
public:
    Archiver(std::unique_ptr<ICompressor> s) : m_strategy(std::move(s)) {}
    void setStrategy(std::unique_ptr<ICompressor> s) { m_strategy = std::move(s); }
    void store(const std::string& data, const std::string& path) {
        auto compressed = m_strategy->compress(data);
        // write...
    }
};

// Usage:
Archiver a(std::make_unique<ZstdCompressor>());
a.store("...", "blob.zst");
a.setStrategy(std::make_unique<NoneCompressor>());   // swap at runtime
```

Best when the strategy is non-trivial: multiple methods, internal state, complex configuration.

<br>

#### Variant B: `std::function` (modern, lightweight)

For single-method strategies, skip the interface — pass a callable.

```cpp
class Archiver {
    std::function<std::string(const std::string&)> m_compress;
public:
    explicit Archiver(decltype(m_compress) c) : m_compress(std::move(c)) {}
    void store(const std::string& data, const std::string& path) {
        auto compressed = m_compress(data);
    }
};

Archiver a([](const std::string& s) { return zstd_compress(s); });
```

Less ceremony. Loses the type safety of "this is a *Compressor*, not just any string-to-string function" — pick based on whether that distinction matters.

<br>

#### Variant C: Templates (compile-time strategy / policy-based design)

```cpp
template<typename Compressor>
class Archiver {
    Compressor m_compressor;
public:
    void store(const std::string& data, const std::string& path) {
        auto compressed = m_compressor.compress(data);
    }
};

Archiver<ZstdCompressor> a;
```

Zero runtime cost (no virtual dispatch), but the strategy is fixed per instantiation — you cannot swap at runtime. This is the "policy-based design" approach (see Day 52).

<br>

#### Picking a variant

| Need | Use |
|------|-----|
| Multi-method strategy, runtime swap | Interface |
| Single-method, runtime swap | `std::function` |
| Single-method, compile-time fixed, performance-critical | Template |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 19.3 Strategy vs Inheritance</h2>

Pre-Strategy, the natural reflex was inheritance:

```cpp
class Archiver { virtual std::string compress(...) = 0; };
class GzipArchiver  : public Archiver { /* ... */ };
class ZstdArchiver  : public Archiver { /* ... */ };
```

Problems:
- **Combinatorial explosion** — add an `Encryptor` axis and you need `GzipEncryptedArchiver`, `ZstdEncryptedArchiver`, ...
- **Cannot change behavior after construction** — you'd need a new object
- **Tightly couples policy and host class** — the archiver hierarchy carries every algorithm

Strategy uses **composition** to solve all three: each axis is a separate strategy, slot them together, swap at any time.

<br><br>

---
---

# PART 2: COMMAND — REQUESTS AS OBJECTS

---
---

<br>

<h2 style="color: #2980B9;">📘 19.4 The Core Idea</h2>

A Command is a **request packaged as an object**. Instead of calling `editor.type('a')`, you create a `TypeCommand('a')` and call `editor.execute(cmd)`. The command holds the operation and its parameters together, and survives long enough to be:

- **Queued** (job systems, undo stacks)
- **Logged** (replay, auditing)
- **Undone** (`undo()` reverses the operation)
- **Composed** (macros — a command made of commands)

```
Direct call:                            Command:
  editor.type('a');                       auto cmd = std::make_unique<TypeCommand>('a');
                                          editor.execute(std::move(cmd));   // runs and stored
                                          editor.undo();                    // reverses it
```

<br>

#### When to reach for it

| Use case | Why Command fits |
|----------|------------------|
| **Undo / redo** | Each command knows how to reverse itself |
| **Job/task queues** | Push commands; worker thread `execute()`s them |
| **Macro recording** | Capture sequence of commands; replay |
| **GUI menus / shortcuts** | Bind keystroke → command object |
| **Transactional operations** | Run on success; the failure path knows how to roll back |

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 19.5 Anatomy of a Command</h2>

```cpp
class ICommand {
public:
    virtual ~ICommand() = default;
    virtual void execute() = 0;
    virtual void undo()    = 0;
    virtual std::string name() const = 0;   // for debugging / logging
};

class TypeCommand : public ICommand {
    TextEditor& m_editor;
    std::string m_text;
    std::size_t m_position = 0;     // captured for undo
public:
    TypeCommand(TextEditor& ed, std::string text) : m_editor(ed), m_text(std::move(text)) {}
    void execute() override {
        m_position = m_editor.cursor();
        m_editor.insertAt(m_position, m_text);
    }
    void undo() override {
        m_editor.eraseRange(m_position, m_position + m_text.size());
    }
    std::string name() const override { return "Type(\"" + m_text + "\")"; }
};
```

The command **closes over** everything it needs to undo itself. Once `execute()` has run, the command's job is to remember the inverse transformation.

<br>

#### Three approaches to undo

| Approach | How `undo()` works | Memory |
|----------|-------------------|--------|
| **Inverse operation** | Compute and apply the inverse (delete the inserted text) | Cheapest — store only the delta |
| **Memento (state snapshot)** | Snapshot before; restore on undo | Easy to write; expensive on big state |
| **Event log replay** | Replay all commands from start to N-1 | No live undo state; slow for deep histories |

Inverse-operation is the typical default. Use mementos when the operation isn't easily invertible (e.g., random shuffles, complex multi-cell edits).

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 19.6 The Undo / Redo Stack</h2>

Two stacks: one for done commands (the **undo stack**), one for undone commands available to redo (the **redo stack**).

```
State after typing "hello", "world", undoing once:

  undo stack: [TypeCmd("hello")]            ← top
  redo stack: [TypeCmd("world")]            ← top

  redo() pops "world", executes it again, pushes back to undo stack.
  Typing a new command CLEARS the redo stack.
```

```cpp
class TextEditor {
    std::string m_text;
    std::size_t m_cursor = 0;
    std::vector<std::unique_ptr<ICommand>> m_undoStack;
    std::vector<std::unique_ptr<ICommand>> m_redoStack;

public:
    void execute(std::unique_ptr<ICommand> cmd) {
        cmd->execute();
        m_undoStack.push_back(std::move(cmd));
        m_redoStack.clear();    // any new edit invalidates the redo branch
    }

    void undo() {
        if (m_undoStack.empty()) return;
        auto cmd = std::move(m_undoStack.back());
        m_undoStack.pop_back();
        cmd->undo();
        m_redoStack.push_back(std::move(cmd));
    }

    void redo() {
        if (m_redoStack.empty()) return;
        auto cmd = std::move(m_redoStack.back());
        m_redoStack.pop_back();
        cmd->execute();
        m_undoStack.push_back(std::move(cmd));
    }
};
```

The "clear redo stack on new command" rule is what most editors do. Some IDEs implement **branching undo** (a tree of edit history) — uncommon, but powerful for power users.

<br><br>

---
---

# PART 3: STRATEGY + COMMAND TOGETHER

---
---

<br>

<h2 style="color: #2980B9;">📘 19.7 Two Patterns, One System</h2>

In our text editor:
- **Each user action** (Type, Delete, Paste) is a **Command** — captured for undo
- **How the buffer is rendered** (plain text, markdown, syntax-highlighted) is a **Strategy** — interchangeable

```cpp
class IRenderer {                                // Strategy
public:
    virtual ~IRenderer() = default;
    virtual std::string render(const std::string& text) const = 0;
};

class PlainRenderer : public IRenderer {
public:
    std::string render(const std::string& t) const override { return t; }
};

class MarkdownRenderer : public IRenderer {
public:
    std::string render(const std::string& t) const override {
        return "**" + t + "**";   // ridiculously simplified
    }
};

class TextEditor {
    std::unique_ptr<IRenderer> m_renderer = std::make_unique<PlainRenderer>();
    /* ... commands, stacks, etc. ... */
public:
    void setRenderer(std::unique_ptr<IRenderer> r) { m_renderer = std::move(r); }
    std::string display() const { return m_renderer->render(m_text); }
};
```

The strategy chooses **how to display**; the command chooses **what was done**. They don't compete — they cover orthogonal axes.

<br><br>

---
---

# PART 4: NUANCES & EDGE CASES

---
---

<br>

<h2 style="color: #2980B9;">📘 19.8 Macro Commands (Composite Pattern overlap)</h2>

A "replace all" operation might be many `Delete + Type` commands. Wrap them in a `MacroCommand`:

```cpp
class MacroCommand : public ICommand {
    std::vector<std::unique_ptr<ICommand>> m_subs;
public:
    void add(std::unique_ptr<ICommand> c) { m_subs.push_back(std::move(c)); }

    void execute() override {
        for (auto& c : m_subs) c->execute();
    }
    void undo() override {
        // Undo in REVERSE order — each undo restores the state before its execute
        for (auto it = m_subs.rbegin(); it != m_subs.rend(); ++it) (*it)->undo();
    }
    std::string name() const override { return "Macro(" + std::to_string(m_subs.size()) + ")"; }
};
```

The reverse-order undo is critical. If you typed `a` then `b`, undoing `a` first then `b` would corrupt cursor positions. Always: forward execute, reverse undo.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 19.9 Non-Invertible Commands</h2>

Some operations can't be undone purely from the command itself:

```cpp
class ShuffleCommand : public ICommand {
    std::vector<int>& m_data;
    std::vector<int>  m_snapshot;   // memento — captures before-state
public:
    void execute() override {
        m_snapshot = m_data;        // snapshot for undo
        std::random_shuffle(m_data.begin(), m_data.end());
    }
    void undo() override {
        m_data = m_snapshot;
    }
};
```

Memento-style undo trades memory for simplicity. For huge state, use a copy-on-write data structure or persistent collections.

<br>

#### Commands with side effects

```cpp
class SendEmailCommand : public ICommand {
public:
    void execute() override { sendEmail(...); }
    void undo() override    { /* ??? — you can't unsend an email */ }
};
```

You can't always undo. For irreversible side effects, either:
- Make the command refuse to enter the undo stack (`bool isUndoable() override { return false; }`)
- Implement compensating actions (`SendApologyEmail` rather than `unsendEmail`)
- Don't model it as a command — just call the function

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 19.10 Memory in the Undo Stack</h2>

Undo stacks grow without bound by default. In production:

- **Cap depth** — keep last N commands; drop the oldest
- **Coalesce** — collapse repeated keystrokes into one command (typing "hello" → one TypeCommand of "hello", not five)
- **Periodic snapshot** — keep recent commands + an old snapshot; older history reconstructed from snapshot + log if asked

VS Code, Git, and most IDEs do all three.

<br><br>

---
---

# PART 5: ANTI-PATTERNS & TRADE-OFFS

---
---

<br>

<h2 style="color: #2980B9;">📘 19.11 Common Mistakes</h2>

#### Anti-pattern 1: Strategy with one implementation

```cpp
class IFoo { virtual void bar() = 0; };
class TheOnlyFoo : public IFoo { void bar() override { /* ... */ } };
```

If there's exactly one strategy and no plan for more, you've added an interface for nothing. Inline the call.

<br>

#### Anti-pattern 2: Command with no undo

If you only ever `execute()` and never `undo()`, you don't have a Command — you have `std::function<void()>`. Use the Command pattern when at least one of: undo, queueing, logging, or replay is required.

<br>

#### Anti-pattern 3: Command that captures references to mutable shared state

```cpp
class RenameCommand : public ICommand {
    Item& m_item;          // dangling if Item is destroyed before undo
    std::string m_oldName;
};
```

Either the command shares ownership (`shared_ptr`) or the editor guarantees lifetimes. Don't store raw references to long-lived shared state across an undo stack.

<br>

#### Anti-pattern 4: `undo()` that's not really an inverse

```cpp
void execute() override { x += 1; }
void undo()    override { x  = 0; }   // wrong if x was 7 to start with
```

`undo()` must restore the **exact** prior state, not "a default state". Test this with a property test: for a random sequence of execute/undo/redo, the final state must match a sequence with all undos applied.

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 19.12 Exercise: Text Editor with Undo/Redo</h2>

A small editor with `type`, `del`, `undo`, `redo`. Each user action becomes a Command on the undo stack.

<br>

#### Skeleton

```cpp
// editor.h
#pragma once
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>
#include <vector>

class TextEditor;

class ICommand {
public:
    virtual ~ICommand() = default;
    virtual void execute() = 0;
    virtual void undo()    = 0;
    virtual std::string name() const = 0;
};

class TextEditor {
public:
    void type(const std::string& text);
    void del(std::size_t n);
    void undo();
    void redo();

    const std::string& text() const { return m_text; }
    std::size_t cursor() const      { return m_cursor; }

    // Mutators commands call into
    void insertAt(std::size_t pos, const std::string& text);
    void eraseRange(std::size_t from, std::size_t to);
    void setCursor(std::size_t c) { m_cursor = c; }

    void dumpHistory() const;

private:
    void execute(std::unique_ptr<ICommand> cmd);

    std::string m_text;
    std::size_t m_cursor = 0;
    std::vector<std::unique_ptr<ICommand>> m_undoStack;
    std::vector<std::unique_ptr<ICommand>> m_redoStack;
};

// ----- concrete commands -----

class TypeCommand : public ICommand {
public:
    TypeCommand(TextEditor& ed, std::string text)
        : m_ed(ed), m_text(std::move(text)) {}

    void execute() override {
        m_pos = m_ed.cursor();
        m_ed.insertAt(m_pos, m_text);
        m_ed.setCursor(m_pos + m_text.size());
    }
    void undo() override {
        m_ed.eraseRange(m_pos, m_pos + m_text.size());
        m_ed.setCursor(m_pos);
    }
    std::string name() const override { return "Type(\"" + m_text + "\")"; }

private:
    TextEditor& m_ed;
    std::string m_text;
    std::size_t m_pos = 0;
};

class DeleteCommand : public ICommand {
public:
    DeleteCommand(TextEditor& ed, std::size_t n) : m_ed(ed), m_n(n) {}

    void execute() override {
        std::size_t cur = m_ed.cursor();
        if (m_n > cur) m_n = cur;          // can't delete past start
        m_pos = cur - m_n;
        m_removed = m_ed.text().substr(m_pos, m_n);
        m_ed.eraseRange(m_pos, m_pos + m_n);
        m_ed.setCursor(m_pos);
    }
    void undo() override {
        m_ed.insertAt(m_pos, m_removed);
        m_ed.setCursor(m_pos + m_removed.size());
    }
    std::string name() const override { return "Delete(" + std::to_string(m_n) + ")"; }

private:
    TextEditor& m_ed;
    std::size_t m_n = 0;
    std::size_t m_pos = 0;
    std::string m_removed;
};
```

```cpp
// editor.cpp
#include "editor.h"

void TextEditor::insertAt(std::size_t pos, const std::string& text) {
    m_text.insert(pos, text);
}

void TextEditor::eraseRange(std::size_t from, std::size_t to) {
    if (from > m_text.size() || to > m_text.size() || from > to)
        throw std::out_of_range("eraseRange");
    m_text.erase(from, to - from);
}

void TextEditor::execute(std::unique_ptr<ICommand> cmd) {
    cmd->execute();
    m_undoStack.push_back(std::move(cmd));
    m_redoStack.clear();
}

void TextEditor::type(const std::string& text) {
    execute(std::make_unique<TypeCommand>(*this, text));
}

void TextEditor::del(std::size_t n) {
    execute(std::make_unique<DeleteCommand>(*this, n));
}

void TextEditor::undo() {
    if (m_undoStack.empty()) return;
    auto cmd = std::move(m_undoStack.back());
    m_undoStack.pop_back();
    cmd->undo();
    m_redoStack.push_back(std::move(cmd));
}

void TextEditor::redo() {
    if (m_redoStack.empty()) return;
    auto cmd = std::move(m_redoStack.back());
    m_redoStack.pop_back();
    cmd->execute();
    m_undoStack.push_back(std::move(cmd));
}

void TextEditor::dumpHistory() const {
    std::cout << "  text: \"" << m_text << "\" (cursor=" << m_cursor << ")\n";
    std::cout << "  undo: [";
    for (const auto& c : m_undoStack) std::cout << c->name() << " ";
    std::cout << "]\n  redo: [";
    for (const auto& c : m_redoStack) std::cout << c->name() << " ";
    std::cout << "]\n";
}
```

<br>

#### Test driver

```cpp
// main.cpp
#include "editor.h"
#include <cassert>

int main() {
    TextEditor ed;

    ed.type("hello");
    assert(ed.text() == "hello");

    ed.type(" world");
    assert(ed.text() == "hello world");

    ed.undo();                     // remove " world"
    assert(ed.text() == "hello");

    ed.redo();                     // re-add " world"
    assert(ed.text() == "hello world");

    ed.del(5);                     // delete "world"
    assert(ed.text() == "hello ");

    ed.undo();                     // restore "world"
    assert(ed.text() == "hello world");

    ed.undo();                     // remove " world"
    ed.undo();                     // remove "hello"
    assert(ed.text().empty());

    // Re-type AFTER undos clears redo stack
    ed.redo();                     // back to "hello"
    ed.type("!");                  // new branch — wipes redo of " world"
    assert(ed.text() == "hello!");
    ed.redo();                     // nothing to redo
    assert(ed.text() == "hello!");

    std::cout << "All assertions passed.\n";
    ed.dumpHistory();

    // Demonstrate Strategy axis: a renderer
    // (left as a bonus exercise)

    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day19 main.cpp editor.cpp && ./day19
```

<br>

#### Expected output pattern

```
All assertions passed.
  text: "hello!" (cursor=6)
  undo: [Type("hello") Type("!") ]
  redo: [ ]
```

<br>

#### Bonus Challenges

1. **Renderer Strategy** — add `IRenderer` with `PlainRenderer` and `UpperCaseRenderer`. Make the editor's `display()` route through the strategy. Swap renderers at runtime.

2. **Macro recording** — add `startMacro()` / `endMacro()`. While recording, executed commands collect into a `MacroCommand` instead of going to the undo stack directly. After `endMacro()`, the whole macro is one undoable unit.

3. **Coalescing** — collapse consecutive `TypeCommand`s into one when undoing (typing "hello" produces five commands; undoing once removes all five). Hint: when pushing a `TypeCommand`, check whether the top-of-stack is a recent `TypeCommand` and merge.

4. **Bounded history** — cap `m_undoStack` at, say, 100. When over, drop the oldest. Confirm undo still works correctly within the cap.

5. **Persistence** — write the undo stack to disk as a JSON log of command names + parameters. Restore it on startup. (Requires commands to be serializable.)

6. **Branching undo (advanced)** — replace the linear redo-stack-clear-on-new-edit with a tree of edit history. Add `undoBranch()` / `redoBranch()`.

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 19.13 Q&A</h2>

<br>

#### Q1: "When does Strategy degenerate into a `std::function`?"

When the strategy has exactly one method and no internal state, an interface buys you nothing over `std::function`. Stick with `std::function` until you need:
- Multiple methods on the strategy
- Strategy-level state that persists across calls
- `dynamic_cast` / RTTI (rare)

<br>

#### Q2: "When should I use Strategy vs Template policy-based design?"

- **Strategy** (virtual): runtime swap, dynamic selection from config/UI
- **Template policy**: compile-time fixed, zero-overhead — pick when the strategy is set per-instantiation and performance matters

You can also combine them: a template default with a virtual override hook.

<br>

#### Q3: "Why not use `std::function` for commands too?"

For execute-only operations, you can. The moment you need `undo()`, you're pairing two callables — at which point a `Command` interface (or a `struct { std::function<void()> exec, undo; }`) is clearer.

<br>

#### Q4: "How do I compose commands without losing undo correctness?"

Use `MacroCommand`. Always undo in **reverse order** of execution. If sub-commands have inter-dependencies (one's output is another's input), capture the dependency in the parameters at execute time, not at construction.

<br>

#### Q5: "What happens if `execute()` fails partway through?"

For atomic commands, exception out of `execute()` and ensure no state was changed before the throw. For composite commands, you need a transaction model — a partial-execute should roll back the steps that did succeed:

```cpp
void MacroCommand::execute() {
    std::size_t completed = 0;
    try {
        for (auto& c : m_subs) { c->execute(); ++completed; }
    } catch (...) {
        for (std::size_t i = completed; i-- > 0; ) m_subs[i]->undo();
        throw;
    }
}
```

<br>

#### Q6: "How do I unit-test commands?"

Create a fake editor, execute a command, assert the editor's state, undo, assert the state matches the pre-execute snapshot. Property-based testing shines here:

```cpp
auto pre = ed.text();
cmd.execute();
cmd.undo();
assert(ed.text() == pre);   // round-trip invariant
```

<br>

#### Q7: "Doesn't Strategy violate encapsulation by exposing the algorithm?"

The opposite — it **encapsulates** the algorithm in a separate object so the host class doesn't have to know its internals. The strategy itself is encapsulated; what's exposed is its interface, which is the whole point.

<br>

#### Q8: "Where do GUI shortcut keys fit?"

Keystroke handling typically maps a key combo → a Command factory. The handler creates the command, calls `editor.execute(cmd)`. This is why most GUIs have a "Command Manager" / "Action Registry" — it's the lookup table from input event to command.

<br><br>

---

## Reflection Questions

1. What's the difference between Strategy and Command? When are they used together?
2. Why does Command shine for undo/redo specifically? What does "the command captures what was done" mean?
3. Why must macro commands undo their sub-commands in reverse order?
4. When does Strategy degenerate into "just use `std::function`"?
5. How do non-invertible operations (like sending an email) fit (or not fit) into a Command system?
6. What memory pressure does an undo stack create, and how do real editors mitigate it?

---

## Interview Questions

1. "Implement a text editor with `type`, `delete`, `undo`, `redo`. Each action is a command."
2. "Walk through how `redo()` interacts with new edits. What invariant does the redo stack maintain?"
3. "Implement a `MacroCommand` that batches sub-commands and undoes them atomically."
4. "How would you handle a command whose `execute()` partially fails midway through?"
5. "Compare interface-based Strategy with template-based policy design. When would you pick each?"
6. "What's the difference between Strategy and the State pattern (Day 20)?"
7. "How would you cap your undo stack at 100 commands while still allowing undo within that window?"
8. "How would you coalesce a stream of single-character typing into one undoable unit?"
9. "How do you test that `undo()` is the true inverse of `execute()`?"
10. "Where does Command sit relative to other patterns like Memento, Composite, and Observer?"

---

**Next**: Day 20 — State Machine →
