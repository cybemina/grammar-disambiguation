# Grammar Disambiguation by Minimum-Cost Tree Selection

A small, dependency-free Python tool that takes **any context-free grammar** and **any input string** and always returns **exactly one parse tree** (or reports that the string is not in the language).

Ambiguous grammars give some strings several parse trees. Instead of deleting "bad" trees (which can leave zero or several), this tool gives every tree a *cost* and picks the cheapest one. Because bad patterns are only penalised, never forbidden outright, a tree always exists for every string in the language.

```text
$ python3 disambig.py examples/expr.gr 'a + 3 * b'
(a + (3 * b))

$ python3 disambig.py examples/expr.gr 'a - b - c'
((a - b) - c)

$ python3 disambig.py examples/dangling_eps.gr 'if c if c a else a'
(if c (if c a (else a)) ε)
```

## Features

- Works for **any CFG**, including grammars with **ε-rules** (and the empty string) and **cycles** such as `A -> A`.
- **One tree per string**, deterministic and reproducible.
- Operator **precedence and associativity** (`@2L`, `@1R`, `@1N`) are turned into penalty patterns automatically.
- Custom **`forbid:` patterns** for anything else, including deeper patterns that look at siblings, grandchildren and the *right spine* of a subtree (needed for the dangling else with an optional `else`).
- Real source text can be tokenised with regex tokens (`token INT = [0-9]+`).
- Output as an operator tree, indented tree, bracketed tree or JSON.
- An interactive **notebook** that draws all candidate trees and highlights the chosen one.

## How it works

1. Every parse tree `t` gets a cost, compared lexicographically:

   ```text
   cost(t) = ( violations(t), size(t), canonical_key(t) )
   ```

   - **violations**: number of forbidden `(parent rule, child position, child rule)` patterns used in the tree. These are soft penalties, so a tree always exists.
   - **size**: number of non-terminal nodes. This removes useless cycles such as `A -> A`.
   - **canonical key**: rule order and split points; it breaks the remaining ties deterministically.
2. The minimum-cost tree is found with a **CYK-style dynamic program over input spans**, with a fixed-point iteration inside each span to handle ε-rules and unit cycles.
3. For patterns that look deeper than direct children, each subtree publishes a small **profile** (its root rule, selected grandchild rules, and watched rules on its right spine). The table stores the best tree *per profile*, so the dynamic program stays exact.

The implementation runs in roughly `O(n^4 · C)` time, where `C` depends only on the grammar and the patterns. It is meant for tools that must accept arbitrary grammars and still return one reproducible tree, not as a replacement for linear-time LR parsing in production compilers.

## Installation

Requires Python 3. The command-line tool uses only the standard library.

```bash
git clone <your-repo-url>
cd <your-repo>
python3 disambig.py --help
```

The notebook additionally needs `matplotlib` and Jupyter:

```bash
pip install matplotlib jupyter
```

## Usage

```bash
# grammar from a file, input as an argument
python3 disambig.py examples/expr.gr 'a + 3 * b'

# inline grammar (any argument containing "->" is treated as grammar text)
python3 disambig.py 'S -> S S | a' 'a a a'

# several inputs from a file or from stdin
python3 disambig.py examples/expr.gr -F inputs.txt
echo 'a - b - c' | python3 disambig.py examples/expr.gr -f json

# the empty string
python3 disambig.py 'S -> a S b | ε' --empty

# interactive mode
python3 disambig.py examples/expr.gr -i
```

Options:

| Option | Meaning |
|---|---|
| `-f {ast,tree,bracket,json}` | output format (default `ast`) |
| `--mode {auto,space,chars}` | tokenisation: match terminals/tokens in raw text, split on spaces, or one character per token |
| `-F FILE` | read one input string per line from a file |
| `--empty` | also/only test the empty string |
| `-i` | interactive mode |
| `--help-format` | print the grammar file format |

If a string is not in the language the tool prints `NOT IN LANGUAGE: ...` and exits with status 1. If the chosen tree still violates some pattern, the number of violations is printed after the tree.

## Grammar file format

```text
// comment (everything after // is ignored)
start: E                          // optional; defaults to the left side of the first rule
E -> E + E @1L | E * E @2L        // alternatives separated by ' | ' (spaces around the bar!)
E -> INT | Id | ( E )
token INT = [0-9]+                // regex token classes
token Id  = [A-Za-z_]\w*
A -> ε                            // empty alternative (also: eps, epsilon)
```

- Symbols that never appear on a left-hand side are terminals.
- `@<level>L|R|N` sets the precedence level (higher binds tighter) and associativity (left, right, non-associative).
- `#label` names an alternative so that `forbid:` lines can refer to it.
- `forbid: parent pos child` penalises the child rule at the 0-based right-hand-side position `pos` of the parent rule.
- Deeper patterns use conditions in brackets: `x[3=els]` means "rule `x` whose child 3 uses rule `els`", and `[~=none]` means "rule `none` on the right spine".

### Example: dangling else with an optional (ε) else

```text
// dangling else with an optional (ε) else: the else attaches to the nearest if
start: S
S -> if c S O #x | a
O -> else S #els | ε #none
forbid: x[3=els] 2 x[~=none]
```

The pattern says: in an `if` that has an else-branch, the then-branch (child 2) must not end, along its right edge, in an `if` without else. For `if c if c a else a` the else therefore binds to the inner `if`.

## Notebook

`grammar_disambiguation.ipynb` walks through the examples and draws every candidate tree next to the chosen one (green frame):

1. precedence (`a + 3 * b`)
2. reversed order and parentheses
3. left associativity (`a - b - c`)
4. dangling else
5. a cyclic grammar (`A -> A`)
6. no disambiguation rule at all (the unique key decides)
7. ε-rules (`S -> a S b | ε`)
8. ε and ambiguity together (optional parts)
9. dangling else with an optional else and a deep forbidden pattern
10. ε combined with a cycle

The last section lets you try your own grammar and string.

## Limitations

- **"Correct" is defined by your patterns.** The tool returns the best tree *under the rules you wrote*. If the patterns are incomplete or contradictory, the final choice falls back to the canonical key, which is deterministic but arbitrary.
- **Patterns look downward only:** direct children, siblings, grandchildren and the right spine. Conditions on the left spine or at arbitrary depth are not supported; write the grammar so that the distinction falls within these patterns.
- **Speed:** roughly `O(n^4 · C)`, so it is slow for large inputs.
- **It does not decide whether a grammar is ambiguous** (that question is undecidable); it works per input string.

## Repository layout

```text
disambig.py                     # disambiguator + command-line interface
grammar_disambiguation.ipynb    # interactive notebook with tree drawings
examples/
  expr.gr                       # arithmetic expressions with precedence/associativity
  dangling_eps.gr               # dangling else with an optional (ε) else
```

## Using it from Python

```python
from disambig import parse_grammar, tokenize, disambiguate

G, lex = parse_grammar(open("examples/expr.gr", encoding="utf-8").read())
tokens = tokenize("a + 3 * b", G, lex, "auto")
result = disambiguate(G, [sym for sym, _ in tokens])

if result is None:
    print("not in the language")
else:
    violations, size, key, tree = result
    print(tree)   # nested tuples: (nonterminal, child, child, ...)
```
