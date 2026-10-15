# Morse Code Tree

> An interactive, single-file Morse code trainer rendered as a circuit-board trace tree. Press short for a dot, long for a dash, and watch the signal travel down the tree until it lands on a letter.

---

![Single File](https://img.shields.io/badge/single--file-yes-blue?style=for-the-badge&labelColor=1a1a1a)
![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen?style=for-the-badge&labelColor=1a1a1a)
![Build](https://img.shields.io/badge/build-not%20required-blue?style=for-the-badge&labelColor=1a1a1a)
![Stack](https://img.shields.io/badge/stack-html%20%7C%20css%20%7C%20js-blue?style=for-the-badge&labelColor=1a1a1a)

![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=for-the-badge&labelColor=1a1a1a)
![ITU-R](https://img.shields.io/badge/ITU--R-M.1677--1-blue?style=for-the-badge&labelColor=1a1a1a)
![GNU Terry Pratchett](https://img.shields.io/badge/GNU-Terry%20Pratchett-blueviolet?style=for-the-badge&labelColor=1a1a1a)

## What This Is

A browser-based Morse code decoder that teaches the *structure* of Morse, not just the output. Instead of showing you a lookup table, it draws the standard Morse tree as a PCB-style schematic — and lights up the exact path your input takes through it.

Every letter in the International Morse alphabet lives at a specific position in a binary tree:

- A **short press** (dot) takes the right branch
- A **long press** (dash) takes the left branch

Type `...` and you walk to **S**. Add one more dot (`....`) and you arrive at **H**. Add a dash instead (`...-`) and you arrive at **V**. The tree makes the relationship between letters visible and physical.

---

## How To Use

| Action | Input |
|---|---|
| Dot (`.`) | Short press — **Space** or **Left Click** |
| Dash (`-`) | Long press — **Space** or **Left Click** |
| Commit a letter | Stop pressing for ~1 second |

There is no "send" button and no backspace. The tree validates your input live: if you type a sequence that doesn't correspond to a real letter, the input is rejected and you stay where you are.

### Timing

Two thresholds control the feel of the app:

- **Dash threshold:** how long a press must be to count as a dash rather than a dot
- **Letter timeout:** how long you can pause before the current letter is finalized

Both are tuned for beginners. If you're an experienced operator, they'll feel slow — that's intentional. This is a learning tool, not a paddle. Users are free to shorten the delay times as needed. 

---

## Reading The Screen

### Node Shapes

The shape of each node is not decorative. It tells you what *kind of input* reaches it:

| Shape | Meaning |
|---|---|
| **Circle** | You arrive here with a **dot** (short press) |
| **Rectangle** | You arrive here with a **dash** (long press) |

So the shape alone tells you the last input required. **S** is a circle because you reach it with a dot. **V** is a rectangle because you reach it with a dash. **H** is a circle because you reach it with a dot.

This distinction matters. Before it was fixed, **V** was mistakenly drawn as a circle — which visually implied that `...` followed by a *dot* would produce V, when in reality V requires `...-` (a dash). Two letters appearing to share the same command is impossible in a binary tree, and the shape system exists precisely to make that impossible-looking.

### The "Traces"

Lines are routed Manhattan-style — vertical, horizontal, vertical — the way a real circuit board routes copper. Junction dots appear wherever a parent splits into two children. This is slightly different from the units I've seen for sale with a '18 speed gear shift' pattern.

### The Banners

Two readouts sit below the tree:

**Morse Output** — shows the raw dots and dashes you're currently typing. Resets when a letter is committed.

**Ticker Tape** — a rolling buffer of the last 16 letters you've decoded. The most recent letter is highlighted; older letters fade back.

---

## Why The Code Looks Like This

This project went through several distinct architectural phases, and the current design is a direct response to what broke in each one. If you're curious about the "why" behind the structure, this is the short version.

### Phase 1 — Hand-placed coordinates

The first version stored every letter as a flat object with hardcoded `x` and `y` percentages. Nodes were HTML `<div>`s layered on top of a separate SVG for the traces.

This produced the now-infamous *squished tree in the corner* bug: the nodes were positioned with percentages, but the SVG traces were drawn in raw pixel units. The two coordinate systems never agreed, so the traces rendered as a tiny tree in the top-left corner while the nodes sprawled across the whole board.

### Phase 2 — Coordinate system reconciliation

Adding `viewBox="0 0 100 100"` and `preserveAspectRatio="none"` to the SVG synced the two systems. This fixed the corner-tree but exposed the next problem: nodes were now overlapping.

### Phase 3 — Manual nudging

Every fix from here was a manual coordinate adjustment. Widening the board, increasing the height, shifting `C` left and `D` right, fanning out `B` and `J`. Each adjustment worked — until the next one broke something else. Moving `D` to make room for `C` pushed `D` into `W`. Moving `W` out of the way pushed `W` into `R`.

This is the classic symptom of a layout with no underlying rule. Every fix is a patch, and patches conflict.

### Phase 4 — The rewrite

The current version throws out hand-placed coordinates entirely.

The tree is now a **nested data structure**: each node knows only its own letter and its two children (dash child, dot child). No positions. No pixel values.

A **recursive layout engine** walks the tree and assigns positions automatically:

- Leaf nodes get sequential horizontal slots
- Parent nodes are placed at the average of their children's positions
- Vertical position is derived from depth

Because siblings get unique slots and parents sit between their children, **overlap is mathematically impossible**. The tree also recalculates on window resize, so it can never break at any screen size.

Rendering moved fully into SVG, built element-by-element with `createElementNS` rather than injected as an HTML string. This guarantees correct namespace handling and makes traces, nodes, and labels share one coordinate space.

### Phase 4.1 — Semantic correctness / Final review

Two bugs survived the rewrite, both in the *data* rather than the layout:

- **Q** was attached to the wrong parent, producing `---.` instead of `--.-`
- **V** was assigned the wrong shape, making it visually indistinguishable in intent from **H**

Both were structural or semantic errors that no amount of layout tuning would have caught. They were found by cross-referencing against the actual Morse table, not by looking at the screen.

---

## What This Project Teaches (Structurally)

If you're reading the source out of curiosity, the interesting lesson is this: **most "layout bugs" are actually data-model bugs wearing a disguise.**

The overlapping-nodes phase looked like a CSS problem. It wasn't. It was a symptom of storing positions as data instead of deriving them from relationships. Once the tree knew its own shape, the layout became self-correcting.

The same lesson applies in reverse to the Q and V bugs: those looked like rendering issues, but the renderer was faithfully drawing what the data told it to draw. The data was wrong.

---

## Forward: On Clacks, Beacons, and Small Lights

I built this after rereading some Terry Pratchett. The Discworld novels are full of signaling — the semaphore towers of the Grand Trunk Clacks, the beacons lit along the Ramtops, the whole idea that a message can be broken into discrete pulses and reassembled somewhere far away by someone who was listening correctly. The Clacks operators don't send words. They send *states* — the arm up, the arm down, the light on, the light off — and the meaning emerges from the pattern.

There is a detail in the later books that has stayed with me for years. The Clacks code **GNU** stands for *"do not log off"* — a way of keeping a name in the header of every message, passed from tower to tower, never discarded. When Pratchett died, his fans used it as a memorial. For a while, the overhead of many websites carried the name in a hidden header, riding along with every request, the way it would ride along a Clacks line.

I think that's the thing I find moving about Morse, and about all signaling systems. A dot and a dash are not information. They are *physics* — a light on, a light off, a hand held down for a moment longer than usual. And yet somebody on the other end, listening patiently, reconstructs a thought out of them. The tree in this project is just a map of that reconstruction: one short press, then another, then a longer one, and there — a letter appears.

If you're learning Morse with this, I hope the tree helps it feel less like memorization and more like walking down a path you already know. That's all signaling ever is: knowing where you are, and knowing which way the next pulse goes.

*GNU Terry Pratchett.*

---

## Files

```
index.html    — the entire app. No build step, no dependencies local or in the cloud. 
```

Open it in any modern browser. That's the whole setup. If there are ideas for improvements, feel free to open a PR. My only potential refinement is adding a long pause for word separation and extending the ticker. If I get incredibly ambitious perhaps a more stylized UI. 

---

## Credits

Morse tree structure per the International Morse Code standard (ITU-R M.1677-1). Visual language inspired by PCB silkscreen training tools and, indirectly, by a fictional semaphore network that refused to forget anyone.
