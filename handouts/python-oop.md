# Object-Oriented Programming in Python

## Why this topic, and why now

Up to this point, most of the Python you have written has probably been a script: a file that runs from top to bottom, loading some data, transforming it, and printing or saving a result. That is a perfectly good way to answer a question. It is not a good way to build a system that has to keep growing, and the portfolio tracker you will build this semester is exactly that kind of system. As a reminder, the tracker records buy and sell transactions for financial assets and computes what you currently hold, what you paid for it, and how much you have gained or lost.

Before writing more Python, this week changes *how you think about what a program is*. Instead of a script that runs top to bottom, you will start to see a program as a world of **objects** that each know certain things and can do certain things, and that interact with each other. That shift in thinking matters more than any single piece of syntax in this handout. It is the foundation for every Python module that follows, and later in your career it is what will let you look at code someone else (or an AI tool) wrote and judge whether it is organized well.

The handout builds up in this order:

1. Programming paradigms: the different "lenses" for telling a computer what to do, and where object-oriented programming fits among them.
2. The core idea of object-oriented programming: a program models the real world, and the difference between a *class* and an *object*.
3. Building your first class in Python, piece by piece.
4. Creating objects from that class and using them.
5. The four principles of object-oriented programming, with **inheritance** covered in depth.

## Part 1: Programming paradigms

There isn't one correct way to write code. A **programming paradigm** is a style, or lens, for describing to a computer what you want it to do. Different paradigms suit different problems, and you have already used more than one of them without naming them.

### Imperative: telling the computer *how*, step by step

In the **imperative** paradigm, you write explicit, ordered instructions: do this, then this, then this. You are responsible for spelling out *how* the result gets produced.

The analogy is a recipe. Preheat the oven, mix the dry ingredients, add the eggs, bake for 30 minutes. The order matters, and every step is spelled out.

Most of the Python you have written is imperative. A typical pandas script loads a CSV, filters some rows, computes a new column, and prints a summary, in exactly that sequence. A plain loop is imperative too:

```python
prices = [190, 410, 145]
total = 0
for p in prices:
    total = total + p
print(total)
```

You told Python exactly how to get the total: start at zero, walk through the list, add each value.

### Declarative: telling the computer *what*

In the **declarative** paradigm, you describe the result you want and leave it to the system to work out how to produce it.

The analogy is ordering at a restaurant. You say "I'd like the pasta." You don't tell the kitchen to boil water, salt it, and set a timer. You describe the outcome, and the kitchen handles the steps.

SQL is the example you already know:

```sql
SELECT * FROM transactions WHERE amount > 1000;
```

This query never tells the database how to search. It doesn't say "open the table, read the first row, check the amount, move to the next row." It only says what you want back. The database decides how to find it.

### Object-oriented: organizing a program around objects

The third lens is the **object-oriented** paradigm, usually shortened to **OOP**. In OOP, a program is organized around *objects* that bundle together data (what the object knows) and behavior (what the object can do). The rest of this handout is about what that means and how to do it in Python.

### Paradigms are lenses, not rival camps

These paradigms are not mutually exclusive, and nobody has to pick one for life. Most real programs mix them: an object-oriented program still contains step-by-step imperative code inside its functions, and it may run declarative SQL queries against a database. A paradigm is a choice about how to *organize and think about* a program. Starting this week, this course and the portfolio tracker adopt OOP as the primary lens.

## Part 2: A program models the real world

Here is the central idea of object-oriented programming:

> **A computer program is modeled after the real world. Our world is made up of objects that interact with each other, and so is a program.**

In the real world, a brokerage account, a stock, and a trade are all distinct "things." Each has its own facts (an account has a balance, a stock has a ticker and a price, a trade has a date and a quantity), and each can do or have certain things done to it. OOP asks you to structure your program the same way: identify the things in the problem, describe what each one knows and does, and let them interact.

To do that, you need two ideas that are easy to mix up: the **class** and the **object**.

### Plato's chair: the idea versus the thing

The Greek philosopher Plato had a theory that turns out to be a useful way to understand classes. He argued that every chair you encounter in the physical world is an instance of an abstract *Form*: the idea of "Chair." The Form describes what a chair is like and what it does (it has a seat, it has legs, you can sit on it). But the Form is not any particular chair. It is the description that all particular chairs share.

Map that directly onto programming:

- The **Form** ("Chair" as an idea) is the **class**.
- A **particular chair** (the one you are sitting on right now) is an **object**.

You have never sat on the Form of a chair. You only ever sit on particular chairs. In the same way, a program never really "uses" a class directly to hold data. It uses objects that were made from the class.

### The blueprint and the house

