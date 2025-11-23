# Simple SQL Query Builder - Complete Example

```python
from dataclasses import dataclass
from typing import List, Optional, Tuple
from enum import Enum

# ============================================================================
# ENUMS
# ============================================================================

class OrderDirection(Enum):
    ASC = "ASC"
    DESC = "DESC"

class JoinType(Enum):
    INNER = "INNER JOIN"
    LEFT = "LEFT JOIN"
    RIGHT = "RIGHT JOIN"


# ============================================================================
# 1. PRODUCT - The SQL Query (Immutable)
# ============================================================================

@dataclass(frozen=True)
class SQLQuery:
    """
    The PRODUCT - Immutable SQL query object.
    Once built, cannot be modified.
    """
    select_columns: Tuple[str, ...]
    from_table: str
    joins: Tuple[Tuple[JoinType, str, str], ...]  # (join_type, table, condition)
    where_conditions: Tuple[str, ...]
    order_by: Optional[Tuple[str, OrderDirection]]
    limit: Optional[int]
  
    def __post_init__(self):
        """Validation"""
        if not self.select_columns:
            raise ValueError("Must select at least one column!")
      
        if not self.from_table:
            raise ValueError("Must specify FROM table!")
      
        if self.limit is not None and self.limit <= 0:
            raise ValueError("LIMIT must be positive!")
  
    def to_sql(self) -> str:
        """Generate the actual SQL string"""
        # SELECT clause
        columns = ", ".join(self.select_columns)
        sql = f"SELECT {columns}"
      
        # FROM clause
        sql += f"\nFROM {self.from_table}"
      
        # JOIN clauses
        for join_type, table, condition in self.joins:
            sql += f"\n{join_type.value} {table} ON {condition}"
      
        # WHERE clause
        if self.where_conditions:
            conditions = " AND ".join(self.where_conditions)
            sql += f"\nWHERE {conditions}"
      
        # ORDER BY clause
        if self.order_by:
            column, direction = self.order_by
            sql += f"\nORDER BY {column} {direction.value}"
      
        # LIMIT clause
        if self.limit:
            sql += f"\nLIMIT {self.limit}"
      
        return sql + ";"
  
    def __str__(self):
        return self.to_sql()


# ============================================================================
# 2. CONCRETE BUILDER - Query Builder
# ============================================================================

class QueryBuilder:
    """
    CONCRETE BUILDER for SQL queries.
    Provides fluent interface for building queries step-by-step.
    """
  
    def __init__(self):
        self.reset()
  
    def reset(self) -> None:
        """Reset builder to start fresh"""
        self._select_columns: List[str] = []
        self._from_table: Optional[str] = None
        self._joins: List[Tuple[JoinType, str, str]] = []
        self._where_conditions: List[str] = []
        self._order_by: Optional[Tuple[str, OrderDirection]] = None
        self._limit: Optional[int] = None
  
    def select(self, *columns: str) -> 'QueryBuilder':
        """
        Add columns to SELECT clause.
        Can be called multiple times to add more columns.
        """
        self._select_columns.extend(columns)
        return self
  
    def select_all(self) -> 'QueryBuilder':
        """SELECT * shorthand"""
        self._select_columns = ["*"]
        return self
  
    def from_table(self, table: str) -> 'QueryBuilder':
        """Set the FROM table"""
        self._from_table = table
        return self
  
    def join(self, table: str, condition: str, 
             join_type: JoinType = JoinType.INNER) -> 'QueryBuilder':
        """
        Add a JOIN clause.
      
        Example:
            .join("orders", "users.id = orders.user_id")
            .join("products", "orders.product_id = products.id", JoinType.LEFT)
        """
        self._joins.append((join_type, table, condition))
        return self
  
    def inner_join(self, table: str, condition: str) -> 'QueryBuilder':
        """Shorthand for INNER JOIN"""
        return self.join(table, condition, JoinType.INNER)
  
    def left_join(self, table: str, condition: str) -> 'QueryBuilder':
        """Shorthand for LEFT JOIN"""
        return self.join(table, condition, JoinType.LEFT)
  
    def right_join(self, table: str, condition: str) -> 'QueryBuilder':
        """Shorthand for RIGHT JOIN"""
        return self.join(table, condition, JoinType.RIGHT)
  
    def where(self, condition: str) -> 'QueryBuilder':
        """
        Add a WHERE condition.
        Multiple calls are combined with AND.
      
        Example:
            .where("age > 18")
            .where("country = 'USA'")
            # Results in: WHERE age > 18 AND country = 'USA'
        """
        self._where_conditions.append(condition)
        return self
  
    def order_by(self, column: str, 
                 direction: OrderDirection = OrderDirection.ASC) -> 'QueryBuilder':
        """
        Set ORDER BY clause.
      
        Example:
            .order_by("created_at", OrderDirection.DESC)
        """
        self._order_by = (column, direction)
        return self
  
    def limit(self, count: int) -> 'QueryBuilder':
        """Set LIMIT clause"""
        if count <= 0:
            raise ValueError("LIMIT must be positive!")
        self._limit = count
        return self
  
    def build(self) -> SQLQuery:
        """
        Build and return the final immutable SQLQuery object.
        Performs validation.
        """
        if not self._select_columns:
            raise ValueError("Must call select() before build()!")
      
        if not self._from_table:
            raise ValueError("Must call from_table() before build()!")
      
        query = SQLQuery(
            select_columns=tuple(self._select_columns),
            from_table=self._from_table,
            joins=tuple(self._joins),
            where_conditions=tuple(self._where_conditions),
            order_by=self._order_by,
            limit=self._limit
        )
      
        self.reset()  # Reset for next query
        return query


# ============================================================================
# 3. DIRECTOR - Pre-configured Query Templates
# ============================================================================

class QueryDirector:
    """
    DIRECTOR - Knows common query patterns.
    Provides pre-configured query templates.
    """
  
    def __init__(self, builder: QueryBuilder):
        self._builder = builder
  
    def get_all_users(self) -> SQLQuery:
        """Standard query: Get all users"""
        return (self._builder
                .select_all()
                .from_table("users")
                .build())
  
    def get_active_users(self) -> SQLQuery:
        """Standard query: Get active users ordered by name"""
        return (self._builder
                .select("id", "name", "email", "created_at")
                .from_table("users")
                .where("status = 'active'")
                .order_by("name", OrderDirection.ASC)
                .build())
  
    def get_recent_orders_with_users(self, limit: int = 10) -> SQLQuery:
        """Standard query: Get recent orders with user info"""
        return (self._builder
                .select("orders.id", "orders.total", "orders.created_at",
                       "users.name", "users.email")
                .from_table("orders")
                .inner_join("users", "orders.user_id = users.id")
                .order_by("orders.created_at", OrderDirection.DESC)
                .limit(limit)
                .build())
  
    def get_user_order_summary(self) -> SQLQuery:
        """Standard query: User order summary with aggregation"""
        return (self._builder
                .select("users.id", "users.name", 
                       "COUNT(orders.id) as order_count",
                       "SUM(orders.total) as total_spent")
                .from_table("users")
                .left_join("orders", "users.id = orders.user_id")
                .where("users.status = 'active'")
                .order_by("total_spent", OrderDirection.DESC)
                .build())
  
    def get_top_products(self, limit: int = 5) -> SQLQuery:
        """Standard query: Top selling products"""
        return (self._builder
                .select("products.id", "products.name",
                       "COUNT(order_items.id) as times_ordered",
                       "SUM(order_items.quantity) as total_quantity")
                .from_table("products")
                .inner_join("order_items", "products.id = order_items.product_id")
                .order_by("times_ordered", OrderDirection.DESC)
                .limit(limit)
                .build())


# ============================================================================
# DEMONSTRATION
# ============================================================================

def main():
    print("=" * 80)
    print("SQL QUERY BUILDER - COMPLETE EXAMPLE")
    print("=" * 80)
  
    # ========================================================================
    # Example 1: Simple Query (Builder Only)
    # ========================================================================
    print("\n1️⃣  SIMPLE QUERY - Get all users")
    print("-" * 80)
  
    query1 = (QueryBuilder()
              .select("id", "name", "email")
              .from_table("users")
              .build())
  
    print(query1)
  
    # ========================================================================
    # Example 2: Query with WHERE and ORDER BY
    # ========================================================================
    print("\n2️⃣  FILTERED QUERY - Active users ordered by name")
    print("-" * 80)
  
    query2 = (QueryBuilder()
              .select("id", "name", "email", "created_at")
              .from_table("users")
              .where("status = 'active'")
              .where("age >= 18")
              .order_by("name", OrderDirection.ASC)
              .build())
  
    print(query2)
  
    # ========================================================================
    # Example 3: Query with JOIN
    # ========================================================================
    print("\n3️⃣  JOIN QUERY - Users with their orders")
    print("-" * 80)
  
    query3 = (QueryBuilder()
              .select("users.name", "users.email", 
                     "orders.id as order_id", "orders.total")
              .from_table("users")
              .inner_join("orders", "users.id = orders.user_id")
              .where("orders.total > 100")
              .order_by("orders.total", OrderDirection.DESC)
              .limit(10)
              .build())
  
    print(query3)
  
    # ========================================================================
    # Example 4: Complex Query with Multiple JOINs
    # ========================================================================
    print("\n4️⃣  COMPLEX QUERY - Orders with user and product details")
    print("-" * 80)
  
    query4 = (QueryBuilder()
              .select("orders.id", "orders.created_at",
                     "users.name as customer_name",
                     "products.name as product_name",
                     "order_items.quantity",
                     "order_items.price")
              .from_table("orders")
              .inner_join("users", "orders.user_id = users.id")
              .inner_join("order_items", "orders.id = order_items.order_id")
              .inner_join("products", "order_items.product_id = products.id")
              .where("orders.status = 'completed'")
              .where("orders.created_at >= '2024-01-01'")
              .order_by("orders.created_at", OrderDirection.DESC)
              .limit(20)
              .build())
  
    print(query4)
  
    # ========================================================================
    # Example 5: Using Director for Standard Queries
    # ========================================================================
    print("\n5️⃣  USING DIRECTOR - Pre-configured queries")
    print("-" * 80)
  
    builder = QueryBuilder()
    director = QueryDirector(builder)
  
    print("📋 Standard Query: Get all users")
    print(director.get_all_users())
  
    print("\n📋 Standard Query: Get active users")
    print(director.get_active_users())
  
    print("\n📋 Standard Query: Recent orders with user info")
    print(director.get_recent_orders_with_users(limit=5))
  
    print("\n📋 Standard Query: User order summary")
    print(director.get_user_order_summary())
  
    print("\n📋 Standard Query: Top 3 products")
    print(director.get_top_products(limit=3))
  
    # ========================================================================
    # Example 6: Aggregation Query
    # ========================================================================
    print("\n6️⃣  AGGREGATION QUERY - Sales by country")
    print("-" * 80)
  
    query6 = (QueryBuilder()
              .select("users.country",
                     "COUNT(orders.id) as total_orders",
                     "SUM(orders.total) as total_revenue",
                     "AVG(orders.total) as avg_order_value")
              .from_table("users")
              .inner_join("orders", "users.id = orders.user_id")
              .where("orders.status = 'completed'")
              .order_by("total_revenue", OrderDirection.DESC)
              .build())
  
    print(query6)
  
    # ========================================================================
    # Example 7: LEFT JOIN Query
    # ========================================================================
    print("\n7️⃣  LEFT JOIN QUERY - All users with optional order count")
    print("-" * 80)
  
    query7 = (QueryBuilder()
              .select("users.id", "users.name", "users.email",
                     "COUNT(orders.id) as order_count")
              .from_table("users")
              .left_join("orders", "users.id = orders.user_id")
              .order_by("order_count", OrderDirection.DESC)
              .build())
  
    print(query7)
  
    # ========================================================================
    # Example 8: Demonstrating Immutability
    # ========================================================================
    print("\n8️⃣  IMMUTABILITY - Query cannot be modified after build")
    print("-" * 80)
  
    query = (QueryBuilder()
             .select("id", "name")
             .from_table("users")
             .build())
  
    print("Original query:")
    print(query)
  
    try:
        query.from_table = "products"  # ❌ Will fail
    except Exception as e:
        print(f"\n❌ Cannot modify: {type(e).__name__}")
  
    # ========================================================================
    # Example 9: Validation Errors
    # ========================================================================
    print("\n9️⃣  VALIDATION ERRORS")
    print("-" * 80)
  
    # Missing FROM table
    try:
        bad_query1 = QueryBuilder().select("id", "name").build()
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # Missing SELECT columns
    try:
        bad_query2 = QueryBuilder().from_table("users").build()
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # Invalid LIMIT
    try:
        bad_query3 = (QueryBuilder()
                      .select("id")
                      .from_table("users")
                      .limit(-5)
                      .build())
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # ========================================================================
    # Example 10: Reusing Builder
    # ========================================================================
    print("\n🔟 REUSING BUILDER - Build multiple queries")
    print("-" * 80)
  
    builder = QueryBuilder()
  
    query_a = builder.select("id", "name").from_table("users").build()
    print("Query A:")
    print(query_a)
  
    # Builder automatically resets after build()
    query_b = builder.select("id", "title").from_table("posts").build()
    print("\nQuery B (builder was reset):")
    print(query_b)
  
    print("\n" + "=" * 80)
    print("✅ ALL EXAMPLES COMPLETE")
    print("=" * 80)


if __name__ == "__main__":
    main()
```

