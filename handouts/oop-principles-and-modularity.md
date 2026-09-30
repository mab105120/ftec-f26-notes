# OOP Principles, Modularity, and the Python Ecosystem

## Why this topic, and why now

This course is moving you from using Python as a *scripting tool* (a file that runs top to bottom to answer one question) to using it as a *programming language*: a way to build a system that keeps growing. The system you are building this semester is a personal portfolio tracker. It records buy and sell transactions for assets such as stocks, organizes them into portfolios, and computes what you hold, what it cost, and how much you have gained or lost.

Last week introduced object-oriented programming (OOP): a program as a set of interacting objects, the difference between a class (the blueprint) and an object (the house built from it), and how to write a class with a constructor, `self`, properties, and methods. It also named the four principles of OOP and developed one of them, inheritance, in depth.

This week has one idea running through it: **expose the "what," hide the "how."** It appears at three scales:

1. **Inside a class.** The remaining three OOP principles (encapsulation, abstraction, and polymorphism) are all ways of letting code use an object without knowing or touching its internals.
2. **Across files.** Modularity means splitting code into modules that other code imports and uses without rewriting.
3. **Across the whole Python world.** Thousands of developers publish modules you can install and reuse. The ecosystem (PyPI, pip, virtual environments) is how that reuse is managed safely.

The handout follows that order: the OOP principles first, then modularity and imports, then the tooling around them.

## Part 1: Inheritance, continued — overriding and overloading

### A quick recap

Inheritance lets a **child** class be a specialized version of a **parent** class. The child automatically gets everything the parent has, and only states what is different. It is correct only when the "is-a" test holds: a `Stock` *is an* `Asset`, so inheritance fits. A transaction *has an* asset but is not one, so it uses composition instead.

This week adds two ideas about what a child can do with the methods it inherits.

### Overriding: the child replaces an inherited behavior

Every asset in the tracker has a market value. For a stock, it is simply quantity × price. Bonds are different: bond prices are quoted as a *percentage of face value*. A bond priced at 98.5 with a $1,000 face value is worth $985, not $98.50. Using the parent's quantity × price formula for a bond would be badly wrong.

The fix is **overriding**: the child defines a method with *the same name* as the parent's, and its version takes priority.

```python
class Asset:
    def __init__(self, ticker, quantity, price):
        self.ticker = ticker
        self.quantity = quantity
        self.price = price

    def market_value(self):
        return self.quantity * self.price


class Stock(Asset):
    pass  # a stock's value is exactly what Asset already computes


class Bond(Asset):
    def __init__(self, ticker, quantity, price, face_value):
        super().__init__(ticker, quantity, price)
        self.face_value = face_value

    def market_value(self):
        # Bond prices are quoted as a percent of face value (98.5 = 98.5%).
        return self.quantity * self.face_value * self.price / 100
```

`Stock` adds nothing (`pass` means "no body"), so it uses `Asset`'s `market_value` unchanged. `Bond` overrides it. When you call `market_value()` on a `Bond`, Python looks in the `Bond` class first, finds a `market_value` there, and uses it. It only goes up to the parent if the child doesn't define the method itself.

In business terms, this is the department addendum from last week's analogy saying: "the company-wide expense policy applies to us, *except* for travel, where our rule is this instead." Everything else still comes from the company policy.

### Overloading: same name, different inputs

**Overloading** is a different idea that is easy to confuse with overriding. It means defining *several versions of the same method in the same class*, each taking different parameters. The language picks the version that matches the arguments you pass. In languages such as Java or C#, a bank account class might have:

```
deposit(amount)
deposit(amount, currency)
```

Calling `deposit(100)` runs the first version, and `deposit(100, "EUR")` runs the second.

**Python does not support overloading directly.** If you define two methods with the same name in one class, the second one replaces the first, and only the last definition survives:

```python
def deposit(amount):
    return "one argument"

def deposit(amount, currency):   # silently replaces the version above
    return "two arguments"

deposit(100)   # TypeError: deposit() missing 1 required positional argument: 'currency'
```

You need to recognize overloading as a general OOP term, because you will meet it in other languages and in documentation. You are not expected to write it in Python.

