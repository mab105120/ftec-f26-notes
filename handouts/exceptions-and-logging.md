# Exceptions and Logging

## Why this topic, and why now

This course is moving you from using Python as a *scripting tool* to using it as a *programming language*. A scripting tool is a file that runs top to bottom to answer one question. A programming language is a way to build a system that keeps growing. The system you are building this semester is a personal portfolio tracker. The user deposits cash, creates named portfolios (one per investment strategy, such as "Tech growth" or "Dividend income"), adds assets such as stocks, buys and sells them inside a portfolio, and sees what they hold and how much they have gained or lost.

So far you have learned to model that world with classes and objects, to organize code into modules and packages, and to manage the tools around it with virtual environments, pip, and `requirements.txt`. All of that assumed things go right. This handout is about what happens when they don't.

A script that crashes on bad input is an inconvenience: you fix the input and run it again. An application that crashes is a failure. A user who types "abc" where a number belongs, or tries to sell shares they don't own, should get a clear message and carry on, not lose everything they entered. And when something does go wrong, someone needs to find out what happened afterwards, without having been there to watch.

Those are two different jobs, and Python has a tool for each:

- **Exceptions** decide *what happens* when something goes wrong.
- **Logging** records *what happened*, so it can be investigated later.

Part A covers exceptions; Part B covers logging. A final section follows one failed operation through both.

---

# Part A: Exceptions

## 1. What an exception is

When Python hits a line it cannot carry out, it stops normal execution and creates an **exception**: an object describing what went wrong. Three common examples:

```python
int("abc")            # ValueError: invalid literal for int() with base 10: 'abc'
{"AAPL": 190}["MSFT"] # KeyError: 'MSFT'
100 / 0               # ZeroDivisionError: division by zero
```

If no code deals with the exception, the program stops and prints a **traceback**. That is Python's default: an unhandled problem ends the program. The goal of this part is to make that a deliberate choice instead of an accident.

An exception is a regular Python object, created from a class. `ValueError`, `KeyError`, and `ZeroDivisionError` are all classes, and the exception you see is an instance of one of them. This matters later: because exceptions are classes, you can write your own, and inheritance works on them like any other class.

### Common built-in exceptions

| Exception | Typical cause | Example |
| --- | --- | --- |
| `ValueError` | The right type of value, but an unacceptable one | `int("abc")`, `float("12,5")` |
| `TypeError` | An operation on the wrong type | `"Price: " + 190` |
| `KeyError` | A dictionary key that doesn't exist | `prices["MSFT"]` when MSFT was never added |
| `IndexError` | A list position that doesn't exist | `holdings[5]` on a three-item list |
| `ZeroDivisionError` | Dividing by zero | average cost when the quantity is 0 |
| `AttributeError` | An attribute or method the object doesn't have | `account.cash_balanse` (a typo) |
| `NameError` | A variable name that was never defined | using `totl` instead of `total` |
| `FileNotFoundError` | Opening a file that isn't there | `open("prices.csv")` in the wrong folder |
| `ModuleNotFoundError` | An import Python can't find | `import rich` outside the virtual environment |

You don't need to memorize this table. You need to recognize the names when you see them and know where to look.

## 2. Reading a traceback

A traceback looks intimidating, but it has a fixed structure, and you read it **bottom-up**. Here is a small program with a mistake in it:

```python
prices = {"AAPL": 190.0}

def get_price(ticker):
    return prices[ticker]

def market_value(ticker, quantity):
    return get_price(ticker) * quantity

print(market_value("MSFT", 10))
```

Running it produces:

```
Traceback (most recent call last):
  File "/Users/you/tracker/value.py", line 9, in <module>
    print(market_value("MSFT", 10))
          ^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/you/tracker/value.py", line 7, in market_value
    return get_price(ticker) * quantity
           ^^^^^^^^^^^^^^^^^
  File "/Users/you/tracker/value.py", line 4, in get_price
    return prices[ticker]
           ~~~~~~^^^^^^^^
KeyError: 'MSFT'
```

