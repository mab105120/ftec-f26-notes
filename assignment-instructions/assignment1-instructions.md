# Assignment 1 — Portfolio Tracker: Command-Line Edition

**Due:** Wednesday 10/21/26, before class starts (7:00 PM)
**Weight:** 20% of the course grade
**Submitted as:** a pull request in your own GitHub repository

## Objective

This semester you build one application in stages: a **personal investment portfolio tracker**. The user deposits cash, organizes investments into named portfolios (one per strategy, such as "Tech growth" or "Dividend income"), buys and sells assets inside a portfolio, and tracks gains and losses.

In this first milestone you build it as a **command-line application**. Later milestones replace the command line with a web API and then a web page. The business rules should survive those changes untouched, so the way you structure your code now matters as much as whether it runs.

You start from an empty folder. Nothing from class is handed out; build it yourself.

## Features

When the app starts, it shows a **welcome screen** followed by a **main menu**. The user picks an option, the app carries it out, and the menu comes back until the user chooses to quit. The app keeps all data in memory; it's fine that everything is lost when it exits.

### Assets
- **Add an asset** with a ticker and a name. A ticker that already exists is refused.
- **View assets**: every asset's ticker, name, and current price (or a clear indication that no price has been set).
- **Set an asset's current price.** An unknown ticker is refused. The price must be zero or more.

### Portfolios
- **Create a portfolio** with a name and an investment-strategy description. A name that already exists is refused.
- **View portfolios**: every portfolio's name and strategy, and which one is currently selected.
- **Select a portfolio.** All buys and sells go to the selected portfolio. Trading with no portfolio selected is refused with a clear message.
- **Delete a portfolio.** Refused while the portfolio still holds any shares. If all its positions have been sold, deletion is allowed. Its trade history and realized gains are removed with it, and the cash balance doesn't change. Deleting the selected portfolio clears the selection.
- **View the selected portfolio**: for each asset held, the quantity, average cost, current price, market value, and unrealized gain/loss. Then the portfolio's totals: market value, unrealized gain/loss, and realized gain/loss.

### Cash and trading
- **Deposit funds.** The amount must be greater than zero. There are no withdrawals. The user must be able to see their cash balance.
- **Buy** an asset in the selected portfolio, entering the quantity and price per share. The ticker must exist, and quantity and price must both be greater than zero. A buy that costs more than the cash balance is refused. A successful buy reduces cash by quantity × price.
- **Sell** an asset from the selected portfolio, entering the quantity and price per share. The quantity must be greater than zero and no more than the shares held **in that portfolio**. The price must be zero or more (selling a worthless position at $0 is allowed). A successful sell increases cash by quantity × price.

### Robustness
No input can crash the app: not an invalid menu choice, not text typed where a number belongs, not a broken rule. Every refusal tells the user what went wrong in plain language. Quitting exits cleanly.

## Calculation rules

These are fixed and not open to interpretation:

- **Average cost method.** A buy recalculates the position's average cost as the weighted average of what was paid. A sell reduces the quantity but leaves average cost unchanged.
- **Unrealized gain** = (current price − average cost) × quantity held.
- **Realized gain** on a sell = (sell price − average cost) × quantity sold. It stays in the portfolio's totals even after the position is fully sold.
- **Positions are per portfolio.** The same asset held in two portfolios is two independent positions, each with its own quantity and average cost.
- **One cash balance** is shared by all portfolios, and it can never go negative.

## Your decisions

Make each of these choices yourself, then state it and justify it in your README. Any reasonable choice earns full credit if your code follows it consistently and your reasoning is sound.

1. Are fractional shares allowed?
2. Is `aapl` the same asset as `AAPL`?
3. Do fully sold positions still appear in the portfolio view?
4. How does the portfolio view handle an asset with no current price set? (It must not crash.)
5. Which errors does the user see, and which are only written to the log?

## How your code must be built

This is where most of the grade comes from. Each practice below was covered in class; the goal is to show that you understand why it exists.

