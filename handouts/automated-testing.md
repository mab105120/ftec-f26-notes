# Automated Testing

## Why this topic, and why now

The project this semester is a personal portfolio tracker. The user deposits cash, creates named portfolios (one per investment strategy, such as "Tech growth"), adds assets such as stocks, buys them inside a portfolio, and sees each portfolio's holdings and gain or loss. You have built it as a command-line application, in layers: the command line (the interface) calls one **service module** per business area (account, portfolios, assets, transactions, holdings, prices), where the business rules and calculations live; the services call a **storage layer**, which keeps the data. Broken rules raise custom exceptions such as a duplicate-name error, and the interface catches them. Over the last two weeks the storage layer moved into a MySQL database, accessed through SQLAlchemy.

That move changed the storage modules, the models and the database connection code. Afterwards, the way we checked that the app still worked was to run it and try a few menu options. Many things went unchecked: creating a duplicate portfolio, buying an unknown ticker, depositing zero, viewing a holding with no current price. Any of them could have broken without anyone noticing.

This handout is about making that checking automatic: writing code that checks your code, the same way every time, in a few seconds. It also connects to the larger aim of this course. Later you will direct an AI coding agent to build software, and you will need to judge whether what it produced is right. Tests are the main tool for that judgment, and tests themselves can be wrong in ways that look right. The last part of this handout is about telling the difference.

The handout has six parts:

- **Part A** explains why manual testing doesn't scale.
- **Part B** writes and runs the first tests, on a calculation with no database.
- **Part C** tests code that uses the database, without touching the real data.
- **Part D** tests the business rules, including the operations the app must refuse.
- **Part E** replaces the price source with a stand-in the test controls.
- **Part F** reads tests critically: tests that pass and prove nothing.

The code shown is the version built in class. Your own project will use different names and structure in places; the ideas carry over unchanged.

---

# Part A: Why automate testing

## 1. Manual testing doesn't scale

Every time you change code, something that already worked might stop working. The only way to know is to check it again. With manual testing, that means running the program and trying everything: every menu option, every rule, every error message. The tracker has a handful of features today; each new feature adds to the list of things to re-check after every change. The cost grows with every feature, so in practice people stop re-checking everything, and check only what they just changed.

That gap is where **regressions** come from. A regression is something that used to work and broke because of a change somewhere else. The change you made works, so you don't notice that it broke something two modules away. Weeks of changes to storage, models and the database connection are the classic setup for one.

Accounting solved the same problem long ago. A bank doesn't trust its books because someone looked at them once. It runs the same reconciliation every night, automatically, and investigates whatever doesn't match. An **automated test** is the nightly reconciliation for code: code that runs your code with known inputs and checks the result against what *should* happen. A full set of tests (a **test suite**) runs in seconds and checks the same things the same way every time, no matter who runs it or how rushed they are.

## 2. What tests buy you, and their limit

Tests give you three things.

**Confidence to change code.** Later this semester the command-line interface will be replaced by a web API. With a test suite, you can make a change of that size and know within seconds whether the business rules still hold. Without one, every change is a risk you can't measure, and teams that can't measure risk stop making changes.

**An executable specification.** A test named `test_portfolio_names_are_unique` states a business rule in plain language, and it is checked every time the suite runs. A rule written in a requirements document can drift out of date without anyone noticing. A rule written as a test can't: the moment the code stops following it, the test fails.

**A way to judge code you didn't write.** An AI agent will report "done" whether or not the code is right. Running tests is how you check. The tests it writes need checking too, which is the subject of Part F.

The limit: tests can show that a bug is present, never that bugs are absent. A test suite checks exactly what someone thought to check and nothing more. A passing suite tells you the checked behaviors work; it says nothing about the rest.

---

# Part B: First tests with pytest

## 3. Setting up

**pytest** is the most widely used test runner for Python. It finds your tests, runs them, and reports what passed and what failed. Install it in the project's virtual environment and record it in `requirements.txt`, like any other dependency:

```
pip install pytest
pip freeze > requirements.txt
```

Tests live in a `tests` folder next to the application package, not inside it. The tests are not part of the application; they check it from the outside.

```
portfolio-tracker/
├── portfolio_tracker/        the application package
│   ├── models/
│   ├── services/
│   └── storage/
├── tests/                    the test suite
│   ├── test_holding.py
│   └── ...
└── requirements.txt
```

pytest finds tests by name. It looks for files named `test_*.py`, and inside them, for functions named `test_*`. A test in a file called `holding_tests.py`, or a function called `check_market_value`, is silently ignored.

## 4. A first test: a calculation with no moving parts

