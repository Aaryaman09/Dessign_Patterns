# Production Object Pool: Complete Breakdown

Let me break down this code piece by piece, connecting it to what you already learned.

---

## The Big Picture First

This code has **4 layers**, each building on the previous:

```
┌─────────────────────────────────────────────────────────────────┐
│                        ARCHITECTURE OVERVIEW                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   LAYER 4: PostgresConnectionPool (What you use)                │
│       │    "Give me a database connection"                      │
│       │                                                         │
│       ▼                                                         │
│   LAYER 3: ObjectPool (Generic pool logic)                      │
│       │    "Manage any type of reusable object"                 │
│       │                                                         │
│       ▼                                                         │
│   LAYER 2: PooledPostgresConnection (Database wrapper)          │
│       │    "A database connection that knows how to reset"      │
│       │                                                         │
│       ▼                                                         │
│   LAYER 1: Poolable + Config + Stats (Contracts & Settings)     │
│            "Rules every pooled object must follow"              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Now let's understand each layer.

---

## Layer 1: The Foundation (Contracts & Configuration)

### 1.1 The Poolable Contract

```python
class Poolable(ABC):
    @abstractmethod
    def reset(self) -> None:
        pass
    
    @abstractmethod
    def validate(self) -> bool:
        pass
    
    @abstractmethod
    def dispose(self) -> None:
        pass
```

**What is this?**

A **contract** (interface) that says: "If you want to be managed by my pool, you MUST have these 3 abilities."

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE POOLABLE CONTRACT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Any object in the pool MUST be able to:                       │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  reset()                                                │   │
│   │  ────────                                               │   │
│   │  "Clean yourself for the next user"                     │   │
│   │                                                         │   │
│   │  WHEN: Called before returning to pool                  │   │
│   │  WHY:  Prevent data leakage between users               │   │
│   │                                                         │   │
│   │  Example: Rollback uncommitted transactions             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  validate() -> bool                                     │   │
│   │  ──────────────────                                     │   │
│   │  "Are you still alive and working?"                     │   │
│   │                                                         │   │
│   │  WHEN: Called before giving to a user                   │   │
│   │  WHY:  Don't give dead connections to users             │   │
│   │                                                         │   │
│   │  Example: Run "SELECT 1" to test connection             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  dispose()                                              │   │
│   │  ─────────                                              │   │
│   │  "Destroy yourself permanently"                         │   │
│   │                                                         │   │
│   │  WHEN: Called when removing from pool forever           │   │
│   │  WHY:  Clean up resources (close sockets, free memory)  │   │
│   │                                                         │   │
│   │  Example: Close the database connection                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why do we need this?**

The pool doesn't know what kind of objects it's managing. It could be:
- Database connections
- HTTP clients
- Thread workers
- File handles

But the pool needs to reset, validate, and dispose them. This contract guarantees every pooled object has these abilities.

---

### 1.2 Pool Configuration

```python
@dataclass
class PoolConfig:
    min_size: int = 2
    max_size: int = 10
    acquire_timeout: float = 30.0
    idle_timeout: float = 300.0
    max_lifetime: float = 3600.0
    validation_on_acquire: bool = True
    validation_query: str = "SELECT 1"
