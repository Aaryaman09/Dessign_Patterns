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

# Builder Pattern with Dataclasses - Complete Example

```python
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum

# ============================================================================
# 1. ENUMS FOR TYPE SAFETY
# ============================================================================

class PizzaSize(Enum):
    SMALL = "small"
    MEDIUM = "medium"
    LARGE = "large"
    EXTRA_LARGE = "extra_large"

class CrustType(Enum):
    THIN = "thin"
    REGULAR = "regular"
    THICK = "thick"
    STUFFED = "stuffed"

class Topping(Enum):
    CHEESE = "cheese"
    PEPPERONI = "pepperoni"
    MUSHROOMS = "mushrooms"
    OLIVES = "olives"
    ONIONS = "onions"
    BACON = "bacon"
    SAUSAGE = "sausage"
    BELL_PEPPERS = "bell_peppers"
    PINEAPPLE = "pineapple"


# ============================================================================
# 2. IMMUTABLE PIZZA (THE PRODUCT)
# ============================================================================

@dataclass(frozen=True)  # ⭐ frozen=True makes it IMMUTABLE
class Pizza:
    """Immutable Pizza object - cannot be modified after creation"""
    size: PizzaSize
    crust: CrustType
    toppings: tuple[Topping, ...]  # Tuple is immutable
    extra_cheese: bool = False
    sauce_amount: str = "normal"  # "light", "normal", "extra"
  
    def __post_init__(self):
        """Validation after dataclass initialization"""
        # This runs AFTER the object is created but BEFORE it's frozen
        if not self.toppings:
            raise ValueError("Pizza must have at least one topping!")
      
        if len(self.toppings) > 8:
            raise ValueError("Maximum 8 toppings allowed!")
  
    @property
    def price(self) -> float:
        """Calculate price based on size and toppings"""
        base_prices = {
            PizzaSize.SMALL: 8.99,
            PizzaSize.MEDIUM: 10.99,
            PizzaSize.LARGE: 12.99,
            PizzaSize.EXTRA_LARGE: 14.99
        }
      
        price = base_prices[self.size]
        price += len(self.toppings) * 1.50  # $1.50 per topping
      
        if self.extra_cheese:
            price += 2.00
      
        if self.crust == CrustType.STUFFED:
            price += 3.00
      
        return round(price, 2)
  
    def __str__(self):
        toppings_str = ", ".join(t.value for t in self.toppings)
        return (f"{self.size.value.title()} {self.crust.value} crust pizza "
                f"with {toppings_str} - ${self.price}")


# ============================================================================
# 3. PIZZA BUILDER (MUTABLE DURING CONSTRUCTION)
# ============================================================================

class PizzaBuilder:
    """Builder for constructing Pizza objects step-by-step"""
  
    def __init__(self):
        # Mutable state during construction
        self._size: Optional[PizzaSize] = None
        self._crust: CrustType = CrustType.REGULAR  # Default
        self._toppings: List[Topping] = []  # Mutable list during building
        self._extra_cheese: bool = False
        self._sauce_amount: str = "normal"
      
        # Track construction order
        self._size_set = False
        self._crust_set = False
  
    # ========================================================================
    # STEP 1: SIZE (REQUIRED FIRST)
    # ========================================================================
  
    def set_size(self, size: PizzaSize) -> 'PizzaBuilder':
        """Set pizza size - MUST be called first"""
        self._size = size
        self._size_set = True
        return self
  
    # ========================================================================
    # STEP 2: CRUST (MUST BE AFTER SIZE)
    # ========================================================================
  
    def set_crust(self, crust: CrustType) -> 'PizzaBuilder':
        """Set crust type - must set size first"""
        if not self._size_set:
            raise ValueError("Must set size before setting crust!")
      
        # VALIDATION: Stuffed crust only available for large pizzas
        if crust == CrustType.STUFFED and self._size == PizzaSize.SMALL:
            raise ValueError("Stuffed crust not available for small pizzas!")
      
        self._crust = crust
        self._crust_set = True
        return self
  
    # ========================================================================
    # STEP 3: TOPPINGS (MUST BE AFTER CRUST)
    # ========================================================================
  
    def add_topping(self, topping: Topping) -> 'PizzaBuilder':
        """Add a topping - must set crust first"""
        if not self._crust_set:
            raise ValueError("Must set crust before adding toppings!")
      
        if topping in self._toppings:
            raise ValueError(f"{topping.value} already added!")
      
        if len(self._toppings) >= 8:
            raise ValueError("Maximum 8 toppings allowed!")
      
        self._toppings.append(topping)
        return self
  
    def add_toppings(self, *toppings: Topping) -> 'PizzaBuilder':
        """Add multiple toppings at once"""
        for topping in toppings:
            self.add_topping(topping)
        return self
  
    # ========================================================================
    # OPTIONAL CUSTOMIZATIONS
    # ========================================================================
  
    def with_extra_cheese(self) -> 'PizzaBuilder':
        """Add extra cheese"""
        self._extra_cheese = True
        return self
  
    def set_sauce_amount(self, amount: str) -> 'PizzaBuilder':
        """Set sauce amount: 'light', 'normal', or 'extra'"""
        if amount not in ["light", "normal", "extra"]:
            raise ValueError("Sauce amount must be 'light', 'normal', or 'extra'")
        self._sauce_amount = amount
        return self
  
    # ========================================================================
    # BUILD METHOD - CREATES IMMUTABLE PIZZA
    # ========================================================================
  
    def build(self) -> Pizza:
        """
        Validate and build the final immutable Pizza object.
        This is where all final validation happens.
        """
        # Validation: Required fields
        if not self._size_set:
            raise ValueError("Size is required! Call set_size() first.")
      
        if not self._crust_set:
            raise ValueError("Crust is required! Call set_crust() after set_size().")
      
        if not self._toppings:
            raise ValueError("At least one topping is required!")
      
        # Complex validation: Business rules
        if (Topping.PINEAPPLE in self._toppings and 
            Topping.BACON not in self._toppings):
            raise ValueError("Hawaiian pizza must have both pineapple AND bacon!")
      
        # Create immutable Pizza (convert list to tuple)
        return Pizza(
            size=self._size,
            crust=self._crust,
            toppings=tuple(self._toppings),  # ⭐ Convert to immutable tuple
            extra_cheese=self._extra_cheese,
            sauce_amount=self._sauce_amount
        )


# ============================================================================
# 4. SPECIALIZED BUILDERS (INHERITANCE)
# ============================================================================

class VegetarianPizzaBuilder(PizzaBuilder):
    """Builder that only allows vegetarian toppings"""
  
    VEGETARIAN_TOPPINGS = {
        Topping.CHEESE,
        Topping.MUSHROOMS,
        Topping.OLIVES,
        Topping.ONIONS,
        Topping.BELL_PEPPERS
    }
  
    def add_topping(self, topping: Topping) -> 'VegetarianPizzaBuilder':
        """Override to only allow vegetarian toppings"""
        if topping not in self.VEGETARIAN_TOPPINGS:
            raise ValueError(f"{topping.value} is not vegetarian!")
        return super().add_topping(topping)


class MeatLoversPizzaBuilder(PizzaBuilder):
    """Pre-configured builder for meat lovers"""
  
    def __init__(self):
        super().__init__()
        # Pre-set defaults for meat lovers
        self._sauce_amount = "extra"
  
    def build(self) -> Pizza:
        """Ensure at least 2 meat toppings"""
        meat_toppings = {Topping.PEPPERONI, Topping.BACON, Topping.SAUSAGE}
        meat_count = sum(1 for t in self._toppings if t in meat_toppings)
      
        if meat_count < 2:
            raise ValueError("Meat lovers pizza must have at least 2 meat toppings!")
      
        return super().build()


# ============================================================================
# 5. DEMONSTRATION
# ============================================================================

def main():
    print("=" * 70)
    print("BUILDER PATTERN WITH DATACLASSES - DEMONSTRATION")
    print("=" * 70)
  
    # ========================================================================
    # Example 1: Basic Pizza (Enforced Order)
    # ========================================================================
    print("\n1️⃣  BASIC PIZZA (Step-by-step construction)")
    print("-" * 70)
  
    pizza1 = (PizzaBuilder()
              .set_size(PizzaSize.LARGE)           # Step 1: Size first
              .set_crust(CrustType.THIN)           # Step 2: Crust second
              .add_topping(Topping.CHEESE)         # Step 3: Toppings
              .add_topping(Topping.PEPPERONI)
              .add_topping(Topping.MUSHROOMS)
              .with_extra_cheese()
              .build())
  
    print(f"✅ {pizza1}")
    print(f"   Toppings: {pizza1.toppings}")
    print(f"   Is immutable: {pizza1.__class__.__dataclass_fields__['size'].metadata}")
  
    # Try to modify (will fail because frozen=True)
    try:
        pizza1.size = PizzaSize.SMALL  # ❌ Will raise error
    except Exception as e:
        print(f"   ❌ Cannot modify: {type(e).__name__}")
  
    # ========================================================================
    # Example 2: Wrong Order (Will Fail)
    # ========================================================================
    print("\n2️⃣  WRONG ORDER - Validation Enforces Steps")
    print("-" * 70)
  
    try:
        pizza2 = (PizzaBuilder()
                  .add_topping(Topping.CHEESE)     # ❌ Can't add topping first!
                  .set_size(PizzaSize.MEDIUM)
                  .build())
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # ========================================================================
    # Example 3: Complex Validation
    # ========================================================================
    print("\n3️⃣  COMPLEX VALIDATION - Business Rules")
    print("-" * 70)
  
    # Stuffed crust not allowed for small pizzas
    try:
        pizza3 = (PizzaBuilder()
                  .set_size(PizzaSize.SMALL)
                  .set_crust(CrustType.STUFFED)    # ❌ Not allowed!
                  .add_topping(Topping.CHEESE)
                  .build())
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # Hawaiian pizza must have both pineapple AND bacon
    try:
        pizza4 = (PizzaBuilder()
                  .set_size(PizzaSize.MEDIUM)
                  .set_crust(CrustType.REGULAR)
                  .add_topping(Topping.CHEESE)
                  .add_topping(Topping.PINEAPPLE)  # ❌ Missing bacon!
                  .build())
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # ========================================================================
    # Example 4: Vegetarian Pizza Builder
    # ========================================================================
    print("\n4️⃣  VEGETARIAN PIZZA BUILDER")
    print("-" * 70)
  
    veggie_pizza = (VegetarianPizzaBuilder()
                    .set_size(PizzaSize.MEDIUM)
                    .set_crust(CrustType.THICK)
                    .add_toppings(Topping.MUSHROOMS, Topping.OLIVES, Topping.BELL_PEPPERS)
                    .with_extra_cheese()
                    .build())
  
    print(f"✅ {veggie_pizza}")
  
    # Try to add meat (will fail)
    try:
        bad_veggie = (VegetarianPizzaBuilder()
                      .set_size(PizzaSize.LARGE)
                      .set_crust(CrustType.REGULAR)
                      .add_topping(Topping.PEPPERONI)  # ❌ Not vegetarian!
                      .build())
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # ========================================================================
    # Example 5: Meat Lovers Pizza
    # ========================================================================
    print("\n5️⃣  MEAT LOVERS PIZZA BUILDER")
    print("-" * 70)
  
    meat_pizza = (MeatLoversPizzaBuilder()
                  .set_size(PizzaSize.EXTRA_LARGE)
                  .set_crust(CrustType.STUFFED)
                  .add_toppings(Topping.PEPPERONI, Topping.BACON, Topping.SAUSAGE)
                  .add_topping(Topping.CHEESE)
                  .build())
  
    print(f"✅ {meat_pizza}")
  
    # ========================================================================
    # Example 6: Multiple Toppings at Once
    # ========================================================================
    print("\n6️⃣  SUPREME PIZZA (Multiple Toppings)")
    print("-" * 70)
  
    supreme = (PizzaBuilder()
               .set_size(PizzaSize.LARGE)
               .set_crust(CrustType.REGULAR)
               .add_toppings(
                   Topping.CHEESE,
                   Topping.PEPPERONI,
                   Topping.SAUSAGE,
                   Topping.MUSHROOMS,
                   Topping.ONIONS,
                   Topping.BELL_PEPPERS
               )
               .set_sauce_amount("extra")
               .build())
  
    print(f"✅ {supreme}")
  
    # ========================================================================
    # Example 7: Immutability Demonstration
    # ========================================================================
    print("\n7️⃣  IMMUTABILITY - Cannot Modify After Creation")
    print("-" * 70)
  
    pizza = (PizzaBuilder()
             .set_size(PizzaSize.MEDIUM)
             .set_crust(CrustType.THIN)
             .add_topping(Topping.CHEESE)
             .build())
  
    print(f"Original: {pizza}")
  
    # All these will fail:
    modifications = [
        ("pizza.size = PizzaSize.LARGE", lambda: setattr(pizza, 'size', PizzaSize.LARGE)),
        ("pizza.toppings.append(Topping.BACON)", lambda: pizza.toppings.append(Topping.BACON)),
        ("pizza.extra_cheese = True", lambda: setattr(pizza, 'extra_cheese', True))
    ]
  
    for desc, func in modifications:
        try:
            func()
        except Exception as e:
            print(f"   ❌ {desc} → {type(e).__name__}")
  
    print("\n" + "=" * 70)
    print("✅ All pizzas are immutable and validated!")
    print("=" * 70)


if __name__ == "__main__":
    main()
```

---

# 📝 Key Takeaways from This Example

## 1️⃣ **Immutability with `frozen=True`**

```python
@dataclass(frozen=True)  # Makes object immutable after creation
class Pizza:
    size: PizzaSize
    toppings: tuple[Topping, ...]  # Tuple (immutable) not list
```

## 2️⃣ **Enforced Construction Order**

```python
# Must call in this order:
.set_size()      # Step 1
.set_crust()     # Step 2 (checks if size was set)
.add_topping()   # Step 3 (checks if crust was set)
.build()         # Final validation
```

## 3️⃣ **Complex Validation**

- **In setters**: Basic validation (e.g., stuffed crust only for large pizzas)
- **In `build()`**: Complex business rules (e.g., Hawaiian pizza needs pineapple + bacon)

## 4️⃣ **Builder is Mutable, Product is Immutable**

- `PizzaBuilder`: Mutable during construction (uses `List`)
- `Pizza`: Immutable after creation (uses `tuple`, `frozen=True`)

## 5️⃣ **Specialized Builders via Inheritance**

- `VegetarianPizzaBuilder`: Restricts to veggie toppings
- `MeatLoversPizzaBuilder`: Enforces meat topping rules

---

**Run this code and see all the validation in action!** 🍕