A more practical version of the same idea: a contractor builds a house by following a blueprint.

- The **blueprint** is the **class**.
- The **house** is the **object**.

This distinction is the most important point in the whole topic: **the blueprint is not itself a house.** You cannot live in a blueprint. It is a description of what a house will look like and what it will contain. Only once the contractor builds from the blueprint do you get a house you can walk into. In programming terms, the class is the description, and once the class is **instantiated** (built), you get an object. An object is also called an **instance** of its class. The two words mean the same thing.

The analogy extends naturally. One blueprint can produce an entire subdivision of houses. Every house follows the same description, but each one is its own house: independently owned, painted a different color, with different people living in it. Painting one house blue doesn't paint the others. The same holds in code: **one class can produce many objects, and each object holds its own data.**

### More examples of classes and objects

The distinction becomes second nature with repetition. For each class below, notice that the class is a *general kind of thing*, and each object is one *specific* example of it.

| Class (the description) | Objects (specific instances) |
|---|---|
| `Car` | the specific car in your driveway; the specific car in your neighbor's driveway |
| `Employee` | each individual person on the company payroll |
| `BankAccount` | each customer's individual account, each with its own balance |
| `Asset` | 10 shares of AAPL; 5 shares of MSFT |

A common mistake is naming a specific thing as the class, such as offering "my Honda" as a class. "My Honda" is an object. The class is `Car`, the general description that your Honda, and every other car, is an instance of. A quick test: if you can point to it, it's an object. If it describes what all things of that kind have in common, it's a class.

### The classes behind the portfolio tracker

The portfolio tracker will be built around three classes, each modeling something real in the investing world:

- **`Asset`**: a tradable instrument, such as a stock, identified by its ticker.
- **`Transaction`**: a single buy or sell of an asset.
- **`Position`**: what you currently hold in a given asset, built up from your transactions.

The rest of this handout builds a first version of `Asset`.

## Part 3: Building a class in Python

### How the code is organized

The convention used in class is **one class per file, with the file named after the class it contains.** The `Asset` class lives in its own file, named `asset.py`. Keeping each class in its own clearly named file means anyone opening the project can find a class by its name.

### What a class defines: properties and behavior

Before looking at syntax, hold on to one framing, because every line of the class falls into one of these two categories. A class defines two things about an object:

1. **Properties**: the data the object knows about itself. For an asset: its ticker, how many shares, and the price.
2. **Behavior**: the functions available to the object, meaning what the object can *do*. For an asset: calculate its own market value.

That is all a class ever is: a description of what its objects know and what they can do.

### The finished class

Here is the complete class. Each piece is explained below.

```python
class Asset:
    def __init__(self, ticker, quantity, price):
        self.ticker = ticker
        self.quantity = quantity
        self.price = price

    def market_value(self):
        return self.quantity * self.price
```

### Step 1: declaring the class

```python
class Asset:
```

This line says "here is a description of a kind of thing called `Asset`." It is the blueprint, not a house. Running this line does not create any asset. It only defines what an asset *would* look like. Class names are written with each word capitalized (`Asset`, `BankAccount`), which makes them easy to tell apart from ordinary variables and functions.

Everything indented underneath belongs to the class.

### Step 2: the constructor, `__init__`

```python
    def __init__(self, ticker, quantity, price):
```

This is a special function called the **constructor**. The name is `__init__` (two underscores on each side, short for "initialize"). Three things about it:

- **It runs automatically, exactly once, at the moment an object is created.** You never call `__init__` by name yourself. Python calls it for you when you create an object (shown in Part 4).
- **Its job is to make sure every new object starts out valid**, meaning it holds all the information it needs to be a real instance of this class. An asset without a ticker, quantity, or price wouldn't make sense, so the constructor requires all three up front.
- **Its parameters are the information you must supply to create an object**, just like the parameters of any function you have written before. Here: `ticker`, `quantity`, and `price`. (The first parameter, `self`, is special and is explained in Step 4.)

A business comparison: the constructor is like an account-opening form at a bank. The bank won't open an account until the required fields are filled in, so every account that exists is guaranteed to have them.

### Step 3: storing the properties

```python
        self.ticker = ticker
        self.quantity = quantity
        self.price = price
```

These three lines are the object's **properties**. Each one takes a value passed into the constructor and stores it *on this particular object*.

Read `self.ticker = ticker` as: "take the `ticker` value that was just handed in, and save it as this object's own `ticker`." The name on the left (`self.ticker`) is the object's permanent storage. The name on the right (`ticker`) is the temporary parameter that only exists while the constructor is running. They happen to share a name, which is the normal convention, but they are two different things. Once the constructor finishes, the parameter is gone, and the value lives on inside the object.

