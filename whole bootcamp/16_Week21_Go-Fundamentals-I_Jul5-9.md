# GO TRACK — FILE 16 of 48
## GO FUNDAMENTALS I
### Setup, Variables, Types, Control Flow, Functions, Packages
**Week 21 | Jul 5 - Jul 9, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHY GO FOR SRE?**
>
> Go is THE language of cloud-native infrastructure:
> - **Kubernetes** — written in Go
> - **Docker** — written in Go
> - **Prometheus** — written in Go
> - **Terraform** — written in Go
> - **etcd, Consul, Vault** — all Go
>
> Google, Apple, and Meta SRE teams use Go heavily.
> It's fast, compiles to a single binary, and has built-in concurrency.
>
> You already know Python. Go will feel different — stricter,
> but that strictness catches bugs at compile time, not at 3 AM.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Setup & Hello World | Installation, go mod, first program, go run/build |
| 2 | Variables & Types | int, string, bool, arrays, slices, maps |
| 3 | Control Flow | if, for, switch, range |
| 4 | Functions | Multiple returns, errors, defer, variadic |
| 5 | Packages & Project Structure | Packages, imports, visibility, go modules |

---

## DAY 1: SETUP & HELLO WORLD
**Jul 5, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: Go philosophy, installation, project setup
> 00:20-01:10 Hands-on: first programs, go run, go build
> 01:10-01:30 Drills: compilation exercises

### Go vs Python — Key Differences

```
Feature          Python                      Go
─────────        ──────                      ──
Typing           Dynamic (runtime)           Static (compile time)
Compilation      Interpreted                 Compiled to binary
Speed            Slow                        Fast (close to C)
Concurrency      Threading/asyncio           Goroutines (built-in!)
Error handling   try/except                  Explicit error returns
Dependencies     pip + venv                  go mod (built-in)
Output           Needs Python installed      Single binary (just copy!)

WHY GO IS GREAT FOR SRE:
  1. Single binary — no runtime dependencies, just scp to server
  2. Fast startup — CLI tools respond instantly
  3. Goroutines — check 1000 servers concurrently with minimal code
  4. Strong typing — compiler catches bugs before production
  5. Standard library — HTTP server, JSON, crypto built-in
```

### Installation & Setup

```bash
# Install Go (Ubuntu/Debian):
sudo apt update
sudo apt install -y golang-go

# Or download latest from go.dev:
wget https://go.dev/dl/go1.22.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.4.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Verify:
go version            # go version go1.22.4 linux/amd64

# Create your first project:
mkdir -p ~/go-projects/hello
cd ~/go-projects/hello
go mod init hello      # creates go.mod (like requirements.txt)
```

### Hello World

```go
// main.go
// Every Go file starts with a package declaration.
// 'package main' means this is an executable program.
package main

// Import packages (like Python's import)
import "fmt"

// main() is the entry point — like Python's if __name__ == "__main__"
func main() {
    fmt.Println("Hello, SRE World!")
}
```

```bash
# Run without compiling:
go run main.go                    # compiles and runs in one step

# Compile to binary:
go build -o hello main.go        # creates ./hello binary
./hello                           # runs instantly, no Go needed!

# Cross-compile for different OS:
GOOS=linux GOARCH=amd64 go build -o hello-linux main.go
GOOS=darwin GOARCH=arm64 go build -o hello-mac main.go
# This is why Go is perfect for SRE tools — build once, run anywhere!
```

### Go Basics — Syntax Overview

```go
package main

import (
    "fmt"
    "time"
    "os"
)

func main() {
    // Variables:
    var name string = "Siddharth"        // explicit type
    age := 30                            // short declaration (type inferred)
    hostname, _ := os.Hostname()         // multiple return values

    // Print:
    fmt.Println("Name:", name)           // with newline
    fmt.Printf("Age: %d\n", age)        // formatted (like Python's f-string)
    fmt.Printf("Host: %s\n", hostname)

    // Current time:
    now := time.Now()
    fmt.Println("Time:", now.Format("2006-01-02 15:04:05"))
    // Go uses a REFERENCE date: Mon Jan 2 15:04:05 MST 2006
    // Yes, it's weird. Just memorize: 1 2 3 4 5 6 7 (month day hour min sec year timezone)
}
```

### DAY 1 EXERCISES

