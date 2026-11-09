# Assignment 2 — Portfolio Tracker: Database and Automated Tests

**Due:** Sunday 11/22/26, 11:59 PM Central Time
**Weight:** 20% of the course grade
**Submitted as:** a pull request in the same GitHub repository you used for Assignment 1

## Objective

This semester you are building one application in stages: a **personal investment portfolio tracker**. The user deposits cash, organizes investments into named portfolios, buys and sells assets inside a portfolio, and tracks gains and losses. In Assignment 1 you built it as a command-line application that kept everything in memory, so every portfolio, asset, and trade disappeared when the app closed.

In this milestone the tracker gets a memory. You replace the in-memory storage with a **MySQL database**, modeled with **SQLAlchemy**, so that everything the user records is still there the next time the app starts. You then add **automated tests** with **pytest** that check your business rules and calculations against a temporary in-memory database, never against your real data.

The command line stays the interface, and every feature and rule from Assignment 1 still applies. The point of this milestone is the payoff of the structure you built last time: the way your data is stored changes completely, while your business rules barely change at all, and your tests prove it. This is also a chance to fix anything flagged in your Assignment 1 feedback.

## What changes for the user

From the user's point of view, the app works the same as in Assignment 1, with one difference: **data survives between runs.** Deposit cash, create a portfolio, add assets, and buy and sell; quit; start the app again. The cash balance, portfolios, assets, and full trade history are all still there, and the portfolio view shows the same holdings and gains it showed before quitting.

All of Assignment 1's features, rules, and calculation methods (average cost, unrealized and realized gain, positions per portfolio, one shared cash balance that can never go negative) stay exactly as they were. Deleting a portfolio still removes its trade history and realized gains, now from the database as well.

**Robustness.** As before, no input can crash the app. In addition, if the app can't reach the database when it starts (for example, the server isn't running or the settings are wrong), it tells the user in plain language and exits cleanly, with the details written to the log.

## Your decisions

Make each of these choices yourself, then state and justify it in your README. Any reasonable choice earns full credit if your code follows it consistently and your reasoning is sound.

1. Are current prices stored in the database, or kept in memory and re-entered each run? (Either way, a later milestone replaces them with a live market-data feed.)
2. How many decimal places do you store for cash, share quantities, and prices, and what does that mean for rounding? For example, what happens to the cash balance when someone buys 3 shares at $33.3333?
3. Which of your business rules does the database itself enforce, in addition to the checks in your Python code, and why those?

## How your code must be built

Most of the grade comes from this section. Each practice below was covered in class; the goal is to show that you understand why it exists.

**Data design**
- Design your tables before you write the code. Store the facts; calculate what can be calculated. Anything your app can work out from stored data (such as holdings and portfolio totals) is calculated, not stored.
- Give every table a key with no business meaning, and protect business identifiers (such as a portfolio's name or an asset's ticker) as unique.
- Money, share quantities, and prices are never stored or calculated as floating-point numbers.

**SQLAlchemy models**
- Every kind of stored data is described by a SQLAlchemy model. Where one table refers to another, the models express that relationship, so your code works with related objects rather than looking up IDs by hand.
- Use the SQLAlchemy 2.0 style shown in class. Don't mix it with the older style you may find in tutorials.
- The app creates any missing tables when it starts. The grader starts from an empty database.

**Configuration and secrets**
- The database connection settings (host, port, user, password, database name) live outside the code, in a `.env` file that is **not committed**.
- Commit an example settings file with placeholder values instead, so someone else can set up the app from your README.

**Storage layer**
- Database code lives in its own storage area, separate from your business-rule (service) modules, organized by business area. Only the storage code talks to the database.
- Storage only saves and returns data. Deciding that something is a business error (a duplicate name, an unknown ticker) stays in the services, where it was in Assignment 1.
- If you write any SQL by hand, values are always passed as parameters. User input is never glued into a query string.
- To check your structure, ask yourself: *if I switched to a different database tomorrow, would I have to touch a single business rule?*

**Transactions**
- Every user operation is all-or-nothing. If any step of an operation fails (a broken rule, a database error), none of its changes are kept. A buy never takes the cash without recording the trade, or the reverse.
- The business operation decides when changes are committed, not the individual storage functions.