```

**What is this?**

Settings that control pool behavior. Think of it as a "settings panel."

```
┌─────────────────────────────────────────────────────────────────┐
│                    POOL CONFIGURATION EXPLAINED                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   min_size = 2                                                  │
│   ──────────────                                                │
│   "Always keep at least 2 connections ready"                    │
│   Created at startup, never goes below this.                    │
│                                                                 │
│   ┌─────┐ ┌─────┐                                               │
│   │Conn1│ │Conn2│  ← Always available                           │
│   └─────┘ └─────┘                                               │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   max_size = 10                                                 │
│   ──────────────                                                │
│   "Never create more than 10 connections"                       │
│   Prevents overwhelming the database server.                    │
│                                                                 │
│   ┌─────┐ ┌─────┐ ┌─────┐ ... ┌─────┐                           │
│   │  1  │ │  2  │ │  3  │     │ 10  │  ← Maximum                │
│   └─────┘ └─────┘ └─────┘     └─────┘                           │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   acquire_timeout = 30.0                                        │
│   ──────────────────────                                        │
│   "Wait max 30 seconds for a connection"                        │
│   If all connections are busy, wait. But not forever.           │
│                                                                 │
│   User requests connection                                      │
│         │                                                       │
│         ▼                                                       │
│   All 10 busy? ──► Wait up to 30s ──► Still busy? ──► ERROR!    │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   max_lifetime = 3600.0 (1 hour)                                │
│   ──────────────────────────────                                │
│   "Replace connections older than 1 hour"                       │
│   Old connections can become stale or hold bad state.           │
│                                                                 │
│   Connection created at 10:00 AM                                │
│   Current time: 11:05 AM                                        │
│   Age: 65 minutes > 60 minutes                                  │
│   Action: Dispose and create fresh connection                   │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   validation_on_acquire = True                                  │
│   ────────────────────────────                                  │
│   "Test connection before giving to user"                       │
│   Runs validation_query ("SELECT 1") to verify it works.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 1.3 Pool Statistics

```python
@dataclass
class PoolStatistics:
    total_connections_created: int = 0
    total_connections_destroyed: int = 0
    total_acquisitions: int = 0
    total_releases: int = 0
    total_validation_failures: int = 0
    total_timeouts: int = 0
    current_pool_size: int = 0
    current_in_use: int = 0
    current_available: int = 0
```

**What is this?**

A counter dashboard for monitoring. Helps you answer:
- "How many connections have we created?"
- "How many times did validation fail?"
- "Are we running out of connections?"

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATISTICS DASHBOARD                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   LIFETIME COUNTERS (since pool started):                       │
│   ├── total_connections_created: 15                             │
│   ├── total_connections_destroyed: 5                            │
│   ├── total_acquisitions: 1,247                                 │
│   ├── total_releases: 1,245                                     │
│   ├── total_validation_failures: 3                              │
│   └── total_timeouts: 0                                         │
│                                                                 │
│   CURRENT STATE (right now):                                    │
│   ├── current_pool_size: 10                                     │
│   ├── current_in_use: 7                                         │
│   └── current_available: 3                                      │
│                                                                 │
│   INTERPRETATION:                                               │
│   • 15 created, 5 destroyed = 10 alive (matches pool_size ✓)    │
│   • 1247 acquired, 1245 released = 2 still in use... wait,      │
│     current says 7? Someone forgot to release! (bug detected)   │
│   • 0 timeouts = pool is sized correctly                        │
│   • 3 validation failures = some connections died (normal)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 1.4 Custom Exceptions

```python
class PoolExhaustedError(Exception):
    """Raised when a connection cannot be acquired within the timeout."""
    pass

class PoolClosedError(Exception):
    """Raised when operations are attempted on a closed pool."""
    pass
```

**What is this?**

Custom error types so you can handle specific problems:

```python
try:
    with pool.connection() as conn:
        conn.execute("SELECT * FROM users")
except PoolExhaustedError:
    # All connections busy, maybe retry later
    return "Server busy, try again"
except PoolClosedError:
    # Pool was shut down
    return "Service unavailable"
```

---

## Layer 2: The Database Connection Wrapper

```python
class PooledPostgresConnection(Poolable):
    ...
```

**What is this?**

A wrapper around the real `psycopg2` connection that implements the `Poolable` contract.

