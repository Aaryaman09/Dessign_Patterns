# Builder Pattern - One Page Notes

## 🎯 Core Concept

**Separate object construction from its representation.** Build complex objects step-by-step using a fluent interface.

---

## ❌ Problems Builder Solves

### 1. **Too Many Constructor Parameters**

```python
# BAD: Hard to read, easy to mess up
User("John", "Doe", "john@email.com", 25, "USA", True, False, "premium")

# GOOD: Clear and explicit
user = UserBuilder().set_name("John").set_email("john@email.com").with_premium().build()
```

### 2. **Complex Interdependent Validation**

```python
# Constructor can't enforce: "If SSL enabled, must provide cert AND key"
# Builder can group related parameters:
config = ConfigBuilder().enable_ssl(cert_path, key_path).build()  # Forces both together
```

### 3. **Creating Immutable Objects**

```python
# Builder constructs, then returns immutable object
pizza = PizzaBuilder().add_cheese().add_pepperoni().build()
# pizza is now immutable - can't be modified after creation
```

---

## ✅ When to Use Builder

| **Use Builder When**           | **Don't Use Builder When** |
| ------------------------------------ | -------------------------------- |
| 5+ parameters (especially optional)  | 2-3 simple parameters            |
| Complex validation between params    | Simple validation                |
| Need immutable objects               | Mutable objects are fine         |
| Building test data                   | One-off object creation          |
| Query/config builders (fluent API)   | Keyword args work fine           |
| Step-by-step construction with order | No construction dependencies     |

---

## 📝 Basic Implementation Pattern

```python
class Product:
    def __init__(self, field1, field2, field3):
        self._field1 = field1  # Immutable (private)
        self._field2 = field2
        self._field3 = field3

class ProductBuilder:
    def __init__(self):
        self.field1 = None  # Defaults
        self.field2 = None
        self.field3 = None
  
    def set_field1(self, value):
        self.field1 = value
        return self  # ⭐ Return self for chaining
  
    def set_field2(self, value):
        self.field2 = value
        return self
  
    def build(self):
        # ⭐ Validation happens here
        if not self.field1:
            raise ValueError("field1 required!")
        return Product(self.field1, self.field2, self.field3)

# Usage
product = ProductBuilder().set_field1("A").set_field2("B").build()
```

---

## 🔑 Key Components

1. **Product**: The complex object being built (usually immutable)
2. **Builder**: Has methods to set each part, returns `self` for chaining
3. **`build()` method**: Validates and constructs the final immutable object

---

## 💡 Real-World Use Cases

### 1. **Query Builders** (Best use case)

```python
query = QueryBuilder().select("name").from_table("users").where("age > 18").build()
```

### 2. **Test Data Builders**

```python
user = UserBuilder().as_admin().with_premium().build()
```

### 3. **Configuration Objects**

```python
config = ConfigBuilder().set_host("localhost").enable_ssl(cert, key).build()
```

### 4. **HTTP Request Builders**

```python
request = RequestBuilder().post("/api").with_json(data).with_auth(token).build()
```

---

## ⚠️ Common Mistakes

| **Mistake**                | **Fix**                                |
| -------------------------------- | -------------------------------------------- |
| Forgetting `return self`       | Always return `self` in builder methods    |
| No validation in `build()`     | Validate all required fields in `build()`  |
| Making Product mutable           | Make Product fields private/immutable        |
| Using Builder for simple objects | Use keyword args instead                     |
| Complex logic in setters         | Keep setters simple, validate in `build()` |

---

## 🐍 Python-Specific Notes

- **Prefer keyword arguments** for simple cases (2-4 params)
- **Use `dataclasses`** for simple data objects
- Builder is **less common** in Python than Java/C++
- Use when **readability & safety** genuinely improve
- **Fluent interfaces** (query builders) are Builder's killer feature in Python

---

## 📊 Quick Decision Tree

```
Do you have 5+ parameters? 
├─ NO → Use keyword arguments
└─ YES → Do parameters have complex validation?
    ├─ NO → Use keyword arguments with defaults
    └─ YES → Do you need immutability?
        ├─ NO → Maybe use keyword arguments
        └─ YES → USE BUILDER PATTERN ✅
```

---

## 🎓 Remember

**Builder is about CLARITY and SAFETY, not just avoiding long parameter lists.**

If keyword arguments work fine, use them. Builder should make code **significantly** more readable and safer, not just "different."

# Simple Builder Pattern Example - Coffee Order ☕