1. **The last line** says *what* went wrong: the exception type (`KeyError`) and its message (`'MSFT'`). Start here.
2. **The entry just above it** says *where*: file, line number, function name, and the line of code itself. This is where the exception happened, though not always where the mistake is.
3. **The entries above that** show *how the program got there*: `market_value` was called from line 9, and it called `get_price`. This chain of calls is the **call stack**. "Most recent call last" means the deepest call is at the bottom.

Reading it this way turns a wall of text into a short story: *line 9 asked for the market value of MSFT, which asked for its price, and MSFT isn't in the price dictionary.* Whenever you're stuck on an error, read it bottom-up first.

## 3. Bugs versus exceptional situations

Not every exception should be handled. The most important judgment in this topic is telling two kinds of problem apart.

A **bug** is a mistake in your code: a typo, wrong math, a misspelled attribute. The traceback for `account.cash_balanse` is telling you to fix the spelling. Wrapping that line in error handling would hide the mistake and leave it in the code. **Bugs get fixed, not handled.**

An **exceptional situation** is the world not cooperating, even though your code is correct. The user types "abc" for a deposit amount. They try to buy a ticker that was never added. A file someone was supposed to provide is missing. You can't prevent these, because they come from outside your code. You can only **anticipate them and decide what should happen.**

The business analogy is a bank teller who gets a request they can't approve, such as a withdrawal above their limit. They don't guess, and they don't quietly ignore it. They **escalate** it to someone with the authority and context to decide. In Python, `raise` is the escalation, and `except` is where someone makes the decision. The rest of Part A covers both.

## 4. Handling an exception: `try` and `except`

A `try` block marks code that might fail. An `except` block says what to do if it does:

```python
text = input("Amount to deposit: ")
try:
    amount = float(text)
except ValueError:
    print(f"'{text}' is not a number.")
```

Python runs the `try` block. If it finishes without a problem, the `except` block is skipped. If a `ValueError` occurs, Python abandons the rest of the `try` block immediately, jumps to the matching `except`, and runs it. Execution then continues after the whole `try`/`except`, and the program doesn't crash.

A common pattern combines this with a loop, so the user keeps being asked until the input is usable:

```python
def ask_number(prompt):
    while True:
        text = input(prompt)
        try:
            return float(text)
        except ValueError:
            print(f"'{text}' is not a number, try again.")
```

`ask_number("Quantity: ")` either returns a real number or asks again. Every place in the interface that needs a number can call it instead of repeating the same handling.

To use the exception object itself, usually to show its message, name it with `as`:

```python
try:
    price = prices[ticker]
except KeyError as error:
    print(f"No price for {error}")
```

### Rule 1: catch specific exception types

Always name the exception you expect. The tempting alternative is a **bare `except:`** with no type:

```python
try:
    amount = float(text)
except:                    # don't do this
    print("Something went wrong")
```

A bare `except` catches *everything*, including bugs. If you misspell `text` as `txt`, the resulting `NameError` is caught and reported as "something went wrong", and the bug disappears from view. It even catches the user pressing Ctrl+C to stop the program. Catching a specific type says exactly which problem you've planned for, and everything else still surfaces.

If two different types call for the same response, list them together in parentheses:

```python
except (ValueError, TypeError):
```

### Rule 2: keep `try` blocks narrow

Wrap only the line that can raise the exception you expect, not a whole block of code:

```python
# Too wide: a ValueError from anywhere in here is reported as "not a number"
try:
    amount = float(text)
    fee = calculate_fee(amount)
    record_deposit(amount - fee)
except ValueError:
    print("Not a number")

# Narrow: only the conversion is protected
try:
    amount = float(text)
except ValueError:
    print("Not a number")
```

In the wide version, if `calculate_fee` has a bug that raises `ValueError`, the user is told they typed something wrong when the program is at fault. A narrow `try` block keeps the handling truthful.

