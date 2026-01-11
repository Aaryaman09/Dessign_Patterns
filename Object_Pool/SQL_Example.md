## 1. What Is the Object Pool Pattern?

The Object Pool is a **creational design pattern** that manages a collection of reusable objects. Instead of creating and destroying objects on demand, the pool:

1. **Pre-creates** objects (or creates them lazily up to a limit)
2. **Lends** objects to clients when requested
3. **Reclaims** objects when clients are done
4. **Recycles** them for future requests

### The Core Metaphor

Think of a library. Books (objects) are expensive to produce. Instead of printing a new book for each reader and burning it when they finish, the library:

- Maintains a collection of books
- Lends them to patrons
- Takes them back when returned
- Lends the same book to the next patron

```
┌─────────────────────────────────────────────────────────────────┐
│                        OBJECT POOL LIFECYCLE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐    acquire()    ┌──────────┐                     │
│   │          │ ──────────────► │          │                     │
│   │   POOL   │                 │  CLIENT  │                     │
│   │          │ ◄────────────── │          │                     │
│   └──────────┘    release()    └──────────┘                     │
│        │                                                        │
│        │ contains                                               │
│        ▼                                                        │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│   │  Obj 1   │  │  Obj 2   │  │  Obj 3   │  ... (reusable)      │
│   └──────────┘  └──────────┘  └──────────┘                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Why Does This Pattern Exist?

### The Problem: Expensive Object Lifecycle

Some objects have **costly creation and/or destruction**:

| Cost Type                      | Examples                           | Typical Overhead |
| ------------------------------ | ---------------------------------- | ---------------- |
| **Network I/O**          | Database connections, HTTP clients | 10-100ms         |
| **System calls**         | Thread creation, file handles      | 1-10ms           |
| **Memory allocation**    | Large buffers, image objects       | 0.1-10ms         |
| **Initialization logic** | ML model loading, config parsing   | 100ms-10s        |
| **Authentication**       | OAuth tokens, TLS handshakes       | 50-500ms         |

### The Cost Breakdown: Database Connection Example

```
Creating a PostgreSQL connection involves:

1. DNS resolution                    ~1-50ms
2. TCP handshake (SYN/SYN-ACK/ACK)   ~1-100ms (depends on latency)
3. TLS handshake (if SSL enabled)   ~10-100ms
4. PostgreSQL protocol negotiation   ~5-20ms
5. Authentication                    ~5-50ms
6. Connection state initialization   ~1-5ms
                                     ─────────
                            TOTAL:   ~23-325ms per connection
```

If your application handles 1000 requests/second and each needs a database query:

- **Without pooling**: 1000 × 100ms = 100 seconds of connection overhead/second (impossible)
- **With pooling**: Amortized across reuse, effectively ~0.1ms per request

---

## 3. When to Use Object Pool

### ✅ Ideal Use Cases

| Use Case                                | Why Pooling Helps                                           |
| --------------------------------------- | ----------------------------------------------------------- |
| **Database connections**          | TCP + auth overhead is 10-100ms; queries are often <10ms    |
| **HTTP client sessions**          | Connection reuse via keep-alive; TLS session resumption     |
| **Thread pools**                  | OS thread creation is expensive; context switching overhead |
| **Socket connections**            | Network I/O establishment cost                              |
| **Large byte buffers**            | Avoid repeated allocation/GC pressure for large arrays      |
| **Graphics/GPU resources**        | Texture, shader, framebuffer allocation is slow             |
| **Expensive computation results** | Pre-computed objects (compiled regex, parsed schemas)       |
| **External service clients**      | gRPC channels, message queue connections                    |

### ❌ When NOT to Use Object Pool

| Scenario                                  | Why Pooling Is Wrong                       |
| ----------------------------------------- | ------------------------------------------ |
| **Cheap object creation**           | Pool overhead exceeds creation cost        |
| **Highly stateful objects**         | Reset complexity introduces bugs           |
| **Unbounded concurrency needs**     | Pool becomes a bottleneck                  |
| **Short-lived applications**        | Pool initialization overhead not amortized |
| **Objects with identity semantics** | Each use requires a unique instance        |
| **Memory-constrained environments** | Pool holds memory even when idle           |

### Decision Framework

```
                    ┌─────────────────────────────┐
                    │ Is object creation > 1ms?  │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────┴───────────────┐
                    │                             │
                   YES                            NO
                    │                             │
                    ▼                             ▼
        ┌───────────────────────┐      ┌─────────────────────┐
        │ Is the object reused │      │ Don't use pool.     │
        │ frequently (>10/sec)?│      │ Create on demand.   │
        └───────────┬───────────┘      └─────────────────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
         YES                  NO
          │                   │
          ▼                   ▼
   ┌─────────────┐    ┌──────────────────────┐
   │ Can object  │    │ Consider lazy init   │
   │ state be    │    │ or singleton instead │
   │ fully reset?│    └──────────────────────┘
   └──────┬──────┘
          │
    ┌─────┴─────┐
    │           │
   YES          NO
    │           │
    ▼           ▼