### Step 4: what `self` means

`self` is the piece that confuses people most, so here it is stated directly:

> **`self` means "whichever object is currently being worked on."**

When Python builds a new `Asset`, it passes that brand-new object into `__init__` as `self`. So `self.ticker = ticker` means "store the ticker on *this* new object." Later, when you ask a specific asset for its market value, Python passes *that* asset in as `self`.

A few facts about `self`:

- It is **not a special keyword** in Python. It is an ordinary parameter, and naming it `self` is a universal convention. Always follow it.
- **You never pass it in yourself.** Python fills it in automatically with the object the call was made on.
- **Each object's `self` refers to that object and nothing else.** This is what allows one blueprint to produce many objects holding different data without interfering with each other. When the AAPL asset stores `self.quantity = 10`, it is writing into the AAPL object's own storage, not into some shared location that the MSFT object could overwrite.

Think back to the subdivision. Every house was built from the same blueprint, but when a homeowner says "paint *my* house blue," there is no confusion about which house is meant. `self` is how a piece of code inside the class says "my."

### Step 5: behavior, the method

```python
    def market_value(self):
        return self.quantity * self.price
```

A function defined inside a class is called a **method**. A method is the object's **behavior**: something the object can do. Methods are ordinary functions like the ones you already know how to write, with one difference: they belong to an object and automatically receive that object as `self`.

Notice what `market_value` reaches for: `self.quantity` and `self.price`. It does not need anyone to pass in the quantity and price, and it doesn't look for variables floating somewhere else in the file. **The object already knows its own quantity and price**, so the method just uses them. The behavior lives right next to the data it operates on.

Compare this to the scripting approach, where you might have a list of tickers, a separate list of quantities, a separate list of prices, and a function somewhere else that you have to remember to call with the right three values in the right order. With a class, the data and the logic that uses it travel together as one unit.

### Reading the class as a sentence

Read the finished class back in plain English:

> An `Asset` knows its **ticker**, **quantity**, and **price**, and it can tell you its **market value**.

The first half is its properties, and the second half is its behavior. If you can describe any class in a sentence like this, you understand what it is for.

## Part 4: Creating and using objects

So far there is only a blueprint. This part builds houses from it.

### Instantiating: creating objects from the class

```python
aapl = Asset("AAPL", 10, 190)
msft = Asset("MSFT", 5, 410)
```

Writing the class name as if it were a function call, `Asset(...)`, is what creates a new object. When Python sees this, it:

1. creates a brand-new, empty `Asset` object,
2. automatically calls `__init__`, passing the new object in as `self` and your values in as `ticker`, `quantity`, and `price`,
3. hands the finished object back, which you store in a variable (`aapl`, `msft`).

Notice that you supply three values, not four. `self` is filled in by Python. The values are matched to the constructor's parameters in order: `"AAPL"` becomes `ticker`, `10` becomes `quantity`, `190` becomes `price`.

These two lines just built two houses from one blueprint. `aapl` and `msft` are two separate objects, each an instance of `Asset`, each holding its own data.

### Accessing properties with the dot

```python
print(aapl.ticker)     # AAPL
print(msft.quantity)   # 5
```

The dot means "reach into this particular object and get the value it holds." `aapl.ticker` asks the `aapl` object for its ticker. `msft.quantity` asks the `msft` object for its quantity. You name *which object* first, then *what you want from it*.

### Calling methods

```python
print(aapl.market_value())   # 1900
print(msft.market_value())   # 2050
```

Calling a method uses the same dot, followed by parentheses. Look at what happens here: **the exact same method returns different numbers.** Nothing about `market_value` changed between the two calls. What changed is the object it was called on. When you write `aapl.market_value()`, Python passes `aapl` in as `self`, so `self.quantity * self.price` is `10 * 190`. When you write `msft.market_value()`, `self` is `msft`, so the calculation is `5 * 410`.

### Putting it all together

Here is the complete, runnable example. You can paste it into a single file and run it:

```python
class Asset:
    def __init__(self, ticker, quantity, price):
        self.ticker = ticker
        self.quantity = quantity
        self.price = price

    def market_value(self):
        return self.quantity * self.price


aapl = Asset("AAPL", 10, 190)
msft = Asset("MSFT", 5, 410)

print(aapl.ticker, aapl.market_value())   # AAPL 1900
print(msft.ticker, msft.market_value())   # MSFT 2050
```

This small program ties the whole idea together. There is **one blueprint** (`Asset`) and **two objects** built from it, each with **independent data**, answering the same question (`market_value()`) with their own answers. Plato's Forms and the contractor's blueprint are now running as Python code. This is also why `self` has to exist: it is the mechanism that lets one shared description serve many independent objects.