> **EXERCISE 1: Hello SRE**
>
> a) Create a new Go project with go mod init
> b) Write a program that prints:
>    - "Hello from [hostname]"
>    - Current date and time
>    - Go version (hint: runtime.Version())
> c) Run it with go run
> d) Build a binary and run it

> **EXERCISE 2: System Info**
>
> Write a program that prints:
> a) Hostname: os.Hostname()
> b) Working directory: os.Getwd()
> c) Number of CPUs: runtime.NumCPU()
> d) OS and Architecture: runtime.GOOS, runtime.GOARCH
> e) Environment variable HOME: os.Getenv("HOME")

> **EXERCISE 3: Cross-Compile**
>
> a) Build your program for Linux, macOS, and Windows
> b) Check the file sizes of each binary
> c) Why is cross-compilation useful for SRE? (deploy tools anywhere)

> **DRILL: 10 Go Setup Challenges (2 min each)**
>
> 1. Create a new project with go mod init
> 2. Write and run a Hello World
> 3. Build a binary and run it
> 4. Print a formatted string with fmt.Printf
> 5. Get the hostname with os.Hostname()
> 6. Import multiple packages with ()
> 7. What does package main mean?
> 8. What does func main() do?
> 9. What is go.mod for? (dependency management)
> 10. How is Go different from Python? (compiled, static typing, single binary)

---

## DAY 2: VARIABLES & TYPES
**Jul 6, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: types, declarations, zero values, collections
> 00:20-01:10 Hands-on: work with all Go types
> 01:10-01:30 Drills: type exercises

### Variable Declarations

```go
package main

import "fmt"

func main() {
    // METHOD 1: var with explicit type
    var name string = "Siddharth"
    var port int = 8080
    var healthy bool = true
    var cpuUsage float64 = 87.5

    // METHOD 2: var with type inference
    var hostname = "web-01"          // Go infers string

    // METHOD 3: short declaration := (most common, ONLY inside functions)
    ip := "10.0.1.1"                 // infers string
    count := 42                      // infers int
    ratio := 3.14                    // infers float64
    active := true                   // infers bool

    // CONSTANTS (cannot be changed):
    const maxRetries = 3
    const defaultPort = 8080
    const version = "1.0.0"

    // ZERO VALUES — Go initializes variables automatically:
    var s string   // "" (empty string)
    var n int      // 0
    var f float64  // 0.0
    var b bool     // false
    // No nil/None for basic types! Go is safe by default.

    fmt.Println(name, port, healthy, cpuUsage, hostname, ip, count, ratio, active)
    fmt.Println("Zero values:", s, n, f, b)
}
```

### Basic Types

```go
// INTEGERS:
var i int       // platform-dependent (32 or 64 bit)
var i8 int8     // -128 to 127
var i16 int16   // -32768 to 32767
var i32 int32   // ~±2 billion
var i64 int64   // ~±9 quintillion
var u uint      // unsigned (0 and up)
var u8 uint8    // 0 to 255 (same as byte)

// FLOATS:
var f32 float32  // ~7 decimal digits precision
var f64 float64  // ~15 decimal digits (use this by default)

// STRING:
var s string     // UTF-8 encoded, immutable
s = "hello"
len(s)           // 5 (bytes, not characters for multibyte)

// BOOL:
var b bool       // true or false (not True/False like Python)

// BYTE and RUNE:
var by byte      // alias for uint8 (raw byte)
var r rune       // alias for int32 (Unicode character)

// TYPE CONVERSION (Go requires explicit conversion — no implicit!):
var x int = 42
var y float64 = float64(x)          // must explicitly convert
var z int = int(y)
var s string = fmt.Sprintf("%d", x) // int to string
```

### Strings

```go
package main

import (
    "fmt"
    "strings"
    "strconv"
)

func main() {
    // String basics:
    hostname := "web-01.prod.company.com"

    // String operations (strings package):
    fmt.Println(strings.ToUpper(hostname))                    // WEB-01.PROD...
    fmt.Println(strings.Contains(hostname, "prod"))           // true
    fmt.Println(strings.HasPrefix(hostname, "web"))           // true
    fmt.Println(strings.HasSuffix(hostname, ".com"))          // true
    fmt.Println(strings.Split(hostname, "."))                 // [web-01 prod company com]
    fmt.Println(strings.Replace(hostname, "prod", "staging", 1)) // web-01.staging...
    fmt.Println(strings.TrimSpace("  hello  "))               // "hello"
    fmt.Println(strings.Join([]string{"a", "b", "c"}, ","))  // "a,b,c"

    // String formatting (like Python f-strings):
    name := "web-01"
    cpu := 87.5
    status := fmt.Sprintf("Server %s: CPU=%.1f%%", name, cpu)
    fmt.Println(status)    // Server web-01: CPU=87.5%

    // Format verbs:
    // %s = string      %d = int       %f = float
    // %v = any value   %T = type      %t = bool
    // %.2f = 2 decimal places        %05d = zero-padded 5 digits

    // String ↔ Number conversion:
    numStr := "8080"
    port, err := strconv.Atoi(numStr)     // string → int
    if err != nil {
        fmt.Println("Invalid number:", err)
    }
    portStr := strconv.Itoa(port)          // int → string
    fmt.Println(port, portStr)
}
```

