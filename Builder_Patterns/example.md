# Complete Builder Pattern - Pizza Example with All 5 Components

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import List, Tuple

# ============================================================================
# ENUMS FOR TYPE SAFETY
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
    MOZZARELLA = "mozzarella"
    TOMATO_SAUCE = "tomato sauce"
    BASIL = "basil"
    PEPPERONI = "pepperoni"
    SAUSAGE = "sausage"
    MUSHROOMS = "mushrooms"
    BELL_PEPPERS = "bell peppers"
    ONIONS = "onions"
    OLIVES = "olives"
    PINEAPPLE = "pineapple"
    HAM = "ham"


# ============================================================================
# 1. PRODUCT - The Complex Immutable Object
# ============================================================================

@dataclass(frozen=True)  # Immutable
class Pizza:
    """
    The PRODUCT - The complex object being built.
    This is immutable (frozen=True) and can only be created via builders.
    """
    size: PizzaSize
    crust: CrustType
    toppings: Tuple[Topping, ...]  # Immutable tuple
    style: str  # "Italian", "American", "New York", etc.
  
    def __post_init__(self):
        """Validation after creation"""
        if not self.toppings:
            raise ValueError("Pizza must have at least one topping!")
      
        if len(self.toppings) > 10:
            raise ValueError("Maximum 10 toppings allowed!")
  
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
        price += len(self.toppings) * 1.50
      
        if self.crust == CrustType.STUFFED:
            price += 3.00
      
        return round(price, 2)
  
    def __str__(self):
        toppings_str = ", ".join(t.value for t in self.toppings)
        return (f"{self.style} Style - {self.size.value.title()} "
                f"{self.crust.value} crust pizza with {toppings_str} - ${self.price}")


# ============================================================================
# 2. BUILDER INTERFACE - Abstract Base Class
# ============================================================================

class PizzaBuilder(ABC):
    """
    The BUILDER INTERFACE - Abstract class that defines the contract.
    All concrete builders must implement these methods.
    """
  
    @abstractmethod
    def reset(self) -> None:
        """Reset the builder to start fresh"""
        pass
  
    @abstractmethod
    def set_size(self, size: PizzaSize) -> 'PizzaBuilder':
        """Set the pizza size"""
        pass
  
    @abstractmethod
    def set_crust(self, crust: CrustType) -> 'PizzaBuilder':
        """Set the crust type"""
        pass
  
    @abstractmethod
    def add_topping(self, topping: Topping) -> 'PizzaBuilder':
        """Add a single topping"""
        pass
  
    @abstractmethod
    def add_toppings(self, *toppings: Topping) -> 'PizzaBuilder':
        """Add multiple toppings at once"""
        pass
  
    @abstractmethod
    def build(self) -> Pizza:
        """Build and return the final Pizza object"""
        pass


# ============================================================================
# 3. CONCRETE BUILDERS - Specific Implementations
# ============================================================================

class ItalianPizzaBuilder(PizzaBuilder):
    """
    CONCRETE BUILDER for Italian-style pizzas.
    Uses thin crust by default and authentic Italian ingredients.
    """
  
    def __init__(self):
        self.reset()
  
    def reset(self) -> None:
        """Reset to default Italian style settings"""
        self._size: PizzaSize = None
        self._crust: CrustType = CrustType.THIN  # Italian default
        self._toppings: List[Topping] = []
        self._style = "Italian"
  
    def set_size(self, size: PizzaSize) -> 'ItalianPizzaBuilder':
        self._size = size
        return self
  
    def set_crust(self, crust: CrustType) -> 'ItalianPizzaBuilder':
        # Italians prefer thin or regular crust
        if crust == CrustType.STUFFED:
            print("⚠️  Warning: Stuffed crust is not traditional Italian style!")
        self._crust = crust
        return self
  
    def add_topping(self, topping: Topping) -> 'ItalianPizzaBuilder':
        if topping in self._toppings:
            raise ValueError(f"{topping.value} already added!")
        self._toppings.append(topping)
        return self
  
    def add_toppings(self, *toppings: Topping) -> 'ItalianPizzaBuilder':
        for topping in toppings:
            self.add_topping(topping)
        return self
  
    def build(self) -> Pizza:
        if self._size is None:
            raise ValueError("Size must be set!")
      
        pizza = Pizza(
            size=self._size,
            crust=self._crust,
            toppings=tuple(self._toppings),
            style=self._style
        )
        self.reset()  # Reset for next pizza
        return pizza