┌────────┐  ┌─────────────────────────────┐
│ USE    │  │ Reconsider design.          │
│ POOL   │  │ Stateful objects in pools   │
└────────┘  │ cause subtle bugs.          │
            └─────────────────────────────┘
```

---

## 4. Real-World Example: PostgreSQL Connection Pool

Here is a production-grade implementation using `psycopg2`:

```python
from __future__ import annotations

import threading
import time
import logging
from abc import ABC, abstractmethod
from contextlib import contextmanager
from dataclasses import dataclass, field
from queue import Queue, Empty
from typing import TypeVar, Generic, Callable, Optional, Any
import psycopg2
from psycopg2.extensions import connection as Psycopg2Connection

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


# ============================================================================
# CORE ABSTRACTIONS
# ============================================================================

class Poolable(ABC):
    """
    Contract for objects that can be managed by the pool.
  
    Every poolable object MUST implement these three methods.
    This is non-negotiable for correct pool behavior.
    """
  
    @abstractmethod
    def reset(self) -> None:
        """
        Reset object to a clean state for the next user.
      
        This method is called BEFORE returning the object to the pool.
        It must undo any state changes made during use.
        """
        pass
  
    @abstractmethod
    def validate(self) -> bool:
        """
        Check if the object is still healthy and usable.
      
        This method is called BEFORE dispensing the object to a user.
        Return False if the object should be discarded.
        """
        pass
  
    @abstractmethod
    def dispose(self) -> None:
        """
        Permanently release all resources held by this object.
      
        Called when the object is removed from the pool forever.
        After this call, the object must not be used.
        """
        pass


@dataclass
class PoolConfig:
    """
    Pool configuration with sensible defaults.
  
    These values should be tuned based on:
    - Expected concurrent load
    - Database server capacity
    - Network latency characteristics
    """
    min_size: int = 2          # Minimum connections to maintain
    max_size: int = 10         # Maximum connections allowed
    acquire_timeout: float = 30.0    # Seconds to wait for a connection
    idle_timeout: float = 300.0      # Seconds before idle connections are pruned
    max_lifetime: float = 3600.0     # Maximum age of a connection (prevents stale)
    validation_on_acquire: bool = True   # Validate before dispensing
    validation_query: str = "SELECT 1"   # Query used for validation
  
    def __post_init__(self):
        if self.min_size < 0:
            raise ValueError("min_size cannot be negative")
        if self.max_size < self.min_size:
            raise ValueError("max_size must be >= min_size")
        if self.max_size < 1:
            raise ValueError("max_size must be at least 1")


@dataclass
class PoolStatistics:
    """Runtime statistics for monitoring and debugging."""
    total_connections_created: int = 0
    total_connections_destroyed: int = 0
    total_acquisitions: int = 0
    total_releases: int = 0
    total_validation_failures: int = 0
    total_timeouts: int = 0
    current_pool_size: int = 0
    current_in_use: int = 0
    current_available: int = 0


class PoolExhaustedError(Exception):
    """Raised when a connection cannot be acquired within the timeout."""
    pass


class PoolClosedError(Exception):
    """Raised when operations are attempted on a closed pool."""
    pass


# ============================================================================
# POSTGRESQL CONNECTION WRAPPER
# ============================================================================