```
┌─────────────────────────────────────────────────────────────────┐
│              PooledPostgresConnection STRUCTURE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              PooledPostgresConnection                   │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │                                                         │   │
│   │   WRAPS:                                                │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │  psycopg2.connection (the REAL connection)      │   │   │
│   │   │  - Actually talks to PostgreSQL                 │   │   │
│   │   │  - Handles SQL queries                          │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   │                                                         │   │
│   │   ADDS:                                                 │   │
│   │   ├── connection_id (for logging/debugging)            │   │
│   │   ├── created_at (to track age)                        │   │
│   │   ├── reset() (clean for next user)                    │   │
│   │   ├── validate() (check if alive)                      │   │
│   │   └── dispose() (close permanently)                    │   │
│   │                                                         │   │
│   │   CONVENIENCE METHODS:                                  │   │
│   │   ├── execute(query, params) → results                 │   │
│   │   ├── execute_many(query, params_list)                 │   │
│   │   ├── commit()                                         │   │
│   │   └── rollback()                                       │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Breaking Down Each Method:

#### `__init__` - Create the connection

```python
def __init__(self, dsn: str, connection_id: int, validation_query: str = "SELECT 1"):
    self._dsn = dsn                    # Database address
    self._connection_id = connection_id # Unique ID for logging
    self._validation_query = validation_query
    self._created_at = time.time()     # Birth timestamp
    self._conn = None                  # Will hold real connection
    
    self._connect()  # Actually connect to database
```

#### `_connect` - The expensive operation

```python
def _connect(self) -> None:
    logger.info(f"Connection {self._connection_id}: Establishing connection...")
    start = time.time()
    
    self._conn = psycopg2.connect(self._dsn)  # THIS IS SLOW (50-200ms)
    self._conn.autocommit = False
    
    elapsed = time.time() - start
    logger.info(f"Connection {self._connection_id}: Connected in {elapsed:.3f}s")
```

**This is WHY we pool.** This operation is slow. We want to do it once, then reuse.

#### `reset` - Clean for next user (CRITICAL)

```python
def reset(self) -> None:
    if self._conn is None:
        return
    
    try:
        # Step 1: Rollback any uncommitted work
        if self._conn.status != psycopg2.extensions.STATUS_READY:
            self._conn.rollback()
        
        # Step 2: Reset ALL session settings
        with self._conn.cursor() as cursor:
            cursor.execute("DISCARD ALL")
        self._conn.commit()
        
    except Exception as e:
        raise  # If reset fails, connection is unusable
```

**Visual explanation of why reset matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY RESET IS CRITICAL                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   WITHOUT PROPER RESET:                                         │
│   ─────────────────────                                         │
│                                                                 │
│   User A gets connection:                                       │
│   ├── SET ROLE admin;                                           │
│   ├── BEGIN TRANSACTION;                                        │
│   ├── UPDATE accounts SET balance = 1000000;                    │
│   └── (forgets to commit, releases connection)                  │
│                                                                 │
│   User B gets SAME connection:                                  │
│   ├── Still has admin role! (SECURITY BUG)                      │
│   ├── Still in User A's transaction! (DATA BUG)                 │
│   └── SELECT * FROM accounts; (sees uncommitted changes!)       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   WITH PROPER RESET:                                            │
│   ───────────────────                                           │
│                                                                 │
│   User A releases connection:                                   │
│   ├── reset() called automatically                              │
│   ├── ROLLBACK (undo uncommitted changes)                       │
│   └── DISCARD ALL (reset role, timezone, everything)            │
│                                                                 │
│   User B gets SAME connection:                                  │
│   ├── Clean slate, default role                                 │
│   ├── No transaction in progress                                │
│   └── Safe to use!                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### `validate` - Check if alive

```python
def validate(self) -> bool:
    if self._conn is None:
        return False
    
    if self._conn.closed:
        return False
    
    try:
        with self._conn.cursor() as cursor:
            cursor.execute(self._validation_query)  # "SELECT 1"
            cursor.fetchone()
        return True
    except Exception:
        return False  # Connection is dead
