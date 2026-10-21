# Databases for Applications

## Why this topic, and why now

The project this semester is a personal portfolio tracker. The user deposits cash, creates named portfolios (one per investment strategy, such as "Tech growth" or "Dividend income"), adds assets such as stocks, buys them inside a portfolio, and sees what each portfolio holds and how much it has gained or lost. You have built it as a command-line application: classes for the business objects, one service module per business area holding the rules, custom exceptions for broken rules, and logging for a record of what happened.

It has one serious flaw. Every portfolio, trade, and dollar of cash lives in Python variables: dictionaries and lists inside the service modules. When the program stops, all of it disappears. A portfolio tracker that forgets your portfolios is not a tracker.

The fix is a database, and you already know relational databases and SQL. This handout is not about SQL itself. It is about the **connection between an application and a database**: where database code belongs in the application, how the application connects, how it sends queries safely, how it keeps related changes together, and how a library called SQLAlchemy removes most of the repetitive work. These are the questions that decide whether database code is reliable, and they are the questions to ask when you review code that someone, or something, else wrote.

The handout has four parts:

- **Part A** designs the data before writing any code.
- **Part B** decides where database code lives in the application.
- **Part C** talks to the database with plain SQL from Python.
- **Part D** does the same job with SQLAlchemy, and compares the two.

---

# Part A: Design around the data

## 1. Data outlives code

The code of an application gets rewritten many times. The portfolio tracker starts as a command-line program; later this semester the command line will be replaced by a web API, and then by a web page. Each change rewrites part of the code. The data does not get rewritten. A trade recorded today must still be there, with the same meaning, after every one of those changes.

Accounting works the same way. A company can replace its accounting software several times in a decade, but the general ledger carries over each time. Nobody designs the ledger around the software's screens. The ledger comes first, and the software is built to serve it.

That is what **data-first design** means: decide what information the application must keep, and how the pieces relate, *before* deciding how the code will work. The data design is the hardest part of the application to change later. A bad screen can be redesigned in an afternoon; a bad table with three years of trades in it cannot.

## 2. Designing the tracker's schema

### What gets stored

Start from the business, not from the code. The tracker deals with four kinds of things:

- **The account.** There is one user, with one cash balance shared across all portfolios.
- **Portfolios.** Each has a unique name and a description of its investment strategy.
- **Assets.** Each has a unique ticker (such as `AAPL`) and a name.
- **Transactions.** Each records one trade: which portfolio, which asset, buy or sell, how many shares, the price per share, and the date.

Each becomes a table. Notice what is *not* on the list.

### What does not get stored

The application shows **holdings**: for each asset in a portfolio, how many shares you hold, your average cost, and the market value. It is tempting to store holdings in a table too. Don't. Holdings are *calculated* from transactions: add up the shares bought (and subtract the shares sold) and you have the quantity held.

If you stored both the transactions and the holdings, you would have the same fact in two places. Sooner or later they would disagree: a bug updates one but not the other, and now the database says two different things about how many AAPL shares you own. A brokerage statement works the same way: positions are derived from the trade history, and the trade history is the record of truth. The rule is to **store facts, and calculate what can be calculated from them**. Portfolio summaries (total market value, total gain or loss) are calculated for the same reason.

Current prices are not in this design either. For now they are typed in by hand and kept in memory. How to store them is a decision you will make yourself in your next assignment.

### Relationships

A portfolio has many transactions; each transaction belongs to exactly one portfolio. An asset appears in many transactions; each transaction is for exactly one asset. Both are **one-to-many** relationships, recorded with a **foreign key**: the `transactions` table has a `portfolio_id` column pointing at a row in `portfolios`, and an `asset_id` column pointing at a row in `assets`.

Between portfolios and assets the relationship is **many-to-many**: a portfolio holds many assets, and the same asset can be held in many portfolios. That doesn't need its own table, because it already *emerges* from the transactions. "Which assets does Tech growth hold?" is answered by looking at Tech growth's transactions.

### Keys

Every table gets an `id` column as its **primary key**: a number the database generates, with no business meaning. Why not use the ticker as the key of `assets`? Because business identifiers change. Companies change tickers, and a portfolio name is something the user might want to rename. If other tables point at an asset by its ticker, every one of those references has to change with it. A meaningless `id` never needs to change, so references to it never break. The business identifiers are still protected, by a `UNIQUE` constraint.

### Constraints are business rules

`UNIQUE`, `NOT NULL`, and foreign keys are business rules that the database enforces itself. "Two portfolios cannot have the same name" is checked in your Python code, but the `UNIQUE` constraint means the database will refuse a duplicate even if a bug skips the check. "A transaction must belong to a real portfolio" is enforced by the foreign key. The database is the last line of defense, like an auditor who checks every entry no matter who made it.

### Types: why money is `DECIMAL`, not `FLOAT`

Try this in Python:

```python
>>> 0.1 + 0.2 == 0.3
False
>>> 0.1 + 0.2
0.30000000000000004
```

`float` stores numbers in binary, and most decimal fractions (including 0.1) cannot be represented exactly in binary. The errors are tiny, but money is the one place tiny errors are unacceptable: a balance that is off by a fraction of a cent fails reconciliation. Databases have a `DECIMAL` type that stores decimal numbers exactly, and Python has a matching `Decimal` type:

