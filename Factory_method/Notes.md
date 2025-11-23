# Factory Method Pattern - Complete Notes with Simple Example

## 🎯 Core Concept

**Factory Method delegates object creation to subclasses.** The client doesn't create objects directly - it asks a "factory" (creator) to create them.

---

## 🔑 Key Difference from Simple Inheritance

| **Simple Inheritance**           | **Factory Method**                            |
| -------------------------------------- | --------------------------------------------------- |
| Client creates objects:`dog = Dog()` | Creator creates objects:`shelter.create_animal()` |
| Client knows concrete classes          | Client only knows creator interface                 |
| Just polymorphism                      | Polymorphism + Delegated creation                   |

---

## ❌ Problem Factory Method Solves

### Without Factory Method:

```python
# BAD: Client creates objects directly
class Application:
    def __init__(self, type: str):
        if type == "email":
            self.notifier = EmailNotifier()  # ❌ Knows concrete class
        elif type == "sms":
            self.notifier = SMSNotifier()    # ❌ Knows concrete class
        # Adding new type = MODIFY this code ❌

# Problems:
# 1. Client tightly coupled to concrete classes
# 2. Adding new type requires modifying existing code
# 3. Violates Open-Closed Principle
```

### With Factory Method:

```python
# GOOD: Creator creates objects
class EmailApplication(Application):
    def create_notifier(self):
        return EmailNotifier()  # ✅ Subclass decides

class SMSApplication(Application):
    def create_notifier(self):
        return SMSNotifier()    # ✅ Subclass decides

# Benefits:
# 1. Client doesn't know concrete classes
# 2. Adding new type = ADD new subclass (no modification)
# 3. Follows Open-Closed Principle
```

---

## 🏗️ Pattern Structure

```
┌─────────────────────────────────────┐
│         Creator (Abstract)          │
│                                     │
│  + factory_method() -> Product     │  ← Factory Method (abstract)
│  + operation()                     │  ← Uses factory_method()
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
┌──────▼──────┐  ┌──────▼──────┐
│ConcreteA    │  │ConcreteB    │
│             │  │             │
│factory()    │  │factory()    │
│ -> ProductA │  │ -> ProductB │
└─────────────┘  └─────────────┘
```

**4 Components:**

1. **Product** (interface) - What the factory creates
2. **Concrete Products** - Specific implementations
3. **Creator** (abstract) - Has factory method
4. **Concrete Creators** - Implement factory method

---

## 📝 Simple Example - Document Generator

```python
from abc import ABC, abstractmethod

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
# 3. CREATOR (Abstract) - Defines factory method
# ============================================================================

class DocumentCreator(ABC):
    """
    Creator - Abstract class with factory method.
  
    Key: create_document() is ABSTRACT - subclasses decide which document to create.
    """
  
    @abstractmethod
    def create_document(self) -> Document:
        """
        FACTORY METHOD - Subclasses implement this.
        Returns a Document object.
        """
        pass
  
    def generate_report(self, title: str, data: str) -> str:
        """
        Business logic that USES the factory method.
  
        This method is the SAME for all creators.
        It doesn't know which document type it will get.
        """
        # ✅ Call factory method (implemented by subclass)
        document = self.create_document()
  
        # Use the document (works with ANY Document type)
        format_type = document.get_format()
        content = document.create_content()
  
        # Generate report
        report = f"""
╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  {title:<50}                                          ║
║ Format: {format_type:<50}                                    ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ {content:<56}                                                ║
║                                                              ║
║ Data: {data:<54}                                             ║
╚══════════════════════════════════════════════════════════════╝
        """
        return report


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
# CLIENT CODE - Uses the factory
# ============================================================================

def main():
    print("=" * 80)
    print("FACTORY METHOD PATTERN - SIMPLE EXAMPLE")
    print("=" * 80)
  
    # ========================================================================
    # Example 1: Using different creators
    # ========================================================================
    print("\n1️⃣  GENERATING REPORTS IN DIFFERENT FORMATS")
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
    # Example 2: Runtime selection
    # ========================================================================
    print("\n2️⃣  RUNTIME SELECTION - Based on user preference")
    print("-" * 80)
  
    def get_creator(format_type: str) -> DocumentCreator:
        """Factory function - returns appropriate creator"""
        creators = {
            "pdf": PDFCreator,
            "word": WordCreator,
            "html": HTMLCreator
        }
        creator_class = creators.get(format_type.lower())
        if not creator_class:
            raise ValueError(f"Unknown format: {format_type}")
        return creator_class()
  
    # Simulate user choosing format
    user_format = "pdf"
    creator = get_creator(user_format)
    print(f"User selected: {user_format.upper()}")
    print(creator.generate_report("User Report", "Data from user"))
  
    # ========================================================================
    # Example 3: Adding new format (NO modification to existing code!)
    # ========================================================================
    print("\n3️⃣  ADDING NEW FORMAT - Markdown (no existing code modified!)")
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
    # Example 4: Polymorphism - treating all creators the same
    # ========================================================================
    print("\n4️⃣  POLYMORPHISM - All creators treated uniformly")
    print("-" * 80)
  
    # List of different creators
    creators = [
        PDFCreator(),
        WordCreator(),
        HTMLCreator(),
        MarkdownCreator()
    ]
  
    # Client code treats them all the same way
    for i, creator in enumerate(creators, 1):
        report = creator.generate_report(
            f"Report #{i}",
            f"Sample data {i}"
        )
        print(report)
  
    print("=" * 80)
    print("✅ DEMONSTRATION COMPLETE")
    print("=" * 80)


if __name__ == "__main__":
    main()
```

