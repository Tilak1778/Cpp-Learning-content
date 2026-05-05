# Day 20: State Machine

[← Back to Study Plan](../lld-study-plan.md) | [← Day 19](day19-strategy-command.md)

> **Time**: ~1.5-2 hours
> **Goal**: A finite state machine (FSM) is a system whose behavior is defined by a fixed set of **states**, **events**, and **transitions** between them. Learn the difference between Mealy and Moore machines, **guards** (conditional transitions) and **entry/exit actions**, the three implementation styles (switch-based, State-pattern classes, transition table), and when to escalate to a hierarchical state machine. Build a TCP connection state machine covering CLOSED, LISTEN, SYN_SENT, ESTABLISHED, FIN_WAIT_*, TIME_WAIT, ...

---
---

# PART 1: WHAT IS A STATE MACHINE AND WHY DOES IT EXIST?

---
---

<br>

<h2 style="color: #2980B9;">📘 20.1 The Core Idea</h2>

A finite state machine is the **smallest, most explicit model of a stateful system**. It says:

> "I am in exactly one state at a time. When event E happens, here is the rule: maybe I move to state S' and run action A; otherwise I ignore E."

```
States:        {CLOSED, LISTEN, ESTABLISHED, ...}
Events:        {open(), close(), recv(SYN), recv(FIN), timeout, ...}
Transitions:   (currentState, event) → (nextState, action)
```

Everything else is consequence.

<br>

#### When to reach for it

| Use case | Why FSM fits |
|----------|--------------|
| **Network protocols** (TCP, TLS) | Behavior is purely a function of state + packet type |
| **UI flows** (login → 2FA → home) | Each screen is a state; navigation is a transition |
| **Parsers / lexers** | State per partial-parse phase |
| **Game / animation logic** | Idle ↔ Walking ↔ Jumping ↔ Falling |
| **Connection lifecycles** | Connecting ↔ Connected ↔ Reconnecting ↔ Closed |

The pattern's payoff: **all valid behavior is enumerable** in a transition table. Bugs become missing rows, not subtle conditions sprinkled across `if` chains.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 20.2 Mealy vs Moore</h2>

Two textbook flavors:

| | Mealy | Moore |
|---|------|------|
| **Action attached to** | The transition (depends on current state **and** event) | The state itself (depends on current state only) |
| **Output computed from** | (state, event) | (state) |
| **Typical use** | Most software FSMs — actions vary per event | Hardware / parsers — outputs are state-driven |

In real software, you usually mix them: actions on transitions (Mealy) plus **entry/exit actions** on states (Moore-like).

```
ESTABLISHED state
  ┌─────────────────┐
  │ on_entry: start_keepalive_timer()
  │ on_exit:  cancel_keepalive_timer()
  └─────────────────┘
       │
   recv(FIN) ── transition ──┐
   action: send(ACK)         │
       ↓                     ↓
  CLOSE_WAIT
```

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 20.3 The Implicit-FSM Anti-Pattern</h2>

Most code already contains state machines — they're just hidden in flag soup:

```cpp
class Connection {
    bool m_open = false;
    bool m_handshakeDone = false;
    bool m_closing = false;
    bool m_authenticated = false;
    /* ... */
};
```

With four bools, you have 2⁴ = 16 nominal states, but probably only 4-5 are reachable. Code looks like:

```cpp
if (m_open && !m_closing && m_handshakeDone && !m_authenticated) { ... }
else if (m_open && !m_closing && m_handshakeDone && m_authenticated) { ... }
```

Bugs live in:
- Unreachable combinations no one accounts for
- Combinations the developer thought were unreachable but aren't
- Subtle ordering — set `m_handshakeDone` before `m_open` and the condition fails

Making the FSM explicit collapses 16 fictitious states into 5 real ones with named transitions. Every change becomes "add a row" rather than "add another flag and audit every conditional".

<br><br>

---
---

# PART 2: IMPLEMENTATION VARIANTS

---
---

<br>

<h2 style="color: #2980B9;">📘 20.4 Variant 1: Switch-Based FSM</h2>

The simplest implementation — an enum for state, a `switch` that handles each event per state.

