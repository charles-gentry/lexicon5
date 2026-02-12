# Code Review Issues

Use `gh issue create` with each section below to create GitHub issues, or create them manually.

---

## Issue 1: Bug - Biased shuffle algorithm produces non-uniform randomness

**Labels:** bug

### Description

The codebase uses `array.sort(() => Math.random() - 0.5)` in three places to shuffle arrays. This is a well-known anti-pattern that does **not** produce a uniform shuffle. The sort comparator must be consistent (transitive), but random return values violate this contract, causing different JS engines to produce different biases.

### Affected locations

- `index.html:206` - dictionary shuffling in `generatePuzzle()`
- `index.html:294` - vertical word selection in `initGame()`
- `index.html:329` - loose letter shuffling in `initGame()`

### Impact

Directly affects gameplay fairness. Some words/letter arrangements will appear more frequently than others.

### Suggested fix

Replace all three instances with a Fisher-Yates shuffle:

```js
function shuffle(arr) {
    for (let i = arr.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [arr[i], arr[j]] = [arr[j], arr[i]];
    }
    return arr;
}
```

---

## Issue 2: Bug - clearRect uses DPR-scaled dimensions instead of CSS pixels

**Labels:** bug

### Description

In `drawGame()` at `index.html:351`:

```js
ctx.clearRect(0, 0, canvas.width, canvas.height);
```

Since `ctx.scale(dpr, dpr)` is applied during initialization, the drawing coordinate space operates in CSS pixels. However, `canvas.width` and `canvas.height` are set to `cssSize * dpr`, so `clearRect` clears a larger area than intended.

### Impact

Not visually broken (clearing too much is harmless), but technically incorrect and could cause confusion during maintenance.

### Suggested fix

```js
ctx.clearRect(0, 0, canvas.width / dpr, canvas.height / dpr);
```

Or store the CSS dimensions and use those directly.

---

## Issue 3: Bug - No window resize handling causes blurry/misaligned canvas

**Labels:** bug

### Description

Canvas scaling is computed once in `initGame()` based on `canvas.clientWidth` at load time. If the user resizes the browser window or rotates a mobile device, the canvas buffer dimensions no longer match the displayed size, resulting in blurry or misaligned rendering.

### Steps to reproduce

1. Open the game on a mobile device in portrait mode
2. Rotate to landscape
3. Observe that the canvas rendering becomes blurry or misaligned

### Suggested fix

Add a `resize` event listener (debounced) that recalculates `CELL_SIZE`, `FONT_SIZE`, rescales the canvas buffer, and calls `drawGame()`:

```js
window.addEventListener('resize', debounce(() => {
    const dpr = window.devicePixelRatio || 1;
    const cssSize = canvas.clientWidth;
    CELL_SIZE = cssSize / GRID_SIZE;
    FONT_SIZE = CELL_SIZE * 0.5;
    canvas.width = cssSize * dpr;
    canvas.height = cssSize * dpr;
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.scale(dpr, dpr);
    drawGame();
}, 200));
```

---

## Issue 4: Bug - Premature win-check message shown on every letter placement

**Labels:** bug

### Description

`checkWinCondition()` is called after every letter tap (`index.html:475`), even when central column cells are obviously still empty. This causes the "Almost there! Keep going..." error-styled message to flash after each placement, which is a distracting UX issue.

### Steps to reproduce

1. Start a new puzzle
2. Tap to place the first letter
3. Observe the "Almost there!" message appears immediately

### Suggested fix

Only call `checkWinCondition()` when all central cells are filled, i.e., when `selectedRow === -1` after `highlightNextEmpty`:

```js
highlightNextEmpty(selectedRow + 1);
drawGame();
if (selectedRow === -1) {
    checkWinCondition();
}
```

---

## Issue 5: Code Quality - `easy_dictionary` is defined but never used

**Labels:** enhancement

### Description

`dictionary.js:1` defines `easy_dictionary` (90 words), but `index.html:185` only references `dictionary`. This is dead code.

### Suggested fix

Either remove `easy_dictionary` or wire it up to a difficulty selector so users can choose between easy and standard modes.