class PooledPostgresConnection(Poolable):
    """
    Wrapper around psycopg2 connection implementing Poolable interface.
  
    This class is responsible for:
    1. Managing the underlying psycopg2 connection
    2. Implementing proper reset semantics
    3. Providing health validation
    4. Tracking connection metadata
    """
  
    def __init__(
        self,
        dsn: str,
        connection_id: int,
        validation_query: str = "SELECT 1"
    ):
        self._dsn = dsn
        self._connection_id = connection_id
        self._validation_query = validation_query
        self._created_at = time.time()
        self._conn: Optional[Psycopg2Connection] = None
      
        # Establish the actual connection
        self._connect()
  
    def _connect(self) -> None:
        """Establish the database connection."""
        logger.info(f"Connection {self._connection_id}: Establishing connection...")
        start = time.time()
      
        self._conn = psycopg2.connect(self._dsn)
        # Set autocommit to False by default for explicit transaction control
        self._conn.autocommit = False
      
        elapsed = time.time() - start
        logger.info(
            f"Connection {self._connection_id}: Connected in {elapsed:.3f}s"
        )
  
    @property
    def connection(self) -> Psycopg2Connection:
        """Access the underlying psycopg2 connection."""
        if self._conn is None:
            raise RuntimeError("Connection has been disposed")
        return self._conn
  
    @property
    def age(self) -> float:
        """Age of this connection in seconds."""
        return time.time() - self._created_at
  
    @property
    def connection_id(self) -> int:
        return self._connection_id
  
    # === Query execution helpers ===
  
    def execute(
        self, 
        query: str, 
        params: Optional[tuple] = None
    ) -> list[dict[str, Any]]:
        """Execute a query and return results as list of dicts."""
        with self.connection.cursor() as cursor:
            cursor.execute(query, params)
            if cursor.description:
                columns = [desc[0] for desc in cursor.description]
                return [dict(zip(columns, row)) for row in cursor.fetchall()]
            return []
  
    def execute_many(
        self, 
        query: str, 
        params_list: list[tuple]
    ) -> int:
        """Execute a query with multiple parameter sets."""
        with self.connection.cursor() as cursor:
            cursor.executemany(query, params_list)
            return cursor.rowcount
  
    def commit(self) -> None:
        """Commit the current transaction."""
        self.connection.commit()
  
    def rollback(self) -> None:
        """Rollback the current transaction."""
        self.connection.rollback()
  
    # === Poolable interface implementation ===
  
    def reset(self) -> None:
        """
        Reset connection state for the next user.
      
        CRITICAL: This method must handle all possible states the connection
        could be in after use. Failure to reset properly causes:
      
        1. Transaction leakage - uncommitted changes visible to next user
        2. Session state leakage - SET commands affecting next user
        3. Cursor leakage - open cursors consuming server resources
        4. Lock leakage - held locks blocking other connections
        """
        if self._conn is None:
            return
      
        try:
            # 1. Rollback any uncommitted transaction
            #    This is the MOST important step
            if self._conn.status != psycopg2.extensions.STATUS_READY:
                logger.warning(
                    f"Connection {self._connection_id}: "
                    "Rolling back uncommitted transaction during reset"
                )
                self._conn.rollback()
          
            # 2. Reset session-level settings
            #    This handles SET commands like timezone, search_path, etc.
            with self._conn.cursor() as cursor:
                cursor.execute("DISCARD ALL")
            self._conn.commit()
          
            logger.debug(f"Connection {self._connection_id}: Reset complete")
          
        except Exception as e:
            logger.error(
                f"Connection {self._connection_id}: Reset failed: {e}"
            )
            # If reset fails, mark connection as unusable
            raise
  
    def validate(self) -> bool:
        """
        Verify the connection is still alive and healthy.
      
        Validation catches:
        1. Network disconnections
        2. Server restarts
        3. Idle connection timeouts (server-side)
        4. Connection corruption
        """
        if self._conn is None:
            return False
      
        if self._conn.closed:
            logger.warning(
                f"Connection {self._connection_id}: Already closed"
            )
            return False
      
        try:
            with self._conn.cursor() as cursor:
                cursor.execute(self._validation_query)
                cursor.fetchone()
            return True
        except Exception as e:
            logger.warning(
                f"Connection {self._connection_id}: Validation failed: {e}"
            )
            return False
  
    def dispose(self) -> None:
        """
        Permanently close the connection and release resources.
      
        After this call, the connection object must not be used.
        """
        if self._conn is not None:
            try:
                if not self._conn.closed:
                    self._conn.close()
                logger.info(
                    f"Connection {self._connection_id}: Disposed "
                    f"(lived {self.age:.1f}s)"
                )
            except Exception as e:
                logger.error(
                    f"Connection {self._connection_id}: Error during dispose: {e}"
                )
            finally:
                self._conn = None


# ============================================================================
# GENERIC OBJECT POOL
# ============================================================================

T = TypeVar('T', bound=Poolable)


@dataclass
class PooledWrapper(Generic[T]):
    """
    Internal wrapper tracking metadata for pooled objects.
  
    Separating metadata from the pooled object itself keeps concerns clean:
    - The pooled object focuses on its domain (e.g., database operations)
    - The wrapper handles pool-specific tracking
    """
    obj: T
    created_at: float = field(default_factory=time.time)
    last_used_at: float = field(default_factory=time.time)
    last_validated_at: float = field(default_factory=time.time)
    use_count: int = 0
  
    def mark_acquired(self) -> None:
        self.last_used_at = time.time()
        self.use_count += 1
  
    def mark_released(self) -> None:
        self.last_used_at = time.time()
  
    def mark_validated(self) -> None:
        self.last_validated_at = time.time()
  
    def is_expired(self, max_lifetime: float) -> bool:
        """Check if connection has exceeded maximum lifetime."""
        return (time.time() - self.created_at) > max_lifetime
  
    def is_idle_too_long(self, idle_timeout: float) -> bool:
        """Check if connection has been idle too long."""
        return (time.time() - self.last_used_at) > idle_timeout