Start testing where the rules are, and where there are no moving parts. The best first target in the tracker is `Holding`, the class that represents one position in a portfolio. It does pure calculation: no database, no files, no user input.

```python
# portfolio_tracker/models/holding.py
class Holding:
    def __init__(self, asset, quantity, average_cost, current_price):
        self.asset = asset
        self.quantity = quantity
        self.average_cost = average_cost
        self.current_price = current_price  # None when no price has been set

    @property
    def market_value(self):
        if self.current_price is None:
            return None
        return self.quantity * self.current_price

    @property
    def unrealized_gain(self):
        if self.current_price is None:
            return None
        return (self.current_price - self.average_cost) * self.quantity
```

The first test checks that market value is quantity times current price:

```python
# tests/test_holding.py
from decimal import Decimal

from portfolio_tracker.models.asset import Asset
from portfolio_tracker.models.holding import Holding

APPLE = Asset(ticker="AAPL", name="Apple Inc.")


def test_market_value_is_quantity_times_current_price():
    holding = Holding(APPLE, quantity=Decimal("10"), average_cost=Decimal("150"), current_price=Decimal("170"))

    assert holding.market_value == Decimal("1700")
```

`assert` is plain Python: it does nothing if the condition is true and raises an `AssertionError` if it is false. pytest treats a test function that finishes without an error as a pass, and one that raises as a fail.

Run the suite from the project folder (the folder that contains both `portfolio_tracker` and `tests`):

```
python -m pytest
```

pytest prints one dot per passing test, grouped by file:

```
tests/test_holding.py .                                                  [100%]

============================== 1 passed in 0.01s ===============================
```

Use `python -m pytest`, not plain `pytest`. Plain `pytest` fails with:

```
E   ModuleNotFoundError: No module named 'portfolio_tracker'
```

This is the import search path from earlier in the semester. When Python looks for `portfolio_tracker`, it searches the folders on `sys.path`. `python -m` adds the current folder to that list, so the package is found; the `pytest` command doesn't, so the test file's `import` fails.

The same file gets three more tests: an unrealized gain (current price above average cost), an unrealized loss, and a holding with no current price.

```python
def test_unrealized_gain_compares_current_price_to_average_cost():
    holding = Holding(APPLE, quantity=Decimal("10"), average_cost=Decimal("150"), current_price=Decimal("170"))

    assert holding.unrealized_gain == Decimal("200")


def test_unrealized_loss_is_negative():
    holding = Holding(APPLE, quantity=Decimal("10"), average_cost=Decimal("150"), current_price=Decimal("140"))

    assert holding.unrealized_gain == Decimal("-100")


def test_no_current_price_means_no_market_value_or_gain():
    holding = Holding(APPLE, quantity=Decimal("10"), average_cost=Decimal("150"), current_price=None)

    assert holding.market_value is None
    assert holding.unrealized_gain is None
```

## 5. The shape of a test: arrange, act, assert

Every test above has the same three steps, separated by blank lines:

1. **Arrange:** set up the situation (a holding of 10 shares bought at $150, now priced at $170).
2. **Act:** do the one thing being tested (read `market_value`).
3. **Assert:** check the result against what should happen.

In the `Holding` tests, arrange and act happen in one line, because reading a property is a single step. In longer tests, later in this handout, the three steps are clearly separate. Keeping that shape makes any test readable at a glance: what was the situation, what happened, what was expected.

## 6. Break it on purpose: reading a failure

A test you have only ever seen pass has not yet shown it can catch anything. So break the code on purpose. In `Holding.unrealized_gain`, swap the two prices:

```python
return (self.average_cost - self.current_price) * self.quantity  # bug: sign flipped
```

Run the tests again:

```
tests/test_holding.py .FF.                                               [100%]

=================================== FAILURES ===================================
_________ test_unrealized_gain_compares_current_price_to_average_cost __________

    def test_unrealized_gain_compares_current_price_to_average_cost():
        holding = Holding(APPLE, quantity=Decimal("10"), average_cost=Decimal("150"), current_price=Decimal("170"))

>       assert holding.unrealized_gain == Decimal("200")
E       AssertionError: assert Decimal('-200') == Decimal('200')

tests/test_holding.py:18: AssertionError
_______________________ test_unrealized_loss_is_negative _______________________
...
=========================== short test summary info ============================
FAILED tests/test_holding.py::test_unrealized_gain_compares_current_price_to_average_cost
FAILED tests/test_holding.py::test_unrealized_loss_is_negative - AssertionErr...
========================= 2 failed, 2 passed in 0.04s ==========================
```

(Shortened slightly; the real report also repeats the details for the second failure.)