### Rule 3: never swallow an exception silently

```python
try:
    amount = float(text)
except ValueError:
    pass                   # the problem vanishes
```

`pass` means "do nothing". The user isn't told, nothing is recorded, and the program carries on as if all were well, often with missing or wrong data. A silent failure is worse than a crash: a crash is at least visible. Every `except` block should do something meaningful, such as tell the user, retry, use a sensible default deliberately, or record the problem (Part B).

### `if` or `try`?

You can often check a condition in advance with `if` instead of using `try`. Both are legitimate:

- Use **`if`** for rules you own and can state directly: "the quantity must be greater than zero."
- Use **`try`** when checking in advance is awkward or unreliable. To know whether `"12.5x"` is a valid number, the simplest reliable test is to try converting it.

### Two optional clauses: `else` and `finally`

A `try` statement can have two more parts. You won't need them often, but you'll see them in other people's code:

```python
try:
    amount = float(text)
except ValueError:
    print("Not a number")
else:
    print(f"Depositing {amount}")      # runs only if no exception occurred
finally:
    print("Done with this input")      # runs no matter what
```

`else` holds code that should run only when the `try` block succeeded, which keeps the `try` block itself narrow. `finally` runs whether or not an exception occurred, which suits cleanup work such as closing a connection.

## 5. Exceptions are a family tree

Since exceptions are classes, they use inheritance. Python's built-in exceptions form a hierarchy. A simplified slice:

```
BaseException
├── KeyboardInterrupt        (the user pressed Ctrl+C)
└── Exception
    ├── ValueError
    ├── TypeError
    ├── ArithmeticError
    │   └── ZeroDivisionError
    └── LookupError
        ├── KeyError
        └── IndexError
```

**An `except` clause catches the named class and every class below it.** `except LookupError` catches both `KeyError` and `IndexError`. `except Exception` catches nearly everything that signals a problem in your program, but not `KeyboardInterrupt`, which sits outside `Exception` so that Ctrl+C still works. A bare `except:` catches even that, which is one more reason to avoid it.

When a `try` has several `except` clauses, Python checks them top to bottom and uses the **first match**. Put the specific ones first:

```python
try:
    ...
except KeyError:
    ...   # specific case
except LookupError:
    ...   # anything else in the lookup family
```

If `LookupError` came first, the `KeyError` clause would never run. The next section uses this same family-tree idea for your own errors.

## 6. Raising your own exceptions

So far, Python has done the raising. Your application has rules Python knows nothing about: a portfolio name must be unique, a deposit must be greater than zero, a ticker must exist before you can buy it. Nothing in Python stops someone from depositing −500. Your code has to check, and when a rule is broken it reports the problem with **`raise`**:

```python
def deposit(amount):
    if amount <= 0:
        raise ValueError(f"Deposit amount must be greater than zero (got {amount})")
    ...
```

`raise` creates the exception and immediately stops the function, so the line that would have changed the balance never runs. Breaking a rule can't half-succeed.

### Why define custom exceptions

Reusing built-in types such as `ValueError` for business rules works, but it is ambiguous. A `ValueError` might mean the user broke a rule, or it might mean a conversion failed somewhere deep in a library. The code catching it can't tell which. The fix is to define your own exception classes, which takes one line each:

```python
class TrackerError(Exception):
    """Base class for every business rule the tracker enforces."""


class DuplicateError(TrackerError):
    """Something with this name or ticker already exists."""


class NotFoundError(TrackerError):
    """No portfolio or asset with this name or ticker exists."""


class InvalidAmountError(TrackerError):
    """A quantity, price, or cash amount is out of range."""
```

A custom exception needs no body beyond a docstring: inheriting from `Exception` gives it everything it needs. Raising one works exactly like raising a built-in:

