# Factory Method Pattern

## 🎯 Core Concept

**Factory Method is a creational pattern that combines a template method with delegated object creation.**

The creator class defines an algorithm (template method) that uses an object, but delegates the instantiation of that object to subclasses via an abstract factory method.

**Key Formula:**
```
Factory Method = Template Method + Creation Hook
   (pattern)    =   (algorithm)   +  (subclass decides which object)
```

---

## 🔑 What Makes It Factory Method (Critical!)

For code to be Factory Method pattern, it MUST have **BOTH**:

1. **Factory Method** (abstract) - Subclasses override this to return different products
2. **Template Method** - Uses the factory method within a larger algorithm

```python
class Creator(ABC):
    @abstractmethod
    def create_product(self) -> Product:
        """FACTORY METHOD - subclass decides which product"""
        pass
    
    def execute_workflow(self) -> str:
        """TEMPLATE METHOD - uses factory method in algorithm"""
        product = self.create_product()  # ← Calls factory method
        # ... rest of algorithm that uses product
        return result
```

**Without the template method, it's just an abstract interface, NOT Factory Method pattern!**

---

## ❌ Problem Factory Method Solves

### **The Real Problem: Code Duplication in Algorithms That Create Objects**

```python
# ❌ WITHOUT Factory Method - Code duplication
class PDFReportGenerator:
    def generate_report(self, title: str, data: str) -> str:
        document = PDFDocument()  # ← Hardcoded concrete class
        
        # Algorithm (duplicated across all generators):
        header = f"=== {title} ==="
        content = document.create_content()
        footer = "--- End ---"
        return f"{header}\n{content}\n{data}\n{footer}"

class WordReportGenerator:
    def generate_report(self, title: str, data: str) -> str:
        document = WordDocument()  # ← Hardcoded concrete class
        
        # Same algorithm (DUPLICATED!):
        header = f"=== {title} ==="
        content = document.create_content()
        footer = "--- End ---"
        return f"{header}\n{content}\n{data}\n{footer}"

# Problems:
# 1. Algorithm is duplicated in every generator class
# 2. Each class hardcodes its document type
# 3. Changing the algorithm requires modifying every class
# 4. Can't share common logic
```

### **✅ WITH Factory Method - Shared Algorithm, Delegated Creation**

```python
class ReportGenerator(ABC):
    """Creator - defines shared algorithm"""
    
    @abstractmethod
    def create_document(self) -> Document:
        """Factory method - subclass decides document type"""
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        """Template method - shared algorithm"""
        document = self.create_document()  # ← Delegates to subclass
        
        # Algorithm (defined ONCE, shared by all):
        header = f"=== {title} ==="
        content = document.create_content()
        footer = "--- End ---"
        return f"{header}\n{content}\n{data}\n{footer}"

class PDFReportGenerator(ReportGenerator):
    def create_document(self) -> Document:
        return PDFDocument()  # ← Only this differs

class WordReportGenerator(ReportGenerator):
    def create_document(self) -> Document:
        return WordDocument()  # ← Only this differs

# Benefits:
# ✅ Algorithm defined once (no duplication)
# ✅ Subclasses only specify which document type
# ✅ Changing algorithm = modify one place
# ✅ Adding new generator = add new subclass (no modification to existing code)
```

---

## 🏗️ Pattern Structure

```
┌─────────────────────────────────────────────────────────┐
│              Creator (Abstract)                         │
│                                                         │
│  + create_product() -> Product  ← FACTORY METHOD       │
│  + template_method()             ← TEMPLATE METHOD      │
│      calls create_product()                             │
└──────────────────┬──────────────────────────────────────┘
                   │
           ┌───────┴────────┐
           │                │
┌──────────▼─────┐  ┌───────▼──────┐
│ ConcreteA      │  │ ConcreteB    │
│                │  │              │
│ create_product │  │create_product│
│  -> ProductA   │  │ -> ProductB  │
└────────────────┘  └──────────────┘
           │                │
           └───────┬────────┘
                   │
           ┌───────▼────────┐
           │    Product     │
           │  (interface)   │
           └────────────────┘
```

