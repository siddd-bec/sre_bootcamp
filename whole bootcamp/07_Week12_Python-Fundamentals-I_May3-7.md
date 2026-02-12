# PYTHON TRACK — FILE 06 of 48
## PYTHON FUNDAMENTALS I
### Variables, Data Types, Operators, Control Flow, Strings, Input/Output
**Week 12 | May 3 - May 7, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHY PYTHON FOR SRE**
>
> Python is THE language for SRE automation at Apple, Google, Meta, and Visa.
> Every SRE team uses Python for:
> - Automation scripts (deployments, cleanups, health checks)
> - Monitoring tools (Prometheus client, custom exporters)
> - API development (FastAPI, Flask for internal tools)
> - Log parsing and data analysis
> - Infrastructure as Code helpers
> - Interview coding challenges
>
> Apple SRE job posting: "Proficiency in Python for scripting and automation"
>
> This is Week 1 of a 12-week Python journey (Files 06-17).
> We start from absolute zero — no prior Python experience needed.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Python Setup & First Program | Install, REPL, print(), comments, variables |
| 2 | Data Types & Operators | int, float, str, bool, arithmetic, comparison |
| 3 | Strings Deep Dive | Indexing, slicing, methods, f-strings, escape chars |
| 4 | Control Flow | if/elif/else, truthiness, match-case |
| 5 | Loops & Practice | for, while, range, break, continue, nested loops |

---

## DAY 1: PYTHON SETUP & FIRST PROGRAM
**May 3, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: install Python, REPL, print, comments, variables
> 00:20-01:10 Hands-on: write your first programs
> 01:10-01:30 Drills: 10 quick exercises

### Installing Python

```
# Check if Python is already installed (it usually is on Linux):
python3 --version          # should show Python 3.10+ 
which python3              # /usr/bin/python3

# If not installed:
sudo apt update && sudo apt install python3 python3-pip python3-venv

# Verify pip (package installer):
pip3 --version

# Create a project directory:
mkdir -p ~/python-bootcamp/week01
cd ~/python-bootcamp/week01
```

### The Python REPL (Interactive Mode)

```python
# Start the REPL (Read-Eval-Print Loop):
# Type 'python3' in your terminal

>>> print("Hello, Siddharth!")
Hello, Siddharth!

>>> 2 + 3
5

>>> type(42)
<class 'int'>

>>> exit()    # or Ctrl+D to quit
```

> **SRE TIP: WHEN TO USE REPL vs SCRIPTS**
>
> REPL = quick experiments, testing one-liners, debugging
> Scripts (.py files) = anything you want to save, reuse, or automate
>
> As SRE, you'll use scripts 90% of the time.
> But REPL is great for: "let me quickly check if this regex works"

### Your First Script

```python
# Create file: hello.py
# Run with: python3 hello.py

# This is a comment — Python ignores it
# Comments explain your code to humans (including future you!)

# print() displays output to the terminal
print("Hello, World!")
print("My name is Siddharth")
print("I am learning Python for SRE")

# Multiple values in one print:
print("Python version:", 3, "is awesome")
# Output: Python version: 3 is awesome

# print() adds a newline at the end by default
# Change that with end=
print("Loading", end="")
print("...", end="")
print(" Done!")
# Output: Loading... Done!
```

### Variables — Storing Data

```python
# Variables store values. No need to declare types — Python figures it out.
# Rules: start with letter or _, no spaces, case-sensitive

name = "Siddharth"          # string (text)
age = 30                     # integer (whole number)
height = 5.9                 # float (decimal number)
is_sre = True                # boolean (True or False)

# Python is CASE SENSITIVE:
Name = "Alice"               # this is a DIFFERENT variable from 'name'

# Print variables:
print(name)                  # Siddharth
print(type(name))            # <class 'str'>
print(type(age))             # <class 'int'>
print(type(is_sre))          # <class 'bool'>

# Reassign freely — no type restriction:
x = 10                       # x is an int
x = "hello"                  # now x is a string (Python doesn't care)

# Multiple assignment:
a, b, c = 1, 2, 3           # a=1, b=2, c=3
x = y = z = 0               # all three are 0

# Naming conventions (PEP 8 — Python style guide):
server_name = "web-01"       # snake_case for variables (THIS is Python style)
ServerName = "web-01"        # CamelCase is for class names (later)
MAX_RETRIES = 5              # ALL_CAPS for constants
```