```python
>>> from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2") == Decimal("0.3")
True
```

`DECIMAL(14, 2)` means up to 14 digits in total, 2 of them after the decimal point. The second number is a **business decision**. The cash balance uses 2 decimal places (cents); share quantities and prices use 4. If you buy 3 shares at $33.3333, the cost is $99.9999, but the stored cash balance is rounded to cents. Choosing the scale of a column is choosing a rounding policy.

### The schema

Here is the design as SQL:

```sql
CREATE TABLE account (
    id           INT PRIMARY KEY,
    cash_balance DECIMAL(14, 2) NOT NULL
);

-- One implicit account, so exactly one row.
INSERT INTO account (id, cash_balance) VALUES (1, 0);

CREATE TABLE portfolios (
    id       INT AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL UNIQUE,
    strategy VARCHAR(255) NOT NULL
);

CREATE TABLE assets (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    ticker VARCHAR(10)  NOT NULL UNIQUE,
    name   VARCHAR(100) NOT NULL
);

CREATE TABLE transactions (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    portfolio_id    INT NOT NULL,
    asset_id        INT NOT NULL,
    side            ENUM('buy', 'sell') NOT NULL,
    quantity        DECIMAL(14, 4) NOT NULL,
    price_per_share DECIMAL(14, 4) NOT NULL,
    trade_date      DATE NOT NULL,
    FOREIGN KEY (portfolio_id) REFERENCES portfolios (id),
    FOREIGN KEY (asset_id) REFERENCES assets (id)
);
```

The `account` table holds a single row, which looks odd for a table. It is the simplest honest model of "one account with a balance." The row is created along with the table, so the application can always assume it exists.

### The type choice reaches back into the code

Once the data comes from the database, every stored amount (cash balance, quantities, prices) arrives in Python as a `Decimal`. But amounts the user types still arrive as text, and if the application converts them with `float(text)`, the two types meet:

```python
>>> Decimal("100.00") + 5.0
TypeError: unsupported operand type(s) for +: 'decimal.Decimal' and 'float'
```

Python refuses to mix them, because doing so would bring the float's inexactness back in. The fix is to convert user input to `Decimal` at the edge, where it enters the program:

```python
from decimal import Decimal, InvalidOperation


def ask_number(prompt):
    while True:
        text = input(prompt)
        try:
            number = Decimal(text)
        except InvalidOperation:
            number = None
        # Decimal also accepts "nan" and "inf", which are not usable amounts.
        if number is not None and number.is_finite():
            return number
        console.print(f"[yellow]'{text}' is not a number, try again.[/yellow]")
```

Two details. First, `Decimal("abc")` raises `InvalidOperation`, *not* `ValueError`, so an `except ValueError` written for `float` would stop catching bad input. Second, `Decimal` accepts the special values `"nan"` (not a number) and `"inf"` (infinity), so the function checks `is_finite()` before accepting the value.

This is data-first design in miniature: one decision in the schema (`DECIMAL`) reached all the way back to the input prompt.

---

# Part B: Where database code lives

## 3. The storage layer

### The problem

Right now each service module holds its data directly. The portfolio service has a dictionary of portfolios, the asset service a dictionary of assets, and so on. The quickest way to add a database would be to put SQL straight into those functions, next to the business rules.

That mixes two different jobs in one place. The business rule "two portfolios cannot share a name" has nothing to do with *how* portfolios are kept. If the SQL sits beside the rule, every change to how data is stored means editing code that holds business rules, and every bug in a query is a bug in the business layer.

### The fix: a separate layer for storage

Split the work the way a company splits it between a manager and its records department. The manager decides: approves the trade, enforces the policy. The records department files and retrieves: it doesn't decide whether a trade is allowed, it just keeps the record and finds it again when asked. The records department can switch from paper files to a database without the manager changing how they work.

In code, the **storage layer** is a package of modules, one per business area, mirroring the services:

```
portfolio_tracker/
    services/
        portfolio_service.py    # rules: duplicate names, not found, ...
        ...
    storage/
        portfolio_store.py      # keeps and finds portfolios
        asset_store.py
        account_store.py
        transaction_store.py
```

The first step is to move the existing dictionaries out of the services into storage modules, *without* adding a database yet:

```python
# storage/portfolio_store.py
portfolios = {}  # name -> Portfolio


def add_portfolio(portfolio):
    portfolios[portfolio.name] = portfolio


def get_portfolio(name):
    return portfolios.get(name)  # None when there is no such portfolio
```

```python
# services/portfolio_service.py
def find_portfolio(name):
    portfolio = portfolio_store.get_portfolio(name)
    if portfolio is None:
        raise NotFoundError(f"No portfolio named '{name}'")
    return portfolio
```

Notice the division of labor. Storage **only stores and returns**: when a portfolio doesn't exist, it returns `None` and doesn't judge. The service decides that a missing portfolio is a business error and raises `NotFoundError`. The rules and the errors stay in one place, the services. The application behaves exactly as before; only the location of the data has moved.