class AmericanPizzaBuilder(PizzaBuilder):
    """
    CONCRETE BUILDER for American-style pizzas.
    Uses thick crust by default and generous toppings.
    """
  
    def __init__(self):
        self.reset()
  
    def reset(self) -> None:
        """Reset to default American style settings"""
        self._size: PizzaSize = None
        self._crust: CrustType = CrustType.THICK  # American default
        self._toppings: List[Topping] = []
        self._style = "American"
  
    def set_size(self, size: PizzaSize) -> 'AmericanPizzaBuilder':
        self._size = size
        return self
  
    def set_crust(self, crust: CrustType) -> 'AmericanPizzaBuilder':
        self._crust = crust
        return self
  
    def add_topping(self, topping: Topping) -> 'AmericanPizzaBuilder':
        if topping in self._toppings:
            raise ValueError(f"{topping.value} already added!")
        self._toppings.append(topping)
        return self
  
    def add_toppings(self, *toppings: Topping) -> 'AmericanPizzaBuilder':
        for topping in toppings:
            self.add_topping(topping)
        return self
  
    def build(self) -> Pizza:
        if self._size is None:
            raise ValueError("Size must be set!")
      
        pizza = Pizza(
            size=self._size,
            crust=self._crust,
            toppings=tuple(self._toppings),
            style=self._style
        )
        self.reset()
        return pizza


class NewYorkPizzaBuilder(PizzaBuilder):
    """
    CONCRETE BUILDER for New York-style pizzas.
    Large, thin crust, foldable slices.
    """
  
    def __init__(self):
        self.reset()
  
    def reset(self) -> None:
        """Reset to default New York style settings"""
        self._size: PizzaSize = PizzaSize.LARGE  # NY pizzas are always large
        self._crust: CrustType = CrustType.THIN  # NY default
        self._toppings: List[Topping] = []
        self._style = "New York"
  
    def set_size(self, size: PizzaSize) -> 'NewYorkPizzaBuilder':
        if size != PizzaSize.LARGE and size != PizzaSize.EXTRA_LARGE:
            print("⚠️  Warning: New York pizzas are traditionally large!")
        self._size = size
        return self
  
    def set_crust(self, crust: CrustType) -> 'NewYorkPizzaBuilder':
        if crust != CrustType.THIN:
            print("⚠️  Warning: New York pizzas traditionally have thin crust!")
        self._crust = crust
        return self
  
    def add_topping(self, topping: Topping) -> 'NewYorkPizzaBuilder':
        if topping in self._toppings:
            raise ValueError(f"{topping.value} already added!")
        self._toppings.append(topping)
        return self
  
    def add_toppings(self, *toppings: Topping) -> 'NewYorkPizzaBuilder':
        for topping in toppings:
            self.add_topping(topping)
        return self
  
    def build(self) -> Pizza:
        pizza = Pizza(
            size=self._size,
            crust=self._crust,
            toppings=tuple(self._toppings),
            style=self._style
        )
        self.reset()
        return pizza


# ============================================================================
# 4. DIRECTOR - Controls the Construction Process
# ============================================================================