> **COMMON BEGINNER MISTAKES**
>
> 1. `my name = "Sid"` — spaces in variable names (use my_name)
> 2. `1server = "web"` — starting with a number (use server1)
> 3. `class = "A"` — using reserved words (class, for, if, etc.)
> 4. Forgetting Python is case sensitive: `Name ≠ name`

### DAY 1 EXERCISES

> **EXERCISE 1: Hello SRE**
>
> Write a script that:
> a) Prints your name, role, and company on separate lines
> b) Stores each in a variable first, then prints the variable
> c) Uses `type()` to print the type of each variable
> d) Prints everything on ONE line using a single print()

> **EXERCISE 2: Variable Practice**
>
> a) Create variables for: hostname, ip_address, port, is_healthy, cpu_usage
> b) Print each with its type
> c) Reassign hostname to a different value, print again
> d) Do multiple assignment: host1, host2, host3 = "web-01", "web-02", "web-03"
> e) Print all three hosts on one line

> **EXERCISE 3: Predict the Output**
>
> Without running, predict what each prints. Then verify.
> a) `print(type(42))`
> b) `print(type(3.14))`
> c) `print(type("hello"))`
> d) `print(type(True))`
> e) `x = 5; x = "five"; print(type(x))`
> f) `print("Hello", "World", sep="-")`
> g) `print("A", end=""); print("B", end=""); print("C")`

> **DRILL: 10 Quick Exercises (2 min each)**
>
> 1. Print "I am an SRE" to the terminal
> 2. Store the number 8080 in a variable called `port`, print it
> 3. Create 3 variables in one line: x=1, y=2, z=3
> 4. Print the type of the value False
> 5. Print two words on the same line using `end=""`
> 6. Store your name, print it 3 times (3 print statements)
> 7. Create a variable `pi = 3.14159`, print its type
> 8. Print "Hello" and "World" separated by a dash using `sep="-"`
> 9. Assign `count = 0`, then change it to `count = 100`, print both steps
> 10. What happens if you type `print(undefined_variable)`? Try it.

---

## DAY 2: DATA TYPES & OPERATORS
**May 4, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: int, float, str, bool, type conversion, operators
> 00:20-01:10 Hands-on: calculations and comparisons
> 01:10-01:30 Drills: operator challenges

### Python Data Types (The Big Four)

```python
# 1. INTEGERS (int) — whole numbers, no limit on size!
count = 42
big_number = 1_000_000_000   # underscores for readability (= 1000000000)
negative = -17

# 2. FLOATS (float) — decimal numbers
cpu_usage = 87.5
pi = 3.14159
tiny = 0.001

# 3. STRINGS (str) — text, enclosed in quotes
name = "Siddharth"           # double quotes
host = 'web-01'              # single quotes (same thing)
message = """This is a
multi-line string"""          # triple quotes for multi-line

# 4. BOOLEANS (bool) — True or False (capital T and F!)
is_healthy = True
is_down = False

# Check type:
print(type(42))              # <class 'int'>
print(type(3.14))            # <class 'float'>
print(type("hello"))         # <class 'str'>
print(type(True))            # <class 'bool'>

# Special value: None (like null in other languages)
result = None                # "no value assigned yet"
print(type(None))            # <class 'NoneType'>
```

### Type Conversion (Casting)

```python
# Python won't auto-convert types for you. You must be explicit.

# String to int:
port_str = "8080"
port_num = int(port_str)     # 8080 (now an integer)

# Int to string:
count = 42
message = "Count is " + str(count)   # must convert to concatenate

# String to float:
cpu = float("87.5")          # 87.5

# Float to int (TRUNCATES, doesn't round!):
int(3.9)                     # 3 (not 4!)
int(3.1)                     # 3

# Round:
round(3.7)                   # 4
round(3.14159, 2)            # 3.14 (2 decimal places)

# Bool conversion:
bool(0)                      # False
bool(1)                      # True
bool("")                     # False (empty string)
bool("hello")                # True (non-empty string)
bool(None)                   # False
```

> **SRE RELEVANCE: WHY TYPES MATTER**
>
> When you read config files or command output, everything comes as STRINGS.
> You must convert to the right type before doing math:
>
> ```python
> # Reading CPU from a monitoring tool:
> cpu_str = "87.5"     # this is a string from stdout
> if float(cpu_str) > 90.0:
>     print("ALERT: CPU critical!")
> ```
>
> Forgetting to convert = bugs. A common SRE interview gotcha.