class ObjectPool(Generic[T]):
    """
    Thread-safe, production-grade object pool.
  
    Key features:
    - Thread-safe acquisition and release
    - Configurable min/max pool size
    - Connection validation before use
    - Automatic disposal of stale connections
    - Comprehensive statistics tracking
    - Context manager support for automatic release
  
    Design decisions:
    - Uses Queue for thread-safe FIFO access (oldest connections used first)
    - Separate tracking of in-use objects for monitoring
    - RLock for reentrant locking in nested operations
    - Lazy creation up to max_size to handle burst load
    """
  
    def __init__(
        self,
        factory: Callable[[], T],
        config: Optional[PoolConfig] = None
    ):
        self._factory = factory
        self._config = config or PoolConfig()
      
        # Thread-safe containers
        self._available: Queue[PooledWrapper[T]] = Queue()
        self._in_use: dict[int, PooledWrapper[T]] = {}
        self._lock = threading.RLock()
      
        # Statistics
        self._stats = PoolStatistics()
      
        # State
        self._closed = False
        self._connection_counter = 0
      
        # Initialize pool with minimum connections
        self._initialize_pool()
  
    def _initialize_pool(self) -> None:
        """Pre-create minimum number of connections."""
        logger.info(
            f"Initializing pool with {self._config.min_size} connections"
        )
        for _ in range(self._config.min_size):
            try:
                wrapper = self._create_new_wrapper()
                self._available.put_nowait(wrapper)
            except Exception as e:
                logger.error(f"Failed to create initial connection: {e}")
                raise
      
        self._update_stats()
  
    def _create_new_wrapper(self) -> PooledWrapper[T]:
        """Create a new wrapped poolable object."""
        self._connection_counter += 1
        obj = self._factory()
        self._stats.total_connections_created += 1
      
        logger.debug(
            f"Created connection #{self._connection_counter}. "
            f"Total created: {self._stats.total_connections_created}"
        )
      
        return PooledWrapper(obj=obj)
  
    def _update_stats(self) -> None:
        """Update current pool statistics."""
        with self._lock:
            self._stats.current_in_use = len(self._in_use)
            self._stats.current_available = self._available.qsize()
            self._stats.current_pool_size = (
                self._stats.current_in_use + self._stats.current_available
            )
  
    def _current_total_size(self) -> int:
        """Get total number of connections (available + in use)."""
        with self._lock:
            return self._available.qsize() + len(self._in_use)
  
    def _should_create_new(self) -> bool:
        """Determine if we should create a new connection."""
        return self._current_total_size() < self._config.max_size
  
    def acquire(self, timeout: Optional[float] = None) -> T:
        """
        Acquire an object from the pool.
      
        Acquisition strategy:
        1. Try to get an available object from the queue
        2. Validate the object before returning
        3. If no objects available and under max_size, create new
        4. If at max_size, wait for a release
        5. Timeout if waiting too long
      
        Args:
            timeout: Maximum seconds to wait. Uses config default if None.
      
        Returns:
            A pooled object ready for use.
      
        Raises:
            PoolExhaustedError: If no object available within timeout.
            PoolClosedError: If pool has been closed.
        """
        if self._closed:
            raise PoolClosedError("Cannot acquire from a closed pool")
      
        timeout = timeout if timeout is not None else self._config.acquire_timeout
        deadline = time.time() + timeout
      
        while True:
            remaining = deadline - time.time()
            if remaining <= 0:
                self._stats.total_timeouts += 1
                self._update_stats()
                raise PoolExhaustedError(
                    f"Could not acquire connection within {timeout:.1f}s. "
                    f"Pool size: {self._current_total_size()}, "
                    f"In use: {len(self._in_use)}, "
                    f"Available: {self._available.qsize()}"
                )
          
            wrapper = self._try_get_wrapper(remaining)
          
            if wrapper is None:
                continue
          
            # Check if connection is too old
            if wrapper.is_expired(self._config.max_lifetime):
                logger.info(
                    f"Connection expired (age: {wrapper.obj.age:.1f}s), disposing"
                )
                self._dispose_wrapper(wrapper)
                continue
          
            # Validate if configured
            if self._config.validation_on_acquire:
                if not wrapper.obj.validate():
                    self._stats.total_validation_failures += 1
                    logger.warning("Connection validation failed, disposing")
                    self._dispose_wrapper(wrapper)
                    continue
                wrapper.mark_validated()
          
            # Success - mark as in use and return
            wrapper.mark_acquired()
            with self._lock:
                self._in_use[id(wrapper)] = wrapper
          
            self._stats.total_acquisitions += 1
            self._update_stats()
          
            logger.debug(
                f"Acquired connection. In use: {self._stats.current_in_use}, "
                f"Available: {self._stats.current_available}"
            )
          
            return wrapper.obj
  
    def _try_get_wrapper(self, timeout: float) -> Optional[PooledWrapper[T]]:
        """
        Attempt to get a wrapper from the queue or create new.
      
        This method encapsulates the acquisition logic:
        1. Non-blocking try from queue
        2. Create new if under limit
        3. Blocking wait with timeout
        """
        # First, try non-blocking get
        try:
            return self._available.get_nowait()
        except Empty:
            pass
      
        # Check if we can create a new connection
        if self._should_create_new():
            with self._lock:
                # Double-check after acquiring lock (prevent race)
                if self._should_create_new():
                    try:
                        return self._create_new_wrapper()
                    except Exception as e:
                        logger.error(f"Failed to create new connection: {e}")
                        # Fall through to wait for existing connection
      
        # Wait for a connection to be released
        try:
            return self._available.get(timeout=min(timeout, 1.0))
        except Empty:
            return None
  
    def _dispose_wrapper(self, wrapper: PooledWrapper[T]) -> None:
        """Dispose a wrapper and its underlying object."""
        try:
            wrapper.obj.dispose()
        except Exception as e:
            logger.error(f"Error disposing connection: {e}")
        finally:
            self._stats.total_connections_destroyed += 1
  
    def release(self, obj: T) -> None:
        """
        Return an object to the pool.
      
        Release process:
        1. Find the wrapper for this object
        2. Reset the object state
        3. Return to available queue
      
        If reset fails, the object is disposed instead of returned.
      
        Args:
            obj: The pooled object to release.
        """
        if self._closed:
            # Pool is closed, just dispose
            try:
                obj.dispose()
            except Exception:
                pass
            return
      
        # Find the wrapper
        wrapper: Optional[PooledWrapper[T]] = None
        with self._lock:
            for wrapper_id, w in list(self._in_use.items()):
                if w.obj is obj:
                    wrapper = w
                    del self._in_use[wrapper_id]
                    break
      
        if wrapper is None:
            logger.warning(
                "Attempted to release object not belonging to this pool"
            )
            return
      
        # Reset and return to pool
        try:
            obj.reset()
            wrapper.mark_released()
            self._available.put_nowait(wrapper)
          
            self._stats.total_releases += 1
            self._update_stats()
          
            logger.debug(
                f"Released connection. In use: {self._stats.current_in_use}, "
                f"Available: {self._stats.current_available}"
            )
          
        except Exception as e:
            logger.error(f"Reset failed, disposing connection: {e}")
            self._dispose_wrapper(wrapper)
  
    @contextmanager
    def acquire_connection(self, timeout: Optional[float] = None):
        """
        Context manager for automatic acquire/release.
      
        THIS IS THE PREFERRED WAY TO USE THE POOL.
      
        Guarantees release even if an exception occurs during use.
      
        Usage:
            with pool.acquire_connection() as conn:
                conn.execute("SELECT * FROM users")
                conn.commit()
            # Connection automatically released here
        """
        obj = self.acquire(timeout=timeout)
        try:
            yield obj
        finally:
            self.release(obj)
  
    def close(self) -> None:
        """
        Close the pool and dispose all connections.
      
        After closing:
        - No new acquisitions are allowed
        - In-use connections will be disposed when released
        - All available connections are immediately disposed
        """
        logger.info("Closing connection pool...")
        self._closed = True
      
        # Dispose all available connections
        disposed_count = 0
        while True:
            try:
                wrapper = self._available.get_nowait()
                self._dispose_wrapper(wrapper)
                disposed_count += 1
            except Empty:
                break
      
        # Mark in-use connections for disposal on release
        with self._lock:
            in_use_count = len(self._in_use)
      
        logger.info(
            f"Pool closed. Disposed {disposed_count} available connections. "
            f"{in_use_count} connections still in use (will be disposed on release)."
        )
      
        self._update_stats()
  
    @property
    def stats(self) -> PoolStatistics:
        """Get current pool statistics."""
        self._update_stats()
        return self._stats
  
    @property
    def is_closed(self) -> bool:
        return self._closed
  
    def __enter__(self):
        return self
  
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False