---

## 📤 Output

```
================================================================================
FACTORY METHOD PATTERN - SIMPLE EXAMPLE
================================================================================

1️⃣  GENERATING REPORTS IN DIFFERENT FORMATS
--------------------------------------------------------------------------------

╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  Sales Report Q4                                      ║
║ Format: PDF                                                  ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ PDF content with formatting and images                       ║
║                                                              ║
║ Data: Revenue: $1.2M                                         ║
╚══════════════════════════════════════════════════════════════╝
  

╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  Project Proposal                                     ║
║ Format: DOCX                                                 ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ Word content with styles and tables                          ║
║                                                              ║
║ Data: Budget: $500K                                          ║
╚══════════════════════════════════════════════════════════════╝
  

╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  Web Dashboard                                        ║
║ Format: HTML                                                 ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ <html><body>HTML content with tags</body></html>             ║
║                                                              ║
║ Data: Users: 10,000                                          ║
╚══════════════════════════════════════════════════════════════╝
  

2️⃣  RUNTIME SELECTION - Based on user preference
--------------------------------------------------------------------------------
User selected: PDF

╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  User Report                                          ║
║ Format: PDF                                                  ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ PDF content with formatting and images                       ║
║                                                              ║
║ Data: Data from user                                         ║
╚══════════════════════════════════════════════════════════════╝
  

3️⃣  ADDING NEW FORMAT - Markdown (no existing code modified!)
--------------------------------------------------------------------------------

╔══════════════════════════════════════════════════════════════╗
║ REPORT GENERATED                                             ║
╠══════════════════════════════════════════════════════════════╣
║ Title:  README                                               ║
║ Format: MD                                                   ║
╠══════════════════════════════════════════════════════════════╣
║ Content:                                                     ║
║ # Markdown content with **bold** and *italic*                ║
║                                                              ║
║ Data: Installation guide                                     ║
╚══════════════════════════════════════════════════════════════╝
  

✅ Added Markdown support without modifying existing code!

4️⃣  POLYMORPHISM - All creators treated uniformly
--------------------------------------------------------------------------------
[Reports for all 4 formats displayed...]

================================================================================
✅ DEMONSTRATION COMPLETE
================================================================================
```

---

## 🎯 Key Takeaways

### **The Pattern in 3 Steps:**

1. **Define Product interface** (Document)

   ```python
   class Document(ABC):
       @abstractmethod
       def create_content(self): pass
   ```
2. **Define Creator with factory method** (DocumentCreator)

   ```python
   class DocumentCreator(ABC):
       @abstractmethod
       def create_document(self) -> Document:  # Factory method
           pass

       def generate_report(self):  # Uses factory method
           doc = self.create_document()  # Calls subclass implementation
   ```
3. **Concrete Creators implement factory method**

   ```python
   class PDFCreator(DocumentCreator):
       def create_document(self) -> Document:
           return PDFDocument()  # Decides which product
   ```

---

## ✅ When to Use Factory Method

| **Use When**                           | **Don't Use When**           |
| -------------------------------------------- | ---------------------------------- |
| Need to add new types without modifying code | Only one product type              |
| Don't know exact type until runtime          | Simple object creation             |
| Want to delegate creation to subclasses      | No variation in creation logic     |
| Have family of related products              | Client can create objects directly |

---

## 🔄 Quick Comparison

```python
# ❌ WITHOUT Factory Method (Simple Inheritance)
doc = PDFDocument()  # Client creates directly
content = doc.create_content()

# ✅ WITH Factory Method
creator = PDFCreator()  # Client creates creator
doc = creator.create_document()  # Creator creates product
content = doc.create_content()

# Key: Extra layer of indirection
# Client → Creator → Product
```

---

## 💡 Remember

**Factory Method = "Don't call us, we'll call you"**

- Client doesn't create products directly
- Client creates creators
- Creators create products
- Adding new product = Add new creator (no modification!)

---

**This is the simplest, most practical explanation of Factory Method!** 🎯