**4 Components:**

1. **Product** (interface) - What the factory creates
2. **Concrete Products** - Specific implementations (PDFDocument, WordDocument)
3. **Creator** (abstract) - Has factory method + template method
4. **Concrete Creators** - Implement factory method only

---

## 📝 Complete Example - Document Generator

```python
from abc import ABC, abstractmethod
from datetime import datetime
from typing import List

# ============================================================================
# 1. PRODUCT INTERFACE - What the factory creates
# ============================================================================

class Document(ABC):
    """Product interface - all documents must implement these methods"""
    
    @abstractmethod
    def create_content(self) -> str:
        """Generate document content"""
        pass
    
    @abstractmethod
    def get_format(self) -> str:
        """Get document format"""
        pass


# ============================================================================
# 2. CONCRETE PRODUCTS - Specific document types
# ============================================================================

class PDFDocument(Document):
    """Concrete Product - PDF document"""
    
    def create_content(self) -> str:
        return "PDF content with formatting and images"
    
    def get_format(self) -> str:
        return "PDF"


class WordDocument(Document):
    """Concrete Product - Word document"""
    
    def create_content(self) -> str:
        return "Word content with styles and tables"
    
    def get_format(self) -> str:
        return "DOCX"


class HTMLDocument(Document):
    """Concrete Product - HTML document"""
    
    def create_content(self) -> str:
        return "<html><body>HTML content with tags</body></html>"
    
    def get_format(self) -> str:
        return "HTML"


# ============================================================================
# 3. CREATOR (Abstract) - Defines factory method + template method
# ============================================================================

class DocumentCreator(ABC):
    """
    Creator - Abstract class with factory method and template method.
    
    This is the heart of Factory Method pattern:
    - create_document() is the FACTORY METHOD (abstract - subclasses implement)
    - generate_report() is the TEMPLATE METHOD (concrete - uses factory method)
    """
    
    @abstractmethod
    def create_document(self) -> Document:
        """
        FACTORY METHOD - Subclasses implement this.
        Returns a Document object.
        
        This is the "hook" that subclasses customize.
        """
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        """
        TEMPLATE METHOD - Defines the algorithm skeleton.
        
        This method is SHARED by all creators.
        It uses the factory method but doesn't know which concrete document it gets.
        
        This is why Factory Method exists - to share this algorithm!
        """
        # Step 1: Create document (delegated to subclass)
        document = self.create_document()  # ← Calls factory method
        
        # Step 2: Get document properties
        format_type = document.get_format()
        content = document.create_content()
        
        # Step 3: Generate report (shared algorithm)
        report = f"""
╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  {title:<50} ║
║ Format: {format_type:<50} ║
║ Time:   {datetime.now().strftime('%Y-%m-%d %H:%M:%S'):<50} ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ {content:<56} ║
║                                                              ║
║ Data: {data:<54} ║
╚══════════════════════════════════════════════════════════════╝
        """
        return report
    
    def _create_header(self, title: str) -> str:
        """
        Hook method - subclasses CAN override (but don't have to).
        Default implementation provided.
        """
        return f"=== {title} ==="
    
    def _create_footer(self) -> str:
        """
        Hook method - subclasses CAN override (but don't have to).
        Default implementation provided.
        """
        return "--- End of Report ---"


# ============================================================================
# 4. CONCRETE CREATORS - Implement factory method
# ============================================================================

class PDFCreator(DocumentCreator):
    """Concrete Creator - Creates PDF documents"""
    
    def create_document(self) -> Document:
        """Factory method implementation - returns PDFDocument"""
        return PDFDocument()


class WordCreator(DocumentCreator):
    """Concrete Creator - Creates Word documents"""
    
    def create_document(self) -> Document:
        """Factory method implementation - returns WordDocument"""
        return WordDocument()


class HTMLCreator(DocumentCreator):
    """Concrete Creator - Creates HTML documents"""
    
    def create_document(self) -> Document:
        """Factory method implementation - returns HTMLDocument"""
        return HTMLDocument()


# ============================================================================
# ADVANCED: Concrete creator with customization
# ============================================================================

class FancyPDFCreator(DocumentCreator):
    """
    Demonstrates that subclasses can:
    1. Implement factory method (required)
    2. Override hook methods (optional)
    3. Have their own state (optional)
    """
    
    def __init__(self, compression: bool = False):
        """Subclass can have its own configuration"""
        self.compression = compression
    
    def create_document(self) -> Document:
        """Factory method - uses instance state"""
        doc = PDFDocument()
        # Could configure document based on self.compression
        return doc
    
    def _create_header(self, title: str) -> str:
        """Override hook method for fancy header"""
        return f"╔══ {title.upper()} ══╗"
    
    def _create_footer(self) -> str:
        """Override hook method for fancy footer"""
        return "╚════════════════════╝"


# ============================================================================
# CLIENT CODE - Uses the factory
# ============================================================================

def main():
    print("=" * 80)
    print("FACTORY METHOD PATTERN - COMPLETE EXAMPLE")
    print("=" * 80)
    
    # ========================================================================
    # Example 1: Basic usage - different creators
    # ========================================================================
    print("\n1️⃣  BASIC USAGE - Different document formats")
    print("-" * 80)
    
    # Client works with DocumentCreator interface
    # Client doesn't know about PDFDocument, WordDocument, HTMLDocument
    
    pdf_creator = PDFCreator()
    print(pdf_creator.generate_report("Sales Report Q4", "Revenue: $1.2M"))
    
    word_creator = WordCreator()
    print(word_creator.generate_report("Project Proposal", "Budget: $500K"))
    
    html_creator = HTMLCreator()
    print(html_creator.generate_report("Web Dashboard", "Users: 10,000"))
    
    # ========================================================================
    # Example 2: Polymorphism - treating all creators uniformly
    # ========================================================================
    print("\n2️⃣  POLYMORPHISM - All creators treated the same")
    print("-" * 80)
    
    # List of different creators
    creators: List[DocumentCreator] = [
        PDFCreator(),
        WordCreator(),
        HTMLCreator(),
    ]
    
    # Client code treats them all identically
    for i, creator in enumerate(creators, 1):
        report = creator.generate_report(
            f"Report #{i}",
            f"Sample data {i}"
        )
        print(report)
    
    # ========================================================================
    # Example 3: Adding new format (Open-Closed Principle)
    # ========================================================================
    print("\n3️⃣  EXTENSIBILITY - Adding Markdown (no existing code modified!)")
    print("-" * 80)
    
    # Step 1: Create new Product
    class MarkdownDocument(Document):
        def create_content(self) -> str:
            return "# Markdown content with **bold** and *italic*"
        
        def get_format(self) -> str:
            return "MD"
    
    # Step 2: Create new Creator
    class MarkdownCreator(DocumentCreator):
        def create_document(self) -> Document:
            return MarkdownDocument()
    
    # ✅ NO modification to existing code!
    # ✅ Just added new classes
    
    markdown_creator = MarkdownCreator()
    print(markdown_creator.generate_report("README", "Installation guide"))
    
    print("\n✅ Added Markdown support without modifying existing code!")
    
    # ========================================================================
    # Example 4: Subclass customization
    # ========================================================================
    print("\n4️⃣  CUSTOMIZATION - Fancy PDF with overridden hooks")
    print("-" * 80)
    
    fancy_creator = FancyPDFCreator(compression=True)
    print(fancy_creator.generate_report("Executive Summary", "Q4 Results"))
    
    print("=" * 80)
    print("✅ DEMONSTRATION COMPLETE")
    print("=" * 80)


if __name__ == "__main__":
    main()
```