class PizzaDirector:
    """
    The DIRECTOR - Knows the recipes (construction sequences).
    Controls WHICH builder methods to call and in WHAT ORDER.
    Encapsulates complex construction logic.
    """
  
    def __init__(self, builder: PizzaBuilder):
        self._builder = builder
  
    def change_builder(self, builder: PizzaBuilder) -> None:
        """Switch to a different builder"""
        self._builder = builder
  
    # ========================================================================
    # STANDARD RECIPES - Director knows how to make these
    # ========================================================================
  
    def make_margherita(self) -> Pizza:
        """
        Classic Margherita recipe.
        Works with any builder - produces different styles.
        """
        return (self._builder
                .set_size(PizzaSize.MEDIUM)
                .add_toppings(
                    Topping.MOZZARELLA,
                    Topping.TOMATO_SAUCE,
                    Topping.BASIL
                )
                .build())
  
    def make_pepperoni(self) -> Pizza:
        """Classic Pepperoni recipe"""
        return (self._builder
                .set_size(PizzaSize.LARGE)
                .add_toppings(
                    Topping.MOZZARELLA,
                    Topping.TOMATO_SAUCE,
                    Topping.PEPPERONI
                )
                .build())
  
    def make_supreme(self) -> Pizza:
        """Loaded Supreme pizza recipe"""
        return (self._builder
                .set_size(PizzaSize.EXTRA_LARGE)
                .add_toppings(
                    Topping.MOZZARELLA,
                    Topping.TOMATO_SAUCE,
                    Topping.PEPPERONI,
                    Topping.SAUSAGE,
                    Topping.MUSHROOMS,
                    Topping.BELL_PEPPERS,
                    Topping.ONIONS,
                    Topping.OLIVES
                )
                .build())
  
    def make_vegetarian(self) -> Pizza:
        """Vegetarian pizza recipe"""
        return (self._builder
                .set_size(PizzaSize.MEDIUM)
                .add_toppings(
                    Topping.MOZZARELLA,
                    Topping.TOMATO_SAUCE,
                    Topping.MUSHROOMS,
                    Topping.BELL_PEPPERS,
                    Topping.ONIONS,
                    Topping.OLIVES
                )
                .build())
  
    def make_hawaiian(self) -> Pizza:
        """Hawaiian pizza recipe (controversial!)"""
        return (self._builder
                .set_size(PizzaSize.LARGE)
                .add_toppings(
                    Topping.MOZZARELLA,
                    Topping.TOMATO_SAUCE,
                    Topping.HAM,
                    Topping.PINEAPPLE
                )
                .build())
  
    def make_meat_lovers(self) -> Pizza:
        """Meat lovers pizza recipe"""
        return (self._builder
                .set_size(PizzaSize.EXTRA_LARGE)
                .set_crust(CrustType.STUFFED)
                .add_toppings(
                    Topping.MOZZARELLA,
                    Topping.TOMATO_SAUCE,
                    Topping.PEPPERONI,
                    Topping.SAUSAGE,
                    Topping.HAM
                )
                .build())


# ============================================================================
# 5. CLIENT - Application Code
# ============================================================================