### Arithmetic Operators

```python
# Basic math:
10 + 3        # 13   addition
10 - 3        # 7    subtraction
10 * 3        # 30   multiplication
10 / 3        # 3.333...  division (ALWAYS returns float!)
10 // 3       # 3    floor division (integer result, rounds DOWN)
10 % 3        # 1    modulo (remainder) — very useful!
10 ** 3       # 1000 exponent (10 to the power of 3)

# Important: / always gives float, // gives int
7 / 2         # 3.5
7 // 2        # 3

# Modulo use cases for SRE:
# Check if number is even: n % 2 == 0
# Rotate through servers: server_index = request_num % num_servers
# Check every Nth iteration: if counter % 100 == 0: print("progress")

# Augmented assignment (shortcuts):
count = 10
count += 5    # count = count + 5 → 15
count -= 3    # count = count - 3 → 12
count *= 2    # count = count * 2 → 24
count //= 4   # count = count // 4 → 6
```

### Comparison & Logical Operators

```python
# Comparison operators (return True or False):
5 == 5        # True    equal
5 != 3        # True    not equal
5 > 3         # True    greater than
5 < 3         # False   less than
5 >= 5        # True    greater or equal
5 <= 3        # False   less or equal

# Logical operators:
True and True    # True
True and False   # False
True or False    # True
not True         # False

# Combine them:
cpu = 85
memory = 70
if cpu > 80 and memory > 60:
    print("System under load")

# Chained comparisons (Python special!):
x = 50
10 < x < 100       # True (is x between 10 and 100?)
# Same as: x > 10 and x < 100

# Identity operators:
x is None           # True if x is None (preferred over == None)
x is not None       # True if x has a value

# Membership operators (preview — very useful!):
"error" in "error: file not found"     # True
5 in [1, 2, 3, 4, 5]                  # True
```

### DAY 2 EXERCISES

> **EXERCISE 1: SRE Calculator**
>
> a) Calculate: total_requests = 1_500_000. error_count = 3_750.
>    What is the error RATE as a percentage? (error_count / total_requests * 100)
> b) Server has 64 GB RAM total. 48.7 GB used. Calculate free GB and usage percentage.
> c) 3 servers handle traffic. 10_000 requests. How many per server? Any remainder?
> d) Disk is 500 GB total, 425 GB used. What percentage free? Round to 1 decimal.

> **EXERCISE 2: Type Conversion Practice**
>
> a) You read "8080" from a config file. Convert to int, add 1, print result.
> b) You read "3.14" from input. Convert to float, multiply by 2, print.
> c) Convert the integer 200 to a string, concatenate with " OK", print.
> d) What does `int("hello")` do? Try it. (Hint: it crashes — this is important!)
> e) What does `bool(0)` return? `bool([])` ? `bool("0")` ? Test each.

> **EXERCISE 3: Comparison Practice**
>
> Write Python expressions that evaluate to True:
> a) Check if 100 is greater than 50
> b) Check if "error" is inside the string "error: timeout"
> c) Check if cpu (85) is between 0 and 100 using chained comparison
> d) Check if a variable is None using `is`
> e) Check if port (443) is equal to 443 AND protocol ("https") equals "https"