Read it the way you read a traceback, from the bottom up. The last line gives the score: 2 failed, 2 passed. The short summary names the failing tests, and because each name is a sentence about a business rule, you already know what broke: unrealized gain and unrealized loss. Above that, each failure shows the test's code, a `>` at the line that failed, and an `E` line with both sides of the comparison: the code produced `-200` where `200` was expected. The last line of each failure gives the file and line number.

Notice which tests did *not* fail: market value and the no-price case, which don't use the flipped line. A good failure report points straight at the broken behavior. Undo the bug, rerun, and everything is green again.

## 7. What makes a good test

**One behavior per test, named as a sentence about the rule.** `test_unrealized_loss_is_negative` tells you what broke without opening the file. A test named `test_holding` that checks five things tells you only that one of them is wrong, and stops at the first failure, so you don't learn about the other four.

**Expected values worked out by hand.** 10 shares × $170 = $1,700. That number comes from the business rule, not from the code. It is tempting to run the code, see what it prints, and paste that into the test. Don't. A value copied from the code's output only checks that the code does what it already does: if the code was wrong when you copied it, the test now protects the bug.

**Deeper dive: test-driven development.** Everything here writes the code first and the tests after. Some teams reverse the order: write a failing test that states the rule, then write just enough code to make it pass. This is **test-driven development (TDD)**. The tool is the same; the difference is that the test is written before the code exists, so it can't be shaped by what the code happens to do.

---

# Part C: Testing code that uses a database

## 8. The problem with the real database

The next target is the account service's `deposit`, which adds cash to the account balance. Unlike `Holding`, it writes to the database. Testing it against the real MySQL database causes three problems:

- **It changes your real data.** Every run of the suite deposits more cash into your actual account.
- **The result depends on what is already there.** A test that expects a $500 deposit to produce a $500 balance fails whenever the balance didn't start at zero, even though `deposit` works.
- **Tests depend on each other.** A test that creates a "Tech growth" portfolio breaks a later test that also needs to create one, because names must be unique. Whether a test passes then depends on which tests ran before it.

The rule that fixes all three: **every test starts from the same known state, and no test affects another.** A test that follows it is **repeatable** (same result on every run) and **independent** (same result no matter which other tests ran, or in what order).

An auditor works this way. Controls are tested on a copy of the books, reset to the same opening balances before every test, never on the live ledger. Each test gets a fresh copy, so one test's entries can't affect another's results.

For the tracker, that means each test needs its own brand-new, empty database. To give it one, the test has to be able to tell the app which database to use.

## 9. Making the database replaceable

The connection code from the databases module looked like this:

```python
# portfolio_tracker/database.py (before)
load_dotenv()

url = URL.create("mysql+pymysql", ...)
engine = create_engine(url)
Session = sessionmaker(engine, expire_on_commit=False)


def transaction():
    return Session.begin()
```

The engine is built at the top level of the module. Recall from earlier in the semester that a module runs from top to bottom the first time it is imported. So any import that touches `database` reads the `.env` file and builds a MySQL engine, whether you want one or not. "Where is the database?" is not a step a test can replace; it just happens, at import time, before the test has a chance to do anything.

The fix is to move that step into a function:

```python
# portfolio_tracker/database.py (after)
import functools
...

@functools.cache  # runs once; later calls get the same engine back
def get_engine():
    """The one place that knows where the app's database is."""
    load_dotenv()
    url = URL.create("mysql+pymysql", ...)  # unchanged
    return create_engine(url)


def create_tables():
    Base.metadata.create_all(get_engine())


def transaction():
    """One unit of work: commit if the block finishes, roll back if it raises."""
    Session = sessionmaker(get_engine(), expire_on_commit=False)
    return Session.begin()
```

The module-level `engine` and `Session` are gone. `get_engine()` is now the single place that decides where the database is, and everything else asks it. `@functools.cache` makes the function run once: the first call builds the engine, and every later call returns that same engine instead of building a new one.

The app still works exactly as before, and the services didn't change: they still call `database.transaction()`.

The judgment behind this change matters more than the change itself. **Code that is hard to test is telling you something about its design.** Here, "connect to the database" was buried in import-time code; now it is one named function that a test can replace. A well-run firm names its custodian in one place in the contract, rather than writing it into every trade ticket. Switching custodians then means changing one line. The same refactor that makes the code testable also makes it easier to change for any other reason.

**Deeper dive.** Without `@functools.cache`, every transaction would build a new engine, each with its own pool of connections. `create_engine` itself is lazy: it records *how* to connect, and no connection is opened until the first query.

## 10. Mocks and `monkeypatch`