```python
def add_asset(ticker, name):
    if ticker in assets:
        raise DuplicateError(f"Asset '{ticker}' already exists")
    assets[ticker] = Asset(ticker, name)
```

### Why a base class

The structure is deliberate: one **project base error** (`TrackerError`) with specific subclasses beneath it. The family-tree rule from Section 5 makes this pay off twice:

- Code that wants to handle *every* broken business rule the same way catches the base: `except TrackerError`. That one clause covers duplicates, missing items, invalid amounts, and any subclass you add later.
- Code that needs to react to *one* situation differently can still catch that subclass specifically.

The base class also draws a clear line. A `TrackerError` always means "a business rule was refused": an expected, explainable outcome. Anything that is *not* a `TrackerError` is a bug or a genuine surprise. That line becomes important at the edge (Section 8).

A company's expense policy works the same way. Every rejected claim gets a "rejected" stamp, so the finance team can count rejections. Each one also carries a specific reason code (missing receipt, over limit, wrong category), so the right person can follow up.

### How specific should an error be?

Make a new subclass when the code catching it might need to **react differently**. "Insufficient funds" and "not found" call for different follow-ups, so they deserve different classes. Duplicate portfolio and duplicate asset are handled the same way, so one `DuplicateError` can cover both. The particulars (which ticker, which amount) go in the **message**, not the class name. A tree of fifty error classes is as unhelpful as one generic error.

Messages are for people. Write them in business language and include the specific value: `"No asset with ticker 'MSFT'"` tells the user exactly what to fix, while `"Lookup failed"` does not.

Common checks can live in a small helper so the rule is written once:

```python
def require_positive(label, value):
    if value <= 0:
        raise InvalidAmountError(f"{label} must be greater than zero (got {value})")
```

## 7. Translating low-level errors into business errors

Some exceptions come from how you *implemented* something, not from the business. If assets are stored in a dictionary, looking up a missing ticker raises `KeyError`. But "KeyError" is a Python detail. The business fact is "there is no asset with ticker MSFT". At the point where the low-level error happens, catch it and raise the meaningful one instead:

```python
def find_asset(ticker):
    try:
        return assets[ticker]
    except KeyError:
        raise NotFoundError(f"No asset with ticker '{ticker}'") from None
```

This is about more than nicer wording. Code that calls `find_asset` shouldn't need to know assets are kept in a dictionary. Later in the semester, assets will be stored in a database, which raises completely different low-level errors when something is missing. Because `find_asset` translates the error, its callers will keep receiving the same `NotFoundError` and won't need to change. This is the "expose the what, hide the how" idea from earlier in the semester, applied to errors.

**Deeper dive.** `from None` tells Python not to attach the original `KeyError` to the new exception's traceback. Without it, the traceback shows both errors with "During handling of the above exception, another exception occurred." The `KeyError` is no longer useful once it has been replaced by something clearer.

## 8. Raise in the core, catch at the edge

You can now raise exceptions and catch them. The remaining question is *where* each one belongs, and the answer is the core design idea of this topic.

### How an exception travels

When an exception is raised and the current function has no matching `except`, the function stops and the exception passes to the function that called it. If that caller doesn't handle it either, it passes up again, and so on up the call stack (the chain you read in the traceback) until some function catches it. If nothing does, the program crashes. So an exception raised deep inside your code can be handled several calls further up, by code that never touched the problem directly.

### The core and the edge

Picture your application in two parts:

- The **core** holds the business rules and calculations: depositing, adding an asset, buying, computing holdings. In the tracker, that's the service modules.
- The **edge** is where the application meets the outside world: the command-line interface (CLI) that asks for input and displays results.

The rule is: **raise in the core, catch at the edge.**

The core knows the business rules, so it is the right place to *detect* a broken one. But it doesn't know who it is talking to, so it can't decide how to *tell* them. The core never prints. It raises an exception and lets the problem travel outwards.

The edge knows exactly how to talk to the user. It catches the exception and presents it in whatever way suits that interface:

```python
def run():
    while True:
        print(MENU)
        choice = input("Choose an option: ").strip()
        try:
            if choice == "1":
                create_portfolio()
            elif choice == "2":
                add_asset()
            # ... the other menu options ...
            elif choice == "0":
                break
            else:
                print(f"'{choice}' is not an option.")
        except TrackerError as error:
            print(f"Sorry: {error}")
```

One `except TrackerError` at the edge handles every broken business rule from every menu option. The user sees the message, the loop continues, and the menu comes back.

### Why this matters

This semester, the tracker changes interfaces twice. It starts as a CLI, becomes a web API, and ends as a web page. With "raise in the core, catch at the edge", the core doesn't change at all. The same `DuplicateError` becomes a printed message in the CLI, an error response from the API, and a red banner on the web page. **Same core, three edges.** If the core printed its own messages, every one of those prints would have to be found and rewritten for each new interface, and that is exactly the kind of tangle that makes software expensive to change.

A useful test for any design: *could I replace the command line with a web page without touching a single business rule?* If the core raises and the edge catches, the answer is yes.

This is also why business-rule code should never call `input()` or `print()`. Asking and telling are the edge's job.

## 9. The one place a broad catch is right

Everything so far handles *expected* problems. Bugs still happen, and an unexpected exception reaching the top of the program crashes it, showing the user a raw traceback that means nothing to them. There is exactly one place where catching everything is correct: the **outermost level**, the entry-point function that starts the application:

```python
def main():
    try:
        run()
    except Exception:
        # record the full details (Part B), then tell the user plainly
        print("Something unexpected went wrong. Details were written to the log.")
```

This is a safety net, not a way to hide problems, and it comes with two conditions:

1. **It's only at the outermost level.** A broad catch anywhere deeper would hide bugs from the code that could react to them.
2. **The full details are always recorded.** The user gets a friendly message, but the traceback must be kept somewhere a developer can find it. Catching everything without recording it is Rule 3's silent failure on a larger scale. Part B shows how to record it.

After an unexpected error, the program stops cleanly rather than carrying on. Something happened that the code didn't plan for, so the data may be in an inconsistent state. Stopping is safer than continuing with bad numbers.

---

# Part B: Logging

## 10. Why `print` isn't enough

Exceptions decide what happens. But when a user reports "it said something went wrong yesterday afternoon", how do you find out what happened? The program has finished, the screen has been cleared, and you weren't watching.

`print` can't solve this, because `print` talks to the **user**, right now, on their screen. What's missing is a record for a different audience: the **developer or operator**, later, investigating what happened. Mixing the two fails both. The user's screen fills with internal detail they don't care about, and the record disappears when the window closes.

The finance analogy is a receipt versus a transaction journal. The customer gets a receipt that is short, relevant to them, and theirs to keep. Operations keeps a journal of every transaction, timestamped, including the rejected ones, so any dispute can be reconstructed afterwards. `print` is the receipt and logging is the journal.

**Logging** is Python's built-in system for keeping that journal. It is in the standard library, so there's nothing to install: `import logging`.

## 11. Log levels

Every log message has a **level** that says how serious it is:

| Level | Use it for | Tracker example |
| --- | --- | --- |
| `DEBUG` | Fine-grained detail, useful only while investigating a problem | "Computing holdings from 14 transactions" |
| `INFO` | Normal, successful operations | "Deposited 500.00; balance is now 1500.00" |
| `WARNING` | Something was refused or looks off, but the program handled it | "Rejected: Asset 'AAPL' already exists" |
| `ERROR` | An operation failed in a way the program didn't plan for | An unexpected exception caught at the outermost level |
| `CRITICAL` | The application can't continue at all | Rarely used in a small app |

There are two separate decisions. **Each message's level** is chosen by the code that writes it. **Which levels actually get recorded** is set once, in the configuration. With the threshold set to `INFO`, `INFO` and everything above it is recorded and `DEBUG` messages are skipped. You can leave detailed `DEBUG` messages in your code and switch them on only when investigating, without editing every file.