# ============================================================================
# POSTGRESQL POOL FACTORY
# ============================================================================

class PostgresConnectionPool:
    """
    High-level PostgreSQL connection pool.
  
    This is a facade that simplifies pool creation and usage
    for PostgreSQL specifically.
    """
  
    def __init__(
        self,
        dsn: Optional[str] = None,
        host: str = "localhost",
        port: int = 5432,
        database: str = "postgres",
        user: str = "postgres",
        password: str = "",
        config: Optional[PoolConfig] = None
    ):
        # Build DSN if not provided
        if dsn is None:
            dsn = (
                f"host={host} port={port} dbname={database} "
                f"user={user} password={password}"
            )
      
        self._dsn = dsn
        self._config = config or PoolConfig()
        self._connection_counter = 0
        self._lock = threading.Lock()
      
        # Create the underlying generic pool
        self._pool: ObjectPool[PooledPostgresConnection] = ObjectPool(
            factory=self._create_connection,
            config=self._config
        )
  
    def _create_connection(self) -> PooledPostgresConnection:
        """Factory method for creating pooled connections."""
        with self._lock:
            self._connection_counter += 1
            conn_id = self._connection_counter
      
        return PooledPostgresConnection(
            dsn=self._dsn,
            connection_id=conn_id,
            validation_query=self._config.validation_query
        )
  
    @contextmanager
    def connection(self, timeout: Optional[float] = None):
        """
        Get a connection from the pool.
      
        Usage:
            with pool.connection() as conn:
                results = conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))
                conn.commit()
        """
        with self._pool.acquire_connection(timeout=timeout) as conn:
            yield conn
  
    @contextmanager
    def transaction(self, timeout: Optional[float] = None):
        """
        Get a connection with automatic commit/rollback.
      
        Commits on success, rolls back on exception.
      
        Usage:
            with pool.transaction() as conn:
                conn.execute("INSERT INTO users (name) VALUES (%s)", ("Alice",))
                conn.execute("INSERT INTO logs (action) VALUES (%s)", ("user_created",))
            # Automatically committed if no exception
        """
        with self._pool.acquire_connection(timeout=timeout) as conn:
            try:
                yield conn
                conn.commit()
            except Exception:
                conn.rollback()
                raise
  
    @property
    def stats(self) -> PoolStatistics:
        return self._pool.stats
  
    def close(self) -> None:
        self._pool.close()
  
    def __enter__(self):
        return self
  
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False