```python
from dataclasses import dataclass
from enum import Enum

# ============================================================================
# ENUMS
# ============================================================================

class CoffeeSize(Enum):
    SMALL = "small"
    MEDIUM = "medium"
    LARGE = "large"


# ============================================================================
# IMMUTABLE COFFEE (THE PRODUCT)
# ============================================================================

@dataclass(frozen=True)  # Immutable
class Coffee:
    """A simple immutable coffee order"""
    size: CoffeeSize
    milk: bool = False
    sugar: int = 0  # Number of sugar cubes
    whipped_cream: bool = False
  
    def __post_init__(self):
        """Simple validation"""
        if self.sugar < 0 or self.sugar > 5:
            raise ValueError("Sugar must be between 0 and 5!")
  
    @property
    def price(self) -> float:
        """Calculate price"""
        base_price = {
            CoffeeSize.SMALL: 2.50,
            CoffeeSize.MEDIUM: 3.00,
            CoffeeSize.LARGE: 3.50
        }
      
        price = base_price[self.size]
        if self.milk:
            price += 0.50
        if self.whipped_cream:
            price += 0.75
      
        return round(price, 2)
  
    def __str__(self):
        extras = []
        if self.milk:
            extras.append("milk")
        if self.sugar > 0:
            extras.append(f"{self.sugar} sugar")
        if self.whipped_cream:
            extras.append("whipped cream")
      
        extras_str = " with " + ", ".join(extras) if extras else ""
        return f"{self.size.value.title()} coffee{extras_str} - ${self.price}"


# ============================================================================
# COFFEE BUILDER
# ============================================================================

class CoffeeBuilder:
    """Simple builder for coffee orders"""
  
    def __init__(self):
        self._size = None
        self._milk = False
        self._sugar = 0
        self._whipped_cream = False
  
    def size(self, size: CoffeeSize) -> 'CoffeeBuilder':
        """Set coffee size"""
        self._size = size
        return self
  
    def with_milk(self) -> 'CoffeeBuilder':
        """Add milk"""
        self._milk = True
        return self
  
    def with_sugar(self, cubes: int) -> 'CoffeeBuilder':
        """Add sugar cubes"""
        self._sugar = cubes
        return self
  
    def with_whipped_cream(self) -> 'CoffeeBuilder':
        """Add whipped cream"""
        self._whipped_cream = True
        return self
  
    def build(self) -> Coffee:
        """Build the coffee"""
        if self._size is None:
            raise ValueError("Size is required!")
      
        return Coffee(
            size=self._size,
            milk=self._milk,
            sugar=self._sugar,
            whipped_cream=self._whipped_cream
        )


# ============================================================================
# DEMONSTRATION
# ============================================================================

def main():
    print("☕ SIMPLE COFFEE BUILDER EXAMPLE")
    print("=" * 60)
  
    # Example 1: Simple black coffee
    print("\n1️⃣  Simple Black Coffee")
    coffee1 = (CoffeeBuilder()
               .size(CoffeeSize.MEDIUM)
               .build())
    print(f"   {coffee1}")
  
    # Example 2: Coffee with milk and sugar
    print("\n2️⃣  Coffee with Milk and Sugar")
    coffee2 = (CoffeeBuilder()
               .size(CoffeeSize.LARGE)
               .with_milk()
               .with_sugar(2)
               .build())
    print(f"   {coffee2}")
  
    # Example 3: Fancy coffee with everything
    print("\n3️⃣  Fancy Coffee (Everything)")
    coffee3 = (CoffeeBuilder()
               .size(CoffeeSize.SMALL)
               .with_milk()
               .with_sugar(1)
               .with_whipped_cream()
               .build())
    print(f"   {coffee3}")
  
    # Example 4: Try to modify (will fail - immutable)
    print("\n4️⃣  Try to Modify (Immutable)")
    try:
        coffee1.sugar = 3  # ❌ Will fail
    except Exception as e:
        print(f"   ❌ Cannot modify: {type(e).__name__}")
  
    # Example 5: Validation error
    print("\n5️⃣  Validation Error (Too Much Sugar)")
    try:
        bad_coffee = (CoffeeBuilder()
                      .size(CoffeeSize.MEDIUM)
                      .with_sugar(10)  # ❌ Too much!
                      .build())
    except ValueError as e:
        print(f"   ❌ Error: {e}")
  
    # Example 6: Missing required field
    print("\n6️⃣  Missing Required Field (Size)")
    try:
        incomplete = (CoffeeBuilder()
                      .with_milk()
                      .build())  # ❌ No size!
    except ValueError as e:
        print(f"   ❌ Error: {e}")
  
    print("\n" + "=" * 60)


if __name__ == "__main__":
    main()
```

---

# 📤 Output

```
☕ SIMPLE COFFEE BUILDER EXAMPLE
============================================================

1️⃣  Simple Black Coffee
   Medium coffee - $3.0

2️⃣  Coffee with Milk and Sugar
   Large coffee with milk, 2 sugar - $4.0

3️⃣  Fancy Coffee (Everything)
   Small coffee with milk, 1 sugar, whipped cream - $3.75

4️⃣  Try to Modify (Immutable)
   ❌ Cannot modify: FrozenInstanceError

5️⃣  Validation Error (Too Much Sugar)
   ❌ Error: Sugar must be between 0 and 5!

6️⃣  Missing Required Field (Size)
   ❌ Error: Size is required!

============================================================
```

---

# 🎯 Why This is Simple

| **Feature**            | **Explanation**                            |
| ---------------------------- | ------------------------------------------------ |
| **Only 4 fields**      | `size`, `milk`, `sugar`, `whipped_cream` |
| **Simple validation**  | Just check sugar range (0-5)                     |
| **Clear methods**      | `with_milk()`, `with_sugar()`, etc.          |
| **One required field** | Only `size` is required                        |
| **No complex order**   | Can call methods in any order                    |
| **Easy to understand** | Everyone knows how to order coffee!              |

---

# 🆚 Compare: Constructor vs Builder

```python
# ❌ WITHOUT BUILDER (Constructor)
coffee = Coffee(
    size=CoffeeSize.LARGE,
    milk=True,
    sugar=2,
    whipped_cream=False  # Why write False explicitly?
)

# ✅ WITH BUILDER (Much Clearer)
coffee = (CoffeeBuilder()
          .size(CoffeeSize.LARGE)
          .with_milk()
          .with_sugar(2)
          .build())
# Only specify what you want!
```

---

# 💡 Key Points

1. **Immutable**: `frozen=True` prevents modification after creation
2. **Fluent**: Each method returns `self` for chaining
3. **Validation**: Happens in `build()` method
4. **Clear**: Only specify what you want (no `False` values)
5. **Safe**: Can't create invalid coffee orders

---

This is the **simplest possible** Builder example. Perfect for understanding the core concept! ☕