Choosing levels is a judgment call. In the tracker, a refused business rule is a `WARNING`: nothing broke, the program did its job, but the attempt is worth noting. An unexpected exception is an `ERROR`. A successful deposit is `INFO`.

## 12. Configure once, at the entry point

Before logging does anything useful, it must be configured: where messages go, the minimum level, and what each line looks like. This is done **once**, in the entry point that starts the application, before anything else logs:

```python
import logging

def main():
    logging.basicConfig(
        filename="tracker.log",
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    )
    ...
```

- **`filename`** sends the log to a file instead of the screen. For a CLI app this is essential: the user's screen stays clean, and the record survives after the program closes.
- **`level`** is the threshold from Section 11.
- **`format`** describes each line. `asctime` is the timestamp, `levelname` the level, `name` the logger that wrote it (Section 13), and `message` the text.

Why only once, and only at the entry point? Configuration is a decision about the *whole application*: one destination, one threshold, one format. If several modules each configured logging, they would compete, and there would be no single place to change it. There's also a practical trap: **`basicConfig` only takes effect the first time it's called.** Later calls are silently ignored. If some module happens to configure logging before `main()` does, your settings never apply and you'll wonder why the log file is empty. One call, at the start of `main()`, avoids this.

If logging is never configured, Python's default is to show only `WARNING` and above, on the screen. If your `INFO` messages seem to vanish, check that `basicConfig` ran first.

## 13. One logger per module

Configuration happens once, but *writing* messages happens everywhere. Each module that logs creates its own **logger** at the top of the file:

```python
import logging

logger = logging.getLogger(__name__)
```

`__name__` is the module's own import name, such as `portfolio_tracker.services.account_service`. Each logger is named after the module it lives in, so every line in the log says which part of the application wrote it:

```
2026-10-14 19:42:10,113 INFO portfolio_tracker.services.account_service: Deposited 500.00; balance is now 500.00
2026-10-14 19:42:31,870 WARNING portfolio_tracker.cli: Rejected: Asset 'AAPL' already exists
```

When you're investigating a problem, you know immediately where to look. All these loggers send their messages to the single destination set up in Section 12.

A small surprise: when you start the app with `python -m portfolio_tracker.main`, the entry module's logger shows up as `__main__`, not `portfolio_tracker.main`. Earlier this semester you learned that the module Python runs directly is named `__main__`; the logger simply reports that name.

## 14. Writing log messages

Each logger has one method per level: `logger.debug(...)`, `logger.info(...)`, `logger.warning(...)`, `logger.error(...)`, `logger.critical(...)`.

```python
def deposit(amount):
    require_positive("Deposit amount", amount)
    account.cash_balance += amount
    logger.info("Deposited %.2f; balance is now %.2f", amount, account.cash_balance)
```

The `%.2f` and `%s` placeholders are filled in from the arguments that follow the message, in order. This style is preferred over an f-string because the message is only assembled if that level is actually recorded. An f-string is always built, even when the message is about to be skipped. Both work; `%` placeholders are the logging convention.

### What to log

Log the **key operations**, both the ones that succeed and the ones that are refused. A good test: from the log alone, could someone reconstruct what the user did and what the application decided? A log of a short session might read:

```
... INFO __main__: Tracker started
... INFO portfolio_tracker.services.portfolio_service: Created portfolio Tech growth
... INFO portfolio_tracker.services.account_service: Deposited 5000.00; balance is now 5000.00
... INFO portfolio_tracker.services.asset_service: Added asset AAPL (Apple Inc.)
... WARNING portfolio_tracker.cli: Rejected: Asset 'AAPL' already exists
... INFO portfolio_tracker.services.price_service: Set current price of AAPL to 190.00
```