# ============================================================================
# USAGE EXAMPLES
# ============================================================================

def example_basic_usage():
    """Basic pool usage demonstration."""
  
    config = PoolConfig(
        min_size=2,
        max_size=10,
        acquire_timeout=30.0,
        validation_on_acquire=True
    )
  
    # Using context manager ensures pool is closed properly
    with PostgresConnectionPool(
        host="localhost",
        port=5432,
        database="myapp",
        user="appuser",
        password="secret",
        config=config
    ) as pool:
      
        # Simple query
        with pool.connection() as conn:
            users = conn.execute("SELECT id, name FROM users LIMIT 10")
            for user in users:
                print(f"User: {user['name']}")
            conn.commit()
      
        # Transaction with automatic commit/rollback
        with pool.transaction() as conn:
            conn.execute(
                "INSERT INTO users (name, email) VALUES (%s, %s)",
                ("Alice", "alice@example.com")
            )
            conn.execute(
                "INSERT INTO audit_log (action) VALUES (%s)",
                ("user_created",)
            )
        # Committed automatically
      
        # Check stats
        stats = pool.stats
        print(f"Total acquisitions: {stats.total_acquisitions}")
        print(f"Current pool size: {stats.current_pool_size}")


def example_concurrent_usage():
    """Demonstrate pool behavior under concurrent load."""
    import concurrent.futures
  
    config = PoolConfig(min_size=2, max_size=5)
  
    with PostgresConnectionPool(
        host="localhost",
        database="myapp",
        user="appuser",
        password="secret",
        config=config
    ) as pool:
      
        def worker(worker_id: int) -> str:
            with pool.connection() as conn:
                # Simulate some work
                result = conn.execute("SELECT pg_sleep(0.1), %s as worker", (worker_id,))
                conn.commit()
                return f"Worker {worker_id} completed"
      
        # Run 20 concurrent tasks with only 5 max connections
        with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
            futures = [executor.submit(worker, i) for i in range(20)]
            for future in concurrent.futures.as_completed(futures):
                print(future.result())
      
        print(f"\nFinal stats:")
        print(f"  Total connections created: {pool.stats.total_connections_created}")
        print(f"  Total acquisitions: {pool.stats.total_acquisitions}")