```cpp
enum class State { Closed, Connecting, Open, Closing };
enum class Event { Connect, Connected, Close, Closed };

class Connection {
    State m_state = State::Closed;

public:
    void on(Event e) {
        switch (m_state) {
        case State::Closed:
            if (e == Event::Connect)   m_state = State::Connecting;
            break;
        case State::Connecting:
            if (e == Event::Connected) m_state = State::Open;
            else if (e == Event::Close) m_state = State::Closed;
            break;
        case State::Open:
            if (e == Event::Close)     m_state = State::Closing;
            break;
        case State::Closing:
            if (e == Event::Closed)    m_state = State::Closed;
            break;
        }
    }
};
```

<br>

#### Trade-offs

| Pros | Cons |
|------|------|
| Zero indirection, trivially fast | Spread across every event handler — hard to see the whole machine |
| Easy to step through in a debugger | Adding states means editing every switch |
| No allocations | Entry/exit actions are awkward |

Fine for small machines (≤ 5 states). For anything bigger, a transition table makes the structure visible.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 20.5 Variant 2: State Pattern (One Class per State)</h2>

The GoF "State" pattern: each state is a class that handles its own events. The context delegates.

```cpp
class Connection;

class IState {
public:
    virtual ~IState() = default;
    virtual void onConnect(Connection&)   {}
    virtual void onConnected(Connection&) {}
    virtual void onClose(Connection&)     {}
    virtual void onClosed(Connection&)    {}
    virtual std::string name() const = 0;
};

class ClosedState : public IState {
public:
    void onConnect(Connection& c) override;
    std::string name() const override { return "Closed"; }
};

class ConnectingState : public IState {
public:
    void onConnected(Connection& c) override;
    void onClose(Connection& c) override;
    std::string name() const override { return "Connecting"; }
};

class Connection {
    std::unique_ptr<IState> m_state = std::make_unique<ClosedState>();
public:
    void setState(std::unique_ptr<IState> s) { m_state = std::move(s); }
    const IState& state() const              { return *m_state; }

    void connect()    { m_state->onConnect(*this); }
    void connected()  { m_state->onConnected(*this); }
    void close()      { m_state->onClose(*this); }
};

void ClosedState::onConnect(Connection& c) {
    c.setState(std::make_unique<ConnectingState>());
}
```

<br>

#### Trade-offs

| Pros | Cons |
|------|------|
| State-specific behavior is local to the state class | Lots of small classes |
| Easy to add per-state state (member variables) | Object lifetime — `setState` destroys the calling state mid-call (subtle!) |
| Works well with polymorphism | Transition matrix is implicit, not visible |

The lifetime gotcha: `c.setState(new X)` destroys `*this` before the calling method returns. After `setState`, **don't touch `this`**.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 20.6 Variant 3: Transition Table</h2>

A data-driven FSM — declare the rules as data, dispatch by lookup.

```cpp
struct Transition {
    State        from;
    Event        on;
    State        to;
    std::function<void()> action;     // optional
    std::function<bool()> guard;      // optional — only transition if guard returns true
};

class TableFSM {
    State m_state;
    std::vector<Transition> m_table;

public:
    explicit TableFSM(State initial, std::vector<Transition> table)
        : m_state(initial), m_table(std::move(table)) {}

    bool dispatch(Event e) {
        for (const auto& t : m_table) {
            if (t.from != m_state || t.on != e) continue;
            if (t.guard && !t.guard())          continue;
            if (t.action)  t.action();
            m_state = t.to;
            return true;
        }
        return false;   // no matching transition — event ignored
    }

    State state() const { return m_state; }
};

// Definition reads like the spec:
TableFSM conn{State::Closed, {
    { State::Closed,     Event::Connect,    State::Connecting, nullptr, nullptr },
    { State::Connecting, Event::Connected,  State::Open,       []{ std::cout << "OPEN\n"; }, nullptr },
    { State::Connecting, Event::Close,      State::Closed,     nullptr, nullptr },
    { State::Open,       Event::Close,      State::Closing,    nullptr, nullptr },
    { State::Closing,    Event::Closed,     State::Closed,     nullptr, nullptr },
}};
```

<br>

#### Trade-offs

| Pros | Cons |
|------|------|
| The whole machine is a single readable artifact | Slightly slower lookup than switch (linear scan) |
| Easy to dump as a graph for visualization | Generic actions/guards via `std::function` cost a few cycles |
| Add a row → add a transition; no scattered edits | Type erasure on actions — harder to attach state-specific data |