**Automated tests**
- Write your tests with pytest, in a test folder that sits next to your package (not inside it). They run with `python -m pytest` from the project folder, and pytest is in `requirements.txt`.
- **Tests never touch your real database.** For each test, the app's database is replaced with a brand-new, empty in-memory database, so every test starts from the same known state. Tests pass in any order, alone or together, every time you run them.
- Shared setup (an empty database, a funded account with a portfolio and an asset) lives in fixtures, not copy-pasted from test to test.
- Test the business rules and calculations, through your service modules:
  - **Every calculation** has tests whose expected values you worked out by hand.
  - **Every rule** has a test for its refusal. A refusal test checks two things: that the specific error was raised, *and* that nothing changed. Test the boundary of each rule (zero, not just a negative number).
  - Where a result depends on current prices, replace the price source with a stand-in that the test controls.
- Each test checks one behavior and is named as a sentence about the business rule it checks.
- A test that would still pass if the code were wrong doesn't count. When grading, your code will be broken on purpose to see whether your tests notice.
- You don't need to test the command-line interface.

**Carried over from Assignment 1.** Everything in Assignment 1's "How your code must be built" section still applies: a package run with `python -m`, a virtual environment and a complete `requirements.txt`, lowercase module names, separate areas for domain objects, services, errors, interface, and entry point, a command-line interface with no business rules in it, a custom error hierarchy raised in the core and caught at the edge, logging configured once and written to a file, readable code, and small, meaningful commits. Database and test changes don't excuse a regression in any of these.

## README

Update the `README.md` at the root of your repository so it covers:

1. how to set up and run the app from scratch, including creating the MySQL database and user and filling in the settings file.
2. how to run the tests.
3. your three decisions from "Your decisions", each with its justification.
4. **How your tests work**, in your own words: how a test gets a fresh database without touching the real one, and why that matters. Then describe one bug you introduced into your code on purpose, which test caught it, and what the failure report said.

## Submission

1. Merge your Assignment 1 pull request into `main` first, so this pull request shows only the Assignment 2 changes.
2. Create a new feature branch from `main` for this assignment and do your work there.
3. Open a pull request from your branch into `main` with a short description of what it contains. **Do not merge it.**
4. Submit the pull request link on Teams by the deadline.

## How you'll be graded

| Area | Weight | What's assessed |
| --- | --- | --- |
| Persistence and features | 20% | The app sets up from the README, creates its tables, and everything from Assignment 1 still works and survives a restart |
| Data design and models | 15% | Facts stored and calculations not; sensible keys, constraints, and types; models and relationships |
| Storage layer, transactions, and configuration | 15% | Only storage talks to the database; rules stay in the services; operations are all-or-nothing; no secrets committed; no SQL built from user input |
| Automated tests | 35% | Tests are isolated and repeatable on an in-memory database; calculations and every rule's refusal are covered; fixtures and stand-ins used well; tests catch deliberately broken code |
| Readability, setup, Git, and README | 15% | Assignment 1 practices kept; meaningful commit history and PR description; decisions justified; a clear account of how your tests work |

**Late work:** there is no grace period. Work submitted up to two weeks after the deadline loses 20%; work submitted later than that loses 30%.

## Out of scope

There is no web interface or API yet, and no live market prices; both come in the next milestone. You don't need to handle changing the table design after there is data in it (dropping and recreating your tables during development is fine). Your tests don't need to run against MySQL, and you don't need to test the command-line interface. There are no new features: no account-wide overview and no trade-history screen. As before, there are no withdrawals, fees, dividends, short selling, multiple users, or multiple currencies.

## AI and collaboration

- **You may not use AI tools to write your code, and that includes your test code.** You *may*, and are encouraged to, use AI as a teacher: ask it to explain a concept (transactions, models, fixtures, mocking) or to help you understand an error message. Later you'll build software *with* AI agents, and you can only judge an agent's code, or the tests it writes to prove its code works, if you understand the fundamentals well enough to have written them yourself.
- You may discuss concepts and approaches with classmates out loud. You may not copy code, look at another student's repository or pull request, or share your solution. Submissions with suspiciously similar code will be reviewed.