Refusals belong in the log as much as successes do. A pattern of repeated rejected operations is often the first sign of a confusing interface or a misunderstood rule.

Don't log sensitive information such as passwords or personal financial identifiers. Log files are often read by more people than the application's users, and they tend to be kept for a long time.

### Log an error where it is handled

An exception that travels up the call stack passes through several functions. Log it **once, at the point where it is handled**, not in every function it passes through. If the core logged a refusal and then raised it, and the edge logged it again when catching it, every refusal would appear twice and the log would overstate what happened. The edge is where the decision is made, so that's where it is recorded:

```python
except TrackerError as error:
    logger.warning("Rejected: %s", error)
    print(f"Sorry: {error}")
```

This mirrors the division of labor from Section 8: the core raises, and the edge catches, tells the user, and records the decision.

### Users see some errors; the log sees all of them

Not every problem needs to be shown to the user, but every problem should be in the log. The user needs messages they can act on: "No asset with ticker 'MSFT'" tells them what to fix. A developer needs the full story, including details that would mean nothing to the user. Deciding which errors the user sees and which are only logged is a design decision, and you should be able to explain the reasoning behind yours.

## 15. Recording unexpected errors: `logger.exception`

This completes the safety net from Section 9. Inside an `except` block, `logger.exception(...)` writes an `ERROR`-level message **plus the full traceback**:

```python
def main():
    logging.basicConfig(
        filename="tracker.log",
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    )
    logger.info("Tracker started")
    try:
        run()
    except Exception:
        logger.exception("Unexpected error; tracker stopped")
        print("Something unexpected went wrong. Details were written to tracker.log.")
```

Suppose the deposit code contains a typo, `account.cash_balanse`. The user sees one calm sentence. The log file holds everything a developer needs: the timestamp, the message, and the complete traceback pointing at the misspelled line. Each audience gets what it needs, and nothing is lost.

`logger.exception` only belongs inside an `except` block, because that is the only place there is a traceback to record.

## 16. Log files and Git

A log file is a record of one run on one machine. It isn't source code, it changes every time the program runs, and it can contain data that shouldn't be shared. **Don't commit log files.** Add a pattern such as `*.log` to your `.gitignore`, alongside your virtual environment folder and `__pycache__`.

---

## 17. Putting it together: one refused operation, end to end

Follow a single mistake through everything in this handout. The user chooses "Add asset" and enters `AAPL`, which already exists.

1. **The edge asks.** The CLI's add-asset function reads the ticker and name with `input()` and calls the core's `add_asset`. It checks no rules itself.
2. **The core detects and raises.** `add_asset` sees `AAPL` is already in its records and runs `raise DuplicateError("Asset 'AAPL' already exists")`. The function stops; nothing is overwritten.
3. **The exception travels.** `add_asset` has no `except`, so the exception passes to the CLI function that called it, which has none either, and then up to the menu loop.
4. **The edge catches.** The menu loop's `except TrackerError` matches, because `DuplicateError` is a subclass of `TrackerError`.
5. **The edge records and tells.** It logs `WARNING ... Rejected: Asset 'AAPL' already exists` and prints `Sorry: Asset 'AAPL' already exists`.
6. **The program carries on.** The loop continues and the menu comes back.

Now the version where something unexpected happens instead, a bug in the core:

1. The core raises an `AttributeError`. It isn't a `TrackerError`, so the menu loop's `except` doesn't match, and that is correct, because this isn't a business rule.
2. It travels all the way up to `main()`, where `except Exception` catches it.
3. `logger.exception` writes the full traceback to the log; the user sees one friendly sentence; the program stops cleanly.
4. A developer opens the log, reads the traceback bottom-up, and fixes the bug.

In both cases each part of the program does one job. The core enforces rules and raises. The edge decides how to tell the user. Logging keeps the record. That separation is what lets the interface change later without touching the rules.

---

## Summary