> **DRILL: 10 Quick Calculations (2 min each)**
>
> 1. What is 17 // 5?  What is 17 % 5?
> 2. What type does 10 / 2 return? (hint: it's not int)
> 3. Convert the string "42" to an integer, add 8, print result
> 4. Is 0 == False? Is "" == False? Test both.
> 5. What does round(2.675, 2) return? (try it — it might surprise you!)
> 6. Calculate 2 ** 10 (important number in computing!)
> 7. What is bool(None)? bool(0)? bool("False")?
> 8. Is "Python" == "python"? Why?
> 9. Use modulo to check if 144 is divisible by 12
> 10. What is the result of: not (5 > 3 and 2 < 1)?

---

## DAY 3: STRINGS DEEP DIVE
**May 5, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: indexing, slicing, methods, f-strings
> 00:20-01:10 Hands-on: string manipulation exercises
> 01:10-01:30 Drills: string challenges

### String Basics

```python
# Strings are SEQUENCES of characters. Immutable (cannot change in place).

message = "Hello, SRE World!"
#          0123456789...        ← index starts at 0

# Length:
len(message)                 # 17

# Indexing (single character):
message[0]                   # 'H'  (first character)
message[7]                   # 'S'
message[-1]                  # '!'  (last character)
message[-2]                  # 'd'  (second to last)

# Slicing [start:stop:step] — stop is EXCLUSIVE
message[0:5]                 # 'Hello' (chars 0,1,2,3,4)
message[7:10]                # 'SRE'
message[:5]                  # 'Hello' (from beginning)
message[7:]                  # 'SRE World!' (to end)
message[-6:]                 # 'orld!'
message[::2]                 # 'Hlo R ol!' (every 2nd char)
message[::-1]                # '!dlroW ERS ,olleH' (REVERSED!)
```

### f-strings (Formatted String Literals) — USE THESE EVERYWHERE

```python
# f-strings are the modern, preferred way to build strings in Python 3.6+

name = "Siddharth"
role = "SRE"
cpu = 87.5

# Basic f-string:
print(f"Hello, {name}!")                    # Hello, Siddharth!
print(f"{name} is a {role}")                # Siddharth is a SRE

# Expressions inside {}:
print(f"CPU: {cpu}%")                       # CPU: 87.5%
print(f"2 + 3 = {2 + 3}")                  # 2 + 3 = 5
print(f"Uppercase: {name.upper()}")         # Uppercase: SIDDHARTH

# Formatting numbers:
print(f"CPU: {cpu:.1f}%")                   # CPU: 87.5%  (1 decimal)
print(f"Requests: {1500000:,}")             # Requests: 1,500,000
print(f"Error rate: {0.025:.2%}")           # Error rate: 2.50%
print(f"{'Status':<15} {'Code':>5}")        # left/right alignment

# Padding (useful for formatted output):
for code in [200, 404, 500]:
    print(f"HTTP {code:>3}")                # right-align in 3 spaces
```

> **SRE TIP: F-STRINGS FOR LOGGING**
>
> ```python
> import datetime
> host = "web-01"
> status = "DOWN"
> ts = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
> print(f"[{ts}] ALERT: {host} is {status}")
> # [2025-05-05 14:30:22] ALERT: web-01 is DOWN
> ```
>
> You'll use this pattern constantly in SRE scripts.

### Essential String Methods

```python
text = "  Error: Connection timeout on web-01  "

# Whitespace:
text.strip()              # 'Error: Connection timeout on web-01' (trim both ends)
text.lstrip()             # remove left whitespace only
text.rstrip()             # remove right whitespace only

# Case:
text.strip().upper()      # 'ERROR: CONNECTION TIMEOUT ON WEB-01'
text.strip().lower()      # 'error: connection timeout on web-01'
text.strip().title()      # 'Error: Connection Timeout On Web-01'

# Search:
text.strip().startswith("Error")     # True
text.strip().endswith("web-01")      # True (after strip!)
"timeout" in text                     # True (membership test)
text.find("timeout")                  # 21 (index where found)
text.find("missing")                  # -1 (not found)
text.count("o")                       # 4

# Replace:
text.replace("web-01", "web-02")     # swap server name
text.replace("Error", "WARNING")     # change severity

# Split and Join:
log_line = "2025-05-05 14:30:22 ERROR web-01 timeout"
parts = log_line.split()             # ['2025-05-05', '14:30:22', 'ERROR', 'web-01', 'timeout']
parts[2]                             # 'ERROR' (the log level)

csv_line = "web-01,192.168.1.10,8080,healthy"
fields = csv_line.split(",")         # ['web-01', '192.168.1.10', '8080', 'healthy']

# Join (opposite of split):
servers = ["web-01", "web-02", "web-03"]
", ".join(servers)                   # 'web-01, web-02, web-03'
"\n".join(servers)                   # each on new line

# Check content:
"12345".isdigit()                    # True
"hello".isalpha()                    # True
"hello123".isalnum()                 # True
```

### Escape Characters & Raw Strings

```python
# Escape characters:
print("Line 1\nLine 2")     # \n = newline
print("Column1\tColumn2")   # \t = tab
print("He said \"hello\"")  # \" = literal quote
print("Path: C:\\Users")    # \\ = literal backslash

# Raw strings (ignore escapes) — useful for regex and paths:
print(r"C:\new\test")       # C:\new\test (no escaping)
# You'll use raw strings A LOT with regex in Week 12 (Python for SRE I)
```

### DAY 3 EXERCISES

> **EXERCISE 1: Log Line Parser**
>
> Given: `log = "2025-05-05 14:30:22 ERROR web-01 Connection refused to db-master:5432"`
> Using only string methods (no regex):
> a) Extract the date ("2025-05-05")
> b) Extract the time ("14:30:22")
> c) Extract the log level ("ERROR")
> d) Extract the hostname ("web-01")
> e) Check if this line contains "ERROR" (boolean)
> f) Replace "web-01" with "web-02"
> g) Convert the log level to lowercase