A **mock** is a stand-in for a real dependency, controlled by the test. Here, the test replaces `get_engine` with a function that returns a test database instead of MySQL.

pytest provides a tool for this called `monkeypatch`:

```python
monkeypatch.setattr(database, "get_engine", replacement)
```

This reads as: for this test only, replace the name `get_engine` in the `database` module with `replacement`. When the test finishes, pytest puts the original back automatically, whether the test passed or failed.

Putting the original back is the reason to use `monkeypatch` rather than assigning directly (`database.get_engine = replacement`). A direct assignment stays in place for every test that runs afterwards, which is exactly the kind of leak between tests that section 8 ruled out.

The mock works because `transaction()` looks up `get_engine` *when it is called*, inside the `database` module. Replace the name there, and the next call to `transaction()` finds the replacement.

## 11. A database that lives in memory

The replacement database is **SQLite in memory**. SQLite is a small database engine built into Python; with an in-memory database, the whole database lives inside the running Python process. There is nothing to install, no server to start, and nothing left behind: when the test ends, the database is gone.

Because the tracker's tables are described by SQLAlchemy models, the same models can create the same tables in SQLite, with the `create_all` call you already know:

```python
engine = create_engine("sqlite://", poolclass=StaticPool)
database.Base.metadata.create_all(engine)
```

`"sqlite://"` with nothing after it means "in memory." `StaticPool` handles one detail: an in-memory SQLite database exists only as long as the connection that created it, so a second connection would see a different, empty database. `StaticPool` makes every session share that one connection.

## 12. Fixtures: preparing what a test needs

Every database test needs the same preparation: build the in-memory database, create the tables, swap it in, and clean up afterwards. Writing that at the top of every test would be repetitive and easy to get wrong. pytest's answer is a **fixture**: a function that prepares something a test needs.

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.pool import StaticPool

from portfolio_tracker import database

# Imported so create_all() knows every table, as in main.py.
from portfolio_tracker.models import account, asset, portfolio, transaction  # noqa: F401


@pytest.fixture
def empty_db(monkeypatch):
    """Point the app at a brand-new, empty database for the length of one test."""
    engine = create_engine("sqlite://", poolclass=StaticPool)
    database.Base.metadata.create_all(engine)
    # The mock: for this test, "connect to the database" returns the in-memory one.
    monkeypatch.setattr(database, "get_engine", lambda: engine)
    yield  # the test runs here
    engine.dispose()  # monkeypatch puts the real get_engine back on its own
```

A test asks for a fixture by **naming it as a parameter**. pytest sees the name, runs the fixture first, and then runs the test:

```python
from decimal import Decimal

from portfolio_tracker.services import account_service


def test_new_account_starts_with_no_cash(empty_db):
    assert account_service.get_cash_balance() == Decimal("0")


def test_deposits_add_up(empty_db):
    account_service.deposit(Decimal("500"))
    account_service.deposit(Decimal("150.25"))

    assert account_service.get_cash_balance() == Decimal("650.25")
```

A few details in the fixture:

- **`yield` splits setup from cleanup.** Everything before `yield` runs before the test; the test runs at the `yield`; everything after runs once the test is done, even if it failed. Here the cleanup closes the in-memory engine.
- **`monkeypatch` is itself a fixture**, built into pytest. `empty_db` asks for it the same way a test does: by naming it as a parameter.
- **`lambda: engine`** is a one-line function with no name that always returns the test engine. It stands in for `get_engine`, which also takes no arguments and returns an engine.
- **The models import** looks unused, but `create_all` can only create tables for models that have been imported. The `# noqa: F401` comment tells the linter the unused import is deliberate.

Run these tests twice and they give the same result both times. Check MySQL afterwards and it is untouched. That is repeatability.

The account service didn't change to make this possible. It calls `database.transaction()` and has no idea which database is behind it. This is the same payoff as moving storage into its own layer: the business rules don't depend on where the data lives, so the data can be swapped (for MySQL, then SQLAlchemy, and now an in-memory test database) without touching them.

These tests go **through the services**, so they also exercise the storage functions, the models and the SQLAlchemy mapping underneath. They test *your* rules ("deposits add up"), not SQLAlchemy itself, which has its own tests.

## 13. Sharing fixtures, and fixtures that build on fixtures

### `conftest.py`

The transaction service's tests need `empty_db` too. Copying the fixture into every test file would mean several copies to keep in sync. Instead, fixtures go in a file named `conftest.py` inside the `tests` folder. pytest loads it automatically, and every fixture defined there is available to every test file, without any import.

### Hidden shared state