---

## 📤 Expected Output

```
================================================================================
FACTORY METHOD PATTERN - COMPLETE EXAMPLE
================================================================================

1️⃣  BASIC USAGE - Different document formats
--------------------------------------------------------------------------------
[PDF report displayed with format, content, data]
[Word report displayed with format, content, data]
[HTML report displayed with format, content, data]

2️⃣  POLYMORPHISM - All creators treated the same
--------------------------------------------------------------------------------
[Reports for all 3 formats displayed uniformly]

3️⃣  EXTENSIBILITY - Adding Markdown (no existing code modified!)
--------------------------------------------------------------------------------
[Markdown report displayed]
✅ Added Markdown support without modifying existing code!

4️⃣  CUSTOMIZATION - Fancy PDF with overridden hooks
--------------------------------------------------------------------------------
[Fancy PDF report with custom header/footer]

================================================================================
✅ DEMONSTRATION COMPLETE
================================================================================
```

---

## 🎯 The Pattern in 3 Steps

### **Step 1: Define Product Interface**

```python
class Document(ABC):
    @abstractmethod
    def create_content(self) -> str:
        pass
    
    @abstractmethod
    def get_format(self) -> str:
        pass
```

### **Step 2: Define Creator with Factory Method + Template Method**

```python
class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        """FACTORY METHOD - subclass implements"""
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        """TEMPLATE METHOD - uses factory method"""
        doc = self.create_document()  # ← Calls factory method
        # ... shared algorithm
        return result
```

