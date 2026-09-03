# The Software Development Lifecycle

## Why this topic comes first

Almost everything you've done in Python so far probably followed the same pattern: you had a question, opened a notebook or a `.py` file, and wrote code until you got an answer. If it worked, you moved on. There was no real gap between "figuring out what you wanted" and "writing the code that does it" — both happened in the same few minutes, usually in your head.

That works fine for a script that computes a return series or cleans a dataset. It breaks down once software has to do more than one thing, be used by more than one person, or still be correct six months from now. At that point, "figure it out while you type" stops being a convenience and becomes a liability: decisions that used to be reversible in the next line of code are now buried three files deep, used by code you forgot existed, and wrong in a way nobody notices until it's a live problem.

The **software development lifecycle (SDLC)** names the distinct activities real software passes through on its way from an idea to something reliable that other people depend on. It's not a checklist to memorize or a project-management ritual. It's a description of work that happens whether or not anyone names it — the only real choice is whether you do each piece *deliberately*, or let it happen accidentally, badly, and usually too late.

This matters here for two reasons. First, everything you build this semester — a personal portfolio tracker that records buy/sell transactions and computes holdings and gain/loss — will pass through every one of these phases, and the syllabus follows the SDLC on purpose, so each module gives you the tool the next phase needs. Second, and more importantly for your career: an AI coding agent is very good at one of these phases and cannot do the other five for you. Knowing which phase is which, and what "done well" looks like in each, is what lets you direct an agent instead of just hoping it got things right.

## The six phases

Different sources slice the SDLC differently — some list five phases, some seven. This course uses six, because each maps to a distinct kind of mistake you make if you skip it.

### 1. Requirements — deciding what the system must do, before deciding how

Every piece of software exists to satisfy some set of expectations. **Requirements** is the phase where you write those expectations down, explicitly, before writing code — not out of distrust, but because expectations that live only in your head tend to quietly shift as you build, and you won't notice until the software does the wrong thing in a way that felt reasonable at the time.

A requirement for the portfolio tracker: *"If a user tries to sell more shares than they hold, the system must reject the transaction with a clear error — never silently succeed, never allow a negative holding."* That's not code and it's not an implementation detail. It's a decision about correct behavior, in plain language, that the finished system can be held accountable to.

Why does this need to be its own deliberate step rather than something you decide "in the moment" while writing the function? Because in the moment, you're focused on making the function work — not on asking "what should this system guarantee?" as its own question. Requirements is where correctness-thinking happens before typing-thinking.

Beginners tend to skip this phase because it doesn't feel like "real work" — no code comes out of it. That instinct is backwards: a missing requirement is invisible until real usage exposes it, and by then it's a bug report instead of a five-minute design conversation.

### 2. Design — deciding how the pieces fit together, before writing them

Once you know *what* the system must do, **design** decides *how* it will be structured to do it — the major pieces, what each is responsible for, how they talk to each other. Still not code; it's the blueprint implementation will follow.

The clearest example here: splitting the portfolio tracker into a **service layer** (business logic — "record this transaction," "compute current holdings") and a separate **data-access layer (DAO)** responsible only for storing and retrieving data. Nothing about "correctly compute gain and loss" *requires* that split — a single function could do it all. You keep them separate because you already know two things are coming: you'll swap in-memory storage for a real database later, and you'll need to test the business logic without a database running. Making that split now is what makes both of those cheap instead of expensive.

That's the general shape of design: a small, deliberate cost now (thinking about structure instead of just writing) in exchange for a much larger cost avoided later. Bad design rarely fails immediately — it fails months later, when a change that should touch one file ends up touching twelve.

### 3. Implementation — writing the code

This is what most people mean by "developing software," and the phase you already have real experience with. Implementation turns requirements and design into working code.

It's worth being precise about what it isn't: not the phase where you figure out what the system should do (requirements already decided that), and not where you figure out how the pieces fit together (design already decided that). Implementation executes those decisions. That doesn't make it mechanical — writing correct, readable, well-organized code is a real skill, and it's most of what the Python modules ahead teach. But implementation goes faster and produces better results when it isn't also carrying undecided requirements or undesigned structure. If you find yourself making a real "what should this do" decision *while* coding, that's a sign a step got skipped earlier — not that implementation is where that decision belongs.

### 4. Testing — verifying the system does what requirements said it should

Once code exists, how do you know it's correct — not just "ran without crashing," but actually does what requirements demanded, including the cases that are easy to forget? **Testing** is the deliberate phase where you check.