The in-memory database resets everything stored in the database. But current prices are not in the database: they are kept in a plain dictionary inside the price storage module. A price set in one test would still be there in the next. So `empty_db` resets that too:

```python
monkeypatch.setattr(price_store, "current_prices", {})  # prices are stored in memory: start empty too
```

This is the most common reason tests **pass when run alone and fail when run together**: some piece of state (a module-level dictionary, a list, a cached value) survives from one test to the next. When you write a fixture that promises a clean start, look for every place the app keeps state, not only the database.

### Fixtures that use fixtures

Most trade tests start from the same position: some cash, a portfolio, an asset. That starting position is itself a fixture, and it builds on `empty_db` by asking for it:

```python
@pytest.fixture
def funded_portfolio(empty_db):
    """A typical starting point: $1,000 in cash, one portfolio, one asset."""
    account_service.deposit(Decimal("1000"))
    portfolio_service.create_portfolio("Tech growth", "Large-cap tech")
    asset_service.add_asset("AAPL", "Apple Inc.")
    return "Tech growth"
```

pytest runs `empty_db` first, then `funded_portfolio`, then the test. A test receives whatever the fixture returns, here the portfolio's name:

```python
def test_buy_takes_the_cost_out_of_cash(funded_portfolio):
    transaction_service.buy(funded_portfolio, "AAPL", Decimal("3"), Decimal("100"))

    assert account_service.get_cash_balance() == Decimal("700")
```

$1,000 minus 3 shares at $100 is $700. The fixture is the "arrange" step, shared; the test keeps only the act and the assert that are specific to it.

### The trade-off of testing on SQLite

The tests run on SQLite; the app runs on MySQL. That is a deliberate trade: SQLite is fast and needs no setup, but it is not identical to MySQL. Types, constraints and SQL details differ between the two, so a bug that only appears on MySQL can pass these tests. Professional teams usually also run their tests against the same database engine the app uses. For a course project, testing on SQLite is a reasonable trade, as long as you know what it doesn't cover.

**Deeper dive: scope and `autouse`.** By default a fixture runs once per test. `@pytest.fixture(scope="session")` builds it once for the whole run: faster, but the tests share it, so the shared-state problem is back. `@pytest.fixture(autouse=True)` applies a fixture to every test without being named, a safety net against forgetting to ask for `empty_db`.

---

# Part D: Testing the business rules

## 14. Refusals are half the rules

Most of the tracker's business rules are refusals: a portfolio name that already exists, a ticker that doesn't, an amount that isn't positive. Each is enforced by raising one of the custom errors. A suite that only tests the successful path (deposits that work, buys that go through) tests half the app.

`pytest.raises` checks that a block of code raises a specific error. The test passes only if that error is raised:

```python
import pytest

from portfolio_tracker.errors import DuplicateError
from portfolio_tracker.services import portfolio_service


def test_portfolio_names_are_unique(empty_db):
    portfolio_service.create_portfolio("Tech growth", "Large-cap tech")

    with pytest.raises(DuplicateError):
        portfolio_service.create_portfolio("Tech growth", "A different strategy")
```

Name the **specific** error. This is a benefit of the custom error hierarchy you built earlier in the semester: each rule has its own error type, so a test can tell "refused for the right reason" from "crashed for some other reason." `pytest.raises(Exception)` would also pass if the code crashed with a `TypeError` from a typo, and the test would report the rule as working.

## 15. A refusal has two halves

When the app refuses an operation, two things must be true: the error was raised, *and* nothing changed. A buy that raises "unknown ticker" but has already taken the money out of cash is a serious bug, and a test that only checks for the error won't catch it.

```python
def test_buying_an_unknown_ticker_changes_nothing(funded_portfolio):
    with pytest.raises(NotFoundError):
        transaction_service.buy(funded_portfolio, "ZZZZ", Decimal("3"), Decimal("100"))

    assert account_service.get_cash_balance() == Decimal("1000")
```

This connects to database transactions. A `buy` debits cash and records a trade inside one transaction, so either both happen or neither does. This test checks that behavior from the outside: it doesn't care about the order of steps inside `buy`. If someone later rearranges `buy` so the debit happens before the ticker lookup, the transaction must roll the debit back, and this test proves it did, on every run. A test that checks *behavior* rather than *implementation* keeps working, and keeps protecting you, through any rewrite of the code underneath.

## 16. Boundaries and `parametrize`

A rule has an edge. "Deposits must be positive" has its edge at zero: zero is refused, and so is anything below it. Testing only `-50` checks the easy case; a bug such as `if amount < 0` instead of `if amount <= 0` lets zero through and goes unnoticed. Always test the boundary itself, plus a value clearly past it.