This is the recommended default for non-trivial FSMs. For lookup speed, key the table on `(state, event)` in a `std::unordered_map` or a 2D array (when states/events are dense small enums).

<br><br>

---
---

# PART 3: GUARDS, ENTRY/EXIT ACTIONS, AND INTERNAL TRANSITIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 20.7 Guards — Conditional Transitions</h2>

A guard is a predicate evaluated **before** taking a transition. If false, the transition is skipped (and another may be tried, or the event is ignored).

```cpp
{ State::Authenticating, Event::CredentialsOk, State::Authorized,
  /*action*/ nullptr,
  /*guard*/  [&]{ return m_failedAttempts < 3; } },

{ State::Authenticating, Event::CredentialsOk, State::Locked,
  nullptr,
  [&]{ return m_failedAttempts >= 3; } },
```

Two transitions on the same `(state, event)` differ only by guard. The dispatcher tries them in order; first matching guard wins. This keeps complex policy logic visible in the table instead of buried in actions.

<br>

#### Guards vs actions — a common confusion

- **Guard** answers "should we transition?" — pure predicate, **no side effects**
- **Action** answers "what do we do **as part of** the transition?" — side effects allowed

A guard with side effects creates double-fires when the guard runs but the transition is skipped. Keep them pure.

<br><br>

---

<br>

<h2 style="color: #2980B9;">📘 20.8 Entry and Exit Actions</h2>

Code attached to **entering** or **leaving** a state, regardless of which transition triggered it. This is where Moore-style behavior lives.

```cpp
struct StateInfo {
    std::function<void()> onEntry;
    std::function<void()> onExit;
};

class FSMWithActions {
    State m_state;
    std::map<State, StateInfo> m_stateInfo;
    /* ... transition table ... */

public:
    void transition(State newState) {
        if (auto it = m_stateInfo.find(m_state); it != m_stateInfo.end() && it->second.onExit)
            it->second.onExit();
        m_state = newState;
        if (auto it = m_stateInfo.find(m_state); it != m_stateInfo.end() && it->second.onEntry)
            it->second.onEntry();
    }
};
```

Examples in TCP:
- **ESTABLISHED on entry**: start keepalive timer
- **ESTABLISHED on exit**: cancel keepalive timer
- **TIME_WAIT on entry**: schedule 2*MSL timer
- **TIME_WAIT on exit**: clean up control block

These would be duplicated across every transition that arrives at/leaves the state if not factored out.

<br>

#### Self-transition: re-enter or stay?

If state X transitions to itself on event E, two semantics are possible:

- **External transition** — exit X, enter X again (re-fires entry/exit actions)
- **Internal transition** — stay in X (skip entry/exit, run only the transition action)

Pick the right one explicitly. Re-entering on every keystroke (when the state has timer entry actions) will fire timers per stroke; internal transitions don't.

<br><br>

---
---

# PART 4: TCP CONNECTION STATE MACHINE

---
---

<br>

<h2 style="color: #2980B9;">📘 20.9 The TCP State Diagram (RFC 793)</h2>

TCP is the canonical FSM example — it has 11 states with well-defined transitions. We'll model the simplified version sufficient for this exercise:

```
                ┌─────────┐
       passive  │ CLOSED  │  active open: send SYN
       open     │         │  ─────────────────────►
       ─────►   └────┬────┘                        ┌────────────┐
                     │ recv SYN                    │ SYN_SENT   │
                     ▼                             │            │
                 ┌────────┐  recv SYN+ACK / send ACK│            │
                 │ LISTEN │ ◄──────────────────────────┐         │
                 └───┬────┘                            │         │
                     │ recv SYN /                      │         │
                     │ send SYN+ACK                    │         │
                     ▼                                 ▼         │
                ┌──────────┐                     ┌─────────────┐ │
                │ SYN_RCVD │ recv ACK ──────────►│ ESTABLISHED │ │
                └──────────┘                     └──────┬──────┘ │
                                                        │        │
                                       close / send FIN │        │
                                                        ▼        │
                                                 ┌─────────────┐ │
                                                 │ FIN_WAIT_1  │ │
                                                 └──────┬──────┘ │
                                                        │ recv ACK
                                                        ▼
                                                 ┌─────────────┐
                                                 │ FIN_WAIT_2  │
                                                 └──────┬──────┘
                                                        │ recv FIN / send ACK
                                                        ▼
                                                 ┌─────────────┐
                                                 │  TIME_WAIT  │ ── 2*MSL ──► CLOSED
                                                 └─────────────┘

  (Plus passive close path: ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED)
```