For now, the class and the code that uses it live in one file. The next step is giving this code a proper home: organizing classes into files and folders that a real project can grow into, along with the tools professional Python projects use.

## Part 5: The four principles of object-oriented programming

Object-oriented programming is usually described in terms of four principles. Think of them as the vocabulary for everything you will do with classes for the rest of this course. This handout names and defines all four, then develops **inheritance** in depth. The other three will get fuller treatment in later sessions.

### Encapsulation

**Encapsulation** means bundling data together with the behavior that operates on it, in one place, and controlling what outside code is allowed to touch directly.

The analogy: picture a row in a trade blotter that also knows how to update itself. In a spreadsheet, the data lives in the row, and the logic that edits it lives in a separate macro somewhere else in the workbook, and nothing forces the two to stay in sync. With encapsulation, the row and its update logic travel together as one unit.

You have already done this. The `Asset` class bundles an asset's data (ticker, quantity, price) with the behavior that uses that data (`market_value`). That is encapsulation in action.

### Abstraction

**Abstraction** means exposing *what* something does while hiding *how* it does it, so the people using it only deal with a simple surface.

The analogy: a vendor's order form. To buy from a supplier, you fill in the fields they accept (item, quantity, delivery address). You don't need to know how their warehouse is laid out or how they pick and pack your order. The form is the simple surface, and the warehouse is the hidden detail.

You have already seen this too. Anyone who calls `aapl.market_value()` gets the market value without needing to know or care that it's calculated as quantity times price. If the calculation later became more sophisticated, code that calls `market_value()` wouldn't have to change.

### Polymorphism

**Polymorphism** (Greek for "many forms") means different kinds of objects responding to the same request, each in its own way.

The analogy: at a company, "submit an expense report" is the same request whether it comes from a contractor or a salaried employee, but what actually happens behind the scenes differs for each (different approval paths, different reimbursement rules). The person making the request uses the same words either way.

Polymorphism is the natural companion to inheritance: specialized versions of a class can each respond to the same method call in their own way. You will see it put to real use later in the course.

## Inheritance

### What it is

**Inheritance** lets one class be defined as a specialized version of another class. The new class automatically receives all the properties and behavior of the class it inherits from, and it can then add to them or change them.

Some vocabulary. The general class is called the **parent** class, and the specialized class is called the **child** class. You will also see "base class" or "superclass" for the parent and "derived class" or "subclass" for the child. They mean the same thing, and this handout sticks to parent and child.

### A business analogy: the company policy and the department addendum

Think about how a company handles its employee policies. There is a company-wide handbook covering the things that apply to everyone: working hours, the leave policy, the code of conduct. The trading desk might have a department addendum on top of that, covering what's specific to them, such as personal trading restrictions. The addendum does not repeat the whole handbook. It says "everything in the company handbook applies, plus the following."

That is inheritance:

