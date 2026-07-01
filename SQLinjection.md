# SQL Injection (SQLi) Notes
Source: [PortSwigger Web Security Academy](https://portswigger.net/web-security)

### 1. Definition
SQL Injection is not a type of "malware injection." It is a logic flaw where user input is treated as code rather than text.
By manipulating input, we can hijack the "conversation" between the website and the database.

---

### 2. "Sentence-Editor" Analogy
The database processes commands as sentences. We can edit these sentences to change their meaning.

* **The Single Quote (`'`):** Acts as a "gatekeeper" or "terminator" that breaks out of the expected text field.
* **The Logic (`OR 1=1`):** A tautology that forces the database to evaluate the query as "True," often returning all data.
* **The Eraser (`--`):** Comments out the rest of the original query to prevent syntax errors.

**Example:**
* **Original Code:** `"SELECT * FROM items WHERE name = '" + USER_INPUT + "';"`
* **My Payload:** `Laptop' OR 1=1 --`
* **Resulting Code:** `...WHERE name = 'Laptop' OR 1=1 --'`
* **The Purpose:** The `--` tells the database to ignore the leftover quote. The database then checks:
                  *"Is the name 'Laptop' OR is 1=1?"* Since `1=1` is always true, it returns every item in the database.

---

### 3. Blind SQLi Techniques

#### The Boolean Test ("Twenty Questions")
* **Payloads:** `id=1 AND 1=1` (True) vs `id=1 AND 1=2` (False).
* **The Logic:** If I can flip the website between True/False states, I can extract data character-by-character:
  * `...AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='admin')='A'`
  * If the page loads normally, I've confirmed the first letter is 'A'.

#### Time Delay ("Patience")
* **Payload:** `id=1; IF (1=1) WAITFOR DELAY '0:0:10'--`
* **The Logic:** If the page takes 10 seconds to load, the database processed a "True" condition. If it loads instantly, the condition was "False."

#### OAST ("Hidden Spy")
* **The Problem:** The website is "blind", it shows no data and has no delays.
* **The Solution:** Force the database to "ping" an external server I control (e.g., `http://my-secret-server.com`).
* **The Proof:** If my server receives a request from the target's IP, I have confirmed the code executed.

---

### 4. Common Attack Vectors
SQL Injection isn't limited to search boxes. It can occur anywhere a website processes user input. I categorize these by the database command being hijacked:
* **SELECT (The Reader):** Where: Search bars, filters, product categories, or sort-by buttons.
    * **Impact:** Extracting sensitive data (credentials, PII) or dumping entire tables.
* **INSERT (The Creator):** Where: Registration forms, "Contact Us" forms, or blog comment sections.
    * **Impact:** Adding unauthorized data (like creating an admin account via a sign-up form).
* **UPDATE (The Editor):** Where: "Edit Profile" pages, password change forms, or shopping cart quantity fields.
    * **Impact:** Modifying existing data (like changing another user’s email or elevating my own account privileges).
* **ORDER BY (The Sorter):** Where: Any feature that allows sorting results (e.g., "Sort by Price").
    * **Impact:** Probing the database structure by injecting logic that changes the sort order based on hidden data.

      ---

### Lab 1: SQL injection in WHERE clause (Hidden Data)
* **Goal:** Display unreleased products.
* **Vulnerability Location:** Category filter (`/filter?category=Gifts`). = SELECT * FROM products WHERE category = 'Gifts' AND released = 1
* **Payload:** `' OR 1=1 --`
* **Why it worked:** * The `'` broke out of the category string.
  * The `OR 1=1` made the database condition always `True`.
  * The `--` commented out the `AND released = 1` constraint.
    
---


### Lab 2: SQL injection UNION attack (Determining Columns)
* **Goal:** Determine the number of columns returned by the query.
* **Technique Used:**
    * **`ORDER BY`**: Used as a "measuring tape" to find the number of columns. An error indicates the limit was exceeded.
    * **`UNION SELECT NULL`**: Used as a "bridge" to retrieve data.
* **Key Insights:**
    * **Pulling, not Sending:** `UNION` is an "In-Band" retrieval technique. Im not writing to the database, im forcing it to pull hidden data and display it in                                my current browser window.
    * **The "NULL" Strategy:** `NULL` is a universal placeholder. It's compatible with almost every data type, making it the perfect tool to map out the number of                             columns without triggering a "data type mismatch" error.
    * **Data Pipe:** By testing with text values (e.g., `'TEST2'`), I successfully identified which column is mapped to the webpage's display, creating a "Data                      Pipe" for future data extraction.

---
   
