
# Builder Pattern - Complete One Page Notes

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

## 🏗️ The 5 Components (Full GoF Pattern)

| **Component**            | **Role**                         | **Required?**                   |
| ------------------------------ | -------------------------------------- | ------------------------------------- |
| **1. Product**           | Final immutable object being built     | ✅ Always                             |
| **2. Builder Interface** | Abstract contract (defines methods)    | ⚠️ Optional (for multiple builders) |
| **3. Concrete Builder**  | Actual implementation of builder       | ✅ Always                             |
| **4. Director**          | Knows "recipes" - controls build order | ⚠️ Optional (for standard configs)  |
| **5. Client**            | Your code that uses the pattern        | ✅ Always                             |

---

## 📝 Basic Implementation Pattern

```python
from dataclasses import dataclass

# 1. PRODUCT (Immutable)
@dataclass(frozen=True)  # frozen=True makes it immutable
class Product:
    field1: str
    field2: int
    field3: bool = False

# 3. CONCRETE BUILDER (Mutable during construction)
class ProductBuilder:
    def __init__(self):
        self.field1 = None  # Defaults
        self.field2 = 0
        self.field3 = False
  
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

# 5. CLIENT
product = ProductBuilder().set_field1("A").set_field2(10).build()
```

---

## 🎭 Director Pattern (Optional Component)

**Director = Recipe Keeper** - Knows HOW to build standard products

```python
# 2. BUILDER INTERFACE (Optional - for multiple builders)
from abc import ABC, abstractmethod

class PizzaBuilder(ABC):
    @abstractmethod
    def add_topping(self, topping): pass
  
    @abstractmethod
    def build(self): pass

# 3. CONCRETE BUILDERS
class ItalianPizzaBuilder(PizzaBuilder):
    def __init__(self):
        self._toppings = []
        self._crust = "thin"  # Italian default
  
    def add_topping(self, topping):
        self._toppings.append(topping)
        return self
  
    def build(self):
        return Pizza(self._crust, tuple(self._toppings))

class AmericanPizzaBuilder(PizzaBuilder):
    def __init__(self):
        self._toppings = []
        self._crust = "thick"  # American default
  
    def add_topping(self, topping):
        self._toppings.append(topping)
        return self
  
    def build(self):
        return Pizza(self._crust, tuple(self._toppings))

# 4. DIRECTOR (Knows recipes)
class PizzaDirector:
    def __init__(self, builder: PizzaBuilder):
        self._builder = builder
  
    def make_margherita(self):
        """Standard recipe for Margherita"""
        return (self._builder
                .add_topping("mozzarella")
                .add_topping("tomato")
                .add_topping("basil")
                .build())
  
    def make_pepperoni(self):
        """Standard recipe for Pepperoni"""
        return (self._builder
                .add_topping("mozzarella")
                .add_topping("pepperoni")
                .build())

# 5. CLIENT (Uses Director)
italian_builder = ItalianPizzaBuilder()
director = PizzaDirector(italian_builder)
pizza = director.make_margherita()  # Client doesn't know recipe!

# OR Client can use builder directly (skip Director)
custom_pizza = italian_builder.add_topping("cheese").build()
```

**When to use Director:**

- ✅ You have **standard recipes/configurations** reused across codebase
- ✅ You want to **hide construction complexity** from clients
- ❌ Skip it if every object is **custom** or clients need **full control**

---

## ✅ When to Use Builder

| **Use Builder When**                   | **Don't Use Builder When** |
| -------------------------------------------- | -------------------------------- |
| 5+ parameters (especially optional)          | 2-3 simple parameters            |
| Complex validation between params            | Simple validation                |
| Need immutable objects                       | Mutable objects are fine         |
| Building test data                           | One-off object creation          |
| Query/config builders (fluent API)           | Keyword args work fine           |
| Step-by-step construction with order         | No construction dependencies     |
| **Multiple builders for same product** | Only one way to build            |
| **Standard recipes (use Director)**    | Every build is unique            |

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

### 5. **Multiple Representations (Director)**

```python
# Same recipe, different builders produce different results
director = ReportDirector(PDFBuilder())
pdf_report = director.make_sales_report()

director.change_builder(ExcelBuilder())
excel_report = director.make_sales_report()  # Same data, different format
```

---

## ⚠️ Common Mistakes