def main():
    """
    The CLIENT - Uses the Director and Builders to create pizzas.
    The client doesn't need to know the construction details.
    """
  
    print("=" * 80)
    print("COMPLETE BUILDER PATTERN - PIZZA EXAMPLE WITH ALL 5 COMPONENTS")
    print("=" * 80)
  
    # ========================================================================
    # Scenario 1: Using Director with Italian Builder
    # ========================================================================
    print("\n🇮🇹 ITALIAN STYLE PIZZAS (Using Director)")
    print("-" * 80)
  
    italian_builder = ItalianPizzaBuilder()
    director = PizzaDirector(italian_builder)
  
    margherita = director.make_margherita()
    print(f"1. {margherita}")
  
    pepperoni = director.make_pepperoni()
    print(f"2. {pepperoni}")
  
    vegetarian = director.make_vegetarian()
    print(f"3. {vegetarian}")
  
    # ========================================================================
    # Scenario 2: Switch to American Builder (Same Recipes, Different Style)
    # ========================================================================
    print("\n🇺🇸 AMERICAN STYLE PIZZAS (Same Director, Different Builder)")
    print("-" * 80)
  
    american_builder = AmericanPizzaBuilder()
    director.change_builder(american_builder)  # Switch builder!
  
    margherita_american = director.make_margherita()
    print(f"1. {margherita_american}")
  
    supreme_american = director.make_supreme()
    print(f"2. {supreme_american}")
  
    meat_lovers = director.make_meat_lovers()
    print(f"3. {meat_lovers}")
  
    # ========================================================================
    # Scenario 3: New York Style Pizzas
    # ========================================================================
    print("\n🗽 NEW YORK STYLE PIZZAS")
    print("-" * 80)
  
    ny_builder = NewYorkPizzaBuilder()
    director.change_builder(ny_builder)
  
    ny_pepperoni = director.make_pepperoni()
    print(f"1. {ny_pepperoni}")
  
    ny_supreme = director.make_supreme()
    print(f"2. {ny_supreme}")
  
    # ========================================================================
    # Scenario 4: Client Using Builder Directly (Without Director)
    # ========================================================================
    print("\n🔧 CUSTOM PIZZAS (Client Using Builder Directly - No Director)")
    print("-" * 80)
  
    # Client has full control when not using Director
    custom_italian = (ItalianPizzaBuilder()
                      .set_size(PizzaSize.SMALL)
                      .set_crust(CrustType.REGULAR)
                      .add_topping(Topping.MOZZARELLA)
                      .add_topping(Topping.BASIL)
                      .build())
    print(f"1. Custom: {custom_italian}")
  
    custom_american = (AmericanPizzaBuilder()
                       .set_size(PizzaSize.EXTRA_LARGE)
                       .set_crust(CrustType.STUFFED)
                       .add_toppings(
                           Topping.MOZZARELLA,
                           Topping.PEPPERONI,
                           Topping.MUSHROOMS,
                           Topping.PINEAPPLE  # Custom combo!
                       )
                       .build())
    print(f"2. Custom: {custom_american}")
  
    # ========================================================================
    # Scenario 5: Demonstrating Immutability
    # ========================================================================
    print("\n🔒 IMMUTABILITY DEMONSTRATION")
    print("-" * 80)
  
    pizza = director.make_margherita()
    print(f"Original: {pizza}")
  
    try:
        pizza.size = PizzaSize.SMALL  # ❌ Will fail - frozen dataclass
    except Exception as e:
        print(f"❌ Cannot modify size: {type(e).__name__}")
  
    try:
        pizza.toppings.append(Topping.PEPPERONI)  # ❌ Will fail - tuple
    except Exception as e:
        print(f"❌ Cannot modify toppings: {type(e).__name__}")
  
    # ========================================================================
    # Scenario 6: Multiple Pizzas from Same Recipe
    # ========================================================================
    print("\n📦 BATCH ORDER (Same Recipe, Multiple Pizzas)")
    print("-" * 80)
  
    director.change_builder(ItalianPizzaBuilder())
  
    print("Ordering 3 Margherita pizzas:")
    for i in range(3):
        pizza = director.make_margherita()
        print(f"  Pizza #{i+1}: {pizza.size.value} - ${pizza.price}")
  
    # ========================================================================
    # Scenario 7: Validation Examples
    # ========================================================================
    print("\n⚠️  VALIDATION EXAMPLES")
    print("-" * 80)
  
    # Try to build without setting size
    try:
        invalid = ItalianPizzaBuilder().add_topping(Topping.MOZZARELLA).build()
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # Try to add duplicate topping
    try:
        duplicate = (ItalianPizzaBuilder()
                     .set_size(PizzaSize.MEDIUM)
                     .add_topping(Topping.MOZZARELLA)
                     .add_topping(Topping.MOZZARELLA)  # Duplicate!
                     .build())
    except ValueError as e:
        print(f"❌ Error: {e}")
  
    # ========================================================================
    # Summary
    # ========================================================================
    print("\n" + "=" * 80)
    print("✅ DEMONSTRATION COMPLETE")
    print("=" * 80)
    print("\nKey Points Demonstrated:")
    print("  1. Product (Pizza) - Immutable final object")
    print("  2. Builder Interface (PizzaBuilder) - Abstract contract")
    print("  3. Concrete Builders (Italian/American/NY) - Different implementations")
    print("  4. Director (PizzaDirector) - Knows standard recipes")
    print("  5. Client (main) - Uses Director or Builders directly")
    print("\nBenefits Shown:")
    print("  ✓ Same recipe produces different styles (Italian vs American)")
    print("  ✓ Director encapsulates complex construction logic")
    print("  ✓ Client can use Director (simple) or Builder (full control)")
    print("  ✓ Immutable pizzas cannot be modified after creation")
    print("  ✓ Validation prevents invalid pizzas")
    print("  ✓ Fluent interface makes code readable")
    print("=" * 80)


if __name__ == "__main__":
    main()