> **EXERCISE 2: f-string Formatting**
>
> a) Print a server health report:
>    `"[2025-05-05] web-01 | CPU: 87.5% | Memory: 64.2% | Status: OK"`
>    Use f-strings with variables for each value.
> b) Print a table header with aligned columns:
>    `"Host            CPU     Memory  Status"`
>    `"web-01          87.5%   64.2%   OK"`
>    `"web-02          23.1%   45.8%   OK"`
>    Use f-string alignment (:<15, :>7, etc.)
> c) Format the number 1234567 with commas: `1,234,567`
> d) Format 0.0375 as a percentage with 1 decimal: `3.8%`

> **EXERCISE 3: String Manipulation**
>
> a) Reverse the string "Python" using slicing
> b) Count vowels in "Site Reliability Engineering"
> c) Given CSV: "web-01,10.0.1.1,8080,healthy" — split and extract each field
> d) Given list: ["ERROR", "WARN", "INFO"] — join with " | " separator
> e) Check if an IP address string "192.168.1.1" starts with "192.168"
> f) Remove all whitespace from "  hello   world  " (strip + replace)

> **DRILL: 10 String Challenges (2 min each)**
>
> 1. Get the last 3 characters of "kubernetes"
> 2. Check if "nginx" is in "nginx.conf"
> 3. Split "key=value" by "=" to get key and value separately
> 4. Make "hello world" into "Hello World" (title case)
> 5. Count how many times "e" appears in "Site Reliability Engineering"
> 6. Replace all spaces with underscores in "my log file.txt"
> 7. Check if "8080" is all digits using a string method
> 8. Concatenate "Hello" and "World" with a space between (use join)
> 9. Extract "ERROR" from "[ERROR] disk full" using find + slicing
> 10. Print the string "tab\tseparated" — what does \t do?

---

## DAY 4: CONTROL FLOW
**May 6, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: if, elif, else, truthiness, ternary, match-case
> 00:20-01:10 Hands-on: decision-making scripts
> 01:10-01:30 Drills: conditional challenges

### if / elif / else

```python
# Python uses INDENTATION to define code blocks (not braces {})
# Standard: 4 spaces per level (your editor should handle this)

cpu_usage = 87.5

if cpu_usage > 90:
    print("CRITICAL: CPU above 90%!")
    print("Sending alert...")
elif cpu_usage > 80:
    print("WARNING: CPU above 80%")
elif cpu_usage > 60:
    print("INFO: CPU elevated")
else:
    print("OK: CPU normal")

# Output: WARNING: CPU above 80%

# IMPORTANT: Python checks conditions TOP to BOTTOM
# It runs the FIRST block that matches, then skips the rest
```

### Truthiness — What Python Considers True/False

```python
# "Falsy" values (evaluate to False in conditions):
#   False, 0, 0.0, "", [], {}, (), None, set()

# "Truthy" values (evaluate to True):
#   Everything else! Any non-zero number, non-empty string/list/dict

# This is VERY useful in SRE scripts:

error_message = ""        # empty string
if error_message:         # False (empty = falsy)
    print(f"Error: {error_message}")
else:
    print("No errors")

servers = ["web-01", "web-02"]
if servers:               # True (non-empty list = truthy)
    print(f"Found {len(servers)} servers")

result = None
if result is None:        # preferred way to check for None
    print("No result yet")
```

### Ternary Expression (One-Line if)

```python
# Syntax: value_if_true if condition else value_if_false

status = "healthy" if cpu_usage < 80 else "degraded"
print(status)             # "degraded" (cpu is 87.5)

# Useful for quick assignments:
log_level = "ERROR" if error_count > 0 else "INFO"
color = "red" if status == "down" else "green"
```

