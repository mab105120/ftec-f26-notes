# Syllabus — Special Topics in Finance Technology (FTEC-6V97)

## Course Details

- **Instructor:** Dr. Mohamad Bourji
- **Course title:** FTEC-6V97 Special Topics in Finance Technology
- **Meeting time:** Every Wednesday at 7:00 PM CST
- **Classroom:** JSOM 1.517
- **Office hours:** No predetermined office hours. All office hours are on demand.

## Course description

This course introduces graduate students in the Finance Technology and Analytics program to the principles and practice of software development. Students entering the course are assumed to have prior experience with Python in a scripting or data-analysis context, but no formal background in software engineering.

Rather than training students as professional software developers, the course develops a deeper understanding of how well-engineered software systems are designed and built — including application structure, data persistence, services, and user interfaces. Students will progress from writing individual scripts to building a complete, well-tested application with a layered architecture, following practices used in professional software development. This foundation prepares students to critically evaluate and direct the development of software in future coursework, including work involving AI-assisted development tools.

## Learning objectives

- Understand the software development lifecycle and how professional teams structure work
- Use Git/GitHub confidently, including a PR-based submission/review workflow
- Write modular, object-oriented Python with proper exception handling, logging, and separation of concerns (e.g., data modules vs. service modules)
- Model and connect to relational databases from application code (e.g., SQLAlchemy) — not SQL itself, which students already know
- Write automated tests for Python applications to verify correctness and support safe changes — a practice students will lean on directly when evaluating and iterating on AI-agent-generated code
- Understand how to expose local functionality as a web service/API
- Build a simple UI (e.g., React) on top of an application's services
- Develop the engineering judgment needed to evaluate whether software — including software written by an AI agent — is well-structured, correct, and maintainable

## Topics covered (full term outline — actual pace may vary)

1. Intro to software development & the software development lifecycle (SDLC)
2. Version control with Git/GitHub — branching, PRs, code review (all assignments submitted as PRs)
3. Python as a programming language, part 1 — modularity & imports, code organization
4. Python as a programming language, part 2 — object-oriented programming
5. Python as a programming language, part 3 — exception handling & logging
6. Python as a programming language, part 4 — separation of duties (e.g., data modules vs. service modules)
7. Relational databases for applications — connecting to a database, modeling data with SQLAlchemy (not SQL itself)
8. Automated testing — writing automated tests for a Python application (e.g., the data layer built in the databases module)
9. Web services — exposing local functionality over the internet (APIs)
10. UI technologies — building a simple UI (e.g., React) on top of an application's services, and connecting it to the API

Both web services and UI technologies are covered over three sessions each.

## Course project

All three assignments are milestones toward one semester-long build: a personal investment **portfolio tracker** (record buy/sell transactions, compute holdings and gain/loss). The project is intentionally scoped to stay approachable — no live market data, no authentication, single-page UI only. Full project details will be provided separately as each assignment is assigned.

## Weekly breakdown

| Week | Date | Topic | What's covered |
| --- | --- | --- | --- |
| 1 | 08/26 | SDLC intro | What "developing software" actually involves beyond writing code: requirements, design, implementation, testing, deployment, maintenance. Introduce the course project. |
| 2 | 09/02 | Git & GitHub | Repos, commits, branches; why version control exists (an audit trail + safe experimentation, not just backup); pull requests and code review as the assignment-submission workflow |
| 3 | 09/09 | Python as a programming language: modularity | Packages/modules/imports; organizing code into files instead of one script; drafting the project's domain classes as a package |
| 4 | 09/16 | Python as a programming language: OOP | Classes, encapsulation, composition vs. inheritance; modeling `Asset`, `Transaction`, `Position` |
| 5 | 09/23 | Python as a programming language: exceptions & logging | Custom exceptions vs. letting errors propagate; structured logging; the project's "sell more than you own" error as the running example |
| 6 | 09/30 | Python as a programming language: separation of duties | Service layer vs. data-access layer; why the boundary matters (testability, swappable storage); designing the DAO interface |
| 7 | 10/07 | **Midterm exam** | Covers modules 1–6 |
| 8 | 10/14 | Databases (SQLAlchemy) | Relational storage for applications; connecting to a database; SQLAlchemy models and relationships; swapping the in-memory DAO for a SQLAlchemy-backed one behind the same interface |
| 9 | 10/21 | Automated testing | Why manual testing doesn't scale; unit tests for the service layer; fixtures/test doubles for the repository layer |
| 10 | 10/28 | Web services, part 1 | What an API is; HTTP basics; wrapping the service layer in REST endpoints |
| 11 | 11/04 | Web services, part 2 | Request/response design, validation, error handling over HTTP |
| 12 | 11/11 | Web services, part 3 | Finishing the API surface |
| 13 | 11/18 | UI technologies (React), part 1 | Fundamentals: components, rendering, state — from scratch, assuming no prior UI experience |
| 14 | 11/25 | **No class — Thanksgiving week** | — |
| 15 | 12/02 | UI technologies (React), part 2 | Building the holdings table and summary card from live data; building the add-transaction form and handling user input |
| 16 | 12/09 | UI technologies (React), part 3 | Calling the API from the UI; handling errors/loading states |
| 17 | 12/16 | **Final exam / project presentations** | — |

## Grading breakdown

- Assignment 1: 20%
- Assignment 2: 20%
- Assignment 3: 20%
- Midterm exam: 20%
- Final exam: 20%

## Policies

### Late work

No grace period. Penalty applies from the deadline:

- **20% off** the grade if submitted within 2 weeks of the deadline
- **30% off** the grade if submitted more than 2 weeks after the deadline

### Academic integrity

Assignments are submitted as GitHub pull requests. Students may discuss concepts and approach verbally, but must not:

- Copy code from another student
- View another student's repository or pull request
- Share solutions or working code with classmates

Submissions that appear to share code with another student's submission will be flagged for manual review.

### Accommodations

Students who need accommodations should contact the university's Office of Disability Services. Once documentation is provided, accommodations will be made accordingly.