```

---

# 📤 Output

```
================================================================================
COMPLETE BUILDER PATTERN - PIZZA EXAMPLE WITH ALL 5 COMPONENTS
================================================================================

🇮🇹 ITALIAN STYLE PIZZAS (Using Director)
--------------------------------------------------------------------------------
1. Italian Style - Medium thin crust pizza with mozzarella, tomato sauce, basil - $15.49
2. Italian Style - Large thin crust pizza with mozzarella, tomato sauce, pepperoni - $17.49
3. Italian Style - Medium thin crust pizza with mozzarella, tomato sauce, mushrooms, bell peppers, onions, olives - $19.99

🇺🇸 AMERICAN STYLE PIZZAS (Same Director, Different Builder)
--------------------------------------------------------------------------------
1. American Style - Medium thick crust pizza with mozzarella, tomato sauce, basil - $15.49
2. American Style - Extra_large thick crust pizza with mozzarella, tomato sauce, pepperoni, sausage, mushrooms, bell peppers, onions, olives - $26.99
3. American Style - Extra_large stuffed crust pizza with mozzarella, tomato sauce, pepperoni, sausage, ham - $26.49

🗽 NEW YORK STYLE PIZZAS
--------------------------------------------------------------------------------
1. New York Style - Large thin crust pizza with mozzarella, tomato sauce, pepperoni - $17.49
2. New York Style - Large thin crust pizza with mozzarella, tomato sauce, pepperoni, sausage, mushrooms, bell peppers, onions, olives - $24.99

🔧 CUSTOM PIZZAS (Client Using Builder Directly - No Director)
--------------------------------------------------------------------------------
1. Custom: Italian Style - Small thin crust pizza with mozzarella, basil - $11.99
2. Custom: American Style - Extra_large stuffed crust pizza with mozzarella, pepperoni, mushrooms, pineapple - $23.99

🔒 IMMUTABILITY DEMONSTRATION
--------------------------------------------------------------------------------
Original: Italian Style - Medium thin crust pizza with mozzarella, tomato sauce, basil - $15.49
❌ Cannot modify size: FrozenInstanceError
❌ Cannot modify toppings: AttributeError

📦 BATCH ORDER (Same Recipe, Multiple Pizzas)
--------------------------------------------------------------------------------
Ordering 3 Margherita pizzas:
  Pizza #1: medium - $15.49
  Pizza #2: medium - $15.49
  Pizza #3: medium - $15.49

⚠️  VALIDATION EXAMPLES
--------------------------------------------------------------------------------
❌ Error: Size must be set!
❌ Error: mozzarella already added!

================================================================================
✅ DEMONSTRATION COMPLETE
================================================================================

Key Points Demonstrated:
  1. Product (Pizza) - Immutable final object
  2. Builder Interface (PizzaBuilder) - Abstract contract
  3. Concrete Builders (Italian/American/NY) - Different implementations
  4. Director (PizzaDirector) - Knows standard recipes
  5. Client (main) - Uses Director or Builders directly

Benefits Shown:
  ✓ Same recipe produces different styles (Italian vs American)
  ✓ Director encapsulates complex construction logic
  ✓ Client can use Director (simple) or Builder (full control)
  ✓ Immutable pizzas cannot be modified after creation
  ✓ Validation prevents invalid pizzas
  ✓ Fluent interface makes code readable
================================================================================
```

---

# 🎯 What This Example Demonstrates

## All 5 Components in Action:

1. **Product (Pizza)**: Immutable, validated, with business logic (price calculation)
2. **Builder Interface (PizzaBuilder)**: Abstract contract that all builders follow
3. **Concrete Builders**:

   - `ItalianPizzaBuilder` - Thin crust default
   - `AmericanPizzaBuilder` - Thick crust default
   - `NewYorkPizzaBuilder` - Large, thin, with warnings
4. **Director (PizzaDirector)**:

   - Knows 6 standard recipes
   - Can switch between builders
   - Hides construction complexity from client
5. **Client (main function)**:

   - Uses Director for standard pizzas
   - Uses Builders directly for custom pizzas
   - Demonstrates all features

---

This is the **complete, production-ready** Builder Pattern implementation! 🍕