### Slices — Go's Dynamic Arrays

```go
package main

import "fmt"

func main() {
    // ARRAYS (fixed size — rarely used directly):
    var fixedPorts [3]int = [3]int{80, 443, 8080}
    fmt.Println(fixedPorts)              // [80 443 8080]
    fmt.Println(len(fixedPorts))         // 3

    // SLICES (dynamic — this is what you use!):
    // A slice is like Python's list. It can grow and shrink.
    servers := []string{"web-01", "web-02", "db-01"}
    fmt.Println(servers)                 // [web-01 web-02 db-01]
    fmt.Println(len(servers))            // 3

    // Access:
    fmt.Println(servers[0])              // web-01 (0-indexed)
    fmt.Println(servers[len(servers)-1]) // db-01 (last element)

    // Append:
    servers = append(servers, "cache-01")
    fmt.Println(servers)                 // [web-01 web-02 db-01 cache-01]

    // Slice of slice:
    first2 := servers[0:2]               // [web-01 web-02]
    fromIdx1 := servers[1:]              // [web-02 db-01 cache-01]

    // Create with make:
    metrics := make([]float64, 0, 10)    // length 0, capacity 10
    metrics = append(metrics, 87.5, 62.0, 45.2)

    // Iterate:
    for i, server := range servers {
        fmt.Printf("%d: %s\n", i, server)
    }

    // Iterate (ignore index):
    for _, server := range servers {
        fmt.Println(server)
    }

    fmt.Println(first2, fromIdx1, metrics)
}
```

### Maps — Go's Dictionaries

```go
package main

import "fmt"

func main() {
    // MAP = key:value pairs (like Python dict)
    // Syntax: map[keyType]valueType

    // Create with literal:
    serverPorts := map[string]int{
        "web-01": 8080,
        "web-02": 8080,
        "db-01":  5432,
    }
    fmt.Println(serverPorts)              // map[db-01:5432 web-01:8080 web-02:8080]

    // Access:
    fmt.Println(serverPorts["web-01"])    // 8080

    // Check if key exists (IMPORTANT — Go pattern):
    port, exists := serverPorts["cache-01"]
    if exists {
        fmt.Println("Port:", port)
    } else {
        fmt.Println("cache-01 not found")
    }

    // Add / update:
    serverPorts["cache-01"] = 6379

    // Delete:
    delete(serverPorts, "db-01")

    // Iterate:
    for server, port := range serverPorts {
        fmt.Printf("%s → %d\n", server, port)
    }
    // NOTE: map iteration order is RANDOM in Go!

    // Create with make:
    metrics := make(map[string]float64)
    metrics["cpu"] = 87.5
    metrics["memory"] = 62.0
    metrics["disk"] = 45.2

    // Length:
    fmt.Println("Servers:", len(serverPorts))
}
```

### Structs — Custom Types

```go
package main

import "fmt"

// STRUCT = named collection of fields (like a Python class with only attributes)
type Server struct {
    Name        string
    IP          string
    Port        int
    Environment string
    Healthy     bool
    CPU         float64
}

func main() {
    // Create:
    web1 := Server{
        Name:        "web-01",
        IP:          "10.0.1.1",
        Port:        8080,
        Environment: "prod",
        Healthy:     true,
        CPU:         87.5,
    }

    // Access fields:
    fmt.Println(web1.Name)       // web-01
    fmt.Println(web1.CPU)        // 87.5

    // Modify:
    web1.CPU = 45.2
    web1.Healthy = false

    // Print:
    fmt.Printf("%+v\n", web1)   // %+v shows field names

    // Slice of structs:
    fleet := []Server{
        {Name: "web-01", IP: "10.0.1.1", Port: 8080, Environment: "prod"},
        {Name: "web-02", IP: "10.0.1.2", Port: 8080, Environment: "prod"},
        {Name: "db-01",  IP: "10.0.2.1", Port: 5432, Environment: "prod"},
    }

    for _, s := range fleet {
        fmt.Printf("%s (%s:%d)\n", s.Name, s.IP, s.Port)
    }
}
```