```

---

## 5. Critical Implementation Considerations

### 5.1 Thread Safety

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREAD SAFETY REQUIREMENTS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MUST BE THREAD-SAFE:                                           │
│  ├── acquire() - multiple threads requesting simultaneously     │
│  ├── release() - multiple threads returning simultaneously      │
│  ├── Pool size tracking - prevent over-creation                 │
│  └── Statistics updates - accurate counts                       │
│                                                                 │
│  STRATEGIES:                                                    │
│  ├── Queue (built-in thread-safe) for available objects         │
│  ├── Lock/RLock for shared mutable state                        │
│  └── Atomic operations where possible                           │
│                                                                 │
│  COMMON MISTAKE:                                                │
│  ├── Check-then-act without locking                             │
│  │   if pool.size < max:     # Thread A checks                  │
│  │       create_new()        # Thread B also checks, both create│
│  │                                                              │
│  CORRECT:                                                       │
│  └── with lock:                                                 │
│          if pool.size < max:                                    │
│              create_new()                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 The Reset Contract

This is the **most critical** aspect of object pooling:

```python
# WHAT RESET MUST DO:

class PooledConnection(Poolable):
    def reset(self) -> None:
        """
        Reset checklist for database connections:
      
        1. TRANSACTIONS
           - Rollback any uncommitted transaction
           - Clear savepoints
      
        2. SESSION STATE  
           - Reset search_path
           - Reset timezone
           - Reset role/user context
           - Clear session variables
           - Reset statement timeout
      
        3. CURSORS
           - Close all open cursors
           - Clear prepared statements (if not desired to keep)
      
        4. LOCKS
           - Release advisory locks
           - (Regular locks released by rollback)
      
        5. NOTIFICATIONS
           - Clear pending notifications (LISTEN/NOTIFY)
      
        PostgreSQL's DISCARD ALL does most of this:
        """
        self._conn.rollback()  # Always first!
        with self._conn.cursor() as cur:
            cur.execute("DISCARD ALL")
        self._conn.commit()
```

### 5.3 Validation Strategy

```python
# VALIDATION APPROACHES:

# 1. ALWAYS VALIDATE (safest, slowest)
config = PoolConfig(validation_on_acquire=True)
# Pro: Never get a dead connection
# Con: Extra round-trip on every acquire

# 2. VALIDATE ON IDLE TIMEOUT (balanced)
def acquire(self):
    wrapper = self._get_wrapper()
    idle_time = time.time() - wrapper.last_used_at
    if idle_time > 30:  # Validate if idle > 30s
        if not wrapper.obj.validate():
            self._dispose(wrapper)
            return self.acquire()  # Retry
    return wrapper.obj

# 3. BACKGROUND VALIDATION (complex, efficient)
def _background_validator(self):
    """Separate thread that periodically validates idle connections."""
    while not self._closed:
        time.sleep(30)
        self._validate_idle_connections()

# 4. LAZY VALIDATION (validate on first use failure)
# Let the actual operation fail, then handle reconnection
# Pro: No validation overhead
# Con: First operation after disconnect fails
```

### 5.4 Size Tuning

```
┌─────────────────────────────────────────────────────────────────┐
│                     POOL SIZING GUIDELINES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FORMULA (starting point):                                      │
│                                                                 │
│    max_pool_size = (core_count * 2) + effective_spindle_count   │
│                                                                 │
│  For SSD (no spindles): max_pool_size ≈ core_count * 2          │
│                                                                 │
│  FACTORS TO CONSIDER:                                           │
│  ├── Query duration (longer queries = more connections needed)  │
│  ├── Concurrent requests (match your web server workers)        │
│  ├── Database server limits (max_connections setting)           │
│  ├── Memory per connection (~5-10MB for PostgreSQL)             │
│  └── Other applications sharing the database                    │
│                                                                 │
│  ANTI-PATTERN:                                                  │
│  ├── Setting max_size = 100 "just in case"                      │
│  │   → Wastes memory                                            │
│  │   → Can overwhelm database                                   │
│  │   → Hides actual bottlenecks                                 │
│                                                                 │
│  MONITORING:                                                    │
│  ├── Track wait time for connections                            │
│  ├── Track pool exhaustion events                               │
│  └── Adjust based on actual usage patterns                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Common Mistakes and How to Avoid Them

### Mistake 1: Forgetting to Release

```python
# ❌ WRONG: Manual acquire without guaranteed release
def get_user(user_id):
    conn = pool.acquire()
    result = conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    pool.release(conn)  # Never reached if execute() throws!
    return result

# ✅ CORRECT: Use context manager
def get_user(user_id):
    with pool.connection() as conn:
        result = conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        conn.commit()
        return result
    # Automatically released, even on exception
```

### Mistake 2: Connection Leakage via References