The **valid behavior** of a TCP endpoint is exactly this diagram — every received packet that isn't on a defined transition is dropped or RST'd. The protocol's robustness comes from the FSM being explicit and complete.

<br><br>

---
---

# PART 5: ANTI-PATTERNS & TRADE-OFFS

---
---

<br>

<h2 style="color: #2980B9;">📘 20.10 Common Mistakes</h2>

#### Anti-pattern 1: Boolean flag explosion (the implicit FSM)

Already covered in §20.3 — the canonical case for converting to an explicit FSM.

<br>

#### Anti-pattern 2: Side effects in guards

```cpp
[&]{ ++m_attempts; return m_attempts < 3; }   // BAD — counter advances even when transition fails
```

Guards are evaluated speculatively. Mutate state in the action, never the guard.

<br>

#### Anti-pattern 3: Touching `this` after `setState`

In the State-pattern variant, `setState(new X)` destroys the current state object. Code after `setState` in a method of the old state runs on a destroyed object → UB.

```cpp
void ConnectingState::onClose(Connection& c) {
    c.setState(std::make_unique<ClosedState>());
    m_attempts = 0;     // BAD — `this` is destroyed
}
```

Either swap last (set state at the end of the method) or use trampoline-style dispatch where the FSM owns the transition.

<br>

#### Anti-pattern 4: State explosion → time for hierarchical FSMs

If your machine grows to 30 states with cross-cutting concerns (e.g., "in any operating state, on Disconnect → Closed"), you've hit state explosion. **Hierarchical State Machines (HSMs)** group related states under a parent that handles common transitions:

```
+- Operating
|  +- Idle
|  +- Working
|  +- Paused
|  ─── on Disconnect → Closed (handled by parent) ───
+- Closed
```

UML statecharts and the Boost.SML / hsm-cpp libraries support HSMs. Reach for them when flat FSMs become unmaintainable.

<br>

#### Anti-pattern 5: Forgetting unhandled events

```cpp
case State::Closed:
    if (e == Event::Connect) m_state = State::Connecting;
    // what about all other events? silently dropped → bug magnet
```

Decide explicitly: log unhandled events, throw, or treat as ignored. Whatever you choose, document it. Silent drops are the worst case.

<br><br>

---
---

# PART 6: BUILD EXERCISE

---
---

<br>

<h2 style="color: #2980B9;">📘 20.11 Exercise: TCP Connection State Machine</h2>

Build a simplified TCP FSM using the transition-table style. Cover the active-open path (client) and the close path.

<br>

#### Skeleton

