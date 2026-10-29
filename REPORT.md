# Lab 2 report: Extend the MiniLang parser

Name: Fuad Abbasov
Partner (if you worked in a pair): -

## 1. Test output

Paste the last 8 lines that `npm test` (or the Playground) printed for `01_minilang_parser.ts`.

```
REGRESSION         17/17  ok
STEP 1             6/6  ok
STEP 2             7/7  ok
STEP 3             7/7  ok
STEP 4 (stretch)   5/5  ok
STEP 5 (stretch)   6/6  ok
STEP 6 (stretch)   8/8  ok

CORE DONE
```

## 2. Your grammar

Paste the GRAMMAR comment from your file, as it is now.

```
// ---------- Grammar (EBNF). Edit this first in every step. [for core assignments] ----------
//   expr    = "if" expr "then" expr "else" expr | compare ;
//   compare = sum [ ">" sum ] ;
//   sum     = term { ( "+" | "-" ) term } ;
//   term    = primary { "*" primary } ;
//   primary = INT | STRING | IDENT | "(" expr ")" ;
//
// ---------- Grammar (EBNF). Edit this first in every step. [for home assignments] ----------
//   expr    = "if" expr "then" expr "else" expr | compare ;
//   compare = sum [ ( ">" | ">=" | "==" ) sum ] ;
//   sum     = term { ( "+" | "-" ) term } ;
//   term    = unary { "*" unary } ;
//   unary   = "-" unary | primary ;
//   primary = INT | STRING | IDENT | "true" | "false" | "(" expr ")" ;
```

## 3. Precedence and associativity

1. Which line of your code makes `*` bind tighter than `+` and `-`? Give the line number and explain in one sentence.
   - **Lines 159 and 162** (`let left = parseTerm(p);` and `const right = parseTerm(p);` in `parseSum`): `parseSum` obtains each operand of `+` and `-` by calling `parseTerm(p)`, which parses any `*` operations first at a deeper recursion level and returns the whole product as a single child node of the `+` or `-` AST node.

2. Which line makes `8 - 3 - 2` group as `(8 - 3) - 2`? What would you change to get `8 - (3 - 2)`, and why is that wrong for `-`?
   - **Which line**: The `while` loop in `parseSum` (**lines 160–164**, specifically **line 163**: `left = op === "+" ? { kind: "add", left, right } : { kind: "sub", left, right };`), which repeatedly wraps the already-parsed `left` subtree as the left child of each new `"sub"` node.
   - **What to change to get `8 - (3 - 2)`**: Replace the `while` loop with a right-recursive call `const right = parseSum(p);` (corresponding to `sum = term [ ( "+" | "-" ) sum ] ;`).
   - **Why that is wrong for `-`**: Subtraction is not associative; right-grouping turns `8 - 3 - 2` into `8 - (3 - 2) = 7` instead of the mathematically correct left-associative `(8 - 3) - 2 = 3`.

## 4. Typing rules

Write the rules you added in the notation of Lecture 2 (premises above, conclusion below), then point to the `case` in `check()` that implements each one.

- **T-Sub** — implemented in `check()` at `case "sub"` (lines 235–240):
  ```
  Gamma |- e1 : int    Gamma |- e2 : int
  -------------------------------------- (T-Sub)
          Gamma |- e1 - e2 : int
  ```

- **T-Mul** — implemented in `check()` at `case "mul"` (lines 241–246):
  ```
  Gamma |- e1 : int    Gamma |- e2 : int
  -------------------------------------- (T-Mul)
          Gamma |- e1 * e2 : int
  ```

- **Stretch rules** (`T-Bool`, `T-Neg`, `T-Ge`, `T-Eq`):
  - **T-Bool** (`case "bool"`, lines 216–217):
    ```
    ------------------------ (T-Bool)
    Gamma |- b : bool
    ```
  - **T-Neg** (`case "neg"`, lines 223–227):
    ```
    Gamma |- e : int
    -------------------- (T-Neg)
    Gamma |- -e : int
    ```
  - **T-Ge** (`case "ge"`, lines 253–258):
    ```
    Gamma |- e1 : int    Gamma |- e2 : int
    -------------------------------------- (T-Ge)
           Gamma |- e1 >= e2 : bool
    ```
  - **T-Eq** (`case "eq"`, lines 259–264):
    ```
    Gamma |- e1 : T    Gamma |- e2 : T
    ---------------------------------- (T-Eq)
         Gamma |- e1 == e2 : bool
    ```

## 5. Stretch (Home)

- **Step 4 (true/false): why is `trueish` still an identifier?**
  - In `matchAt`, the `"ident"` regex `/[A-Za-z_]\w*/y` greedily matches the entire contiguous word `"trueish"` (`m[0] === "trueish"`), and `KW.has(m[0])` checks exact set membership on the full matched lexeme. Because only `"true"` and `"false"` are in `KW` (and not `"trueish"`), its token kind stays `"ident"`.

- **Step 5 (unary minus): how does your parser tell unary `-` from binary `-`?**
  - By grammatical position in the recursive-descent parser: `parseUnary` checks for `"-"` when the parser is at the start of an operand (before a primary/unary expression, e.g. at the start of an expression or right after another operator or `"("`), whereas `parseSum` checks for `"-"` inside its `while` loop only *after* a complete `term` (the left operand) has already been parsed.

- **Step 6 (`>=`, `==`): where did `">="` have to go in `RULES`, and why? Your rule T-Eq: which operand types does `==` accept, and why?**
  - **Where `">="` goes in `RULES`**: `[">=", />=/y]` (line 75) must be placed **before** `[">", />/y]` (line 76). Because `matchAt` returns the **first** rule in `RULES` that matches at `pos`, putting `">"` first would cause `/>/y` to match only the `>` prefix of `">="`, leaving `"="` behind to fail with `bad character '='`.
  - **Rule `T-Eq`**: `==` accepts any two operands of the **same type** `T` (`int == int`, `string == string`, or `bool == bool`) and returns `bool`, throwing a `CheckError` if `l !== r`. Equality is meaningful for all three primitive types in MiniLang, while requiring `l === r` statically prevents nonsensical cross-type comparisons (such as `1 == "1"` or `true == 1`) without implicit coercion.