- Everyone in the department **inherits** the general policy by default (the child gets everything the parent has).
- The addendum only states **what is different** (the child only defines what's specific to it).
- Nobody rewrites the full handbook for each department (no duplication).
- When the company-wide policy changes, **every department picks up the change automatically** (fix the parent, and every child benefits).

### The test: "is-a"

When is inheritance the right tool? Apply the **"is-a" test**: inheritance is appropriate only when the child genuinely **is a** kind of the parent.

- A `SavingsAccount` **is a** `BankAccount`. ✔
- A `Manager` **is an** `Employee`. ✔
- A `Stock` **is an** `Asset`. ✔

If the sentence "a [child] is a [parent]" is true, and it would stay true in everyone's understanding of the business, inheritance is a candidate. If the sentence sounds wrong, it isn't.

### A worked example: stocks and bonds

Stocks and bonds are both assets. Both have a ticker, a quantity held, and a price, and both have a market value. But they differ in how they produce income: a stock may pay a dividend per share, while a bond pays interest (a coupon) based on its face value.

With inheritance, the shared parts live once in `Asset`, and only the differences live in the children. In Python, you write the parent's name in parentheses after the child's name to say "this class inherits from that one":

```python
class Asset:
    def __init__(self, ticker, quantity, price):
        self.ticker = ticker
        self.quantity = quantity
        self.price = price

    def market_value(self):
        return self.quantity * self.price


class Stock(Asset):
    def __init__(self, ticker, quantity, price, dividend_per_share):
        super().__init__(ticker, quantity, price)
        self.dividend_per_share = dividend_per_share

    def annual_income(self):
        return self.quantity * self.dividend_per_share


class Bond(Asset):
    def __init__(self, ticker, quantity, price, face_value, coupon_rate):
        super().__init__(ticker, quantity, price)
        self.face_value = face_value
        self.coupon_rate = coupon_rate

    def annual_income(self):
        return self.quantity * self.face_value * self.coupon_rate
```

Walk through what is happening:

- `class Stock(Asset):` declares that `Stock` is a child of `Asset`. It automatically has everything `Asset` has, including the `market_value` method, without that method being written again.
- Inside the child's constructor, `super().__init__(ticker, quantity, price)` means "run the parent's constructor for these values." `super()` refers to the parent class. This line lets `Asset` handle the setup it already knows how to do (storing ticker, quantity, and price) so the child doesn't repeat it. The child then adds only what's new: a stock stores its dividend per share, and a bond stores its face value and coupon rate.
- Each child adds its own `annual_income` method, calculated the way that makes sense for that kind of asset.

Using them:

```python
ko = Stock("KO", 100, 60, 1.90)
tbond = Bond("UST10Y", 5, 980, 1000, 0.04)

print(ko.market_value())       # 6000   (inherited from Asset)
print(tbond.market_value())    # 4900   (inherited from Asset)
print(ko.annual_income())      # 190.0  (defined in Stock)
print(tbond.annual_income())   # 200.0  (defined in Bond)
```

Neither `Stock` nor `Bond` defines `market_value`, yet both can use it, because they inherited it. If you later decide market value should be calculated differently, you change it in one place, `Asset`, and both children pick up the change.

A child can also **replace** a behavior it inherited by defining a method with the same name. Its own version then takes priority over the parent's. That ability ties directly to polymorphism: the same method call can do different things depending on what kind of object receives it.

### The caution: inheritance is often overused

Inheritance is one of the most frequently misused ideas in object-oriented programming. It is only correct when the "is-a" test genuinely holds.

Many relationships between classes are not "is-a" but **"has-a."** Take a transaction in the portfolio tracker. A transaction **has an** asset (the thing being bought or sold). A transaction is not a kind of asset. It would be wrong to make `Transaction` a child of `Asset` just because a transaction involves an asset and it would be convenient to reach its ticker.

For "has-a" relationships, the right tool is **composition**: one object holds a reference to another object as one of its properties. In a sketch:

```python
class Transaction:
    def __init__(self, asset, side, quantity, price_per_share, date):
        self.asset = asset          # a Transaction HAS an Asset
        self.side = side            # "buy" or "sell"
        self.quantity = quantity
        self.price_per_share = price_per_share
        self.date = date
```

Here, `asset` is an `Asset` object stored as one of the transaction's properties. The transaction can reach the asset's ticker with `self.asset.ticker`, without pretending to be an asset itself.

Why does getting this wrong matter? A class structure is a statement about how the business works. If the structure claims something false, such as "a transaction is an asset," every future change has to work around that false claim. The parent's properties and methods get forced onto a child where they make no sense, and changes to the parent start breaking children that should never have been connected to it. Class structures built on false "is-a" relationships become painful to change, and they rarely get cheaper to fix over time.

### The takeaway

**Inheritance is a modeling decision before it is a syntax feature.** Before reaching for it, ask: "Is this genuinely a kind of that?" If yes, inheritance removes duplication and keeps the code aligned with the business. If the honest answer is "no, it *has* one of those," use composition instead.

## Summary

- **Paradigms are lenses.** Imperative code says *how* (a recipe), declarative code says *what* (ordering at a restaurant), and object-oriented code organizes a program around objects. Real programs mix them, and this course uses OOP as its main lens.
- **OOP models the real world.** A program is made up of objects that interact, just like the world.
- **A class is a description, and an object is an instance of it.** Plato's Form of a chair versus a particular chair, or a blueprint versus a house. You can't live in a blueprint.
- **A class defines properties and behavior**: what an object knows about itself, and what it can do.
- **`__init__` is the constructor.** It runs automatically when an object is created and makes sure every object starts out with the data it needs.
- **`self` means "whichever object is currently being worked on."** It is what lets one class produce many independent objects.
- **Create an object** by calling the class like a function (`Asset("AAPL", 10, 190)`), **read a property** with a dot (`aapl.ticker`), and **call a method** with a dot and parentheses (`aapl.market_value()`).
- **The four principles** are encapsulation (bundle data with its behavior), abstraction (show what, hide how), inheritance (a specialized version of a class), and polymorphism (same request, different responses).
- **Inheritance** is for genuine "is-a" relationships. It removes duplication and makes change safe. For "has-a" relationships, use composition instead.