| **Mistake**                                 | **Fix**                                           |
| ------------------------------------------------- | ------------------------------------------------------- |
| Forgetting `return self`                        | Always return `self` in builder methods               |
| No validation in `build()`                      | Validate all required fields in `build()`             |
| Making Product mutable                            | Make Product fields private/immutable (`frozen=True`) |
| Using Builder for simple objects                  | Use keyword args instead                                |
| Complex logic in setters                          | Keep setters simple, validate in `build()`            |
| **Not using Director for standard recipes** | Use Director to centralize common build sequences       |
| **Using Director when not needed**          | Skip Director if every build is custom                  |

---

## 🐍 Python-Specific Notes

- **Prefer keyword arguments** for simple cases (2-4 params)
- **Use `dataclasses`** with `frozen=True` for immutable products
- Builder is **less common** in Python than Java/C++
- Use when **readability & safety** genuinely improve
- **Fluent interfaces** (query builders) are Builder's killer feature in Python
- **Director is rare** in modern Python - most code uses builders directly

---

## 📊 Quick Decision Tree

```
Do you have 5+ parameters? 
├─ NO → Use keyword arguments
└─ YES → Do parameters have complex validation?
    ├─ NO → Use keyword arguments with defaults
    └─ YES → Do you need immutability?
        ├─ NO → Maybe use keyword arguments
        └─ YES → Do you have standard build recipes?
            ├─ YES → USE BUILDER + DIRECTOR ✅
            └─ NO → USE BUILDER (skip Director) ✅
```

---

## 🔑 Key Takeaways

1. **Builder = Mutable during construction** → **Product = Immutable after build()**
2. **Director is optional** - use only for standard recipes/configurations
3. **Client can use Builder directly** or through Director
4. **Multiple Concrete Builders** can implement same Builder Interface
5. **Validation in `build()`** - keep setters simple
6. **Return `self`** from all builder methods for fluent chaining
7. **Director knows "how"** - Client just asks for what it wants

---

## 🎓 Remember

**Builder is about CLARITY, SAFETY, and REUSABILITY.**

- Use **Builder** when construction is complex
- Use **Director** when you have reusable recipes
- Skip both if keyword arguments work fine

**The pattern should make code significantly more readable and safer, not just "different."**


# 🎯 The 5 Components Explained - Complete pizza example

## 1️⃣ **Product** (What we've been using)

The final immutable object being built.

```python
@dataclass(frozen=True)
class Pizza:  # This is the PRODUCT
    size: str
    toppings: tuple
```

---

## 2️⃣ **Builder Interface** (Abstract - defines contract)

An abstract class/interface that defines **what methods** all builders must have.

```python
from abc import ABC, abstractmethod

class PizzaBuilderInterface(ABC):  # BUILDER INTERFACE
    """Defines the contract for all pizza builders"""
  
    @abstractmethod
    def set_size(self, size):
        pass
  
    @abstractmethod
    def add_topping(self, topping):
        pass
  
    @abstractmethod
    def build(self) -> Pizza:
        pass
```

---

## 3️⃣ **Concrete Builder** (What we've been using)

The actual implementation of the builder.

```python
class ItalianPizzaBuilder(PizzaBuilderInterface):  # CONCRETE BUILDER
    """Concrete implementation for Italian-style pizza"""
  
    def __init__(self):
        self._size = None
        self._toppings = []
  
    def set_size(self, size):
        self._size = size
        return self
  
    def add_topping(self, topping):
        self._toppings.append(topping)
        return self
  
    def build(self) -> Pizza:
        return Pizza(self._size, tuple(self._toppings))
```

---

## 4️⃣ **Director** (NEW - Controls the building process)

A class that **knows the recipe** - it controls **which methods to call and in what order**.

```python
class PizzaDirector:  # DIRECTOR
    """Controls HOW to build different types of pizzas"""
  
    def __init__(self, builder: PizzaBuilderInterface):
        self._builder = builder
  
    def make_margherita(self) -> Pizza:
        """Director knows the RECIPE for Margherita"""
        return (self._builder
                .set_size("medium")
                .add_topping("cheese")
                .add_topping("tomato")
                .add_topping("basil")
                .build())
  
    def make_pepperoni(self) -> Pizza:
        """Director knows the RECIPE for Pepperoni"""
        return (self._builder
                .set_size("large")
                .add_topping("cheese")
                .add_topping("pepperoni")
                .build())
  
    def make_veggie(self) -> Pizza:
        """Director knows the RECIPE for Veggie"""
        return (self._builder
                .set_size("medium")
                .add_topping("cheese")
                .add_topping("mushrooms")
                .add_topping("peppers")
                .add_topping("olives")
                .build())
```