```

**When does a connection die?**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHY CONNECTIONS DIE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. NETWORK ISSUES                                             │
│      ├── WiFi disconnected                                      │
│      ├── VPN dropped                                            │
│      └── Firewall killed idle connection                        │
│                                                                 │
│   2. DATABASE SERVER                                            │
│      ├── Server restarted                                       │
│      ├── Server killed idle connections (timeout)               │
│      └── Server ran out of memory                               │
│                                                                 │
│   3. TIME                                                       │
│      ├── Connection sat idle for hours                          │
│      └── TCP keepalive failed                                   │
│                                                                 │
│   SOLUTION: Validate before giving to user                      │
│   Run "SELECT 1" - if it fails, connection is dead              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### `dispose` - Permanent destruction

```python
def dispose(self) -> None:
    if self._conn is not None:
        try:
            if not self._conn.closed:
                self._conn.close()  # Close the actual connection
        except Exception as e:
            logger.error(f"Error during dispose: {e}")
        finally:
            self._conn = None  # Clear reference
```

---

## Layer 3: The Generic Object Pool

This is the **core engine** that manages any `Poolable` objects.

### 3.1 The PooledWrapper

```python
@dataclass
class PooledWrapper(Generic[T]):
    obj: T                                              # The actual object
    created_at: float = field(default_factory=time.time)
    last_used_at: float = field(default_factory=time.time)
    last_validated_at: float = field(default_factory=time.time)
    use_count: int = 0
```

**What is this?**

A "box" that holds the object plus metadata about it.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY WE NEED A WRAPPER                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   The CONNECTION knows how to:                                  │
│   ├── Execute queries                                           │
│   ├── Commit/rollback                                           │
│   └── Reset itself                                              │
│                                                                 │
│   The CONNECTION does NOT know:                                 │
│   ├── When was I created?                                       │
│   ├── When was I last used?                                     │
│   ├── How many times have I been borrowed?                      │
│   └── Am I too old?                                             │
│                                                                 │
│   The WRAPPER tracks this metadata:                             │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    PooledWrapper                        │   │
│   ├─────────────────────────────────────────────────────────┤   │
│   │  obj: PooledPostgresConnection  ◄── The actual object   │   │
│   │  created_at: 1699500000.0       ◄── Birth time          │   │
│   │  last_used_at: 1699500500.0     ◄── Last borrowed       │   │
│   │  use_count: 47                  ◄── Times borrowed      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   This separation keeps concerns clean:                         │
│   • Connection focuses on database stuff                        │
│   • Wrapper focuses on pool management stuff                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 The ObjectPool Class

Let me explain the key methods:

#### `__init__` - Setup the pool

```python
def __init__(self, factory: Callable[[], T], config: Optional[PoolConfig] = None):
    self._factory = factory      # Function to create new objects
    self._config = config or PoolConfig()
    
    # Storage
    self._available: Queue[PooledWrapper[T]] = Queue()  # Available objects
    self._in_use: dict[int, PooledWrapper[T]] = {}      # Borrowed objects
    self._lock = threading.RLock()                       # Thread safety
    
    # Stats and state
    self._stats = PoolStatistics()
    self._closed = False
    
    # Pre-fill with min_size objects
    self._initialize_pool()
