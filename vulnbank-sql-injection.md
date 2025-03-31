Okay, here is the write-up formatted in Markdown:

# Vuln-Bank SQL Injection Exploit Write-up

## Objective
To identify and exploit a SQL injection vulnerability in the login functionality of the "Vulnerable Bank" web application.

## Lab Environment

*   **Application:** Vulnerable Bank (running locally on `http://localhost:5000`)
*   **Attacker Machine:** Kali Linux (or similar Linux distribution)
*   **Tools Used:**
    *   Web Browser (**Chromium/Chrome** shown)
    *   **Burp Suite** Community Edition
    *   **sqlmap**
    *   Terminal
    *   Text Editor (**nano** shown)

## Step-by-Step Exploitation Process

### 1. Login and Reconnaissance

1.  Navigate to the login page of the Vulnerable Bank application (`http://localhost:5000/login`).
2.  Configure your browser to proxy traffic through **Burp Suite**.
3.  Log in with known test credentials (e.g., username: `test1`, password: `test`).
4.  Observe the successful login and redirection to the dashboard (`/dashboard`).
5.  Go to **Burp Suite** -> Proxy -> HTTP History. Locate the `POST /login` request.

### 2. Identify Potential Injection Point

1.  Examine the `POST /login` request in **Burp Suite**. Notice the request body is in JSON format:
    ```json
    {"username":"test1","password":"test"}
    ```
2.  Send this request to Burp Repeater (Right-click -> Send to Repeater).
3.  In Repeater, send the original request to confirm a successful (200 OK) response with login success details and a token.

### 3. Test for SQL Injection

1.  Modify the `username` value in the JSON payload by appending a single quote (`'`) to break potential SQL syntax:
    ```json
    {"username":"test1'","password":"test"}
    ```
2.  Send the modified request.
3.  Observe the response: An `HTTP/1.0 500 INTERNAL SERVER ERROR` is returned.
4.  Critically, examine the response body. It reveals a database error message:
    ```json
    {
      "error": "syntax error at or near \"test1'\"\\nLINE 1: ...ECT * FROM users WHERE username= 'test1'' AND password='test'\\n...",
      "status": "error"
    }
    ```
5.  **Conclusion:** This error message confirms a SQL injection vulnerability. The application is directly concatenating the `username` input into the SQL query without proper sanitization or parameterization. The backend database appears to be **PostgreSQL** based on the error syntax and later **sqlmap** findings.

### 4. Prepare for Automated Exploitation (sqlmap)

1.  In Burp Repeater, right-click the modified request (the one that caused the error) and choose "Copy to file". Save it as `vulnbank.txt` (or any preferred name) in your working directory. This saves the full HTTP request including headers and the vulnerable JSON body.
2.  Alternatively, manually copy the entire request text (headers + body) and paste it into a file using `nano vulnbank.txt`.

### 5. Run sqlmap

1.  Open a terminal in the directory where you saved `vulnbank.txt`.
2.  Run **sqlmap** to automate the exploitation. Start by identifying the vulnerable parameter:
    ```bash
    sqlmap -r vulnbank.txt -p username --batch
    ```
    *   `-r vulnbank.txt`: Reads the raw HTTP request from the file.
    *   `-p username`: Specifies that the `username` parameter (within the JSON body) should be tested. **sqlmap** is usually smart enough to find parameters, but explicitly stating it for JSON can be more reliable.
    *   `--batch`: Runs **sqlmap** with default options, avoiding interactive prompts.
3.  **Initial Observation:** The first run might report the parameter is not injectable or encounter issues due to HTTP status codes like 401 (Unauthorized) if injection attempts invalidate the session or credentials used in the base request. The video shows this happening.
4.  **Refined sqlmap command:** To handle potential non-fatal error codes like 401, use `--ignore-code`:
    ```bash
    sqlmap -r vulnbank.txt -p username --ignore-code=401 --batch
    ```
    *   `--ignore-code=401`: Tells **sqlmap** to not consider HTTP 401 as a definitive "not injectable" response.
5.  **Second Observation:** This run successfully identifies the backend DBMS as **PostgreSQL** and confirms that the `username` parameter is injectable using various techniques (Boolean-based blind, Error-based, UNION query, etc.).

### 6. Dump Database Contents

1.  To extract data from the database, add the `--dump` flag to the **sqlmap** command:
    ```bash
    sqlmap -r vulnbank.txt -p username --ignore-code=401 --batch --dump
    ```
2.  **sqlmap** will now:
    *   Confirm the vulnerability.
    *   Enumerate databases.
    *   Enumerate tables within the current database (`public`).
    *   Dump the contents of all found tables (`users`, `transactions`, `virtual_cards`, `billers`, `bill_payments`, etc.).
    *   Optionally attempt to crack any found password hashes (using default dictionaries if `--batch` is enabled).
3.  Observe the dumped data outputted to the console and saved in CSV files within the **sqlmap** output directory (usually `~/.local/share/sqlmap/output/` or similar).

## Vulnerability Explanation

The application's login function takes user-provided `username` and `password` via a JSON payload. The backend code directly incorporates the `username` value into a SQL query string without using prepared statements or properly sanitizing the input. When a single quote (`'`) is injected, it prematurely terminates the string literal in the SQL query, causing a syntax error and revealing the vulnerability (Error-Based SQL Injection). **sqlmap** further leverages this to execute various payloads and extract database information.