| | Overriding | Overloading |
|---|---|---|
| Where | Parent and child classes | Within one class |
| Same name? | Yes | Yes |
| Same parameters? | Yes | No, and the parameter list is how versions are told apart |
| In Python? | Yes, used constantly | Not directly supported |

## Part 2: Encapsulation — the object guards its own data

### The idea

**Encapsulation** means bundling data with the behavior that operates on it, and making that behavior the *only sanctioned way* to change the data.

The analogy is a bank teller. You don't walk into the vault and edit your own balance. You ask the teller, and the teller checks the rules first: is there enough money for this withdrawal? The vault is the data. The teller is the method. The rules live with the teller, so every customer is subject to the same checks.

### Why it matters

Take a rule from the portfolio tracker: **you cannot sell more shares than you hold.** Suppose any script can change an asset's quantity directly (`aapl.quantity = aapl.quantity - 100`). Then every piece of code that sells shares has to remember to check the rule itself. If there are five places in the program that sell shares, the rule is copied five times. Eventually someone forgets it in a sixth place, and the portfolio shows −94 shares of AAPL.

With encapsulation, the rule is written **once**, inside the object, next to the data it protects:

```python
class Asset:
    def __init__(self, ticker, quantity, price):
        self.ticker = ticker
        self._quantity = quantity  # leading underscore: "internal — use the methods"
        self.price = price

    def get_quantity(self):
        return self._quantity

    def sell(self, shares):
        if shares > self._quantity:
            raise ValueError(f"Cannot sell {shares} {self.ticker}; only {self._quantity} held")
        self._quantity -= shares

    def market_value(self):
        return self._quantity * self.price
```

Outside code reads the quantity through `get_quantity()` and changes it through `sell()`. There is no other supported path, so there is no way to sell shares while skipping the check. (For now, read `raise ValueError(...)` as "refuse, and stop with this message." What happens to that refusal, and how a program should respond to it, is the next topic in the course.)

### The underscore is a convention, not a lock

The leading underscore in `_quantity` tells other developers: "this is internal; don't touch it directly, use the methods." **Python does not enforce it.** This line runs without complaint:

```python
aapl._quantity = -5   # allowed, but it breaks the contract
```

Some languages have hard access controls. Python trusts developers to respect the convention, and linters and code reviewers treat reaching into `_something` from outside the class as a red flag. When you review code, including code an AI tool writes for you, a line that reaches past the methods into an underscored attribute is a sign that someone went around the rules.

## Part 3: Abstraction — use the "what," ignore the "how"

**Abstraction** means giving users a simple surface (*what* something does) and hiding the details of *how* it does it.

Everyone in this course has used Excel's `=IRR()` function. Very few people could explain the root-finding algorithm behind it, and nobody needs to. You supply cash flows and get a rate back. That is abstraction: the function's inputs and output are the surface, and the algorithm is hidden.

It is the same idea as a vendor's order form. You fill in the fields the vendor accepts (item, quantity, address) and never see how their warehouse is organized.