### DAY 2 EXERCISES

> **EXERCISE 1: Server Inventory**
>
> a) Create a Server struct with: Name, IP, Port, Environment, CPU, Memory
> b) Create a slice of 5 servers
> c) Print all servers in formatted output
> d) Find the server with highest CPU
> e) Count servers per environment

> **EXERCISE 2: Map Practice**
>
> a) Create a map[string]int for service→port mappings (10 entries)
> b) Look up a port that exists and one that doesn't (check exists)
> c) Add 3 new entries, delete 2
> d) Iterate and print all entries sorted by key (hint: sort the keys first)

> **EXERCISE 3: String Processing**
>
> a) Parse a log line: "2025-07-06 14:30:22 ERROR web-01 Connection refused"
>    Split into fields, extract timestamp, level, hostname, message
> b) Convert a string port number to int
> c) Build a formatted status string with Sprintf
> d) Check if a hostname contains "prod"

> **DRILL: 10 Type Challenges (3 min each)**
>
> 1. Declare variables with := for string, int, float64, bool
> 2. What are Go's zero values for each type?
> 3. Create a slice and append 3 elements
> 4. Create a map and check if a key exists
> 5. Create a struct with 4 fields
> 6. Convert int to float64 (explicit conversion!)
> 7. Use strings.Split to parse a comma-separated string
> 8. Use strconv.Atoi to convert string to int (handle error!)
> 9. Iterate a slice with range
> 10. Why does Go require explicit type conversion?

---

## DAY 3: CONTROL FLOW
**Jul 7, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: if, for, switch, range
> 00:20-01:10 Hands-on: control flow exercises
> 01:10-01:30 Drills: flow challenges

### if / else

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    cpu := 87.5

    // Basic if:
    if cpu > 90 {
        fmt.Println("CRITICAL")
    } else if cpu > 70 {
        fmt.Println("WARNING")
    } else {
        fmt.Println("OK")
    }

    // if with initialization statement (Go special!):
    // The variable only exists inside the if block.
    if hostname, err := os.Hostname(); err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Hostname:", hostname)
    }
    // hostname is NOT accessible here — scoped to the if block

    // This pattern is EVERYWHERE in Go for error handling:
    if err := doSomething(); err != nil {
        fmt.Println("Failed:", err)
        return
    }
}
```

### for — The Only Loop

```go
package main

import "fmt"

func main() {
    // Go has ONLY 'for' — no while, no do-while.
    // But 'for' can do everything:

    // Classic for loop:
    for i := 0; i < 10; i++ {
        fmt.Println(i)
    }

    // While-style (no init/post):
    count := 0
    for count < 5 {
        fmt.Println(count)
        count++
    }

    // Infinite loop:
    for {
        // ... monitoring daemon ...
        break    // use break to exit
    }

    // Range over slice:
    servers := []string{"web-01", "web-02", "db-01"}
    for index, server := range servers {
        fmt.Printf("%d: %s\n", index, server)
    }

    // Range — ignore index:
    for _, server := range servers {
        fmt.Println(server)
    }

    // Range over map:
    ports := map[string]int{"http": 80, "https": 443, "ssh": 22}
    for service, port := range ports {
        fmt.Printf("%s → %d\n", service, port)
    }

    // Range over string (gives runes):
    for i, ch := range "hello" {
        fmt.Printf("%d: %c\n", i, ch)
    }

    // Break and continue:
    for i := 0; i < 100; i++ {
        if i%2 == 0 {
            continue    // skip even numbers
        }
        if i > 10 {
            break       // stop at 10
        }
        fmt.Println(i)  // prints: 1 3 5 7 9
    }
}
```

### switch

```go
package main

import "fmt"

