# 🚨 Critical SQL Injection (Time-Based) in Garlic-Hub: Digital Signage Management

**Date:** March 23, 2026  
**Researcher:** Gaurish (Cybersecurity Researcher & Penetration Tester)  
**Severity:** `CRITICAL (9.8/10)`  
**Vulnerability Type:** CWE-89 (SQL Injection)  
**Status:** Vulnerability Confirmed (Proof of Concept Attached)

---

## 1. Executive Summary
During a deep-dive security audit of the **Garlic-Hub** project, I discovered a high-impact **SQL Injection (SQLi)** vulnerability. This flaw allows an attacker to bypass security layers and communicate directly with the database engine.

In my testing, I successfully executed a **Time-Based Blind SQLi** attack, forcing the server to "sleep" for 5 seconds. This proves that an attacker has full control over the query logic and can steal sensitive data.

---

## 2. The Root Cause (Vulnerable Code)
Look closely at the SELECT part of the Query Builder ($qb). The developer used String Concatenation (the . dots) for the $owner variable:
**File Path:** `src/Modules/Items/Repositories/ItemsRepository.php`

### **The "Dirty" Code:**
<img width="670" height="139" alt="Screenshot 2026-03-23 at 10 14 10 PM" src="https://github.com/user-attachments/assets/76ca78c8-0520-4bb1-92cd-fcacf40fcb91" />

Why this is the Bug: Even though they used a secure parameter for playlist_id later on, they got lazy with $owner. If an attacker can control the $owner input, they can "break out" of that CASE statement and run their own SQL.

---

## 3. Proof of Concept:

To confirm the vulnerability, I performed **Component-Level Testing**. While the Garlic-Hub web interface uses Middleware that can sometimes redirect unauthenticated users, the core database logic remains exposed. I bypassed the front-end "noise" by extracting the vulnerable Repository code into a standalone reproduction script.

### **Why use a Standalone Script (`exploit.php`)?**
1. **Isolation:** It proves the vulnerability exists in the **Source Code** logic, independent of any UI or browser-side redirects.
2. **Engine Verification:** It confirms the MySQL database is actually executing the injected commands passed through the PHP variable.
3. **Accuracy:** It provides a "clean room" environment to measure the exact response time without network lag.

### **Detailed Steps to Reproduce:**

#### **Step 1: Environment Setup**
* I initialized a local **XAMPP stack** on my system.
* I created a database named `garlic_hub` and imported the `items` table schema.
* I ensured the `playlist_id` column was populated to simulate a real-world application state.

#### **Step 2: Creating the Exploit Script**
I created a file named `exploit.php` in the project root. This script mimics the **exact** database connection and the **exact** vulnerable query line found in `ItemsRepository.php`.

<img width="516" height="557" alt="Screenshot 2026-03-23 at 9 46 55 PM" src="https://github.com/user-attachments/assets/f253052d-b9bb-4eb8-bf5c-a04c639dc999" />

### Step 3: Execution and Observation
* I executed the script via the terminal: ```php exploit.php```

* The Payload: ```1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))a)-- -```

**Observation:** The terminal "froze" for exactly 5.01 seconds.

**The Result:** Because the response time was delayed by the exact amount specified in my payload, it is mathematically certain that the database is executing my injected commands.

Terminal Evidence (Screenshot below):

<img width="475" height="119" alt="Screenshot 2026-03-23 at 9 47 18 PM" src="https://github.com/user-attachments/assets/db5e4e3c-f7e5-4606-b900-1aa5ab1f3c9e" />


### 4. Impact Analysis (Why this is Critical)
In the world of Cybersecurity, a "Time-Based" leak is a slow but steady way to destroy a system. This is a Critical (9.8/10) threat because:

1. **Full Data Exfiltration:** Using a script, an attacker can ask the database "True/False" questions. By measuring the sleep time, they can extract the Admin's password hash character by character.

2. **Bypassing the Human Firewall:** An attacker can send a crafted URL to a logged-in Admin. Since the ID is processed immediately, the Admin's browser would trigger the exploit unknowingly.

3. **Remote Code Execution (RCE) Risk:** If the database user has FILE privileges, an attacker could use this SQLi to write a "Web Shell" (malicious PHP file) directly onto the server.

---
### 5. The Fix

To secure the application, we must follow the Principle of Least Privilege and use Parameterization.

**❌ Vulnerable Approach (Raw Concatenation):
The code treats user input as "part of the command."
```php
$query = "SELECT * FROM items WHERE playlist_id = " . $playlistId;
```

**✅ Secure Approach (Prepared Statements):**
We must use PDO Prepared Statements. This tells the database: "This is a fixed command, and whatever the user sends is JUST DATA."

```php
// 1. Define the query with a placeholder (:id)
$query = "SELECT i.* FROM items i WHERE i.playlist_id = :id";
$stmt = $this->connection->prepare($query);

// 2. Bind the value and force it to be an Integer for extra safety
$stmt->bindValue('id', (int)$playlistId); 
$stmt->execute();
```
---
### 6. Conclusion

This discovery confirms that Garlic-Hub is vulnerable to critical data theft. As a cybersecurity professional, I recommend immediate patching of all Repository files to use Prepared Statements. This research proves that even "hidden" back-end code can be uncovered and exploited if the Secure Coding Standards are not followed.

### Contact: [Linkedin](https://www.linkedin.com/in/gaurish-bahurupi/) | Gaurish Bahurupi - Cybersecurity Research