### **Step 3: Concrete Creators Implement Factory Method**

```python
class PDFCreator(DocumentCreator):
    def create_document(self) -> Document:
        return PDFDocument()  # ← Only this differs between creators
```

---

## 🔄 Factory Method vs Simple Factory

### **When to Use Simple Factory (More Pythonic)**

```python
# ✅ Use simple factory when:
# - Just need to create objects based on type
# - No complex algorithm that uses the object
# - No need for subclass customization

def create_document(doc_type: str) -> Document:
    """Simple factory function"""
    factories = {
        'pdf': PDFDocument,
        'word': WordDocument,
        'html': HTMLDocument
    }
    factory = factories.get(doc_type.lower())
    if not factory:
        raise ValueError(f"Unknown document type: {doc_type}")
    return factory()

# Usage:
doc = create_document('pdf')
content = doc.create_content()
```

### **When to Use Factory Method (Pattern)**

```python
# ✅ Use Factory Method when:
# - Have a template method (algorithm) that uses the created object
# - Need multiple customization points (not just object creation)
# - Subclasses need their own state/configuration
# - Want type hierarchy for polymorphism

class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        """Complex algorithm that uses document"""
        doc = self.create_document()
        
        # Multi-step algorithm:
        header = self._create_header(title)
        content = doc.create_content()
        footer = self._create_footer()
        validated = self._validate(content)
        
        return f"{header}\n{validated}\n{footer}"
```

### **Decision Matrix**

| **Scenario** | **Use** | **Why** |
|--------------|---------|---------|
| Just creating objects | Simple factory function | Simpler, more Pythonic |
| Algorithm uses created object | Factory Method pattern | Shares algorithm, delegates creation |
| Need multiple hooks | Factory Method pattern | Subclasses customize multiple points |
| Runtime type selection only | Simple factory function | No need for class hierarchy |
| Subclasses need state | Factory Method pattern | Each subclass has configuration |

---

## 💡 Why Factory Method Uses Inheritance Over Composition

### **Reason 1: Multiple Customization Points**

```python
class ReportGenerator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        """Hook 1: Document creation"""
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        """Template method"""
        doc = self.create_document()
        header = self._create_header(title)  # Hook 2
        footer = self._create_footer()        # Hook 3
        return f"{header}\n{doc.create_content()}\n{footer}"
    
    def _create_header(self, title: str) -> str:
        """Hook 2: Header creation (can override)"""
        return f"=== {title} ==="
    
    def _create_footer(self) -> str:
        """Hook 3: Footer creation (can override)"""
        return "--- End ---"

class FancyPDFGenerator(ReportGenerator):
    def create_document(self) -> Document:
        return PDFDocument()
    
    def _create_header(self, title: str) -> str:
        """Override header"""
        return f"╔══ {title} ══╗"
    
    def _create_footer(self) -> str:
        """Override footer"""
        return "╚════════════╝"

# With composition, you'd need to pass multiple functions - messy!
```

### **Reason 2: Type Hierarchy (Polymorphism)**

