# Setting Up MySQL on Your Computer

Your next assignment stores the portfolio tracker's data in a MySQL database running on your own computer. This guide gets you from nothing installed to a working database you can log in to. Allow about 30 minutes. Do it before you start the assignment, not the night it's due.

By the end you will have:

1. The **MySQL server** installed and running.
2. The **`mysql` command** working in your terminal.
3. A database called **`portfolio_tracker`** and a dedicated user called **`tracker`** that your application will log in as.

Connecting to the database *from Python* was covered in class and isn't part of this guide.

## What you are installing

MySQL comes in two parts. The **server** is a program that runs in the background, owns the data, and answers requests. A **client** is any program that connects to the server to send SQL and get results back. The `mysql` command is a client; later, your Python application will be another one. Installing MySQL gives you both.

**Version:** install **MySQL Community Server 9.7 (LTS)**. "LTS" means long-term support, a stable release line. The download page also offers newer "Innovation" releases (numbered like 26.x); don't pick those for this course. If you already have MySQL 8.4 or newer installed, you can use it and skip to [Step 3](#step-3-create-the-database-and-the-application-user).

**Choose the root password before you start, and write it down.** During installation you set a password for `root`, MySQL's administrator account. It is the password for the whole MySQL server (not for one database), and you need it in Step 3. Forgetting it is the most common setup problem; see Troubleshooting if it happens.

---

## Step 1 (Mac): install MySQL

### 1. Download the right package

1. Find out which chip your Mac has: Apple menu → **About This Mac**. "Chip: Apple M…" means **ARM**. "Processor: Intel…" means **x86**.
2. Go to **https://dev.mysql.com/downloads/mysql/**.
3. Under **Select Version**, choose the **9.7.x LTS** release. Under **Select Operating System**, choose **macOS**.
4. Download the **DMG Archive** that matches your chip: **ARM, 64-bit** or **x86, 64-bit**.
5. The site may ask you to log in or sign up. Click **"No thanks, just start my download."** You don't need an account.

### 2. Run the installer

1. Open the downloaded `.dmg`, then double-click the `.pkg` inside it.
2. Click through **Introduction** and **License** (Continue → Agree).
3. On **Installation Type**, click **Install** (keep the defaults). Enter your Mac password if asked.
4. When asked for the **root password**, enter the password you chose and wrote down. Leave the option to **start MySQL Server** after configuration turned on.
5. Click **Finish** / **Close**.

MySQL is installed in `/usr/local/mysql`.

### 3. Check that the server is running

Open **System Settings** and type **MySQL** in the search box (the MySQL settings pane is usually near the bottom of the sidebar). The status should say the server is **running** (green). If it says **stopped**, click **Start MySQL Server**. Leave **"Start MySQL when your computer starts up"** checked so you don't have to start it every time.

### 4. Make the `mysql` command available

The installer doesn't add MySQL's programs to your terminal's search path, so typing `mysql` gives `command not found` until you do this once. Open **Terminal** and run:

```bash
echo 'export PATH="/usr/local/mysql/bin:$PATH"' >> ~/.zshrc
```

Then **close Terminal and open a new window**, and check:

```bash
mysql --version
```