### match-case (Python 3.10+ — Like switch)

```python
# Python 3.10 added match-case (similar to switch in other languages)

http_status = 404

match http_status:
    case 200:
        print("OK")
    case 301 | 302:                # multiple values
        print("Redirect")
    case 400:
        print("Bad Request")
    case 404:
        print("Not Found")
    case 500:
        print("Internal Server Error")
    case _:                        # default (underscore = wildcard)
        print(f"Unknown status: {http_status}")
```

### Nested Conditions & Combining Logic

```python
# SRE scenario: check multiple system metrics

cpu = 85
memory = 72
disk = 91

# Nested:
if disk > 90:
    print("CRITICAL: Disk almost full!")
    if cpu > 80:
        print("AND CPU is high — possible runaway process")
elif cpu > 80 and memory > 80:
    print("WARNING: Both CPU and memory elevated")
elif cpu > 80 or memory > 80:
    print("INFO: One metric elevated")
else:
    print("OK: All systems normal")

# Guard clauses (early returns — you'll use this in functions):
# Check for error conditions first, handle the "happy path" at the end
```

### Getting User Input

```python
# input() reads a STRING from the user
name = input("Enter your name: ")
print(f"Hello, {name}!")

# IMPORTANT: input() ALWAYS returns a string!
# You must convert for numbers:
port = int(input("Enter port number: "))      # convert to int
cpu = float(input("Enter CPU usage: "))        # convert to float

# Safe input with validation:
user_input = input("Enter a number: ")
if user_input.isdigit():
    number = int(user_input)
    print(f"You entered: {number}")
else:
    print("That's not a valid number!")
```

### DAY 4 EXERCISES

> **EXERCISE 1: HTTP Status Checker**
>
> Write a script that:
> a) Asks the user for an HTTP status code (input)
> b) Converts to integer
> c) Prints the meaning:
>    200 = "OK", 301 = "Moved Permanently", 400 = "Bad Request",
>    401 = "Unauthorized", 403 = "Forbidden", 404 = "Not Found",
>    500 = "Internal Server Error", 502 = "Bad Gateway", 503 = "Service Unavailable"
> d) For anything else, print "Unknown status code"
> e) Use match-case OR if/elif/else

> **EXERCISE 2: System Health Evaluator**
>
> Write a script that:
> a) Asks for CPU usage (float), memory usage (float), disk usage (float)
> b) Determines overall status:
>    - CRITICAL if ANY metric > 95
>    - WARNING if ANY metric > 80
>    - OK otherwise
> c) Print each metric with its own status (OK/WARNING/CRITICAL)
> d) Print the overall system status

> **EXERCISE 3: SRE Triage Script**
>
> Given variables:
> ```python
> service_name = "payment-api"
> is_responding = False
> error_rate = 15.5           # percentage
> latency_ms = 2500           # milliseconds
> last_deploy_minutes = 30    # minutes ago
> ```
> Write a script that:
> a) Checks if service is responding. If not, print "P1: Service down!"
> b) If responding but error_rate > 10%, print "P2: High error rate"
> c) If responding but latency > 2000ms, print "P2: High latency"
> d) If last deploy was < 60 minutes ago, suggest "Consider rollback"
> e) If everything is fine, print "Service healthy"

> **DRILL: 10 Conditional Challenges (2 min each)**
>
> 1. Check if a number is positive, negative, or zero
> 2. Check if a string is empty using truthiness (not len())
> 3. Write a ternary: "even" if n%2==0 else "odd"
> 4. Check if a year is a leap year (divisible by 4, but not 100, unless 400)
> 5. Given a temperature in Fahrenheit, classify: cold(<50), mild(50-75), hot(>75)
> 6. Check if a port number is valid (1-65535)
> 7. Check if a string is a valid IP octet (0-255, digits only)
> 8. Write nested if: if age >= 18 AND has_license, print "Can drive"
> 9. Check if a variable is None without using ==
> 10. Given three numbers, print the largest (no max() function — use if/elif)

---

## DAY 5: LOOPS & PRACTICE
**May 7, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: for, while, range, break, continue, enumerate
> 00:20-01:10 Hands-on: loop-based SRE scripts
> 01:10-01:30 Drills: loop challenges + week capstone