```

**Why these specific data structures?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA STRUCTURE CHOICES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   self._available: Queue                                        │
│   ──────────────────────────                                    │
│   WHY QUEUE?                                                    │
│   ├── Thread-safe (multiple threads can access safely)          │
│   ├── FIFO order (oldest connections used first)                │
│   ├── Blocking get (can wait for available object)              │
│   └── Built into Python (no external dependency)                │
│                                                                 │
│   ┌─────┐ ┌─────┐ ┌─────┐                                       │
│   │ Old │ │ Mid │ │ New │  ──► acquire() takes Old first        │
│   └─────┘ └─────┘ └─────┘                                       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   self._in_use: dict[int, PooledWrapper]                        │
│   ──────────────────────────────────────                        │
│   WHY DICT?                                                     │
│   ├── Fast lookup by ID: O(1)                                   │
│   ├── Easy to remove when released                              │
│   └── Can iterate to find wrapper for an object                 │
│                                                                 │
│   {                                                             │
│       140234567890: Wrapper(conn1),  ◄── id(wrapper) as key     │
│       140234567891: Wrapper(conn2),                             │
│   }                                                             │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   self._lock: RLock (Reentrant Lock)                            │
│   ─────────────────────────────────────                         │
│   WHY RLOCK?                                                    │
│   ├── Prevents race conditions                                  │
│   ├── Reentrant = same thread can lock multiple times           │
│   └── Needed because some methods call other methods            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### `acquire` - The Main Logic

This is the heart of the pool. Let me show you the flow:

```
┌─────────────────────────────────────────────────────────────────┐
│                      ACQUIRE FLOW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   acquire() called                                              │
│        │                                                        │
│        ▼                                                        │
│   ┌──────────────────┐                                          │
│   │ Pool closed?     │──YES──► Raise PoolClosedError            │
│   └────────┬─────────┘                                          │
│            │ NO                                                 │
│            ▼                                                    │
│   ┌──────────────────┐                                          │
│   │ Start timeout    │                                          │
│   │ countdown        │                                          │
│   └────────┬─────────┘                                          │
│            │                                                    │
│            ▼                                                    │
│   ┌──────────────────────────────────────────────────────┐      │
│   │                  MAIN LOOP                           │      │
│   │  ┌──────────────────┐                                │      │
│   │  │ Time remaining?  │──NO──► Raise PoolExhaustedError│      │
│   │  └────────┬─────────┘                                │      │
│   │           │ YES                                      │      │
│   │           ▼                                          │      │
│   │  ┌──────────────────┐                                │      │
│   │  │ Try get wrapper  │──NONE──► Loop again (wait)     │      │
│   │  └────────┬─────────┘                                │      │
│   │           │ GOT ONE                                  │      │
│   │           ▼                                          │      │
│   │  ┌──────────────────┐                                │      │
│   │  │ Too old?         │──YES──► Dispose, loop again    │      │
│   │  └────────┬─────────┘                                │      │
│   │           │ NO                                       │      │
│   │           ▼                                          │      │
│   │  ┌──────────────────┐                                │      │
│   │  │ Validation on?   │──YES──┐                        │      │
│   │  └────────┬─────────┘       │                        │      │
│   │           │ NO              ▼                        │      │
│   │           │        ┌──────────────────┐              │      │
│   │           │        │ Validate OK?     │              │      │
│   │           │        └────────┬─────────┘              │      │
│   │           │                 │                        │      │
│   │           │        NO ◄─────┴─────► YES              │      │
│   │           │         │               │                │      │
│   │           │         ▼               │                │      │
│   │           │    Dispose, loop        │                │      │
│   │           │                         │                │      │
│   │           └─────────────────────────┘                │      │
│   │                         │                            │      │
│   │                         ▼                            │      │
│   │              ┌──────────────────┐                    │      │
│   │              │ Mark as in-use   │                    │      │
│   │              │ Update stats     │                    │      │
│   │              │ Return object    │                    │      │
│   │              └──────────────────┘                    │      │
│   │                         │                            │      │
│   └─────────────────────────┼────────────────────────────┘      │
│                             │                                   │
│                             ▼                                   │
│                      SUCCESS! Object returned                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### `_try_get_wrapper` - Where objects come from

```python
def _try_get_wrapper(self, timeout: float) -> Optional[PooledWrapper[T]]:
    # Strategy 1: Try to get from available queue (instant)
    try:
        return self._available.get_nowait()
    except Empty:
        pass
    
    # Strategy 2: Create new if under limit
    if self._should_create_new():
        with self._lock:
            if self._should_create_new():  # Double-check after lock
                try:
                    return self._create_new_wrapper()
                except Exception:
                    pass  # Fall through to wait
    
    # Strategy 3: Wait for someone to release
    try:
        return self._available.get(timeout=min(timeout, 1.0))
    except Empty:
        return None
```