```cpp
// tcp_fsm.h
#pragma once
#include <functional>
#include <iostream>
#include <map>
#include <optional>
#include <string>
#include <vector>

enum class State {
    Closed, Listen, SynSent, SynRcvd, Established,
    FinWait1, FinWait2, CloseWait, LastAck, TimeWait
};

enum class Event {
    PassiveOpen,        // app: listen()
    ActiveOpen,         // app: connect()  -> send SYN
    Close,              // app: close()
    RecvSyn,
    RecvSynAck,
    RecvAck,
    RecvFin,
    Timeout2Msl,        // TIME_WAIT timer fires
};

inline std::string toString(State s) {
    switch (s) {
    case State::Closed:      return "CLOSED";
    case State::Listen:      return "LISTEN";
    case State::SynSent:     return "SYN_SENT";
    case State::SynRcvd:     return "SYN_RCVD";
    case State::Established: return "ESTABLISHED";
    case State::FinWait1:    return "FIN_WAIT_1";
    case State::FinWait2:    return "FIN_WAIT_2";
    case State::CloseWait:   return "CLOSE_WAIT";
    case State::LastAck:     return "LAST_ACK";
    case State::TimeWait:    return "TIME_WAIT";
    }
    return "?";
}

inline std::string toString(Event e) {
    switch (e) {
    case Event::PassiveOpen: return "passive_open";
    case Event::ActiveOpen:  return "active_open";
    case Event::Close:       return "close";
    case Event::RecvSyn:     return "recv(SYN)";
    case Event::RecvSynAck:  return "recv(SYN+ACK)";
    case Event::RecvAck:     return "recv(ACK)";
    case Event::RecvFin:     return "recv(FIN)";
    case Event::Timeout2Msl: return "timeout(2MSL)";
    }
    return "?";
}

struct Transition {
    State from;
    Event on;
    State to;
    std::function<void()> action;
};

class TcpFsm {
public:
    TcpFsm() = default;

    void dispatch(Event e) {
        for (const auto& t : table()) {
            if (t.from == m_state && t.on == e) {
                runExit(m_state);
                std::cout << "  " << toString(m_state) << " --[" << toString(e)
                          << "]--> " << toString(t.to) << "\n";
                if (t.action) t.action();
                m_state = t.to;
                runEntry(m_state);
                return;
            }
        }
        std::cout << "  [ignored: " << toString(e) << " in " << toString(m_state) << "]\n";
    }

    State state() const { return m_state; }

private:
    void runEntry(State s);
    void runExit(State s);

    static const std::vector<Transition>& table() {
        static const std::vector<Transition> t = {
            // active-open path
            { State::Closed,      Event::ActiveOpen,   State::SynSent,      []{ std::cout << "    send SYN\n"; } },
            { State::SynSent,     Event::RecvSynAck,   State::Established,  []{ std::cout << "    send ACK\n"; } },
            { State::SynSent,     Event::Close,        State::Closed,       nullptr },

            // passive-open path
            { State::Closed,      Event::PassiveOpen,  State::Listen,       nullptr },
            { State::Listen,      Event::RecvSyn,      State::SynRcvd,      []{ std::cout << "    send SYN+ACK\n"; } },
            { State::SynRcvd,     Event::RecvAck,      State::Established,  nullptr },

            // active-close path
            { State::Established, Event::Close,        State::FinWait1,     []{ std::cout << "    send FIN\n"; } },
            { State::FinWait1,    Event::RecvAck,      State::FinWait2,     nullptr },
            { State::FinWait2,    Event::RecvFin,      State::TimeWait,     []{ std::cout << "    send ACK\n"; } },
            { State::TimeWait,    Event::Timeout2Msl,  State::Closed,       nullptr },

            // passive-close path
            { State::Established, Event::RecvFin,      State::CloseWait,    []{ std::cout << "    send ACK\n"; } },
            { State::CloseWait,   Event::Close,        State::LastAck,      []{ std::cout << "    send FIN\n"; } },
            { State::LastAck,     Event::RecvAck,      State::Closed,       nullptr },
        };
        return t;
    }

    State m_state = State::Closed;
};

inline void TcpFsm::runEntry(State s) {
    if (s == State::Established) std::cout << "    [enter] start_keepalive_timer\n";
    if (s == State::TimeWait)    std::cout << "    [enter] schedule_2MSL_timer\n";
}

inline void TcpFsm::runExit(State s) {
    if (s == State::Established) std::cout << "    [exit ] cancel_keepalive_timer\n";
    if (s == State::TimeWait)    std::cout << "    [exit ] clear_control_block\n";
}
```

<br>

#### Test driver