```python
def process_reports(generators: List[ReportGenerator]):
    """Accepts ANY ReportGenerator subclass"""
    for gen in generators:
        report = gen.generate_report("Title", "data")
        print(report)

# Can pass different types:
process_reports([
    PDFGenerator(),
    WordGenerator(),
    HTMLGenerator()
])

# All are ReportGenerator types - polymorphic!
```

### **Reason 3: Subclass-Specific State**

```python
class PDFGenerator(ReportGenerator):
    def __init__(self, compression: bool = False, quality: int = 95):
        self.compression = compression  # PDF-specific config
        self.quality = quality
    
    def create_document(self) -> Document:
        return PDFDocument(
            compression=self.compression,
            quality=self.quality
        )

class WordGenerator(ReportGenerator):
    def __init__(self, template_path: str):
        self.template_path = template_path  # Word-specific config
    
    def create_document(self) -> Document:
        return WordDocument(template=self.template_path)

# Each subclass has DIFFERENT configuration needs
```

---

## 🔗 Factory Method and Dependency Inversion Principle

### **How Factory Method Implements DIP**

**Dependency Inversion Principle:**
> High-level modules should not depend on low-level modules. Both should depend on abstractions.

```python
# ❌ VIOLATES DIP (no Factory Method):
class ReportService:
    """High-level module"""
    
    def generate_report(self, title: str, data: str) -> str:
        doc = PDFDocument()  # ❌ Depends on concrete class (low-level)
        content = doc.create_content()
        return f"{title}: {content}"

# Problem: High-level (ReportService) depends on low-level (PDFDocument)


# ✅ FOLLOWS DIP (with Factory Method):
class ReportService(ABC):
    """High-level module - depends on abstraction"""
    
    @abstractmethod
    def create_document(self) -> Document:
        """Factory method - returns abstraction"""
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        doc = self.create_document()  # ✅ Depends on abstraction (Document)
        content = doc.create_content()
        return f"{title}: {content}"

class PDFReportService(ReportService):
    """Low-level module - provides concrete implementation"""
    
    def create_document(self) -> Document:
        return PDFDocument()

# Now:
# - ReportService (high-level) depends on Document (abstraction) ✅
# - PDFDocument (low-level) implements Document (abstraction) ✅
# - Dependency arrow points UP (from concrete to abstract) ✅
```

### **Dependency Flow Visualization**

```
WITHOUT Factory Method (violates DIP):
┌─────────────────┐
│  ReportService  │ (high-level)
└────────┬────────┘
         │ depends on (BAD!)
         ▼
┌─────────────────┐
│  PDFDocument    │ (low-level)
└─────────────────┘


WITH Factory Method (follows DIP):
┌─────────────────┐
│  ReportService  │ (high-level)
└────────┬────────┘
         │ depends on
         ▼
┌─────────────────┐
│    Document     │ (abstraction)
└────────▲────────┘
         │ implements
         │
┌────────┴────────┐
│  PDFDocument    │ (low-level)
└─────────────────┘

Dependency is INVERTED - points upward!
```

---

## ✅ When to Use Factory Method

| **Use Factory Method When** | **Don't Use When** |
|------------------------------|-------------------|
| You have a template method (algorithm) that uses an object | Just need to create objects based on type |
| Need to share algorithm across multiple creators | Creation logic is simple |
| Subclasses need to customize multiple methods | Only one product type exists |
| Want type hierarchy for polymorphism | Client can create objects directly |
| Subclasses have their own state/configuration | No variation in how object is used |

### **Real-World Examples Where Factory Method Fits:**

1. **Django Class-Based Views**
   - `dispatch()` is template method
   - `get_handler()` is factory method
   - Returns different handlers (GET, POST, etc.)

2. **Database Connection Managers**
   - `execute_query()` is template method
   - `create_connection()` is factory method
   - Returns different connections (Postgres, MySQL, SQLite)

3. **Report Generators**
   - `generate_report()` is template method
   - `create_document()` is factory method
   - Returns different documents (PDF, Word, HTML)

4. **Game Character Systems**
   - `run_battle()` is template method
   - `create_weapon()` is factory method
   - Returns different weapons (Sword, Bow, Staff)