### Why this is worth doing

The functions `add_portfolio` and `get_portfolio` are now the only way the rest of the application touches stored portfolios. The services depend on *what* these functions do, not on *how*. That means the storage modules can be rewritten to use a database (in this handout they will be rewritten twice: once with SQL, once with SQLAlchemy) while the business rules stay untouched. Earlier this semester you translated a `KeyError` from a dictionary into a `NotFoundError` so that callers wouldn't depend on how the data was stored. This is where that pays off.

---

# Part C: Talking to the database with SQL

## 4. Connecting to the database

### Client, server, driver

MySQL is a **database server**: a separate program, running in the background, that owns the data and answers requests. Your Python application is a **client**: it connects to the server, sends SQL, and receives results. They talk over a network connection, even when both run on the same laptop.

Python doesn't speak MySQL's network protocol by itself. It needs a **driver**, a library that does. This course uses **PyMySQL**:

```
pip install PyMySQL cryptography python-dotenv
```

(`cryptography` is needed for the login method recent MySQL versions use; `python-dotenv` is explained below. As always, record the packages in `requirements.txt`.)

### Configuration belongs outside the code

To connect, the application needs a host, port, user name, password, and database name. These are **configuration**: they differ between your laptop, a classmate's laptop, and a production server, while the code stays the same. And the password is a secret.

So they do not belong in the code. They go in a file named `.env` in the project folder:

```
DB_HOST=localhost
DB_PORT=3306
DB_USER=tracker
DB_PASSWORD=your-password-here
DB_NAME=portfolio_tracker
```

The `python-dotenv` package reads that file into the program's **environment variables**, and the code reads them from there with `os.environ`. Most importantly, **`.env` goes in `.gitignore`.** Your code is pushed to GitHub; a password committed to a repository is effectively published, and deleting it in a later commit doesn't remove it from the history.

### The connection code

```python
# portfolio_tracker/database.py
import os

import pymysql
from dotenv import load_dotenv

load_dotenv()  # reads DB_HOST, DB_USER, ... from the .env file into os.environ


def connect():
    return pymysql.connect(
        host=os.environ["DB_HOST"],
        port=int(os.environ["DB_PORT"]),
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
        database=os.environ["DB_NAME"],
        cursorclass=pymysql.cursors.DictCursor,  # rows come back as dicts, not tuples
    )
```

`DictCursor` makes each result row a dictionary keyed by column name (`row["name"]`) instead of a tuple (`row[1]`), which is easier to read and doesn't break when someone reorders the columns in a query.

### Who creates the tables

The connection gives the application access to the database; it doesn't create the tables. With plain SQL, you create them yourself, once, by running the schema file through the `mysql` command-line client:

```
mysql -u tracker -p portfolio_tracker < schema.sql
```

## 5. Running queries

With a connection open, you send SQL through a **cursor**, an object for running one statement and reading its results:

```python
# storage/portfolio_store.py
from portfolio_tracker.models.portfolio import Portfolio


def add_portfolio(session, portfolio):
    with session.cursor() as cursor:
        cursor.execute(
            "INSERT INTO portfolios (name, strategy) VALUES (%s, %s)",
            (portfolio.name, portfolio.strategy),
        )
        portfolio.id = cursor.lastrowid


def get_portfolio(session, name):
    with session.cursor() as cursor:
        cursor.execute("SELECT id, name, strategy FROM portfolios WHERE name = %s", (name,))
        row = cursor.fetchone()
    if row is None:
        return None
    return Portfolio(id=row["id"], name=row["name"], strategy=row["strategy"])
```

(The `session` parameter is the open database connection; section 7 explains where it comes from and why it is passed in.)

- `cursor.execute(sql, values)` sends one statement. The `%s` markers are placeholders for the values; section 6 explains why this matters.
- `cursor.fetchone()` returns the next row, or `None` if there are no more. `cursor.fetchall()` returns all of them as a list.
- `cursor.lastrowid` is the `id` the database generated for the row just inserted.

Look at what `get_portfolio` has to do at the end: take a dictionary of column values and build a `Portfolio` object from it, field by field. Saving does the reverse, taking an object apart into column values. The database thinks in rows; the application thinks in objects; someone has to translate. With plain SQL, that someone is you, in every storage function. Reading a portfolio's transactions is where it gets heavy: a `JOIN` to fetch each trade's asset, then a `Transaction` and an `Asset` rebuilt from every row.

```python
def get_transactions(session, portfolio):
    with session.cursor() as cursor:
        cursor.execute(
            """
            SELECT t.side, t.quantity, t.price_per_share, t.trade_date,
                   a.id AS asset_id, a.ticker, a.name
            FROM transactions t
            JOIN assets a ON a.id = t.asset_id
            WHERE t.portfolio_id = %s
            ORDER BY t.id
            """,
            (portfolio.id,),
        )
        rows = cursor.fetchall()
    return [
        Transaction(
            portfolio=portfolio,
            asset=Asset(id=row["asset_id"], ticker=row["ticker"], name=row["name"]),
            side=row["side"],
            quantity=row["quantity"],
            price_per_share=row["price_per_share"],
            trade_date=row["trade_date"],
        )
        for row in rows
    ]
```