```python
# ❌ WRONG: Storing connection reference beyond scope
class UserRepository:
    def __init__(self, pool):
        self.pool = pool
        self.conn = None  # Danger!
  
    def connect(self):
        self.conn = self.pool.acquire()  # Leaked reference
  
    def get_user(self, user_id):
        return self.conn.execute(...)  # Using leaked connection

# ✅ CORRECT: Acquire per operation
class UserRepository:
    def __init__(self, pool):
        self.pool = pool
  
    def get_user(self, user_id):
        with self.pool.connection() as conn:
            return conn.execute(...)
```

### Mistake 3: Inadequate Reset

```python
# ❌ WRONG: Incomplete reset
def reset(self) -> None:
    pass  # "It's fine, the next query will just work"

# CONSEQUENCE:
# User A: SET ROLE admin;
# User A: releases connection
# User B: acquires same connection
# User B: DELETE FROM users;  # Executes as admin!

# ✅ CORRECT: Comprehensive reset
def reset(self) -> None:
    self._conn.rollback()
    with self._conn.cursor() as cur:
        cur.execute("DISCARD ALL")
    self._conn.commit()
```

### Mistake 4: Ignoring Connection Lifetime

```python
# ❌ WRONG: Connections live forever
class NaivePool:
    def acquire(self):
        return self._available.get()  # No age check

# PROBLEMS:
# - Connections can go stale (server timeout, network issues)
# - Memory leaks in long-running connections
# - Load balancer changes not picked up

# ✅ CORRECT: Enforce maximum lifetime
def acquire(self):
    wrapper = self._available.get()
    if wrapper.age > self.max_lifetime:
        self._dispose(wrapper)
        return self.acquire()  # Get fresh connection
    return wrapper.obj
```

### Mistake 5: Pool Exhaustion Without Feedback

```python
# ❌ WRONG: Block forever
def acquire(self):
    return self._available.get()  # Blocks indefinitely

# ✅ CORRECT: Timeout with clear error
def acquire(self, timeout=30.0):
    try:
        return self._available.get(timeout=timeout)
    except Empty:
        raise PoolExhaustedError(
            f"No connection available within {timeout}s. "
            f"Consider increasing pool size or reducing connection hold time."
        )
```

### Mistake 6: Not Handling Validation Failures

```python
# ❌ WRONG: Assume validation always passes
def acquire(self):
    wrapper = self._available.get()
    wrapper.obj.validate()  # Ignoring return value!
    return wrapper.obj

# ✅ CORRECT: Retry on validation failure
def acquire(self):
    while True:
        wrapper = self._available.get(timeout=remaining_time)
        if wrapper.obj.validate():
            return wrapper.obj
        else:
            self._dispose(wrapper)  # Dispose invalid connection
            # Loop continues, tries next connection
```

### Mistake 7: Sharing Connections Across Threads

```python
# ❌ WRONG: Passing connection between threads
def process_batch(items):
    with pool.connection() as conn:
        # Start threads that all use the same connection
        threads = [
            Thread(target=process_item, args=(conn, item))
            for item in items
        ]
        for t in threads:
            t.start()
        for t in threads:
            t.join()

# PROBLEMS:
# - Most database drivers are not thread-safe per connection
# - Transaction state becomes unpredictable
# - Cursor operations interleave

# ✅ CORRECT: Each thread acquires its own connection
def process_batch(items):
    def process_item(item):
        with pool.connection() as conn:  # Each thread gets own connection
            conn.execute(...)
            conn.commit()
  
    threads = [Thread(target=process_item, args=(item,)) for item in items]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
```

### Mistake 8: Mixing Pool and Non-Pool Connections

```python
# ❌ WRONG: Releasing non-pooled connection to pool
external_conn = psycopg2.connect(dsn)  # Created outside pool
pool.release(external_conn)  # Pool doesn't know about this!

# ✅ CORRECT: Only release what you acquired
with pool.connection() as conn:
    # Use conn
    pass
# Automatic release of pool-managed connection
```

---

## 7. Quick Reference Summary

| Aspect                   | Recommendation                                             |
| ------------------------ | ---------------------------------------------------------- |
| **When to use**    | Object creation > 1ms, frequent reuse, bounded concurrency |
| **When to avoid**  | Cheap objects, complex state, unbounded concurrency        |
| **Thread safety**  | Use Queue + Lock, never check-then-act without locking     |
| **Reset**          | ALWAYS rollback transactions, reset session state          |
| **Validation**     | Validate on acquire, especially after idle periods         |
| **Sizing**         | Start with `2 * CPU cores`, tune based on monitoring     |
| **Acquisition**    | ALWAYS use context managers for automatic release          |
| **Lifetime**       | Enforce max age to prevent stale connections               |
| **Error handling** | Dispose on reset/validation failure, don't return to pool  |
| **Monitoring**     | Track acquisitions, wait times, exhaustion events          |