Writing a separate test for each value would repeat the same code. `@pytest.mark.parametrize` runs one test once per value:

```python
@pytest.mark.parametrize("amount", [Decimal("0"), Decimal("-50")])
def test_deposit_must_be_positive(empty_db, amount):
    with pytest.raises(InvalidAmountError):
        account_service.deposit(amount)

    assert account_service.get_cash_balance() == Decimal("0")
```

The first argument names the parameter; the list gives its values. The test receives `amount` like a fixture. Each value is reported as its own test, which you can see with `-v` (verbose):

```
python -m pytest -v
...
tests/test_account_service.py::test_deposit_must_be_positive[amount0] PASSED
tests/test_account_service.py::test_deposit_must_be_positive[amount1] PASSED
```

Every rule you add to your own tracker deserves a test like these: the error is raised, nothing changed, and the boundary is covered.

---

# Part E: Mocking the price source

## 17. When the real thing is unpredictable

In a later assignment, current prices will come from a live market data feed instead of being typed in. Imagine a test that buys 4 shares and expects a market value of $600. It passes today and fails tomorrow, because the price moved. It also fails whenever the internet is down, and it gets slower with every call to the feed.

Use a mock when the real dependency is **slow, unpredictable, external, or costs money**. Paper trading works the same way: the controls and the rules are real, the market is simulated, and nothing real is at stake.

```python
from decimal import Decimal

from portfolio_tracker.services import holding_service, price_service, transaction_service


def test_holdings_are_valued_at_the_current_price(funded_portfolio, monkeypatch):
    # A stand-in price source: the test decides the price, not the market.
    monkeypatch.setattr(price_service, "get_price", lambda ticker: Decimal("150"))
    transaction_service.buy(funded_portfolio, "AAPL", Decimal("4"), Decimal("110"))

    [holding] = holding_service.get_holdings(funded_portfolio)

    assert holding.market_value == Decimal("600")
    assert holding.unrealized_gain == Decimal("160")
```

The stand-in, `lambda ticker: Decimal("150")`, is a price function that answers $150 for any ticker. With the price fixed, the expected values can be worked out by hand: 4 × $150 = $600 market value; ($150 − $110) × 4 = $160 gain.

`[holding] = ...` unpacks a list that should contain exactly one item. If `get_holdings` returned zero holdings or two, the line itself would fail, so it doubles as a check.

This is the same tool as in Part C. There the test replaced "where is the database"; here it replaces "what is the price." It works because the price source is its own replaceable piece of the app, a design choice made when the services were organized. Just as `get_engine()` made the database replaceable, keeping prices behind one function made the market replaceable. **Good separation is what makes code testable.**

## 18. Two rules for mocking

**Mock the dependencies of what you test, never the thing you are testing.** The test above mocks the price, and the holding calculation runs for real. If it also replaced the calculation, it would only be testing its own stand-in. Part F shows what that looks like.

**Replace a name where it is looked up.** The holding service calls `price_service.get_price(...)`, looking the function up in the `price_service` module each time. Replacing it there is enough. If a module instead did `from portfolio_tracker.services.price_service import get_price`, it would have its own copy of the name, made at import time, and replacing the original wouldn't reach it. The tracker's services consistently go through the module (`database.transaction()`, `price_service.get_price()`), so one replacement covers every caller.

A common worry is that mocking is cheating, because the test isn't using the real thing. The answer is that you mock what you are *not* testing. These tests are about the business rules, and the business rules run for real; only where the data lives and what the market does are replaced.

---

# Part F: Reading tests critically

## 19. Tests that pass and prove nothing

Here is a file of six tests for `Holding.unrealized_gain`. All six pass. Now flip the sign in `unrealized_gain`, the same bug as in section 6, and run them again:

```
test_unrealized_gain PASSED
test_unrealized_gain_is_a_decimal PASSED
test_unrealized_gain_is_correct PASSED
test_holding_rejects_bad_input PASSED
test_unrealized_gain_value SKIPPED (flaky)
test_unrealized_gain_with_stub PASSED
========================= 5 passed, 1 skipped =========================
```

Still green, with the bug in place. The real test suite from Part B, run against the same broken code, fails immediately. Each of the six fails to catch the bug for a different reason, and each is a pattern you will see again.

**1. No assert.** The test only proves the code didn't crash.

```python
def test_unrealized_gain():
    holding = make_holding()
    holding.unrealized_gain
```

**2. A weak assert.** Any `Decimal` passes, including the wrong one. `is not None`, `len(...) > 0` and "is the right type" checks have the same problem.