Keep this function in mind; Part D replaces it with one line.

## 6. Parameterized queries and SQL injection

### The tempting mistake

You know f-strings, so the natural way to put a value into SQL looks like this:

```python
def get_portfolio(session, name):
    sql = f"SELECT id, name, strategy FROM portfolios WHERE name = '{name}'"
    with session.cursor() as cursor:
        cursor.execute(sql)
        ...
```

It works when you test it with "Tech growth". It is also the most common serious security flaw in database code.

### Why it breaks

Picture a bank's wire-transfer form with a memo field for the customer's note. Now picture a clerk who reads the memo out loud *as part of the instruction*: "Transfer $500 to account 1234, memo: *and also transfer $10,000 to account 9999*." The clerk can't tell where the bank's instruction ends and the customer's text begins.

An f-string does the same thing. It glues the user's text into the SQL, so the database receives one string and cannot tell which part you wrote and which part the user typed. Three inputs show what happens:

1. **An honest apostrophe.** A user creates a portfolio called `Mom's retirement`. The SQL becomes `... VALUES ('Mom's retirement', ...)`. The apostrophe ends the string early and the database reports a syntax error. No attacker is involved: this is a correctness bug first.
2. **A crafted name.** A user asks for the portfolio named `x' OR '1'='1`. The SQL becomes:

   ```sql
   SELECT id, name, strategy FROM portfolios WHERE name = 'x' OR '1'='1'
   ```

   `'1'='1'` is true for every row, so the query returns *all* portfolios, and `fetchone()` hands back whichever comes first. The user's trade is booked to the wrong portfolio, silently, with no error.
3. **The same trick in an update.** If prices were stored with `f"UPDATE prices SET price = {price} WHERE ticker = '{ticker}'"`, the ticker `x' OR '1'='1` would reprice *every* asset at once, and every portfolio's market value would be wrong.

This is **SQL injection**: user input changing the meaning of a query.

### The fix: placeholders

Never build SQL out of user values. Write the SQL with placeholders and pass the values separately:

```python
cursor.execute(
    "SELECT id, name, strategy FROM portfolios WHERE name = %s",
    (name,),
)
```

The driver turns each value into a safe SQL literal, escaping any quotes inside it, so a value can never change the shape of the query. With this version, the crafted name `x' OR '1'='1` is simply looked up as a (strange) portfolio name, no portfolio matches, and the service raises `NotFoundError`. `Mom's retirement` is stored correctly.

Watch for these pitfalls:

- `%s` looks like Python's old string formatting, but it is not. Don't put quotes around it, and don't use the `%` operator: `"... WHERE name = '%s'" % name` is the f-string bug again.
- The values go in a tuple. For a single value, the trailing comma in `(name,)` is what makes it a tuple.
- Placeholders work for *values* only, not for table or column names.

**The review rule.** In any database code you review, look at every `execute(` call. If the SQL string is assembled from variables (with an f-string, `+`, `%`, or `.format()`), it is injectable.

## 7. Transactions: keeping related changes together

### All or nothing

Buying shares changes two things: the cash balance goes down, and a trade is recorded. Suppose the first change succeeds and the second fails, because the connection drops or a constraint rejects the row. The cash is gone, and there is no record of what it bought.

A trade either settles completely or not at all; a half-settled trade is a reconciliation nightmare. Databases provide exactly this guarantee with **transactions**: a group of changes that are made permanent together (**commit**) or undone together (**rollback**).

With PyMySQL, every connection starts a transaction automatically with the first statement. Nothing is permanent until you call `commit()`. If you call `rollback()` instead, everything since the last commit is undone:

```python
session = database.connect()
try:
    ...                      # debit the cash, record the trade
    session.commit()         # both changes become permanent
except Exception:
    session.rollback()       # neither change happened
    raise                    # let the error reach the edge, as usual
finally:
    session.close()
```

This is the exception handling you already know, put to a new use: an exception partway through the work triggers the rollback, then continues to the interface, which reports it to the user.

### The transaction boundary: who decides when to commit?

A transaction belongs to a **connection**, not to a function or a cursor. Every statement sent on the same connection between two commits is part of the same transaction. That has a design consequence.

Suppose each storage function opened its own connection and committed its own change. `debit` would commit the cash change on one connection; then `add_transaction` would fail on another. The debit is already permanent; nothing can roll it back. **Two connections means two transactions**, however carefully each one commits.

So the decision to commit belongs to the **business operation**, the thing that has to succeed or fail as a whole. For the tracker, "buy shares" is the unit of work, not "update the balance" or "insert a row." The boundary is the deal, not each line item.

In code, the service operation opens one connection with one transaction, and passes it to every function that does part of the work. We call that open connection a **session**: one conversation with the database, inside one transaction. A small helper packages the commit-or-rollback pattern above so every operation can reuse it:

```python
# portfolio_tracker/database.py
from contextlib import contextmanager


@contextmanager
def transaction():
    """One unit of work: commit if the block finishes, roll back if it raises."""
    session = connect()  # a "session": one connection holding one open transaction
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

`@contextmanager` turns the function into something you can use with `with`, like `with open(...)`. Everything before `yield` runs when the `with` block starts; `yield session` hands the session to the block; everything after `yield` runs when the block ends, normally or with an exception. The service code then reads clearly:

```python
# services/transaction_service.py
def buy(portfolio_name, ticker, quantity, price_per_share):
    # One transaction for the whole trade: the debit and the trade record succeed or fail together.
    with database.transaction() as session:
        portfolio = portfolio_service.find_portfolio(session, portfolio_name)
        asset = asset_service.find_asset(session, ticker)
        require_positive("Quantity", quantity)
        require_positive("Price per share", price_per_share)

        account_service.debit(session, quantity * price_per_share)
        transaction_store.add_transaction(
            session,
            Transaction(
                portfolio=portfolio,
                asset=asset,
                side="buy",
                quantity=quantity,
                price_per_share=price_per_share,
                trade_date=date.today(),
            ),
        )
```

If anything inside the `with` block raises, whether a business error such as an unknown ticker or a database error, the debit is rolled back.

This gives a simple rule for which functions open a transaction:

- **Functions the interface calls** (create a portfolio, deposit, buy, show holdings) open the transaction with `with database.transaction() as session:`.
- **Functions other services call** to do part of a larger job (`find_portfolio`, `find_asset`, `debit`) receive the `session` as their first argument, so their work joins the caller's transaction.

Storage functions always receive the session and never commit. They don't know what larger operation they are part of, so they can't know when it is finished.

**Deeper dive.** Two things this course doesn't need but you may meet. *Isolation*: while a transaction is open, other connections may or may not see its uncommitted changes, depending on the database's isolation level. With one user that never comes up; in a multi-user system it matters a great deal. *Schema changes*: MySQL commits automatically before and after `CREATE TABLE` and `ALTER TABLE`, so changes to the structure of the database can't be rolled back.

## 8. What hurts about writing SQL by hand

The application now keeps its data between runs, safely. Three costs show up quickly:

- **Translation by hand.** Every read rebuilds objects from rows; every write takes objects apart into values. The transaction query in section 5 is mostly translation.
- **SQL in strings.** The database's structure is spread through string literals. Rename a column and you search every string for it; a typo in a query isn't found until that line runs.
- **Repetition.** Every storage function repeats the same cursor-execute-fetch pattern.

None of this makes SQL wrong. SQL is the right tool for many jobs (section 15 covers when). But for the everyday work of saving and loading the application's business objects, the translation is repetitive enough that a library can do it for you.

---

# Part D: SQLAlchemy

## 9. What an ORM does

An **object-relational mapper (ORM)** is a library that translates between the application's objects and the database's rows. You describe, once, how a class corresponds to a table. From then on you save objects and get objects back, and the ORM writes the SQL. **SQLAlchemy** is Python's most widely used ORM.

It doesn't replace the driver; it sits on top of it. SQLAlchemy still sends SQL to MySQL through PyMySQL, the same driver as in Part C. What it removes is the translation code you wrote by hand.

```
pip install SQLAlchemy
```

**A note on versions.** SQLAlchemy changed its style substantially in version 2.0. This handout uses the current style (`Mapped`, `mapped_column`, `select()`). Many older tutorials, and a lot of code produced by AI tools, use the older 1.x style, with `session.query(...)`. Both work, but don't mix them in one project, and recognize the old style when you see it.

## 10. Engine and session

```python
# portfolio_tracker/database.py
import os

from dotenv import load_dotenv
from sqlalchemy import URL, create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

load_dotenv()  # reads DB_HOST, DB_USER, ... from the .env file into os.environ

url = URL.create(
    "mysql+pymysql",  # SQLAlchemy talks to MySQL through the same PyMySQL driver
    username=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
    host=os.environ["DB_HOST"],
    port=int(os.environ["DB_PORT"]),
    database=os.environ["DB_NAME"],
)
engine = create_engine(url)  # echo=True prints every SQL statement SQLAlchemy sends
# expire_on_commit=False: objects stay readable after their transaction ends.
Session = sessionmaker(engine, expire_on_commit=False)


class Base(DeclarativeBase):
    """Parent of every model class; it collects their tables."""


def transaction():
    """One unit of work: commit if the block finishes, roll back if it raises."""
    return Session.begin()
```

The configuration is the same `.env` file as before. The pieces:

- **`URL.create`** builds the connection address. It is safer than writing the address as an f-string, because it correctly handles passwords that contain characters such as `@` or `/`.
- **The engine** is the application's gateway to the database. It manages connections and reuses them rather than opening a new one for every request. You create one engine for the whole application.
- **A session** is SQLAlchemy's version of the idea from section 7: one conversation with the database, inside one transaction. `sessionmaker` creates a factory for sessions.
- **`Session.begin()`** opens a session and a transaction, commits when the `with` block finishes, rolls back if it raises, and closes the session either way. That is exactly the `try` / `commit` / `rollback` / `close` you wrote by hand in section 7, so `transaction()` shrinks to one line.

Because `transaction()` still works with `with database.transaction() as session:`, the services don't change at all.

**Deeper dive.** By default SQLAlchemy *expires* every object when a transaction commits: the next time you read an attribute, it reloads the value from the database, in case another connection changed it. That reload needs an open session, and our services read results after the `with` block has closed it. `expire_on_commit=False` keeps the loaded values. The trade-off is that a long-lived object could show data that is out of date; in a single-user application that never happens.

## 11. Models: classes that describe tables

A **model** is a class that describes one table: its name, its columns, and their types. Each instance of the class is one row.

```python
# models/portfolio.py
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from portfolio_tracker.database import Base