It should print a version number starting with `9.7`. Continue to [Step 2](#step-2-log-in-as-root).

---

## Step 1 (Windows): install MySQL

### 1. Download the installer

1. Go to **https://dev.mysql.com/downloads/mysql/**.
2. Under **Select Version**, choose the **9.7.x LTS** release. Under **Select Operating System**, choose **Microsoft Windows**.
3. Download the **MSI Installer** (Windows x86, 64-bit). Don't download the ZIP Archive.
4. If the site asks you to log in, click **"No thanks, just start my download."**

### 2. Install the server

1. Run the downloaded `.msi` file. Accept the license and choose the **Typical** setup. Approve the Windows administrator prompt.
2. At the end, the installer offers to run **MySQL Configurator**. Choose to **run it now**. The server won't start until it has been configured.

### 3. Configure it with MySQL Configurator

Go through the screens in order. Keep the defaults except where noted.

1. **Type and Networking:** set **Config Type** to **Development** (the right choice for a laptop). Keep **TCP/IP** checked, **Port 3306**, and **Open Windows Firewall port** checked.
2. **Accounts and Roles:** enter the **root password** you chose and wrote down, twice. Don't add other users here; you'll do that in Step 3.
3. **Windows Service:** keep **Configure MySQL Server as a Windows Service** and **Start the MySQL Server at System Startup** checked, and **Standard System Account**. Note the service name it shows (something like `MySQL97`).
4. **Server File Permissions:** keep the default.
5. **Sample Databases:** not needed; leave unchecked.
6. **Apply Configuration:** click **Execute** and wait until every step has a green check, then **Finish**.

MySQL is installed in `C:\Program Files\MySQL\MySQL Server 9.7\`.

### 4. Make the `mysql` command available

Windows doesn't add MySQL's programs to the command search path, so do it once:

1. Press the Windows key, type **environment variables**, and open **"Edit the system environment variables."**
2. Click **Environment Variables…**.
3. Under **System variables**, select **Path** and click **Edit…**.
4. Click **New** and enter: `C:\Program Files\MySQL\MySQL Server 9.7\bin`
5. Click **OK** on every window to close them. Don't delete or change any of the existing entries.
6. **Open a new** Command Prompt or PowerShell window (windows that were already open won't see the change), and check:

```
mysql --version
```

It should print a version number starting with `9.7`.

---

## Step 2: log in as root

This step is the same on Mac and Windows. In a terminal (Terminal on Mac, Command Prompt or PowerShell on Windows), run:

```
mysql -u root -p
```

Enter the root password when asked. Nothing appears on screen as you type; that's normal. You should see a `mysql>` prompt. Check the connection:

```sql
SELECT VERSION();
```

You're now connected to the server as its administrator. Stay logged in for Step 3.

## Step 3: create the database and the application user

Your application should not log in as `root`. Root can do anything to every database on the server, and the application only needs its own database. Give it its own user with access to exactly that. It's the same principle as giving a new employee access to the systems their job needs, not the administrator password.

**First pick a password for the `tracker` user** (different from root's), and write it down: your application will need it. Then, at the `mysql>` prompt, run these three commands, **replacing `choose-a-password` with that password** (keep the single quotes around it):

```sql
CREATE DATABASE portfolio_tracker;
CREATE USER 'tracker'@'localhost' IDENTIFIED BY 'choose-a-password';
GRANT ALL PRIVILEGES ON portfolio_tracker.* TO 'tracker'@'localhost';
```

What each line does:

- `CREATE DATABASE` creates an empty database (a container for tables) named `portfolio_tracker`.
- `CREATE USER` creates a login named `tracker` that can only connect from this computer (`localhost`), with the password you chose.
- `GRANT` gives `tracker` full control of the `portfolio_tracker` database, and nothing else on the server.

Each command should answer `Query OK`. Type `exit` to log out of root.

## Step 4: check that the application user works

Log in as the new user, directly into its database:

```
mysql -u tracker -p portfolio_tracker
```

Enter the **tracker** password. At the `mysql>` prompt:

```sql
SELECT DATABASE(), CURRENT_USER();
SHOW TABLES;
```

You should see `portfolio_tracker` and `tracker@localhost`, and `SHOW TABLES` should report an empty set: the database exists but has no tables yet. Your application creates them. Type `exit`.

**You're done.** The values your application will need are: host `localhost`, port `3306`, user `tracker`, the tracker password, and database `portfolio_tracker`. Never put the root password in your project.

## Optional: a graphical client

If you'd like to browse tables in a window instead of the terminal, **MySQL Workbench** is a free graphical client for Mac and Windows, from the same download site (choose **MySQL Workbench**). Create a connection with host `localhost`, port `3306`, and user `tracker`. It's optional: everything in the course works with the `mysql` command.

---

## Troubleshooting

**`mysql: command not found` (Mac) or `'mysql' is not recognized...` (Windows)**
The `mysql` folder isn't on your search path. Redo the "Make the `mysql` command available" step, and make sure you **opened a new terminal window** afterwards.

**`Can't connect to local MySQL server through socket '/tmp/mysql.sock'` (Mac) or `Can't connect to MySQL server on 'localhost'` (Windows)**
The server isn't running. This error appears *after* the password prompt, because `mysql` asks for the password before it tries to connect, so it does **not** mean your password is wrong.
- Mac: System Settings → MySQL → **Start MySQL Server**.
- Windows: press the Windows key, open **Services**, find the service whose name starts with **MySQL**, and click **Start**. Set its startup type to **Automatic** so it starts with Windows.

**`Access denied for user 'root'@'localhost' (using password: YES)`**
The server is running, but the password is wrong. Check Caps Lock and try again. If you've really forgotten the root password, see below.

**`Access denied for user 'tracker'@'localhost'`**
The tracker password you typed doesn't match the one in `CREATE USER`. A common cause is running the command with `choose-a-password` still in it, in which case that *is* the password. To set the password you meant, log in as root and run (with your password):

```sql
ALTER USER 'tracker'@'localhost' IDENTIFIED BY 'your-real-password';
```

**`ERROR 1007: Can't create database 'portfolio_tracker'; database exists` or `ERROR 1396: Operation CREATE USER failed`**
You already ran that command once; the database or user exists. Skip that line. If you're unsure what the tracker password is, use the `ALTER USER` command above.

**Port 3306 is already in use / the server won't start**
Another MySQL (from Homebrew, XAMPP, MAMP, or an older install) is probably running on the same port. Stop or uninstall the other one, then start this one again. If you already have a working MySQL 8.4 or newer, you can use it instead.

**Forgotten root password**
- **If nothing on this MySQL server matters to you** (for example, you just installed it), the quickest fix is a fresh start. On Mac: System Settings → MySQL → **Initialize Database**, which **erases all databases on this server** and lets you set a new root password. On Windows: uninstall **MySQL Server** (Settings → Apps), delete the folder `C:\ProgramData\MySQL` (it holds the old data and password; `ProgramData` is a hidden folder, so type the path into File Explorer's address bar), then reinstall from Step 1. Either way, **all existing data is lost**.
- **If you need to keep existing data**, follow MySQL's official reset procedure: search for "MySQL reset root password" on dev.mysql.com (the page is titled "How to Reset the Root Password").

If you're still stuck, note the exact error message and the step you were on, and ask in class or office hours. Bring a screenshot.