This is the other half of requirements: a requirement is only useful if something eventually checks it's satisfied. The over-sell requirement above is worthless as a sentence if nothing verifies the code enforces it. You'll write **automated tests** — code that checks other code — so that verification doesn't depend on a human remembering to try the over-sell case by hand every time the software changes. "Attempting to over-sell raises the expected error" is a requirement, expressed as code, running automatically forever.

Testing has to be its own pass rather than something folded into implementation, because the two put you in different mental modes: implementing, you're thinking about the case where it works; testing asks "what would make this wrong, and did I check for it?" That's most effective when done by re-reading the original requirement — not by re-reading the code you just wrote, which tends to confirm what you already believe it does.

### 5. Deployment — making the system usable by someone other than its author

Correct software that only runs on your laptop, in a state only you know how to reproduce, hasn't delivered value to anyone else yet. **Deployment** is the phase that takes a system from "works when I run it" to "works when someone else runs it."

For most professional software this means real infrastructure — servers, production databases, monitoring. This course's project stays local all semester, so you won't deploy at that scale, but the same question still applies every time you hand off code: *could a classmate get it running from your README alone, or does it only work because of a setup step that exists only in your head?* That gap — "works for me" vs. "works for anyone who wasn't there while I built it" — is what deployment as a discipline closes, just at a larger scale than one laptop.

### 6. Maintenance — the system's life after "done"

Software is rarely finished the way a bridge is finished. Once a system is in use, requirements change, bugs surface, and the code has to keep evolving without breaking what already works. **Maintenance** is that ongoing phase, and in real systems it's usually the *most expensive* one — most of a system's total cost is typically spent maintaining it, not building it.

This is where design decisions stop being abstract and start paying rent. If storage logic is cleanly separated from business logic, a change like "switch databases" touches one layer and leaves business rules alone — cheap. If those concerns were tangled from the start, the same change means carefully untangling code that was never meant to be mixed — expensive, risky, and exactly the kind of work that makes people afraid to touch old code. Good design is, among other things, a bet that maintenance will be cheaper later because of care taken now.

## How the phases relate to each other

Laid out this way, it's tempting to read the SDLC as a strict one-way sequence: finish requirements, then design, then implementation, never looking back. That's a real way of organizing work — historically called **waterfall** — simple to reason about and easy to plan around, which is part of why this course walks the phases roughly in order.

It's not the only way real teams work, though. Most modern teams work **iteratively** ("agile"): instead of finishing requirements for an entire system before any design begins, they cycle through small versions of all six phases repeatedly, for one small piece of functionality at a time, adjusting as they learn. Iteration is common because requirements are hard to get exactly right up front — building even a rough version often reveals requirements nobody thought to write down. Waterfall assumes you can nail requirements once, correctly, before implementation starts; iterative approaches assume you can't.

This course runs the phases largely once, in order — not because that's the "correct" way to build software, but because it's the clearest way to teach six distinct kinds of work to people who haven't done any of them. Once you know what each phase is for, you're equipped to see why a real team might instead cycle through miniature versions of all six every two weeks. Don't read the shape of this syllabus as an endorsement of waterfall over agile — it's a teaching simplification.

## Why this framework matters beyond this course

Here's the throughline the rest of this course depends on. An AI coding agent today is extremely good at one of these six phases: implementation. Given a clear, well-specified task, it can produce working code quickly, often faster than you'd type it. What it can't do is decide what the system *should* do (requirements), decide how the pieces *should* fit together (design), or tell you with confidence that what it produced actually satisfies either (testing). Those remain judgment calls that depend on understanding the problem, not just the syntax.

That's why this course isn't primarily about learning to write code by hand, and why it's built around one real application instead of disconnected exercises. The SDLC is the vocabulary for the parts of software development that stay your job even when an agent is doing your typing: knowing what a good requirement looks like, recognizing good and bad design, and telling whether a test actually proves what it claims. Every module ahead is, in one way or another, teaching you to do one of these six things well — so that later, directing an agent instead of writing every line yourself, you have the judgment to know whether what comes back is actually good.

## Check your understanding

Use these to test yourself, not as a substitute for the explanations above — if a question is hard to answer without re-reading a section, that's the section to reread.

- Why isn't "requirements" the same as "implementation, but thinking about it first"? What's different about doing it as its own explicit step?
- Give a design decision from the portfolio tracker (other than the service/DAO split) and explain what maintenance problem it prevents.
- Why does a requirement need a corresponding test to be useful in practice?
- What's the difference between "works when I run it" and "deployed," even for software that never leaves your laptop?
- In one or two sentences: why doesn't an AI agent being good at implementation make the other five phases unnecessary?