**Key Point:** The Director **encapsulates the construction logic**. It knows "to make a Margherita, you need medium size, cheese, tomato, and basil."

---

## 5️⃣ **Client** (Your application code)

The code that **uses** the Director and Builder to create objects.

```python
# CLIENT CODE
def main():  # This is the CLIENT
    # Client creates a builder
    builder = ItalianPizzaBuilder()
  
    # Client creates a director and gives it the builder
    director = PizzaDirector(builder)
  
    # Client asks director to make specific pizzas
    margherita = director.make_margherita()
    pepperoni = director.make_pepperoni()
  
    print(margherita)
    print(pepperoni)
```

---

# 🔄 Complete Example with All 5 Components

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List

# ============================================================================
# 1. PRODUCT
# ============================================================================

@dataclass(frozen=True)
class Pizza:
    """The complex object being built"""
    size: str
    crust: str
    toppings: tuple
  
    def __str__(self):
        return f"{self.size} {self.crust} pizza with {', '.join(self.toppings)}"


# ============================================================================
# 2. BUILDER INTERFACE (Abstract)
# ============================================================================

class PizzaBuilder(ABC):
    """Abstract interface - defines what ALL builders must implement"""
  
    @abstractmethod
    def reset(self):
        """Reset builder to start fresh"""
        pass
  
    @abstractmethod
    def set_size(self, size: str):
        pass
  
    @abstractmethod
    def set_crust(self, crust: str):
        pass
  
    @abstractmethod
    def add_topping(self, topping: str):
        pass
  
    @abstractmethod
    def build(self) -> Pizza:
        pass


# ============================================================================
# 3. CONCRETE BUILDERS (Implementations)
# ============================================================================

class ItalianPizzaBuilder(PizzaBuilder):
    """Concrete builder for Italian-style pizzas"""
  
    def __init__(self):
        self.reset()
  
    def reset(self):
        self._size = None
        self._crust = "thin"  # Italian default
        self._toppings = []
  
    def set_size(self, size: str):
        self._size = size
        return self
  
    def set_crust(self, crust: str):
        self._crust = crust
        return self
  
    def add_topping(self, topping: str):
        self._toppings.append(topping)
        return self
  
    def build(self) -> Pizza:
        pizza = Pizza(self._size, self._crust, tuple(self._toppings))
        self.reset()  # Reset for next pizza
        return pizza


class AmericanPizzaBuilder(PizzaBuilder):
    """Concrete builder for American-style pizzas"""
  
    def __init__(self):
        self.reset()
  
    def reset(self):
        self._size = None
        self._crust = "thick"  # American default
        self._toppings = []
  
    def set_size(self, size: str):
        self._size = size
        return self
  
    def set_crust(self, crust: str):
        self._crust = crust
        return self
  
    def add_topping(self, topping: str):
        self._toppings.append(topping)
        return self
  
    def build(self) -> Pizza:
        pizza = Pizza(self._size, self._crust, tuple(self._toppings))
        self.reset()
        return pizza


# ============================================================================
# 4. DIRECTOR (Controls the construction process)
# ============================================================================

class PizzaDirector:
    """
    Director knows the RECIPES.
    It controls WHICH builder methods to call and in WHAT ORDER.
    """
  
    def __init__(self, builder: PizzaBuilder):
        self._builder = builder
  
    def change_builder(self, builder: PizzaBuilder):
        """Can switch to a different builder"""
        self._builder = builder
  
    def make_margherita(self) -> Pizza:
        """Recipe for Margherita pizza"""
        return (self._builder
                .set_size("medium")
                .add_topping("mozzarella")
                .add_topping("tomato")
                .add_topping("basil")
                .build())
  
    def make_pepperoni(self) -> Pizza:
        """Recipe for Pepperoni pizza"""
        return (self._builder
                .set_size("large")
                .add_topping("mozzarella")
                .add_topping("pepperoni")
                .build())
  
    def make_supreme(self) -> Pizza:
        """Recipe for Supreme pizza"""
        return (self._builder
                .set_size("large")
                .add_topping("mozzarella")
                .add_topping("pepperoni")
                .add_topping("sausage")
                .add_topping("mushrooms")
                .add_topping("peppers")
                .add_topping("onions")
                .build())


# ============================================================================
# 5. CLIENT (Your application code)
# ============================================================================