---

## 🚫 Common Mistakes and Anti-Patterns

### **Mistake 1: No Template Method (Not Factory Method!)**

```python
# ❌ This is NOT Factory Method - just an abstract interface
class AnimalFactory(ABC):
    @abstractmethod
    def create_animal(self) -> Animal:
        pass
    # ❌ No method that USES create_animal()

class DogFactory(AnimalFactory):
    def create_animal(self) -> Animal:
        return Dog()

# This is just an abstract factory interface, not the pattern!
```

### **Mistake 2: Template Method Has No Logic (Defeats Purpose)**

```python
# ❌ Template method is too simple - use simple factory instead
class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    def generate_report(self, data: str) -> str:
        doc = self.create_document()
        return doc.create_content()  # ❌ Too simple! Just one line!

# If template method is this simple, use a simple factory function instead
```

### **Mistake 3: All Logic in Product (Wrong Responsibility)**

```python
# ❌ Anti-pattern: Template method does nothing, product does everything
class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        doc = self.create_document()
        return doc.generate(title, data)  # ❌ Product does all the work!

class PDFDocument(Document):
    def generate(self, title: str, data: str) -> str:
        # ❌ All the algorithm is HERE, not in creator!
        header = self._create_header(title)
        content = self._format_content(data)
        footer = self._create_footer()
        return f"{header}\n{content}\n{footer}"

# This defeats the purpose! The algorithm should be in the creator's
# template method, not in the product.
```

### **Mistake 4: Factory Method Has Conditional Logic**

```python
# ❌ Factory method shouldn't have if/else - that's a simple factory!
class PDFCreator(DocumentCreator):
    def __init__(self, compression: bool):
        self.compression = compression
    
    def create_document(self) -> Document:
        # ❌ Conditional logic in factory method
        if self.compression:
            return CompressedPDFDocument()
        else:
            return PDFDocument()

# If you need conditionals, the factory method itself might be wrong.
# Consider whether you need separate creator subclasses instead.
```

---

## 🎓 Key Takeaways

### **1. Factory Method = Template Method + Creation Hook**

The pattern exists to share an algorithm while delegating object creation.

### **2. Must Have BOTH Components**

- Factory method (abstract - subclass implements)
- Template method (concrete - uses factory method)

Without both, it's not Factory Method pattern.

### **3. Prefer Simple Factory for Simple Cases**

If you just need to create objects, use a simple factory function. Factory Method is for when you have a template algorithm.

### **4. Inheritance Provides Multiple Benefits**

- Multiple customization points
- Type hierarchy (polymorphism)
- Subclass-specific state
- Open-Closed Principle compliance

### **5. Implements Dependency Inversion Principle**

High-level modules depend on abstractions, low-level modules provide implementations.

---

## 🔄 Pattern Comparison Quick Reference

```python
# Simple Factory (not a GoF pattern):
def create_document(type: str) -> Document:
    """Just returns objects"""
    return {'pdf': PDFDocument, 'word': WordDocument}[type]()


# Factory Method (GoF pattern):
class DocumentCreator(ABC):
    """Template method + factory method"""
    @abstractmethod
    def create_document(self) -> Document: pass
    
    def generate_report(self):
        doc = self.create_document()  # Uses created object in algorithm
        # ... complex algorithm


# Abstract Factory (GoF pattern):
class DocumentFactory(ABC):
    """Creates families of related objects"""
    @abstractmethod
    def create_document(self) -> Document: pass
    @abstractmethod
    def create_template(self) -> Template: pass
    @abstractmethod
    def create_stylesheet(self) -> Stylesheet: pass
```

---

## 💡 Mental Model

**Think of Factory Method as:**

> "I have an algorithm (template method) that needs to create and use an object, but I don't want to hardcode which type of object. So I'll let my subclasses decide (via factory method) which object to create, while I keep the algorithm the same."

**The Hollywood Principle:** *"Don't call us, we'll call you"*

- Client doesn't create products directly
- Client creates creators
- Creators create products (via factory method)
- Creators use products (in template method)