func main() {
    // Switch on value:
    severity := "critical"
    switch severity {
    case "critical":
        fmt.Println("🔴 Page on-call immediately!")
    case "warning":
        fmt.Println("🟡 Investigate within 1 hour")
    case "info":
        fmt.Println("🟢 No action needed")
    default:
        fmt.Println("Unknown severity")
    }
    // NOTE: No break needed! Go switches DON'T fall through by default.
    // Use 'fallthrough' keyword if you want fall-through (rare).

    // Switch on conditions (like if-else chain but cleaner):
    cpu := 87.5
    switch {
    case cpu > 90:
        fmt.Println("CRITICAL")
    case cpu > 70:
        fmt.Println("WARNING")
    case cpu > 50:
        fmt.Println("ELEVATED")
    default:
        fmt.Println("OK")
    }

    // Multiple values per case:
    statusCode := 503
    switch statusCode {
    case 200, 201, 204:
        fmt.Println("Success")
    case 301, 302:
        fmt.Println("Redirect")
    case 400, 401, 403, 404:
        fmt.Println("Client Error")
    case 500, 502, 503:
        fmt.Println("Server Error")
    default:
        fmt.Printf("Unknown: %d\n", statusCode)
    }
}
```

### DAY 3 EXERCISES

> **EXERCISE 1: Health Status Checker**
>
> Write a program that:
> a) Has a slice of Server structs with CPU, Memory, Disk values
> b) Loops through each server
> c) Uses if/else to determine status (OK/WARNING/CRITICAL)
> d) Prints a formatted status report
> e) Counts totals: N healthy, N warning, N critical

> **EXERCISE 2: HTTP Status Classifier**
>
> a) Create a map of status codes and their counts
> b) Loop through them with range
> c) Use switch to classify each as Success/Redirect/Client Error/Server Error
> d) Print summary counts per category

> **EXERCISE 3: Number Processing**
>
> a) Loop 1-100, print FizzBuzz (divisible by 3: Fizz, by 5: Buzz, both: FizzBuzz)
> b) Find all prime numbers up to 100
> c) Sum all even numbers from 1-1000

> **DRILL: 10 Control Flow Challenges (2 min each)**
>
> 1. Write an if with an init statement
> 2. Write a for loop counting 1 to 10
> 3. Write a while-style for loop
> 4. Use range to iterate a slice
> 5. Use range to iterate a map
> 6. Use _ to ignore the index in range
> 7. Write a switch with 4 cases
> 8. Write a conditional switch (no value)
> 9. Use break to exit an infinite loop
> 10. What does 'continue' do?

---

## DAY 4: FUNCTIONS
**Jul 8, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: functions, multiple returns, errors, defer
> 00:20-01:10 Hands-on: build SRE utility functions
> 01:10-01:30 Drills: function challenges

### Function Basics

```go
package main

import "fmt"

// Basic function:
func greet(name string) string {
    return fmt.Sprintf("Hello, %s!", name)
}

// Multiple parameters of same type:
func add(a, b int) int {
    return a + b
}

// MULTIPLE RETURN VALUES (Go's killer feature for error handling):
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil    // nil = no error (like Python's None)
}

func main() {
    msg := greet("Siddharth")
    fmt.Println(msg)

    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Result: %.2f\n", result)
    }
}
```

### Error Handling — The Go Way

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

// Go does NOT have try/except. Instead:
// 1. Functions return (result, error)
// 2. Caller checks if error != nil
// 3. Handle error immediately

func readPort(filename string) (int, error) {
    // Read file:
    data, err := os.ReadFile(filename)
    if err != nil {
        return 0, fmt.Errorf("reading file: %w", err)  // wrap error with context
    }

    // Parse port:
    port, err := strconv.Atoi(string(data))
    if err != nil {
        return 0, fmt.Errorf("parsing port: %w", err)
    }

    // Validate:
    if port < 1 || port > 65535 {
        return 0, fmt.Errorf("invalid port: %d", port)
    }

    return port, nil
}

func main() {
    port, err := readPort("port.txt")
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
    fmt.Println("Port:", port)
}

// PATTERN: Check error, handle immediately, continue if OK.
// This is verbose compared to Python, but it forces you to think
// about every failure case. Your code is more robust.
```

### defer — Cleanup Guaranteed

```go
package main

import (
    "fmt"
    "os"
)

// defer schedules a function to run when the enclosing function returns.
// Like Python's try/finally but cleaner.

func readFile(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()    // will run no matter what happens below!

    // ... read and process file ...
    // Even if this panics, file.Close() still runs.

    return nil
}

// COMMON defer PATTERNS:
// defer file.Close()           // close files
// defer conn.Close()           // close network connections
// defer mutex.Unlock()         // release locks
// defer func() { ... }()       // cleanup logic

func main() {
    // Multiple defers run in LIFO order (stack):
    defer fmt.Println("1 — last")
    defer fmt.Println("2 — middle")
    defer fmt.Println("3 — first")
    // Output: 3, 2, 1
}
```

