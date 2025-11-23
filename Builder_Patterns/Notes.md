
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