```python
def test_unrealized_gain_is_a_decimal():
    assert isinstance(make_holding().unrealized_gain, Decimal)
```

**3. A swallowed failure.** A failed `assert` raises `AssertionError`, and `except Exception` catches it. The test can never fail.

```python
def test_unrealized_gain_is_correct():
    try:
        assert make_holding().unrealized_gain == Decimal("200")
    except Exception:
        pass
```

**4. A too-broad `raises`.** The constructor call is missing its `current_price` argument, so Python raises a `TypeError` before any business rule runs. `pytest.raises(Exception)` accepts it, and the test reports the input as "rejected."

```python
def test_holding_rejects_bad_input():
    with pytest.raises(Exception):
        Holding(APPLE, Decimal("10"), Decimal("150"))
```

**5. Skipped.** The one test that checks the right value is marked to be skipped, labeled "flaky." It never runs. A skipped test shows up as an `s` in the output, not an `F`, and is easy to overlook.

```python
@pytest.mark.skip(reason="flaky")
def test_unrealized_gain_value():
    assert make_holding().unrealized_gain == Decimal("200")
```

**6. Mocking the thing under test.** The calculation itself is replaced with the expected answer, so the test checks its own stand-in.

```python
def test_unrealized_gain_with_stub(monkeypatch):
    monkeypatch.setattr(Holding, "unrealized_gain", Decimal("200"))
    assert make_holding().unrealized_gain == Decimal("200")
```

These patterns matter for this course because they are what an AI agent tends to produce when it is told to "make the tests pass": weaken an assertion, wrap a failure in `try`/`except`, skip the test that fails, or mock away the code that's wrong. Each one turns the suite green without fixing anything. "All tests pass" means nothing until you have read what the tests check.

## 20. The quick check: break the code on purpose

The fastest way to find out whether a test protects anything is the one from section 6: introduce a bug in the code it claims to test and run it. If the test still passes, it isn't testing that code. Flip a sign, change `<=` to `<`, delete a line that enforces a rule. A trustworthy test fails.

## 21. What to test, and how much

**Test the services and the calculations.** That is where the rules live. The command-line interface is deliberately thin: it reads input, calls a service, and prints the result or the error message (raise in the core, catch at the edge). There is little logic in it to test, which is part of why it was designed that way.

**Test storage through the services.** The service tests already run the storage functions underneath. Test a storage function directly only when it has logic of its own.

**Enough tests means coverage of the rules, not a number.** Aim for one test per calculation and per rule, plus the refusal for each rule, with its boundary. The question to ask about any line of business logic is: *if I broke this line, would a test fail?* If not, that line is unprotected.

## 22. Reviewing tests someone else wrote

When you review a test suite (a teammate's, or later an AI agent's), check:

1. **Does every test assert something specific?** Exact expected values, not types or "not None."
2. **Were the expected values worked out from the rule, or copied from the output?**
3. **Is any failure swallowed?** Look for `try`/`except` around asserts.
4. **Do error tests name the specific error,** and check that nothing changed?
5. **Is anything skipped?** Find out why, and whether that test is the one that matters.
6. **What is mocked?** Dependencies only, never the code under test.
7. **Is each test independent?** A fresh database, no state leaking between tests.
8. **Which rules have no test at all?**
9. **Does breaking the code make a test fail?**

A note on the course's AI policy: the rule that AI may not write your code covers test code too. Using AI to explain a concept, or to understand an error message, is still fine.

---

## Common errors

| What you see | What it means |
|---|---|
| `ModuleNotFoundError: No module named 'portfolio_tracker'` | You ran `pytest` instead of `python -m pytest`, or ran it from the wrong folder. Run `python -m pytest` from the project folder. |
| `collected 0 items` | A file or function isn't named `test_...`, so pytest didn't find it. |
| `fixture 'empty_db' not found` | The fixture is in another test file instead of `conftest.py`, or `conftest.py` isn't inside `tests/`. |
| The test changed your real data, or passes once and then fails | The test didn't ask for `empty_db`, so it ran against MySQL. |
| Tests pass one at a time but fail together | State is leaking between tests, like the in-memory price dictionary. |
| A test still fails right after you undid a bug | Python may be running stale compiled files. Delete the `__pycache__` folders and rerun. |

## Summary