class Portfolio(Base):
    __tablename__ = "portfolios"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)
    strategy: Mapped[str] = mapped_column(String(255))
```

- Inheriting from **`Base`** registers the class with SQLAlchemy.
- **`__tablename__`** names the table.
- Each **`Mapped[...]`** line is a column. The type in brackets is the Python type (`int`, `str`, `Decimal`, `date`), and `mapped_column(...)` adds the database details: the SQL type (`String(100)`, `Numeric(14, 2)`), the primary key, `unique=True`. A column is `NOT NULL` unless its type is written as optional.

Models get a constructor automatically that takes keyword arguments: `Portfolio(name="Tech growth", strategy="Growth stocks")`. The `id` is left out; the database assigns it when the row is saved.

Not every class becomes a model. A `Holding` (quantity, average cost, market value) is calculated, not stored (section 2), so it stays a plain Python class. The ORM is for the facts you store.

## 12. Relationships

The `transactions` table has two foreign keys. The model declares them as columns, and adds **relationships**, which let you work with the objects the foreign keys point to:

```python
# models/transaction.py
class Transaction(Base):
    __tablename__ = "transactions"

    id: Mapped[int] = mapped_column(primary_key=True)
    portfolio_id: Mapped[int] = mapped_column(ForeignKey("portfolios.id"))
    asset_id: Mapped[int] = mapped_column(ForeignKey("assets.id"))
    side: Mapped[str] = mapped_column(Enum("buy", "sell"))
    quantity: Mapped[Decimal] = mapped_column(Numeric(14, 4))
    price_per_share: Mapped[Decimal] = mapped_column(Numeric(14, 4))
    trade_date: Mapped[date]

    # The foreign keys above are the columns; these are the objects they point to.
    portfolio: Mapped[Portfolio] = relationship()
    asset: Mapped[Asset] = relationship(lazy="selectin")  # load the asset along with the trade
```

`portfolio_id` holds a number; `portfolio` holds a `Portfolio` object. You create a trade with objects, `Transaction(portfolio=portfolio, asset=asset, ...)`, and SQLAlchemy fills in the ID columns when it saves. When you read a trade back, `trade.asset.ticker` just works: no `JOIN` and no rebuilding an `Asset` from columns.

### Lazy loading, and why `lazy="selectin"` is there

By default, SQLAlchemy loads a related object only the first time you touch it. Reading `trade.asset` sends a small query for that asset at that moment. That is **lazy loading**, and it saves work when you never look at the asset. But it needs the session to still be open. The tracker reads each trade's asset when it calculates holdings, after the transaction has closed, so a lazy `asset` would fail with an error saying the object is no longer attached to a session.

`lazy="selectin"` changes the loading strategy for that relationship: whenever trades are loaded, their assets are loaded right along with them, in one extra query for all of them together. The asset is already there when you need it.

**Deeper dive.** Lazy loading is also the source of a classic performance problem, the **N+1 query problem**. Load 100 trades (1 query), then touch each trade's asset in a loop (up to 100 more queries). Each query is fast, but hundreds of them add up. Choosing a loading strategy such as `selectin` is how you avoid it. When reviewing ORM code that loops over objects and touches their relationships, ask how many queries it sends; `echo=True` on the engine will show you.

## 13. Creating the tables

SQLAlchemy can create the tables from the models:

```python
def create_tables():
    # Creates any missing tables for the models that have been imported (see main.py).
    Base.metadata.create_all(engine)
```

The application calls `create_tables()` once at startup. `create_all` creates only the tables that don't exist yet, so running it every time is safe. The schema file from Part A is kept as the readable record of the design. The tables SQLAlchemy creates from these models match it, column for column.

Two cautions:

- **`create_all` only knows about models that have been imported.** If a model module is never imported, its table is silently skipped. So the entry point imports every model explicitly:

  ```python
  # main.py
  # Imported so create_tables() knows every table, even though main.py never uses them by name.
  from portfolio_tracker.models import account, asset, portfolio, transaction  # noqa: F401
  ```

  Ruff would report these imports as unused (rule `F401`), and `ruff check --fix` would delete them, silently breaking table creation. The `# noqa: F401` comment tells ruff the import is deliberate.

- **`create_all` never changes a table that already exists.** Add a column to a model after the table has been created and the database won't get it. Changing the structure of a database that already holds data is called a **migration**. Real projects use a migration tool (for SQLAlchemy, a companion library called Alembic). It is out of scope for this course, but it's the answer to "what happens when the design changes after there's data in it?"

## 14. Saving and fetching

Here are the storage modules again, now with SQLAlchemy:

```python
# storage/portfolio_store.py
from sqlalchemy import select

from portfolio_tracker.models.portfolio import Portfolio


def add_portfolio(session, portfolio):
    session.add(portfolio)


def get_portfolio(session, name):
    return session.scalars(select(Portfolio).filter_by(name=name)).first()
```

```python
# storage/transaction_store.py
def add_transaction(session, transaction):
    session.add(transaction)


def get_transactions(session, portfolio):
    return session.scalars(select(Transaction).filter_by(portfolio=portfolio)).all()
```

Compare `get_transactions` with the version in section 5. The `JOIN` and the row-by-row rebuilding are gone; the relationship from section 12 supplies each trade's asset.

Four operations cover the storage layer:

| To... | Write | Notes |
| --- | --- | --- |
| Save a new object | `session.add(obj)` | Written to the database when the transaction commits. |
| Fetch by primary key | `session.get(Account, 1)` | Returns the object, or `None`. |
| Fetch by anything else | `session.scalars(select(Model).filter_by(field=value))` then `.first()` or `.all()` | `.first()` gives one object or `None`; `.all()` gives a list. |
| Update | Change an attribute: `account.cash_balance = new_balance` | The session notices the change and writes the `UPDATE` on commit. |

**Why `scalars`?** A `select` can ask for several things per row: `select(Portfolio, Asset)` returns rows that each hold a portfolio *and* an asset. So results always come back as rows, even when you asked for one thing. `scalars()` says "I selected one thing per row; give me those things, not one-item rows."

**Updates without an `UPDATE`.** Here is the account store:

```python
# storage/account_store.py
ACCOUNT_ID = 1  # the single implicit account


def get_account(session):
    account = session.get(Account, ACCOUNT_ID)
    if account is None:  # first run against empty tables
        account = Account(id=ACCOUNT_ID, cash_balance=Decimal("0"))
        session.add(account)
    return account


def get_cash_balance(session):
    return get_account(session).cash_balance


def set_cash_balance(session, amount):
    get_account(session).cash_balance = amount  # the session notices and writes the UPDATE
```

The session keeps track of every object it has loaded. When the transaction commits, it compares each object with what it loaded and writes an `UPDATE` for whatever changed. (The `if account is None` branch exists because `create_all` creates an empty `account` table, without the row the schema file inserts.)

**One object per row.** Within one session, SQLAlchemy keeps exactly one Python object for each database row. Buy AAPL twice in the same portfolio and load both trades: `trade1.asset is trade2.asset` is `True`, because there is one AAPL row. The plain-SQL version in section 5 builds a new `Asset` from every result row, giving two separate objects describing the same row.

## 15. The payoff, and when SQL is still the right tool

### The services didn't change

Between the plain-SQL version and the SQLAlchemy version, **not one line of the service modules changed**. The storage modules, the models, and the database module were rewritten; every business rule, error, and log message stayed the same. Earlier the storage moved from dictionaries to a database, and the rules were untouched then too. Each layer has one job, and changing how one layer does its job leaves the others alone. Later this semester the command-line interface will be replaced by a web API, and the same separation will protect the rules again.

### When to write SQL anyway

Use the ORM for the everyday work on individual business objects: load a portfolio, record a trade, list a portfolio's transactions. Write SQL directly when the work is about **many rows at once**:

- **Reports and aggregations.** "Total invested per portfolio per month" is a `GROUP BY` that SQL expresses clearly. The ORM can express it too, often less readably.
- **Bulk changes.** Updating the price of 5,000 tickers at the end of the day is one `UPDATE` in SQL. Through the ORM, it is 5,000 objects loaded, changed, and written back.
- **Performance-critical queries,** where you need control over exactly what SQL runs.
- **Database-specific features** that the ORM doesn't cover.

The tracker's holdings calculation is an example. It loads every transaction and adds them up in Python. With a database, it could be one `GROUP BY` query that returns the totals directly. For a personal tracker with a few hundred trades the Python loop is fine. For a brokerage with millions of trades, it isn't.

SQLAlchemy runs plain SQL too, in the same session and transaction, with the same placeholder protection (named placeholders, written `:name`):

```python
from sqlalchemy import text

session.execute(text("SELECT name FROM portfolios WHERE name = :name"), {"name": name})
```

Writing `text(f"...")` with an f-string brings the injection hole straight back.

### Reviewing database code

When you review database code, whether a classmate's or code generated by a tool, these questions catch most problems:

1. **Is any SQL built from variables?** f-strings, `+`, `%`, or `.format()` inside `execute(...)` or `text(...)` means injection.
2. **Where does it commit?** Commits inside storage functions, or separate connections for steps of one operation, mean the operation isn't all-or-nothing.
3. **Where is the database code?** SQL or session calls mixed into business rules or the interface mean the layers have collapsed.
4. **Are secrets in the code?** A password in a `.py` file, or a `.env` file missing from `.gitignore`, is a leak waiting to happen.
5. **Is money a float?** `FLOAT` columns or `float()` conversions on amounts mean rounding errors.
6. **How many queries does a loop send?** An ORM loop touching relationships may send one query per item.
7. **Does it store what it could calculate?** Stored totals or positions alongside the transactions they come from will eventually disagree.