**Visual explanation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  _try_get_wrapper STRATEGIES                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   STRATEGY 1: Grab from shelf (instant)                         │
│   ─────────────────────────────────────                         │
│                                                                 │
│   Available Queue: [Conn1] [Conn2] [Conn3]                      │
│                       ▲                                         │
│                       └── Pop this one, return immediately      │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   STRATEGY 2: Create new (if allowed)                           │
│   ─────────────────────────────────────                         │
│                                                                 │
│   Available: [] (empty)                                         │
│   In Use: 3                                                     │
│   Max Size: 10                                                  │
│                                                                 │
│   Total = 0 + 3 = 3 < 10 ──► Create new connection!             │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   STRATEGY 3: Wait for release                                  │
│   ─────────────────────────────                                 │
│                                                                 │
│   Available: [] (empty)                                         │
│   In Use: 10                                                    │
│   Max Size: 10                                                  │
│                                                                 │
│   Total = 0 + 10 = 10 = 10 ──► Can't create, must wait          │
│                                                                 │
│   Block on queue.get(timeout=1.0)                               │
│   When someone releases, we'll get it                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why double-check after lock?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE RACE CONDITION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   WITHOUT DOUBLE-CHECK:                                         │
│   ─────────────────────                                         │
│                                                                 │
│   Thread A                    Thread B                          │
│   ────────                    ────────                          │
│   size = 9                    size = 9                          │
│   9 < 10? YES                 9 < 10? YES                       │
│   create_new()                create_new()                      │
│   size = 10                   size = 11  ← EXCEEDED MAX!        │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   WITH DOUBLE-CHECK:                                            │
│   ──────────────────                                            │
│                                                                 │
│   Thread A                    Thread B                          │
│   ────────                    ────────                          │
│   size = 9                    size = 9                          │
│   9 < 10? YES                 9 < 10? YES                       │
│   acquire lock ✓              acquire lock (BLOCKED)            │
│   size = 9? YES               ...waiting...                     │
│   create_new()                ...waiting...                     │
│   size = 10                   ...waiting...                     │
│   release lock                acquire lock ✓                    │
│                               size = 10? NO (10 < 10 is false)  │
│                               don't create, wait instead        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### `release` - Return object to pool

```python
def release(self, obj: T) -> None:
    if self._closed:
        obj.dispose()  # Pool closed, just destroy
        return
    
    # Find the wrapper for this object
    wrapper = None
    with self._lock:
        for wrapper_id, w in list(self._in_use.items()):
            if w.obj is obj:  # Found it!
                wrapper = w
                del self._in_use[wrapper_id]
                break
    
    if wrapper is None:
        logger.warning("Object not from this pool!")
        return
    
    # Reset and return to available
    try:
        obj.reset()
        wrapper.mark_released()
        self._available.put_nowait(wrapper)
    except Exception:
        # Reset failed, object is corrupted
        self._dispose_wrapper(wrapper)
```

**Visual flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                       RELEASE FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   release(conn) called                                          │
│        │                                                        │
│        ▼                                                        │
│   ┌──────────────────┐                                          │
│   │ Pool closed?     │──YES──► conn.dispose(), return           │
│   └────────┬─────────┘                                          │
│            │ NO                                                 │
│            ▼                                                    │
│   ┌──────────────────┐                                          │
│   │ Find wrapper in  │                                          │
│   │ _in_use dict     │                                          │
│   └────────┬─────────┘                                          │
│            │                                                    │
│   ┌────────┴────────┐                                           │
│   │                 │                                           │
│   NOT FOUND         FOUND                                       │
│   │                 │                                           │
│   ▼                 ▼                                           │
│   Log warning    Remove from _in_use                            │
│   Return         │                                              │
│                  ▼                                              │
│            ┌──────────────────┐                                 │
│            │ obj.reset()      │                                 │
│            └────────┬─────────┘                                 │
│                     │                                           │
│            ┌────────┴────────┐                                  │
│            │                 │                                  │
│         SUCCESS           FAILED                                │
│            │                 │                                  │
│            ▼                 ▼                                  │
│   Put in _available     Dispose wrapper                         │
│   Update stats          (object is broken)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### `acquire_connection` - The Context Manager