### for Loops

```python
# for loop iterates over a SEQUENCE (list, string, range, etc.)

# Loop over a list:
servers = ["web-01", "web-02", "web-03", "db-01"]
for server in servers:
    print(f"Checking {server}...")

# Loop over a string:
for char in "ERROR":
    print(char)              # prints E, R, R, O, R on separate lines

# Loop with range():
for i in range(5):           # 0, 1, 2, 3, 4
    print(f"Attempt {i + 1}")

for i in range(1, 6):        # 1, 2, 3, 4, 5
    print(f"Attempt {i}")

for i in range(0, 100, 10):  # 0, 10, 20, ..., 90 (step of 10)
    print(i)

# Loop with enumerate() — get index AND value:
for i, server in enumerate(servers):
    print(f"{i}: {server}")
# 0: web-01
# 1: web-02
# 2: web-03
# 3: db-01

# Loop with enumerate starting at 1:
for num, server in enumerate(servers, start=1):
    print(f"Server #{num}: {server}")
```

### while Loops

```python
# while loop runs as long as condition is True

count = 0
while count < 5:
    print(f"Count: {count}")
    count += 1               # DON'T forget this or infinite loop!

# SRE pattern: retry with backoff
import time

max_retries = 3
attempt = 0
while attempt < max_retries:
    print(f"Attempt {attempt + 1} of {max_retries}...")
    # Imagine checking if service is up here
    success = False          # simulate failure
    if success:
        print("Success!")
        break                # exit the loop early
    attempt += 1
    wait_time = 2 ** attempt  # exponential backoff: 2, 4, 8
    print(f"Failed. Waiting {wait_time}s before retry...")
    # time.sleep(wait_time)  # uncomment in real scripts

if attempt == max_retries:
    print("All retries exhausted!")
```

### break, continue, else

```python
# break — exit the loop immediately
for server in servers:
    if server == "db-01":
        print("Found database server!")
        break                # stops the loop here
    print(f"Checked {server}")

# continue — skip to next iteration
for server in servers:
    if server.startswith("db"):
        continue             # skip database servers
    print(f"Restarting {server}")

# for-else — else runs if loop completed WITHOUT break
for server in servers:
    if server == "cache-01":
        print("Found cache server!")
        break
else:
    print("No cache server found!")   # this runs (no break happened)
```

### Nested Loops

```python
# Loop inside a loop

environments = ["dev", "staging", "prod"]
services = ["api", "web", "worker"]

for env in environments:
    for svc in services:
        print(f"  Deploying {svc} to {env}")

# SRE pattern: check services across environments
for env in environments:
    print(f"\n=== {env.upper()} ===")
    for svc in services:
        # Simulate health check
        status = "healthy"
        print(f"  {svc}: {status}")
```

> **COMMON LOOP MISTAKES**
>
> 1. Infinite while loop (forgot to update counter)
> 2. Off-by-one: `range(5)` gives 0-4, not 1-5
> 3. Modifying a list while looping over it (DON'T do this)
> 4. Using `i` as both loop variable and something else

### Week 1 Capstone: Mini SRE Dashboard

```python
# Combine everything from Days 1-5!
# This is what you should be able to write after this week.

# --- Mini SRE Dashboard ---

print("=" * 50)
print(f"{'SRE SYSTEM DASHBOARD':^50}")
print("=" * 50)

# Server data (you'll use dicts and lists properly next week)
server_names = ["web-01", "web-02", "web-03", "db-01", "cache-01"]
cpu_values = [45.2, 87.1, 23.4, 92.5, 15.0]

# Print header
print(f"\n{'Server':<12} {'CPU':>6} {'Status':<10}")
print("-" * 30)

# Track alerts
critical_count = 0
warning_count = 0

for i in range(len(server_names)):
    server = server_names[i]
    cpu = cpu_values[i]

    if cpu > 90:
        status = "CRITICAL"
        critical_count += 1
    elif cpu > 80:
        status = "WARNING"
        warning_count += 1
    else:
        status = "OK"

    print(f"{server:<12} {cpu:>5.1f}% {status:<10}")

print("-" * 30)
print(f"\nSummary: {critical_count} critical, {warning_count} warning, "
      f"{len(server_names) - critical_count - warning_count} healthy")

if critical_count > 0:
    print("\n*** ALERT: Critical servers detected! ***")
```

### DAY 5 EXERCISES

> **EXERCISE 1: Server Ping Simulator**
>
> Write a script that:
> a) Has a list of 5 server names
> b) Loops through each, prints "Pinging {server}..."
> c) Simulates a random result (use `i % 3 == 0` to simulate some failures)
> d) Counts total UP and DOWN
> e) Prints summary at the end
> f) Exits with a message if ALL servers are down (use break)