- **Manual testing doesn't scale**: the cost of re-checking grows with every feature, so it stops happening, and **regressions** slip through. An automated test suite is the nightly reconciliation for code.
- Tests give you **confidence to change code**, an **executable specification**, and a way to **judge code you didn't write**. They show bugs are present, never that they are absent.
- **pytest** finds `test_*.py` files and `test_*` functions. Run it with `python -m pytest` from the project folder. A test is **arrange, act, assert**, checks **one behavior**, has a **name that states the rule**, and uses **expected values worked out by hand**.
- **Break the code on purpose** to confirm a test can fail, and read the failure report bottom-up.
- **Every test starts from the same known state, and no test affects another.** Tests never touch the real database.
- Code that is hard to test signals a design problem. Moving "where is the database" into **one function, `get_engine()`**, made it replaceable.
- A **mock** is a stand-in for a dependency. `monkeypatch.setattr` replaces a name for one test and puts it back afterwards. Replace a name where it is looked up.
- A **fixture** prepares what a test needs; a test asks for it by naming it as a parameter. `yield` separates setup from cleanup. Fixtures in **`conftest.py`** are shared by all test files, and fixtures can build on other fixtures.
- The **in-memory SQLite database** gives each test a fresh, empty database. Fast and setup-free, but not identical to MySQL.
- A **refusal test** checks two things: the **specific** error was raised, and **nothing changed**. Test the **boundary** of each rule; `parametrize` runs one test per value.
- Mock what is **slow, unpredictable, external, or costs money**, such as a live price feed. Mock dependencies, never the thing under test.
- Tests can pass and prove nothing: **no assert, weak assert, swallowed failure, too-broad `raises`, skipped test, mocked code under test.** "All tests pass" means nothing until you've read what the tests check.

## Check your understanding

1. What is a regression? Why did moving storage to SQLAlchemy create a good setup for one?
2. Tests "show that bugs are present, never that they are absent." Explain what that means for a suite where every test passes.
3. A test file is named `holding_checks.py`. What happens when you run `python -m pytest`, and why?
4. Running `pytest` gives `ModuleNotFoundError: No module named 'portfolio_tracker'`, but `python -m pytest` works. What is the difference between the two commands?
5. A classmate writes their expected values by running the code and pasting what it prints into the asserts. What is wrong with that, given that the tests will pass?
6. Give the three problems with testing `deposit` against the real MySQL database. What single rule addresses all three?
7. Before the refactor, the database engine was created at the top level of `database.py`. Why did that make the database impossible to swap out in a test? What did `get_engine()` change?
8. Why use `monkeypatch.setattr(database, "get_engine", ...)` rather than `database.get_engine = ...`?
9. In the `empty_db` fixture, what runs before the `yield`, what runs after it, and when?
10. Every test uses a fresh in-memory database, yet a test that checks the holdings table fails only when run after a test that sets a price. What is the likely cause, and how is it fixed?
11. A test checks that buying an unknown ticker raises `NotFoundError`, and nothing else. What bug could it miss? What would you add?
12. A deposit rule is written as `if amount < 0: raise InvalidAmountError(...)`. Which test value catches this bug, and which doesn't?
13. Why does the holdings valuation test replace the price source? What would go wrong if it used a live market feed?
14. An AI agent reports "all 30 tests pass." Name four things you would look for in the test code before believing the code works.
15. Why are there no tests for the command-line interface?

## Glossary

- **Automated test**: code that runs your code with known inputs and checks the result against what should happen.
- **Test suite**: the full set of tests for an application, run together.
- **Regression**: something that used to work and broke because of a change elsewhere.
- **pytest**: Python's most widely used test runner; it finds, runs, and reports on tests.
- **Arrange–act–assert**: the three steps of a test: set up the situation, do one thing, check the result.
- **`assert`**: a Python statement that raises `AssertionError` if its condition is false.
- **Test-driven development (TDD)**: writing a failing test before writing the code that makes it pass.
- **Repeatable test**: a test that gives the same result on every run.
- **Independent test**: a test whose result doesn't depend on which other tests ran, or in what order.
- **Mock**: a stand-in for a real dependency, controlled by the test.
- **`monkeypatch`**: a built-in pytest fixture that replaces a name for one test and restores it afterwards.
- **In-memory database**: a database that lives inside the running program and disappears when it ends; here, SQLite.
- **Fixture**: a function that prepares something a test needs; a test requests it by naming it as a parameter.
- **`conftest.py`**: a file in the tests folder whose fixtures are available to every test file.
- **Shared state**: data that survives from one test to the next, such as a module-level dictionary; the usual cause of tests that pass alone and fail together.
- **`pytest.raises`**: checks that a block of code raises a specific error.
- **Boundary**: the edge of a rule, such as zero for "must be positive"; the value most worth testing.
- **`parametrize`**: runs one test once per value in a list, reporting each run separately.
- **`lambda`**: a one-line function without a name, such as `lambda ticker: Decimal("150")`.
