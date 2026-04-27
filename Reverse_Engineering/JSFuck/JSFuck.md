# CTF Writeup — JSFuck Reverse Engineering Challenge

**Category:** Reverse Engineering  
**Flag:** `OVRD{js_i5_a_####ed_up_l4ngu4g3_523d4f}`
**Files:** `file`

---

## Challenge Description

We were given a single file containing a massive wall of JavaScript-looking characters using only `[`, `]`, `(`, `)`, `!`, and `+`. Running it printed `Nothing to see here` — a classic red herring.

---

## Understanding the Obfuscation — JSFuck

[JSFuck](http://www.jsfuck.com/) is a JavaScript obfuscation technique that encodes **any valid JS** using only 6 characters by exploiting JavaScript's type coercion system:

| Expression | Evaluates To |
|---|---|
| `![]` | `false` |
| `!![]` | `true` |
| `+[]` | `0` |
| `+!+[]` | `1` |
| `[]+[]` | `""` |
| `[]+!![]` | `"true"` |

From these primitives, JSFuck constructs characters, strings, and eventually calls `Function(code)()` to execute arbitrary JavaScript.

---

## Layer 1 — Naïve Execution (Decoy)

Running the file directly:

```bash
node file
```

**Output:**
```
Nothing to see here
```

This is intentional misdirection. The challenge author embedded a decoy `console.log` as the visible output.

---

## Layer 2 — Intercepting the Function Constructor

JSFuck always reconstructs `Function` at runtime via the prototype chain:
```
[]['flat']['constructor']  →  Function
```

It then calls `Function(encoded_code)()` to execute the real payload. The trick is that the inner code is **never visible** unless you hook `Function` before JSFuck runs.

We patched `Function.prototype.constructor` using `Object.defineProperty` before executing the blob:

```javascript
const hookedSrc = `
(function() {
  var _real = (function(){}).constructor;
  var _proxy = function() {
    var args = Array.prototype.slice.call(arguments);
    var body = args[args.length - 1];
    if (typeof body === 'string' && body.length > 30) {
      console.log('=== INNER CODE ===');
      console.log(body);
    }
    return _real.apply(this, arguments);
  };
  _proxy.prototype = _real.prototype;
  Object.defineProperty(Array.prototype.filter, 'constructor', {
    get: function() { return _proxy; }, configurable: true
  });
  Object.defineProperty(Function.prototype, 'constructor', {
    get: function() { return _proxy; }, configurable: true
  });
})();
`;

const fs = require('fs');
const src = fs.readFileSync('file', 'utf8');
eval(hookedSrc + src);
```

**Captured inner payload (after decoding octal escapes):**

```javascript
const stuff = [12,21,17,7,56,41,48,28,42,118,28,34,28,96,96,96,96,38,39,28,54,51,28,47,119,45,36,54,119,36,112,28,118,113,112,39,119,37,62];
const decoded_thingy = stuff.map(small_stuff => String.fromCharCode(small_stuff ^ 67));
console.log("Nothing to see here");
```

Key observation: `decoded_thingy` is computed but **never printed**. The only thing printed is the decoy string. This is the second layer of misdirection.

---

## Layer 3 — XOR Decoding the Flag

The inner code maps each integer in `stuff` through `XOR 67` (decimal) to get characters. We simply ran that ourselves:

```javascript
const stuff = [12,21,17,7,56,41,48,28,42,118,28,34,28,96,96,96,96,38,39,28,54,51,28,47,119,45,36,54,119,36,112,28,118,113,112,39,119,37,62];
const decoded = stuff.map(s => String.fromCharCode(s ^ 67)).join('');
console.log(decoded);
```

**Output:**
```
OVRD{js_i5_a_####ed_up_l4ngu4g3_523d4f}
```

---

## Summary of Layers

```
File
 └─ Layer 1: JSFuck obfuscation (6-char JS encoding)
      └─ Layer 2: Function() call hidden inside — outputs decoy "Nothing to see here"
           └─ Layer 3: XOR-encoded integer array — result computed but never printed
                └─ FLAG: OVRD{js_i5_a_####ed_up_l4ngu4g3_523d4f}
```

---

## Key Takeaways

- **Never trust the output at face value** in RE challenges. A "nothing here" message is almost always a red herring.
- **JSFuck always uses `Function()` under the hood** — hooking the constructor before execution is the standard decode technique.
- **Hidden computations** — variables computed but never logged is a classic trick. Always look at what the inner code *does*, not just what it *prints*.
- The flag itself is a leet-speak joke: *"JS is a ####ed up language"* — the `####` is intentionally censored profanity.

---

## Flag

```
OVRD{js_i5_a_####ed_up_l4ngu4g3_523d4f}
```