- An **exception** is an object Python raises when a line can't be carried out. Unhandled, it stops the program and prints a **traceback**, which you read **bottom-up**: what went wrong, then where, then how the program got there.
- **Bugs** get fixed. **Exceptional situations** (bad input, broken business rules, missing files) get anticipated and handled.
- `try`/`except` handles an exception. Catch **specific** types, keep `try` blocks **narrow**, and never **swallow** an exception silently.
- Exceptions form a **hierarchy**. An `except` catches the named class and all its subclasses, so put specific clauses first.
- Use `raise` to report a broken business rule. Define **custom exceptions** under one **project base error**, with subclasses specific enough that the catcher could react differently. Put the details in the message.
- **Translate** low-level errors (such as `KeyError`) into business errors where they happen, so callers don't depend on how the data is stored.
- **Raise in the core, catch at the edge.** The core detects and raises; the interface catches and presents. This is what lets the interface be replaced without touching the rules.
- A broad `except Exception` belongs only at the **outermost level**, and it must always record the full traceback.
- **Logging** keeps a record for developers, separate from what the user sees. Messages have **levels**; the configuration sets which are recorded.
- **Configure logging once**, at the entry point, writing to a file. Create **one logger per module** with `logging.getLogger(__name__)`.
- Log key operations, **successful and refused**. Log an error **once, where it is handled**. Use `logger.exception` in the outermost catch. Keep log files out of Git.

## Check your understanding

1. A traceback ends with `ZeroDivisionError: division by zero`, and the entry above it points to a line in a function called `average_cost`. What do you know about the problem, and where do you look first?
2. A classmate wraps their whole menu function in `try: ... except: pass` "so it never crashes." Name two distinct problems with this.
3. Is a misspelled variable name a bug or an exceptional situation? What about a user typing a negative quantity? What should the code do in each case?
4. Why does the project define one base error class instead of having every rule raise its own unrelated exception type?
5. Your price lookup uses a dictionary and raises `KeyError` for unknown tickers. Why translate that into a `NotFoundError`, and what will that protect when the data moves into a database?
6. Explain "raise in the core, catch at the edge" and how it lets the same business rules work behind a CLI, a web API, and a web page.
7. When is `except Exception` the right choice? What must always accompany it?
8. A colleague's log shows every rejected operation twice. What principle was broken, and how would you fix it?
9. Your `logger.info` messages aren't appearing anywhere. Name two likely causes.
10. Why is `print` the wrong tool for recording what the application did, even if you redirected it to a file?

## Glossary

- **Exception** — an object describing an error, created ("raised") when Python or your code can't continue normally.
- **Traceback** — the report printed when an exception isn't handled; read bottom-up.
- **Call stack** — the chain of function calls that led to the current line.
- **Bug** — a mistake in the code; fixed, not handled.
- **Exceptional situation** — a problem from outside the code (bad input, a broken rule, a missing file); anticipated and handled.
- **`try` / `except`** — the statement that runs code which might fail, and says what to do if it does.
- **Bare `except`** — an `except` with no exception type; catches everything, including bugs and Ctrl+C. Avoid it.
- **`raise`** — the statement that creates an exception and stops the current function.
- **Custom exception** — an exception class you define, usually by inheriting from `Exception` or from your project's base error.
- **Base error** — the single parent class for all of a project's business-rule exceptions.
- **Error translation** — catching a low-level exception and raising a meaningful business exception in its place.
- **Core** — the part of the application holding business rules and calculations.
- **Edge** — the part of the application that talks to the outside world, such as the command-line interface.
- **Logging** — recording what the application does, for developers and operators, separately from what the user sees.
- **Log level** — the seriousness of a log message: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`.
- **Logger** — the object a module uses to write log messages, created with `logging.getLogger(__name__)`.
- **`basicConfig`** — the one-time setup of logging's destination, threshold, and format, done at the entry point.
- **`logger.exception`** — writes an `ERROR` message plus the full traceback; used inside an `except` block.