def main():
    """CLIENT CODE - Uses Director and Builders"""
  
    print("=" * 70)
    print("FULL BUILDER PATTERN - All 5 Components")
    print("=" * 70)
  
    # Client creates builders
    italian_builder = ItalianPizzaBuilder()
    american_builder = AmericanPizzaBuilder()
  
    # Client creates director with Italian builder
    director = PizzaDirector(italian_builder)
  
    # ========================================================================
    # Scenario 1: Italian-style pizzas
    # ========================================================================
    print("\n🇮🇹 ITALIAN STYLE PIZZAS")
    print("-" * 70)
  
    margherita = director.make_margherita()
    print(f"1. {margherita}")
  
    pepperoni = director.make_pepperoni()
    print(f"2. {pepperoni}")
  
    # ========================================================================
    # Scenario 2: Switch to American-style
    # ========================================================================
    print("\n🇺🇸 AMERICAN STYLE PIZZAS (Same recipes, different builder)")
    print("-" * 70)
  
    director.change_builder(american_builder)  # Switch builder!
  
    margherita_american = director.make_margherita()
    print(f"1. {margherita_american}")
  
    supreme_american = director.make_supreme()
    print(f"2. {supreme_american}")
  
    # ========================================================================
    # Scenario 3: Client can also use builder directly (without Director)
    # ========================================================================
    print("\n🔧 CLIENT USING BUILDER DIRECTLY (No Director)")
    print("-" * 70)
  
    custom_pizza = (italian_builder
                    .set_size("small")
                    .set_crust("stuffed")
                    .add_topping("cheese")
                    .add_topping("pineapple")
                    .add_topping("bacon")
                    .build())
  
    print(f"Custom: {custom_pizza}")
  
    print("\n" + "=" * 70)


if __name__ == "__main__":
    main()
```

---

# 📤 Output

```
======================================================================
FULL BUILDER PATTERN - All 5 Components
======================================================================

🇮🇹 ITALIAN STYLE PIZZAS
----------------------------------------------------------------------
1. medium thin pizza with mozzarella, tomato, basil
2. large thin pizza with mozzarella, pepperoni

🇺🇸 AMERICAN STYLE PIZZAS (Same recipes, different builder)
----------------------------------------------------------------------
1. medium thick pizza with mozzarella, tomato, basil
2. large thick pizza with mozzarella, pepperoni, sausage, mushrooms, peppers, onions

🔧 CLIENT USING BUILDER DIRECTLY (No Director)
----------------------------------------------------------------------
Custom: small stuffed pizza with cheese, pineapple, bacon

======================================================================
```

---

# 🎯 Key Differences

| **Component**         | **Role**      | **Example**                                 |
| --------------------------- | ------------------- | ------------------------------------------------- |
| **Product**           | Final object        | `Pizza`                                         |
| **Builder Interface** | Contract (abstract) | `PizzaBuilder` (ABC)                            |
| **Concrete Builder**  | Implementation      | `ItalianPizzaBuilder`, `AmericanPizzaBuilder` |
| **Director**          | Knows recipes/order | `PizzaDirector.make_margherita()`               |
| **Client**            | Uses everything     | `main()` function                               |

---

# 🤔 When Do You Need Director?

## ✅ **Use Director When:**

1. You have **standard recipes/configurations** that are reused
2. You want to **hide complexity** from the client
3. You want **one place** to change how products are built
4. Multiple clients need the **same construction sequences**

```python
# WITH DIRECTOR - Client doesn't need to know the recipe
director = PizzaDirector(builder)
pizza = director.make_margherita()  # Simple!

# WITHOUT DIRECTOR - Client must know the recipe
pizza = (builder
         .set_size("medium")
         .add_topping("mozzarella")
         .add_topping("tomato")
         .add_topping("basil")
         .build())  # Client knows too much!
```

## ❌ **Skip Director When:**

1. Every object is **custom** (no standard recipes)
2. Client needs **full control** over construction
3. Adding Director is **overkill** for simple cases

---

# 📝 Summary

**In my previous examples, I skipped Director because:**

- Most modern Python code doesn't use it
- It adds complexity for simple cases
- Clients usually want direct control

**But the slide is showing the FULL GoF pattern with:**

- **Director** = Recipe keeper (knows how to build standard products)
- **Client** = Your code that uses the pattern

**Bottom line:** Director is optional. Use it only when you have **reusable construction recipes**.