### Named Returns & Variadic Functions

```go
// Named returns (useful for documentation):
func checkHealth(hostname string) (healthy bool, latencyMs int, err error) {
    // ... check logic ...
    return true, 45, nil
}

// Variadic function (variable number of args):
func sum(numbers ...int) int {
    total := 0
    for _, n := range numbers {
        total += n
    }
    return total
}

// Call:
result := sum(1, 2, 3, 4, 5)          // 15
nums := []int{10, 20, 30}
result = sum(nums...)                   // spread slice into args
```

### SRE Function Examples

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func checkPort(host string, port int, timeout time.Duration) (bool, time.Duration, error) {
    addr := fmt.Sprintf("%s:%d", host, port)
    start := time.Now()

    conn, err := net.DialTimeout("tcp", addr, timeout)
    elapsed := time.Since(start)

    if err != nil {
        return false, elapsed, err
    }
    defer conn.Close()
    return true, elapsed, nil
}

func checkMultiplePorts(host string, ports []int) {
    for _, port := range ports {
        open, latency, err := checkPort(host, port, 3*time.Second)
        if open {
            fmt.Printf("  ✓ %s:%d [%v]\n", host, port, latency.Round(time.Millisecond))
        } else {
            fmt.Printf("  ✗ %s:%d [%v]\n", host, port, err)
        }
    }
}

func main() {
    fmt.Println("Checking localhost:")
    checkMultiplePorts("localhost", []int{22, 80, 443, 8080, 5432})
}
```

### DAY 4 EXERCISES

> **EXERCISE 1: Error Handling**
>
> Write functions that:
> a) `parseLogLine(line string) (LogEntry, error)` — parse a log line, return error if malformed
> b) `readConfig(path string) (Config, error)` — read JSON config, wrap errors with context
> c) `validatePort(port int) error` — return error if port not in 1-65535
> d) Chain them: read config → validate → use

> **EXERCISE 2: Port Scanner**
>
> Build a simple port scanner:
> a) `checkPort(host, port, timeout)` returns (open bool, latency, error)
> b) Scan ports 1-1024 on localhost
> c) Print only open ports with latency
> d) Add a --timeout flag

> **EXERCISE 3: Utility Functions**
>
> Write:
> a) `max(nums ...int) int` — find maximum
> b) `contains(slice []string, item string) bool` — check membership
> c) `retry(attempts int, delay time.Duration, fn func() error) error` — retry logic
> d) Test each with multiple inputs

> **DRILL: 10 Function Challenges (3 min each)**
>
> 1. Write a function that returns two values
> 2. Handle an error from os.ReadFile
> 3. Use defer to close a file
> 4. Write a variadic function
> 5. Use fmt.Errorf to wrap an error with context
> 6. What is nil in Go? (like None — the zero value for pointers, errors, maps, slices)
> 7. Why does Go use (result, error) instead of try/except?
> 8. Write a function that returns a named error
> 9. Multiple defers — in what order do they run? (LIFO)
> 10. What does defer file.Close() guarantee?

---

## DAY 5: PACKAGES & PROJECT STRUCTURE
**Jul 9, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: packages, imports, visibility, go modules
> 00:20-01:10 Hands-on: create a multi-package project
> 01:10-01:30 Drills: package challenges

### Packages

```go
// Every Go file belongs to a package.
// package main = executable program
// package anything_else = library (importable by others)

// VISIBILITY RULE (simple!):
// Uppercase first letter = EXPORTED (public) — accessible from other packages
// Lowercase first letter = unexported (private) — only within same package

// server/server.go:
package server

// Server is exported (uppercase S)
type Server struct {
    Name string    // exported
    IP   string    // exported
    port int       // unexported — only accessible within package server
}

// NewServer is exported (uppercase N)
func NewServer(name, ip string, port int) *Server {
    return &Server{Name: name, IP: ip, port: port}
}

// CheckHealth is exported
func (s *Server) CheckHealth() bool {
    return true
}