> **EXERCISE 2: Number Guessing Game**
>
> Write a script that:
> a) Sets a secret number (e.g., 42)
> b) Asks the user to guess (in a while loop)
> c) Says "Too high" or "Too low" after each guess
> d) Congratulates when correct and shows how many attempts
> e) Limits to 7 attempts, then reveals the answer

> **EXERCISE 3: Multiplication Table Generator**
>
> a) Ask user for a number N
> b) Print the multiplication table for N (1 through 10)
> c) Format output in aligned columns
> d) After the table, print sum of all products

> **EXERCISE 4: Password Validator**
>
> Write a script that checks if a password string meets these rules:
> a) At least 8 characters long
> b) Contains at least one uppercase letter (loop through, check .isupper())
> c) Contains at least one digit (check .isdigit())
> d) Does NOT contain spaces
> e) Print which rules pass/fail, and overall PASS/FAIL

> **EXERCISE 5: Log Level Counter**
>
> Given this list of log lines (just paste them as strings):
> ```python
> logs = [
>     "2025-05-07 10:00:01 INFO  Starting service",
>     "2025-05-07 10:00:02 DEBUG Loading config",
>     "2025-05-07 10:00:03 ERROR Connection refused",
>     "2025-05-07 10:00:04 WARN  High latency detected",
>     "2025-05-07 10:00:05 ERROR Timeout on db-01",
>     "2025-05-07 10:00:06 INFO  Request processed",
>     "2025-05-07 10:00:07 ERROR Disk space low",
>     "2025-05-07 10:00:08 INFO  Health check passed",
>     "2025-05-07 10:00:09 WARN  Memory usage high",
>     "2025-05-07 10:00:10 INFO  Request processed",
> ]
> ```
> a) Count how many of each level: INFO, DEBUG, WARN, ERROR
> b) Print the counts
> c) Print only the ERROR lines
> d) What percentage of lines are errors?

> **DRILL: 10 Loop Challenges (3 min each)**
>
> 1. Print numbers 1 to 20, but skip multiples of 3
> 2. Sum all numbers from 1 to 100 using a for loop
> 3. Print a countdown from 10 to 1, then "LAUNCH!"
> 4. Find the first number divisible by 7 AND 5 between 1-200
> 5. Print each character of "Kubernetes" with its index
> 6. Count vowels in a sentence using a for loop
> 7. Print a right triangle of stars (5 rows): *, **, ***, ****, *****
> 8. Given a list of numbers, print only the even ones
> 9. Use a while loop to keep halving a number until it's less than 1
> 10. FizzBuzz: for 1-30, print "Fizz" if divisible by 3, "Buzz" by 5, "FizzBuzz" by both

---

## WEEK 12 SUMMARY

> **FILE 06 COMPLETE — PYTHON FUNDAMENTALS I**
>
> ✓ Python setup: install, REPL, first script, print()
> ✓ Variables: assignment, naming, types, multiple assignment
> ✓ Data types: int, float, str, bool, None
> ✓ Type conversion: int(), float(), str(), bool()
> ✓ Operators: arithmetic (+, -, *, /, //, %, **), comparison, logical
> ✓ Strings: indexing, slicing, f-strings, methods (split, join, replace, strip)
> ✓ Control flow: if/elif/else, truthiness, ternary, match-case
> ✓ Loops: for, while, range(), enumerate(), break, continue
> ✓ Input: input(), type validation
> ✓ 50+ exercises covering all topics
>
> **INTERVIEW PREP:**
> - "What's the difference between `==` and `is`?" (value vs identity)
> - "What values are falsy in Python?" (0, "", [], {}, None, False)
> - "What does `//` do?" (floor division)
> - "How do you format strings?" (f-strings: `f"text {variable}"`)
>
> Next: File 07 — Python Fundamentals II (Week 13, May 10-14)
> Topics: Lists, Tuples, Dictionaries, Sets, Comprehensions