```cpp
// main.cpp
#include "tcp_fsm.h"
#include <cassert>

void testActiveOpenAndClose() {
    std::cout << "=== Active open + active close ===\n";
    TcpFsm c;
    assert(c.state() == State::Closed);

    c.dispatch(Event::ActiveOpen);    assert(c.state() == State::SynSent);
    c.dispatch(Event::RecvSynAck);    assert(c.state() == State::Established);
    c.dispatch(Event::Close);         assert(c.state() == State::FinWait1);
    c.dispatch(Event::RecvAck);       assert(c.state() == State::FinWait2);
    c.dispatch(Event::RecvFin);       assert(c.state() == State::TimeWait);
    c.dispatch(Event::Timeout2Msl);   assert(c.state() == State::Closed);
    std::cout << "\n";
}

void testPassiveOpenAndPassiveClose() {
    std::cout << "=== Passive open + passive close ===\n";
    TcpFsm s;
    s.dispatch(Event::PassiveOpen);   assert(s.state() == State::Listen);
    s.dispatch(Event::RecvSyn);       assert(s.state() == State::SynRcvd);
    s.dispatch(Event::RecvAck);       assert(s.state() == State::Established);
    s.dispatch(Event::RecvFin);       assert(s.state() == State::CloseWait);
    s.dispatch(Event::Close);         assert(s.state() == State::LastAck);
    s.dispatch(Event::RecvAck);       assert(s.state() == State::Closed);
    std::cout << "\n";
}

void testIgnoredEvents() {
    std::cout << "=== Ignored / invalid events ===\n";
    TcpFsm c;
    c.dispatch(Event::RecvAck);       assert(c.state() == State::Closed);   // no transition
    c.dispatch(Event::ActiveOpen);    assert(c.state() == State::SynSent);
    c.dispatch(Event::RecvSyn);       assert(c.state() == State::SynSent);  // no transition (simultaneous open omitted)
    std::cout << "\n";
}

int main() {
    testActiveOpenAndClose();
    testPassiveOpenAndPassiveClose();
    testIgnoredEvents();
    std::cout << "All assertions passed.\n";
    return 0;
}
```

<br>

#### Compile and run

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic -fsanitize=address \
    -o day20 main.cpp && ./day20
```

<br>

#### Expected output pattern

```
=== Active open + active close ===
  CLOSED --[active_open]--> SYN_SENT
    send SYN
  SYN_SENT --[recv(SYN+ACK)]--> ESTABLISHED
    send ACK
    [enter] start_keepalive_timer
    [exit ] cancel_keepalive_timer
  ESTABLISHED --[close]--> FIN_WAIT_1
    send FIN
  FIN_WAIT_1 --[recv(ACK)]--> FIN_WAIT_2
  FIN_WAIT_2 --[recv(FIN)]--> TIME_WAIT
    send ACK
    [enter] schedule_2MSL_timer
    [exit ] clear_control_block
  TIME_WAIT --[timeout(2MSL)]--> CLOSED