```python
@contextmanager
def acquire_connection(self, timeout: Optional[float] = None):
    obj = self.acquire(timeout=timeout)
    try:
        yield obj  # Give to user
    finally:
        self.release(obj)  # ALWAYS release, even on exception
```

**Why this is the SAFEST way:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTEXT MANAGER GUARANTEES RELEASE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   DANGEROUS (manual acquire/release):                           │
│   ───────────────────────────────────                           │
│                                                                 │
│   conn = pool.acquire()                                         │
│   result = conn.execute("SELECT * FROM users")  # Exception!    │
│   pool.release(conn)  # NEVER REACHED! Connection leaked!       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   SAFE (context manager):                                       │
│   ───────────────────────                                       │
│                                                                 │
│   with pool.acquire_connection() as conn:                       │
│       result = conn.execute("SELECT * FROM users")  # Exception!│
│   # finally block ALWAYS runs, connection released!             │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   HOW IT WORKS:                                                 │
│                                                                 │
│   with pool.acquire_connection() as conn:                       │
│        │                                                        │
│        ├── 1. acquire() called                                  │
│        ├── 2. conn given to you                                 │
│        │                                                        │
│        │   ... your code runs ...                               │
│        │                                                        │
│        └── 3. finally: release(conn)  ◄── ALWAYS RUNS           │
│                                           even if exception     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Layer 4: PostgresConnectionPool (The Facade)

This is what users actually interact with. It simplifies everything.

```python
class PostgresConnectionPool:
    def __init__(self, host, port, database, user, password, config):
        # Build connection string
        self._dsn = f"host={host} port={port} dbname={database} ..."
        
        # Create the generic pool with a factory function
        self._pool = ObjectPool(
            factory=self._create_connection,  # How to make connections
            config=config
        )
    
    def _create_connection(self) -> PooledPostgresConnection:
        # This is called by ObjectPool when it needs a new connection
        return PooledPostgresConnection(dsn=self._dsn, ...)
    
    @contextmanager
    def connection(self):
        # Simple wrapper around the pool's context manager
        with self._pool.acquire_connection() as conn:
            yield conn
    
    @contextmanager
    def transaction(self):
        # Adds automatic commit/rollback
        with self._pool.acquire_connection() as conn:
            try:
                yield conn
                conn.commit()  # Success = commit
            except Exception:
                conn.rollback()  # Error = rollback
                raise
```

**Why have this extra layer?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY THE FACADE LAYER?                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   WITHOUT FACADE (user must know internals):                    │
│   ──────────────────────────────────────────                    │
│                                                                 │
│   config = PoolConfig(min_size=2, max_size=10)                  │
│   dsn = "host=localhost port=5432 dbname=myapp ..."             │
│                                                                 │
│   def factory():                                                │
│       return PooledPostgresConnection(dsn=dsn, connection_id=...)│
│                                                                 │
│   pool = ObjectPool(factory=factory, config=config)             │
│                                                                 │
│   with pool.acquire_connection() as conn:                       │
│       ...                                                       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   WITH FACADE (simple and clean):                               │
│   ───────────────────────────────                               │
│                                                                 │
│   pool = PostgresConnectionPool(                                │
│       host="localhost",                                         │
│       database="myapp",                                         │
│       user="postgres",                                          │
│       password="secret"                                         │
│   )                                                             │
│                                                                 │
│   with pool.connection() as conn:                               │
│       ...                                                       │
│                                                                 │
│   BENEFITS:                                                     │
│   ├── User doesn't need to know about factories                 │
│   ├── User doesn't need to know about PooledWrapper             │
│   ├── User doesn't need to build DSN strings                    │
│   ├── Provides transaction() helper                             │
│   └── Cleaner, more intuitive API                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Complete Flow: From User Code to Database

Let me trace a complete request:

```
┌─────────────────────────────────────────────────────────────────┐
│           COMPLETE FLOW: with pool.connection() as conn        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   YOUR CODE:                                                    │
│   with pool.connection() as conn:                               │
│       users = conn.execute("SELECT * FROM users")               │
│       conn.commit()                                             │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   STEP 1: pool.connection() called                              │
│        │                                                        │
│        ▼                                                        │
│   PostgresConnectionPool.connection()                           │
│        │                                                        │
│        ▼                                                        │
│   ObjectPool.acquire_connection()                               │
│        │                                                        │
│        ▼                                                        │
│   ObjectPool.acquire()                                          │
│        │                                                        │
│        ├──► Check: pool closed? NO                              │
│        ├──► _try_get_wrapper()                                  │
│        │         │                                              │
│        │         ├──► Try queue.get_nowait()                    │
│        │         │         │                                    │
│        │         │         └──► Got PooledWrapper!              │
│        │         │                                              │
│        │    ◄────┘                                              │
│        │                                                        │
│        ├──► Check: wrapper.is_expired()? NO                     │
│        ├──► Check: validation_on_acquire? YES                   │
│        │         │                                              │
│        │         └──► wrapper.obj.validate()                    │
│        │                   │                                    │
│        │                   └──► Execute "SELECT 1" ──► OK!      │
│        │                                                        │
│        ├──► wrapper.mark_acquired()                             │
│        ├──► Add to _in_use dict                                 │
│        ├──► Update stats                                        │
│        │                                                        │
│        └──► Return wrapper.obj (PooledPostgresConnection)       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   STEP 2: yield conn (you get the connection)                   │
│        │                                                        │
│        ▼                                                        │
│   YOUR CODE RUNS:                                               │
│   users = conn.execute("SELECT * FROM users")                   │
│        │                                                        │
│        └──► PooledPostgresConnection.execute()                  │
│                  │                                              │
│                  └──► self._conn.cursor().execute(query)        │
│                            │                                    │
│                            └──► psycopg2 sends to PostgreSQL    │
│                                                                 │
│   conn.commit()                                                 │
│        │                                                        │
│        └──► self._conn.commit()                                 │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   STEP 3: Exit 'with' block (finally runs)                      │
│        │                                                        │
│        ▼                                                        │
│   ObjectPool.release(conn)                                      │
│        │                                                        │
│        ├──► Find wrapper in _in_use                             │
│        ├──► Remove from _in_use                                 │
│        ├──► conn.reset()                                        │
│        │         │                                              │
│        │         ├──► rollback() (if needed)                    │
│        │         └──► DISCARD ALL                               │
│        │                                                        │
│        ├──► wrapper.mark_released()                             │
│        ├──► Put wrapper back in _available queue                │
│        └──► Update stats                                        │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   DONE! Connection is back in pool, ready for next user.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary: Function Purpose Map

| Layer | Class/Function | Purpose |
|-------|----------------|---------|
| **1** | `Poolable` | Contract: "You must have reset/validate/dispose" |
| **1** | `PoolConfig` | Settings: min/max size, timeouts |
| **1** | `PoolStatistics` | Counters for monitoring |
| **1** | `PoolExhaustedError` | Error when no connections available |
| **2** | `PooledPostgresConnection` | Wraps psycopg2, implements Poolable |
| **2** | `._connect()` | Establish actual database connection |
| **2** | `.reset()` | Clean connection for next user |
| **2** | `.validate()` | Check if connection is alive |
| **2** | `.dispose()` | Close connection permanently |
| **2** | `.execute()` | Run SQL query |
| **3** | `PooledWrapper` | Holds object + metadata (age, use count) |
| **3** | `ObjectPool` | Core pool logic (acquire/release) |
| **3** | `._initialize_pool()` | Pre-create min_size connections |
| **3** | `.acquire()` | Get connection from pool |
| **3** | `._try_get_wrapper()` | Try queue, create new, or wait |
| **3** | `.release()` | Return connection to pool |
| **3** | `.acquire_connection()` | Context manager (safe acquire/release) |
| **3** | `.close()` | Shutdown pool, dispose all |
| **4** | `PostgresConnectionPool` | User-friendly facade |
| **4** | `.connection()` | Get connection (context manager) |
| **4** | `.transaction()` | Auto commit/rollback |