// helper is unexported
func helper() {
    // only callable within this package
}
```

### Project Structure

```
sre-tool/
├── go.mod                    # module definition + dependencies
├── go.sum                    # dependency checksums (auto-generated)
├── main.go                   # entry point (package main)
├── server/                   # package server
│   ├── server.go             # Server type + methods
│   └── server_test.go        # tests for server package
├── checker/                  # package checker
│   ├── checker.go            # health check logic
│   └── checker_test.go
└── config/                   # package config
    ├── config.go             # configuration loading
    └── config_test.go
```

```go
// go.mod:
module sre-tool

go 1.22

// main.go:
package main

import (
    "fmt"
    "sre-tool/server"          // import our own package
    "sre-tool/checker"
)

func main() {
    s := server.NewServer("web-01", "10.0.1.1", 8080)
    result := checker.Check(s)
    fmt.Println(result)
}
```

### External Dependencies

```bash
# Add external packages:
go get github.com/prometheus/client_golang
go get gopkg.in/yaml.v3

# go.mod updates automatically. go.sum tracks checksums.

# Tidy up (remove unused):
go mod tidy

# Vendor dependencies (include in your repo):
go mod vendor
```

### Testing (Preview)

```go
// server/server_test.go
package server

import "testing"

func TestNewServer(t *testing.T) {
    s := NewServer("web-01", "10.0.1.1", 8080)
    if s.Name != "web-01" {
        t.Errorf("expected web-01, got %s", s.Name)
    }
    if s.IP != "10.0.1.1" {
        t.Errorf("expected 10.0.1.1, got %s", s.IP)
    }
}

// Run tests:
// go test ./...              # test all packages
// go test -v ./server/       # verbose, specific package
// go test -cover ./...       # with coverage
```

### DAY 5 EXERCISES

> **EXERCISE 1: Multi-Package Project**
>
> Create an sre-tool project with:
> a) package main (main.go)
> b) package server (Server struct, NewServer, CheckHealth)
> c) package config (LoadConfig that reads YAML/JSON)
> d) Import and use all packages from main

> **EXERCISE 2: Visibility Practice**
>
> a) Create exported and unexported types, functions, and fields
> b) Try to access unexported items from another package (see the error!)
> c) Use a constructor function (NewXxx) to set unexported fields

> **EXERCISE 3: Write Tests**
>
> a) Write 3 test functions for your Server package
> b) Test edge cases (empty name, invalid port)
> c) Run tests with go test -v
> d) Check coverage with go test -cover

> **DRILL: 10 Package Challenges (2 min each)**
>
> 1. Create a new package (not main)
> 2. Export a function (uppercase first letter)
> 3. Import your own package from main
> 4. What makes a name exported vs unexported?
> 5. Add an external dependency with go get
> 6. Run go mod tidy
> 7. Write a test function (func TestXxx)
> 8. Run go test ./...
> 9. What is go.mod? What is go.sum?
> 10. How does Go's visibility compare to Python's? (Go: Uppercase/lowercase. Python: convention only)

---

## WEEK 21 SUMMARY

> **FILE 16 COMPLETE — GO FUNDAMENTALS I**
>
> ✓ Setup: installation, go mod, go run, go build, cross-compile
> ✓ Variables & Types: int, string, float64, bool, zero values
> ✓ Collections: slices (dynamic arrays), maps (dicts), structs
> ✓ Strings: strings package, formatting, conversion
> ✓ Control flow: if (with init), for (only loop), switch, range
> ✓ Functions: multiple returns, error handling, defer, variadic
> ✓ Packages: visibility, project structure, go modules, testing
> ✓ 40+ exercises including port scanner and multi-package project
>
> **Go vs Python Cheat Sheet:**
> | Python | Go |
> |--------|-----|
> | list | slice ([]Type) |
> | dict | map[KeyType]ValueType |
> | class | struct + methods |
> | try/except | if err != nil |
> | with open() | defer file.Close() |
> | import module | import "package" |
> | pip install | go get |
>
> **INTERVIEW PREP:**
> - "Why use Go for SRE?" → Single binary, fast, built-in concurrency,
>   cloud-native ecosystem (K8s, Docker, Prometheus all in Go)
> - "How does Go handle errors?" → Functions return (result, error),
>   caller checks err != nil immediately
> - "What is defer?" → Schedules cleanup to run when function exits,
>   like try/finally. Used for closing files, connections, locks.
>
> Next: File 17 — Go Fundamentals II (Week 22, Jul 12-16)
> Topics: Pointers, Methods, Interfaces, Goroutines, Channels