...
All assertions passed.
```

<br>

#### Bonus Challenges

1. **Guards** — model the simultaneous-open case: in `SYN_SENT` on `RecvSyn`, transition to `SYN_RCVD`. Add a guard that distinguishes paths if needed.

2. **Per-state class style** — re-implement using the State-pattern (one class per state). Compare LOC, readability, and how many places change when you add a new state.

3. **Hierarchical FSM** — group `ESTABLISHED`, `FIN_WAIT_*`, `CLOSE_WAIT`, `LAST_ACK`, `TIME_WAIT` under a parent "Connected" superstate. Have the parent handle "RST" by transitioning everyone to `CLOSED`.

4. **Statechart export** — write a function that prints the table in [Graphviz DOT](https://graphviz.org/) format. Render it to a PNG and verify the diagram matches RFC 793.

5. **Timeout integration** — replace the manual `Timeout2Msl` event with a real timer (use `std::thread` + `condition_variable`). Demonstrate that the FSM can be driven by external events arriving on different threads (with mutex protection on `dispatch`).

6. **Generic FSM library** — extract the `TcpFsm` machinery into a templated `Fsm<StateEnum, EventEnum>` so you can reuse it for HTTP request lifecycles, parser states, etc.

<br><br>

---
---

# PART 7: DEEP DIVE — COMMON QUESTIONS

---
---

<br>

<h2 style="color: #2980B9;">📘 20.12 Q&A</h2>

<br>

#### Q1: "When should I use a switch-based FSM vs a transition table?"

| | Switch | Table |
|---|--------|-------|
| **States** | ≤ 5 | Many |
| **Self-documenting** | No — spread across handlers | Yes — single readable artifact |
| **Easy to visualize** | No | Yes (export to DOT) |
| **Performance** | Marginally faster | Marginally slower |

Default to a table the moment your machine reaches 6+ states.

<br>

#### Q2: "How is State pattern different from Strategy?"

Both swap an object polymorphically. The difference is semantic:

| | State | Strategy |
|---|------|---------|
| **Why we swap** | Behavior changes because **internal state** changed | Caller wants different **algorithm** |
| **Who swaps it** | The state itself (during transitions) | External code (caller) |
| **Number of swaps over lifetime** | Many — driven by events | Few — set once or rarely |

The State pattern is essentially Strategy where transitions are part of the strategy interface.

<br>

#### Q3: "Where should actions execute — in the transition or in entry/exit?"

- **Transition action**: runs only when this specific (from, on) → to fires. Useful when the action depends on the event ("send ACK because we got a FIN").
- **Entry action**: runs whenever you enter the state, regardless of how. Useful for setup ("start keepalive timer").
- **Exit action**: runs whenever you leave the state. Useful for cleanup ("cancel keepalive timer").

If the same code repeats across multiple transitions into a state, hoist it to entry. If the same code repeats across multiple transitions out, hoist it to exit.

<br>

#### Q4: "Can I dispatch events recursively (an action triggers another event)?"

Possible but dangerous. The FSM must reentrancy-safe: hold no iterators across the action, and ensure `m_state` is updated **before** the action runs (or queue events instead of dispatching synchronously).

The clean pattern is **event queueing**: actions enqueue further events; the dispatcher drains the queue:

```cpp
void dispatch(Event e) {
    m_pending.push_back(e);
    if (m_inDispatch) return;       // currently draining → don't re-enter
    m_inDispatch = true;
    while (!m_pending.empty()) {
        Event next = m_pending.front();
        m_pending.pop_front();
        // ... transition logic, which may push more events ...
    }
    m_inDispatch = false;
}
```

<br>

#### Q5: "How do I make my FSM thread-safe?"

A single `std::mutex` around `dispatch()` is usually enough — events serialize through it. If your FSM emits events that other threads consume, you may need a queue + worker thread (the "active object" pattern).

Beware of deadlocks: if an action calls back into another locked component, lock ordering matters. The same advice as Day 18: do as little under the lock as possible.

<br>

#### Q6: "What about hierarchical state machines (HSMs / statecharts)?"

HSMs add **superstates**: a state can contain sub-states, and transitions can be defined at the superstate level (handled if no sub-state handles them first). This solves state explosion — instead of 30 flat states with shared transitions, you have a tree.

UML statecharts (Harel charts) formalize this. Libraries: Boost.SML, hsm-cpp, QP/C++. For a hand-rolled HSM, store the current state path (not just one state) and walk up the hierarchy on event dispatch.

<br>

#### Q7: "How do I test an FSM?"

Three layers:
1. **Per-transition unit tests**: for every row in the table, fire the event and assert the new state and any side effects.
2. **Sequence tests**: drive sequences of events that mirror real usage (e.g., the full TCP handshake).
3. **Property tests**: random sequences of events should never cause crashes; states should always be one of the enum values; ignored events should not change state.

A good FSM is **easy** to test exhaustively — you have the full transition table to enumerate.

<br>

#### Q8: "When does an FSM stop being the right abstraction?"

When the state needs to be many independent dimensions at once. If you'd need to multiply state count to model two independent axes (say, "online/offline" × "playing/paused"), use **two parallel FSMs** (each with its own state) or a statechart with orthogonal regions. A single flat FSM with the cross product becomes unmanageable.

<br><br>

---

## Reflection Questions

1. What is the difference between Mealy and Moore machines? Where does each show up in software FSMs?
2. Why is an explicit state machine better than a soup of bool flags?
3. When should an action live on a transition vs as an entry/exit action?
4. What's the difference between a guard and an action? Why must guards be pure?
5. When does a flat FSM warrant escalating to a hierarchical state machine?
6. How does the State pattern (one class per state) compare with the transition-table style?

---

## Interview Questions

1. "Implement a simplified TCP connection state machine using a transition table."
2. "What's the difference between the State pattern and the Strategy pattern?"
3. "Explain Mealy vs Moore machines, with software examples."
4. "Why is a guard required to be side-effect-free?"
5. "What is state explosion, and how do hierarchical state machines mitigate it?"
6. "How would you make your state machine thread-safe? What if events can recurse?"
7. "Walk through the canonical bug pattern: 'I have 6 booleans and I think only some combinations are valid.' How does the FSM pattern fix it?"
8. "How would you test an FSM exhaustively?"
9. "Implement entry and exit actions on top of a transition table. Where do you place them in `dispatch()`?"
10. "What goes wrong if you call `setState(new X)` on a State-pattern FSM and then access `this`?"

---

**Next**: Day 21 — Weekly Review & Mini-Project →