In the tracker, code that calls `aapl.market_value()` knows *what* it gets (the position's value) and nothing about *how* it's computed. That ignorance is valuable. You saw `Bond` compute its market value with a completely different formula, and the calling code didn't change at all. If the calculation later changed again, for example to use a live market price, callers still wouldn't change.

Abstraction and encapsulation are closely related but not the same:

- **Encapsulation** is about *protection*: data can only change through methods that enforce the rules.
- **Abstraction** is about *simplicity*: users deal with a small, meaningful surface and not the machinery behind it.

In this course, abstraction is a concept you should understand and recognize. Some languages, and Python itself, have special syntax for defining "abstract" classes. You won't need to write it.

## Part 4: Polymorphism — same request, each object answers its own way

**Polymorphism** ("many forms") means different kinds of objects responding to the same request, each in its own way.

The analogy is end-of-day mark-to-market. Operations asks every position the same question: "What are you worth?" An equity answers from its last trade price. A bond answers as a percentage of face value. Operations doesn't need a different question for each instrument type, and doesn't need to know how each one calculates its answer.

With the `Stock` and `Bond` classes from Part 1, one loop can value a mixed list:

```python
portfolio = [
    Stock("AAPL", 6, 190),
    Stock("MSFT", 5, 410),
    Bond("UST-10Y", 20, 98.5, 1000),
]

for asset in portfolio:
    print(f"{asset.ticker:8} {asset.market_value():>10,.2f}")
```

```
AAPL       1,140.00
MSFT       2,050.00
UST-10Y   19,700.00
```

Look at what the loop *doesn't* contain: there is no `if` that checks "is this a bond?" Each object knows how to answer `market_value()` for itself. Python calls the right version automatically based on the object's actual class: the inherited one for stocks, the overridden one for the bond.

Why this matters: suppose you later add a new kind of asset, such as an option with its own valuation rule. Without polymorphism, you'd hunt down every loop, report, and total in the program and add another `if` branch to each. With polymorphism, you write one new class with its own `market_value()`, and every existing loop already handles it. **Adding a new type means adding a class, not editing everything that uses assets.**

Inheritance set this up, and polymorphism is the payoff. Later in the course, the same idea lets the tracker swap one part of the system for another (for example, where data is stored) without the rest of the program noticing.

### The four principles together

| Principle | One-line meaning | In the tracker |
|---|---|---|
| Encapsulation | The object guards its own data through its methods | `sell()` refuses to oversell; `_quantity` is internal |
| Abstraction | Expose what, hide how | Callers use `market_value()` without knowing the formula |
| Inheritance | A child is a specialized version of a parent | `Stock` and `Bond` are kinds of `Asset` |
| Polymorphism | Same request, each object answers its own way | One loop values stocks and bonds without checking types |

## Part 5: Modularity — why rewrite what someone already wrote?

### Writing it yourself

Net present value is a formula every finance student knows: discount each cash flow back to today and add them up. In Python, it fits in one line:

```python
def npv(rate, cash_flows):
    return sum(cf / (1 + rate) ** t for t, cf in enumerate(cash_flows))

npv(0.08, [-1000, 300, 400, 500])   # 17.63
```

A $1,000 investment returning $300, $400, and $500 over three years is worth $17.63 today at 8%. It works. But do you have to write every function you need yourself?

### Someone already wrote it

No. A published library called `numpy_financial` already contains this function and many others:

```python
import numpy_financial as npf

npf.npv(0.08, [-1000, 300, 400, 500])   # 17.63
```

NPV was easy to write. Now try **IRR**, the rate that makes NPV exactly zero. IRR has no formula you can solve directly. Computing it requires an iterative search (guess a rate, check, adjust, repeat until close enough), with care for edge cases. That's real work, and it has already been done:

```python
npf.irr([-1000, 300, 400, 500])   # 0.089, i.e. about 8.9%
```

This is **modularity**. Developers package routine, reusable work into **modules**, and application developers import those modules and don't rebuild them. It's build vs. buy: no firm writes its own payroll system from scratch. They buy one and spend their own effort on what makes them different. In the portfolio tracker, your effort goes into the portfolio rules, not into re-deriving financial math or building table-formatting code.

### The judgment lesson: read the contract

Try the same NPV in Excel: `=NPV(8%, -1000, 300, 400, 500)` returns **16.32**, not 17.63.

Neither answer is a bug. The two functions have **different contracts**. `npf.npv` treats the first cash flow as happening today (t = 0, undiscounted). Excel's `NPV` treats the first cash flow as happening one period from now (t = 1), so it discounts everything one extra year. Same name, same inputs, different assumptions, and a different answer.

Reusing someone else's code saves you from writing it. It does not save you from understanding what it promises. Read the documentation, and don't assume that a function does what its name suggests to you.

Choosing a dependency is also a form of vendor due diligence. `numpy_financial`, for instance, is only lightly maintained. That's fine for a class exercise, but for production software you'd ask: Is it actively maintained? Who publishes it? How widely is it used? These are the same questions to ask when an AI tool adds a library to your project on its own initiative.

## Part 6: How Python finds modules

### Vocabulary: module and package

- A **module** is a single `.py` file. Any Python file you write is a module that other files can import.
- A **package** is a folder of modules, marked by a file named `__init__.py` (often empty). Packages can contain other packages, which is how larger projects are organized.

A module is not the same thing as a class. A module is a *file*, and one module can hold several classes and functions. A class is a blueprint defined *inside* a module.

### Two ways to import

Suppose a module `rates.py` contains a function `discount_factor`:

```python
import rates
rates.discount_factor(5)          # use it through the module name

from rates import discount_factor
discount_factor(5)                # use it directly
```

`import numpy_financial as npf` is the first form with a shorter alias.

For modules inside a package, use the full path from the top of the project. This is called an **absolute import**:

```python
from portfolio_tracker.models.asset import Asset
```

This reads as: in the `portfolio_tracker` package, in its `models` sub-package, in the module `asset.py`, get the class `Asset`. Python also has *relative* imports (`from .asset import Asset`, where the dot means "this same folder"). You'll see them in other people's code, but this course uses absolute imports because they say exactly where things come from.

### The search: first match wins

When Python meets `import something`, it looks in a fixed order and uses **the first match it finds**. Picture a filing clerk who checks the drawers in the same order every time and hands you the first folder with the right label, even if it's the wrong folder.

1. **Already imported?** Python keeps a record of every module it has already loaded in this run (`sys.modules`). If it's there, Python reuses it. The report is already on your desk.
2. **Built-in modules.** A few modules are compiled into Python itself (for example `sys`).
3. **The folders on `sys.path`**, top to bottom:
   - first, **the folder of the script you ran** (or your current folder, when using `python -m` or the interactive prompt)
   - any folders listed in the `PYTHONPATH` environment variable, if it's set
   - **the standard library**, the modules that ship with Python (`random`, `math`, `csv`, …)
   - **`site-packages`**, where packages you install with pip live

You can see the list for yourself:

```python
import sys

for i, folder in enumerate(sys.path):
    print(i, folder)
```

Entry `0` is your script's folder, and near the bottom you'll find a path ending in `site-packages`. Almost every import error you'll hit is explained by this list.

### A module runs once, on first import

Importing a module *runs* it, top to bottom, the first time. After that, Python reuses the loaded copy. Here is `rates.py`:

```python
print(f"Loading rates.py (__name__ is {__name__!r})")

RISK_FREE_RATE = 0.045


def discount_factor(years):
    return 1 / (1 + RISK_FREE_RATE) ** years


if __name__ == "__main__":
    # Only runs when this file is executed directly, not when imported.
    print("Quick check:", discount_factor(1))
```

And `main.py`, in the same folder:

```python
import rates
import rates  # second import: Python reuses the cached module

from rates import discount_factor

print("5-year discount factor:", round(discount_factor(5), 4))
```

Running `python main.py` prints the "Loading" line **once**, even though `rates` is imported three times, followed by the discount factor. The quick-check line does *not* appear.

Running `python rates.py` directly prints the "Loading" line with `__name__ is '__main__'`, and then the quick check runs.

That's what `__name__` is for. Every module has a built-in variable `__name__`. When a file is imported, `__name__` is the module's name (`'rates'`). When a file is the one you ran directly, `__name__` is `'__main__'`. So:

```python
if __name__ == "__main__":
    ...
```

means **"only do this when this file is run directly, not when it's imported."** Without the guard, anything at the top level of a module (printing, starting a menu, running a test calculation) happens the moment another file imports it, which is almost never what you want. Every file that serves as a program's starting point, its **entry point**, should put its "start the program" code under this guard.

### Three errors the search order explains

**1. Shadowing: your file hides a library.** A student writes a price simulator:

```python
import random

price = 100
for day in range(5):
    price *= 1 + random.uniform(-0.02, 0.02)
    print(f"Day {day + 1}: {price:.2f}")
```

In the same folder, they also have a file of their own called `random.py`. Running the simulator fails:

```
AttributeError: module 'random' has no attribute 'uniform'
```

The script's own folder is first on `sys.path`, before the standard library, so `import random` found *their* `random.py` and never reached Python's. First match wins. The fix: **never name your files after existing modules or libraries** (`random.py`, `csv.py`, `math.py`, `numpy.py`, …).

**2. Wrong environment: "I installed it, but Python says it isn't there."** You install a package and still get `ModuleNotFoundError`. Usually the package went into one Python installation's `site-packages`, and a *different* Python is running your code. Your machine may have several Pythons on it. Part 7 explains why, and how to avoid it.

**3. Wrong entry point: running a file inside a package directly.** Suppose a project looks like this:

```
portfolio-tracker/              ← project root
  portfolio_tracker/            ← the package
    __init__.py
    main.py                     ← contains: from portfolio_tracker.models.asset import Asset
    models/
      __init__.py
      asset.py
```

From the project root, you run:

```
python portfolio_tracker/main.py
```

and get `ModuleNotFoundError: No module named 'portfolio_tracker'`. Apply the search rules: when you run a script by its path, `sys.path[0]` becomes *that script's folder*, which is `portfolio_tracker/`. Inside that folder there is no folder called `portfolio_tracker`, so the absolute import fails.

The fix is to run it as a module of its package, from the project root:

```
python -m portfolio_tracker.main
```

With `-m`, Python puts your current folder (the project root) at the front of `sys.path`, and `portfolio_tracker` is found right there. Note the dots instead of slashes and no `.py`: you're naming a module, not a file path.

## Part 7: The Python ecosystem

### PyPI and pip

**PyPI** (the Python Package Index, at pypi.org) is the public catalog of Python packages. Anyone can publish there, and it hosts hundreds of thousands of packages, including `numpy_financial`. Think of it as a vendor catalog.

**pip** is the procurement tool. It downloads a package from PyPI and installs it into a `site-packages` folder, where imports can find it (the last stop in the search order). The official term for a package your project relies on is a **dependency**, and managing those is pip's job.

Always run pip like this:

```
python -m pip install numpy-financial
```

and not just `pip install ...`. The `python -m` form guarantees the package is installed into *the same Python you're going to run your code with*, which prevents error #2 above. (The name you install, `numpy-financial`, and the name you import, `numpy_financial`, don't always match exactly. PyPI names allow hyphens, and Python module names can't contain them.)

### Virtual environments

Without further setup, every project on your machine shares one global `site-packages`. That causes real problems:

- Project A needs version 13 of a library, and project B needs version 15. Only one can be installed.
- Upgrading a library for one project silently breaks another.
- Your project works on your laptop because of something you installed globally months ago and forgot about. It fails on a teammate's machine, and nobody knows why. This is the classic "works on my machine" problem.

A **virtual environment** (venv) solves this by giving each project its own separate set of installed packages. The analogy is keeping separate books for each legal entity in a group. One subsidiary's entries never land in another's ledger, even though the same accounting team maintains them all.

Mechanically, a venv is not complicated. It's a folder (by convention named `.venv/`, inside your project) that contains its own Python and its own `site-packages`. **Activating the venv puts its `site-packages` on `sys.path` in place of the global one.** That's all it does.

```
python3 -m venv .venv            # create it (once per project)
source .venv/bin/activate        # activate it (macOS/Linux)
.venv\Scripts\activate           # activate it (Windows)
deactivate                       # leave it
```

Once it's active, your terminal prompt usually shows `(.venv)`. `which python` (macOS/Linux) or `where python` (Windows) shows a path inside `.venv`. On Windows, use `python` or `py` where these examples say `python3`. If PowerShell refuses to run the activate script, that's its execution policy blocking scripts, a common first-time hurdle.

The isolation works in both directions, as the class demo showed:

- A library installed **globally** is *not* available inside a fresh venv. Run code that imports it, and you get `ModuleNotFoundError`. Nothing is broken. That's the isolation working. Install it inside the venv with `python -m pip install ...`, and it works.
- A library installed **inside** the venv is *not* available after you `deactivate`. It lives only in that project's `.venv/`.

### `requirements.txt`: the bill of materials

If each project has its own environment, how does a teammate (or a grader, or a server) rebuild yours? You record what's installed in a file called `requirements.txt`:

```
python -m pip freeze > requirements.txt
```

`pip freeze` lists every package installed in the active environment with its exact version, and `>` saves that list to the file. A teammate then creates their own venv and runs:

```
python -m pip install -r requirements.txt
```

to install exactly the same packages at exactly the same versions.

You'll often find packages in `requirements.txt` that you never installed yourself. Install the `rich` library (which draws formatted tables in the terminal), freeze, and the file lists `rich` **plus** `markdown-it-py`, `mdurl`, and `Pygments`:

```
markdown-it-py==...
mdurl==...
Pygments==...
rich==...
```

(Exact version numbers will vary.) Those are *rich's own* dependencies, called **transitive dependencies**. The library you reused was itself built by reusing other libraries. Modularity goes all the way down.

### What goes into Git

**Never commit `.venv/` to Git.** It's large, it's specific to your machine and operating system, and it can be rebuilt at any time. Commit `requirements.txt` instead: share the recipe, not the cooked meal. Add `.venv/` to your project's `.gitignore` file so Git never picks it up by accident.

## Summary

- **One throughline: expose the "what," hide the "how."** It applies inside a class, across files, and across the whole Python ecosystem.
- **Overriding:** a child defines a method with the same name as the parent's, and the child's version wins. `Bond` overrides `market_value()`.
- **Overloading:** multiple versions of one method distinguished by their parameters. It's a general OOP concept (Java, C#). Python doesn't support it directly, because the last definition wins.
- **Encapsulation:** the object guards its own data. Rules live in methods, written once, next to the data. A leading underscore means "internal." It's a convention Python doesn't enforce.
- **Abstraction:** callers use what something does (`=IRR()`, `market_value()`) without knowing how it does it, so the "how" can change freely.
- **Polymorphism:** one request, each object answering its own way. A loop over mixed assets needs no type checks, and a new asset type means a new class, not edits everywhere.
- **Modularity:** reuse packaged work instead of rebuilding it (build vs. buy), but read the contract (`npf.npv` at t = 0 vs. Excel's `NPV` at t = 1) and do due diligence on what you depend on.
- **Module = file; package = folder with `__init__.py`.** Prefer absolute imports.
- **Import search order, first match wins:** already loaded → built-in → `sys.path` (script's folder, `PYTHONPATH`, standard library, `site-packages`).
- **A module runs once on first import.** `if __name__ == "__main__":` means "only when run directly."
- **Three classic errors:** shadowing (don't name files after libraries), wrong environment (use `python -m pip`, activate the venv), wrong entry point (use `python -m package.module` from the project root).
- **PyPI** is the catalog, **pip** installs from it, and a **venv** gives each project its own `site-packages`. **`requirements.txt`** records dependencies so anyone can rebuild the environment. Commit it, never `.venv/`.

## Glossary

- **Absolute import:** an import written with the full path from the top of the project (`from portfolio_tracker.models.asset import Asset`).
- **Dependency:** a package your project needs in order to run. A **transitive dependency** is a dependency of one of your dependencies.
- **Encapsulation / abstraction / inheritance / polymorphism:** the four principles of OOP (see the table in Part 4).
- **Entry point:** the module you run to start a program, with its start-up code under `if __name__ == "__main__":`.
- **Module:** a single `.py` file.
- **Overloading:** several same-named methods in one class, told apart by their parameters. Not directly supported in Python.
- **Overriding:** a child class replacing an inherited method by defining one with the same name.
- **Package:** a folder of modules marked by an `__init__.py` file.
- **pip:** the tool that installs packages from PyPI into `site-packages`. Run it as `python -m pip`.
- **PyPI:** the Python Package Index, the public catalog of installable Python packages.
- **`requirements.txt`:** a file listing a project's dependencies and versions, produced by `pip freeze` and installed with `pip install -r`.
- **Shadowing:** your own file with the same name as a library hiding that library from `import`.
- **`site-packages`:** the folder where installed packages live. Each venv has its own.
- **`sys.path`:** the ordered list of folders Python searches for imports.
- **Virtual environment (venv):** a per-project folder with its own Python and its own `site-packages`, so projects don't interfere with each other.