---

## Issue 6: Code Quality - Global mutable state

**Labels:** enhancement

### Description

Seven top-level `let` variables hold all game state (`index.html:174-181`): `gridData`, `looseLetters`, `solutionGrid`, `validWords`, `selectedRow`, `canvas`, `ctx`, and `looseLettersContainer`.

### Impact

For a small project this works, but encapsulating state in an object or class would reduce the surface for accidental mutation and make the code easier to test.

### Suggested fix

Wrap game state in a class or module pattern:

```js
const game = {
    gridData: [],
    looseLetters: [],
    solutionGrid: [],
    validWords: new Set(),
    selectedRow: 0,
    canvas: null,
    ctx: null,
};
```

---

## Issue 7: Code Quality - `window.onload` assignment can be overwritten

**Labels:** bug

### Description

At `index.html:608`:

```js
window.onload = function() { ... };
```

If any other script or browser extension assigns to `window.onload`, the game initialization is silently replaced.

### Suggested fix

Use `addEventListener` instead:

```js
window.addEventListener('load', function() {
    initGame();
    canvas.addEventListener('pointerdown', handleCanvasClick);
});
```

---

## Issue 8: Code Quality - Magic numbers

**Labels:** enhancement

### Description

Several numeric literals are used inline without named constants:

- `50` max dummy letter generation attempts (`index.html:315`)
- `3000` ms confetti lifetime (`index.html:600`)
- `30` confetti pieces (`index.html:593`)

### Suggested fix

Extract these as named constants alongside the existing ones at the top of the script:

```js
const MAX_DUMMY_ATTEMPTS = 50;
const CONFETTI_COUNT = 30;
const CONFETTI_LIFETIME_MS = 3000;
```

---

## Issue 9: Robustness - Overly strict conflict check in puzzle generation

**Labels:** enhancement

### Description

The puzzle generation algorithm (`index.html:224-233`) rejects any horizontal word that has a fixed letter matching *any* letter in the vertical word. Since validation uses `validWords` (a puzzle-specific set, not the full dictionary), there is no real ambiguity risk from matching letters.

### Impact

This over-constraint makes generation fragile for vertical words with common letters (E, A, S, T, etc.), increasing the chance of exhausting all 10 generation attempts and showing the error message to the user.

### Suggested fix

Relax the conflict check to only reject horizontal words where letter ambiguity could actually lead to an alternate valid solution, rather than blanket-rejecting any letter overlap.

---

## Issue 10: Robustness - Dummy letter generation can silently produce fewer than expected

**Labels:** bug

### Description

At `index.html:313-326`, dummy letter generation loops up to 50 times trying to find letters not already in `looseLetters`. If the vertical word uses many unique letters, most of the alphabet may be rejected, and the loop cap prevents adding the full `NUM_DUMMY_LETTERS` (3) dummies.

### Impact

The game could end up with fewer than 3 dummy letters without any indication, making the puzzle easier than intended.

### Suggested fix

After the loop, check if `dummyLettersAdded < NUM_DUMMY_LETTERS` and either retry with relaxed constraints or log a warning. Alternatively, build the candidate set explicitly:

```js
const candidates = alphabet.split('').filter(l => !looseLetters.includes(l));
shuffle(candidates);
looseLetters.push(...candidates.slice(0, NUM_DUMMY_LETTERS));
```

---

## Issue 11: Security - CDN scripts loaded without Subresource Integrity

**Labels:** enhancement

### Description

At `index.html:7`:

```html
<script src="https://cdn.tailwindcss.com"></script>
```

No `integrity` or `crossorigin` attributes are present. If the CDN is compromised, arbitrary JavaScript runs in the page.

Additionally, `cdn.tailwindcss.com` is intended for development use only. For production, Tailwind recommends a build step or the Tailwind CLI.

### Suggested fix

Either add SRI hashes:

```html
<script src="https://cdn.tailwindcss.com"
  integrity="sha384-..."
  crossorigin="anonymous"></script>
```

Or switch to a Tailwind build step for production deployment.