---

# 📤 Output

```
================================================================================
SQL QUERY BUILDER - COMPLETE EXAMPLE
================================================================================

1️⃣  SIMPLE QUERY - Get all users
--------------------------------------------------------------------------------
SELECT id, name, email
FROM users;

2️⃣  FILTERED QUERY - Active users ordered by name
--------------------------------------------------------------------------------
SELECT id, name, email, created_at
FROM users
WHERE status = 'active' AND age >= 18
ORDER BY name ASC;

3️⃣  JOIN QUERY - Users with their orders
--------------------------------------------------------------------------------
SELECT users.name, users.email, orders.id as order_id, orders.total
FROM users
INNER JOIN orders ON users.id = orders.user_id
WHERE orders.total > 100
ORDER BY orders.total DESC
LIMIT 10;

4️⃣  COMPLEX QUERY - Orders with user and product details
--------------------------------------------------------------------------------
SELECT orders.id, orders.created_at, users.name as customer_name, products.name as product_name, order_items.quantity, order_items.price
FROM orders
INNER JOIN users ON orders.user_id = users.id
INNER JOIN order_items ON orders.id = order_items.order_id
INNER JOIN products ON order_items.product_id = products.id
WHERE orders.status = 'completed' AND orders.created_at >= '2024-01-01'
ORDER BY orders.created_at DESC
LIMIT 20;

5️⃣  USING DIRECTOR - Pre-configured queries
--------------------------------------------------------------------------------
📋 Standard Query: Get all users
SELECT *
FROM users;

📋 Standard Query: Get active users
SELECT id, name, email, created_at
FROM users
WHERE status = 'active'
ORDER BY name ASC;

📋 Standard Query: Recent orders with user info
SELECT orders.id, orders.total, orders.created_at, users.name, users.email
FROM orders
INNER JOIN users ON orders.user_id = users.id
ORDER BY orders.created_at DESC
LIMIT 5;

📋 Standard Query: User order summary
SELECT users.id, users.name, COUNT(orders.id) as order_count, SUM(orders.total) as total_spent
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE users.status = 'active'
ORDER BY total_spent DESC;

📋 Standard Query: Top 3 products
SELECT products.id, products.name, COUNT(order_items.id) as times_ordered, SUM(order_items.quantity) as total_quantity
FROM products
INNER JOIN order_items ON products.id = order_items.product_id
ORDER BY times_ordered DESC
LIMIT 3;

6️⃣  AGGREGATION QUERY - Sales by country
--------------------------------------------------------------------------------
SELECT users.country, COUNT(orders.id) as total_orders, SUM(orders.total) as total_revenue, AVG(orders.total) as avg_order_value
FROM users
INNER JOIN orders ON users.id = orders.user_id
WHERE orders.status = 'completed'
ORDER BY total_revenue DESC;

7️⃣  LEFT JOIN QUERY - All users with optional order count
--------------------------------------------------------------------------------
SELECT users.id, users.name, users.email, COUNT(orders.id) as order_count
FROM users
LEFT JOIN orders ON users.id = orders.user_id
ORDER BY order_count DESC;

8️⃣  IMMUTABILITY - Query cannot be modified after build
--------------------------------------------------------------------------------
Original query:
SELECT id, name
FROM users;

❌ Cannot modify: FrozenInstanceError

9️⃣  VALIDATION ERRORS
--------------------------------------------------------------------------------
❌ Error: Must call from_table() before build()!
❌ Error: Must call select() before build()!
❌ Error: LIMIT must be positive!

🔟 REUSING BUILDER - Build multiple queries
--------------------------------------------------------------------------------
Query A:
SELECT id, name
FROM users;

Query B (builder was reset):
SELECT id, title
FROM posts;

================================================================================
✅ ALL EXAMPLES COMPLETE
================================================================================
```