---

## Summary

- **Data outlives code.** Design what the application stores, and how it relates, before writing code. Store facts; calculate what can be calculated from them.
- **Keys and constraints:** a generated `id` as primary key; business identifiers protected by `UNIQUE`; foreign keys for relationships. Constraints are business rules the database enforces.
- **Money is `DECIMAL`**, and `Decimal` in Python. Convert user input at the edge; `Decimal` and `float` don't mix. The scale of a column is a rounding policy.
- **The storage layer** keeps data; the **services** keep the rules. Storage returns `None` for "not found"; services raise the business error. This is what lets storage be replaced without touching the rules.
- **Configuration and secrets** go in `.env`, read with `python-dotenv`. `.env` goes in `.gitignore`.
- **Never build SQL from values.** Use placeholders (`%s` in PyMySQL, `:name` in SQLAlchemy's `text`) and pass the values separately.
- A **transaction** makes a group of changes permanent together (commit) or undoes them together (rollback). It belongs to a connection or session. The **business operation** owns the boundary: functions the interface calls open the transaction; functions that do part of the work receive the session; storage never commits.
- An **ORM** translates between objects and rows. In SQLAlchemy: one **engine**; a **session** per unit of work (`Session.begin()` commits or rolls back for you); **models** describe tables; **relationships** give you related objects; `create_all` creates missing tables.
- Storage with SQLAlchemy is four operations: `session.add`, `session.get`, `session.scalars(select(...).filter_by(...))`, and changing attributes.
- Use the ORM for everyday object work and **SQL for set-based work** such as reports and bulk changes, always with placeholders.

## Check your understanding

1. Why does the tracker store transactions but not holdings? What would go wrong if it stored both?
2. The `assets` table uses a generated `id` as its primary key even though tickers are unique. Give a business reason for that choice.
3. A classmate stores the cash balance as `FLOAT`. Explain the problem using `0.1 + 0.2`. Then explain what goes wrong in their Python code once the column is changed to `DECIMAL` but the input is still converted with `float(text)`.
4. In the storage layer, `get_portfolio` returns `None` when no portfolio matches, rather than raising `NotFoundError`. Why does the error belong in the service instead?
5. Write out the SQL that `f"SELECT * FROM assets WHERE ticker = '{ticker}'"` produces when the user types `x' OR '1'='1`. What happens, and how do placeholders prevent it?
6. Your `buy` function debits the cash and then records the trade. Each storage function opens its own connection and commits. Describe a failure that leaves the data wrong, and how a single transaction prevents it.
7. Which functions in the tracker open a transaction, and which receive a session? Why does a storage function never call `commit()`?
8. Name the three costs of writing SQL by hand that motivate an ORM. Which one does the `asset` relationship remove?
9. What does `session.scalars(...)` return, and when do you finish it with `.first()` versus `.all()`?
10. A model gains a new column, but the existing table doesn't get it even though `create_all` runs at every startup. Why? What is the general name for the solution?
11. Give two situations where you would write SQL directly instead of using the ORM, and the one rule that applies either way.
12. The services didn't change when the storage moved from plain SQL to SQLAlchemy. Which design decisions made that possible?

## Glossary

- **Data-first design**: deciding what an application stores, and how it relates, before designing the code.
- **Schema**: the structure of a database: its tables, columns, types, keys, and constraints.
- **Primary key**: the column that uniquely identifies each row; here, a generated `id`.
- **Foreign key**: a column that points to a row in another table, such as `transactions.portfolio_id`.
- **Constraint**: a rule the database enforces, such as `UNIQUE`, `NOT NULL`, or a foreign key.
- **`DECIMAL` / `Decimal`**: exact decimal number types, in the database and in Python; used for money.
- **Storage layer**: the modules that keep and retrieve data, separate from the services that hold business rules.
- **Database server**: the program (here, MySQL) that owns the data and answers requests from clients.
- **Driver**: the library that lets Python talk to a particular database (here, PyMySQL).
- **`.env` file**: a local file of configuration values and secrets, kept out of Git.
- **Cursor**: the object used to send one SQL statement and read its results.
- **Placeholder**: a marker in SQL (`%s`, or `:name`) where the driver safely inserts a value.
- **SQL injection**: user input changing the meaning of a query, possible whenever SQL is built from user values.
- **Transaction**: a group of database changes that are made permanent together or undone together.
- **Commit / rollback**: making a transaction's changes permanent / undoing them.
- **Session**: one conversation with the database, inside one transaction.
- **Transaction boundary**: where a unit of work begins and ends; owned by the business operation.
- **ORM (object-relational mapper)**: a library that translates between objects and rows.
- **Engine**: SQLAlchemy's gateway to the database; it manages connections for the whole application.
- **Model**: a class that describes a table; each instance is a row.
- **Relationship**: a model attribute holding the related object(s) a foreign key points to.
- **Lazy loading**: loading a related object only when it is first accessed.
- **N+1 query problem**: one query to load a list, plus one more per item to load something related.
- **`create_all`**: SQLAlchemy's method that creates any tables that don't exist yet.
- **Migration**: a controlled change to the structure of a database that already holds data.