**Project setup**
- The app is a Python package, run with `python -m <your_package>.<entry_module>`. It is not a single script.
- Use a virtual environment. Every third-party library you use goes in `requirements.txt`, and nothing else is needed to run the app.
- Module (file) names are lowercase.
- The virtual environment folder, `__pycache__`, and log files are not committed.

**Structure and separation of responsibilities.** Your package must have separate areas for:
- the **domain objects**: what the things in the system *are*.
- the **business rules and calculations**, one **service** module per area of the business. These are *what the user can do*.
- your **custom errors**.
- the **command-line interface**.
- a small **entry point** that starts the app.

You choose every file, class, function, and variable name. Naming is graded. The command-line interface only asks the user for input and displays results: it contains no business rules and no calculations. The business-rule code never calls `input()` or `print()`. Current prices are kept separate from asset definitions, because in a later milestone they will come from a live market-data feed. To check your structure, ask yourself: *could I replace the command line with a web page without touching a single business rule?*

**Exceptions**

- Define one base error type for your project, with specific subclasses for different broken rules (duplicate, not found, insufficient funds, and so on). Make them specific enough that the code catching them could react differently to each.
- Raise errors where the rule lives, and catch them at the edge (the command-line interface), where the user is told.
- Turn low-level Python errors into meaningful business errors where it helps (for example, a failed dictionary lookup becomes "no asset with ticker MSFT").
- Catch specific exception types, with no bare `except:`. Keep `try` blocks narrow, and never swallow an error silently.
- A broad catch of unexpected errors is allowed only at the outermost level. It must log the full traceback and show the user a friendly message.

**Logging**
- Configure logging once, at the entry point. Write the log to a file so the user's screen stays clean.
- Each module that logs has its own logger.
- Log key operations at sensible levels, both successful ones and rejected ones. Log an error where it is handled, not at every layer it passes through.

**Readability.** Names explain themselves. Functions are small and each does one job. Comments explain *why*, not *what*.

**Git.** Commit in small, meaningful steps as you build. Don't put everything in one final commit; your commit history is part of the grade.

## README

Include a `README.md` at the root of your repository with:

1. how to set up the environment and run the app.
2. your package layout, with one line per module explaining what it is responsible for.
3. your five decisions from "Your decisions", each with its justification.

## Submission

1. Create a GitHub repository and add the instructor as a collaborator (GitHub username: `mab105120`).
2. Do your work on a feature branch, not on `main`.
3. Open a pull request from your branch into `main` with a short description of what it contains. **Do not merge it.**
4. Submit the pull request link on Teams by the deadline.

## How you'll be graded


| Area | Weight | What's assessed |
| --- | --- | --- |
| Features and rules | 35% | Every feature works, rules are enforced, and the cash and gain calculations are correct |
| Structure and separation | 20% | Clear package areas; the interface has no business rules; the business code doesn't talk to the user |
| Exceptions | 15% | A meaningful error hierarchy, raised in the core and caught at the edge, with nothing swallowed |
| Logging | 10% | Configured once, a logger per module, sensible levels, successful and rejected operations recorded |
| Readability and project setup | 10% | Naming, small functions, lowercase modules, venv/requirements |
| Git and README | 10% | Meaningful commit history, a PR description, and decisions stated and justified |

**Late work:** there is no grace period. Work submitted up to two weeks after the deadline loses 20%; work submitted later than that loses 30%.

## Out of scope

There is no database or saving between runs, no live market prices, and no automated tests; those come in later milestones. You also don't need a separate storage layer yet: keeping data inside your business-rule modules is fine for now. There are no withdrawals, fees, dividends, short selling, multiple users, or multiple currencies. There is also no account-wide overview or trade-history screen.

## AI and collaboration

- **You may not use AI tools to write your code.** You *may*, and are encouraged to, use AI as a teacher: ask it to explain a concept (classes, exceptions, logging, imports) or to help you understand an error message. Later you'll build software *with* AI agents, and you can only judge an agent's code if you understand the fundamentals well enough to have written it yourself.
- You may discuss concepts and approaches with classmates out loud. You may not copy code, look at another student's repository or pull request, or share your solution. Submissions with suspiciously similar code will be reviewed.