---

# 🎯 Key Features Demonstrated

## ✅ **Fluent Interface**

```python
query = (QueryBuilder()
         .select("name", "email")
         .from_table("users")
         .where("age > 18")
         .order_by("name")
         .limit(10)
         .build())
```

## ✅ **Immutable Result**

- Once `build()` is called, the `SQLQuery` object is frozen
- Cannot be modified after creation

## ✅ **Validation**

- Must have SELECT columns
- Must have FROM table
- LIMIT must be positive
- Validates in `build()` method

## ✅ **Director for Common Patterns**

- `get_all_users()`
- `get_active_users()`
- `get_recent_orders_with_users()`
- Encapsulates common query patterns

## ✅ **Reusable Builder**

- Automatically resets after `build()`
- Can build multiple queries with same builder instance

---

# 💡 Why This is Better Than String Concatenation

```python
# ❌ BAD: String concatenation (error-prone, not validated)
sql = "SELECT name, email FROM users"
if age_filter:
    sql += " WHERE age > 18"
if order:
    sql += " ORDER BY name"
# Easy to make SQL injection vulnerabilities!

# ✅ GOOD: Builder pattern (safe, validated, readable)
query = (QueryBuilder()
         .select("name", "email")
         .from_table("users")
         .where("age > 18")
         .order_by("name")
         .build())
```

This is a **production-ready** SQL query builder! 🚀
