# Factory Method Pattern - Let's Master This

Alright, let's dive into the Factory Method pattern. I'll be direct with you - this slide gives you the textbook definition, but I'm going to show you **why** you actually need it and **when** to use it in real Python code.

---

## 🎯 The Core Problem Factory Method Solves

**Problem:** You need to create objects, but you don't know **which exact class** to instantiate until runtime.

Let me show you the problem first, then the solution.

---

## ❌ The Problem - Without Factory Method

```python
# BAD: Client code directly creates objects
class Application:
    def __init__(self, notification_type: str):
        # ❌ Client knows about ALL concrete classes
        # ❌ Adding new notification types requires modifying this code
        if notification_type == "email":
            self.notification = EmailNotification()
        elif notification_type == "sms":
            self.notification = SMSNotification()
        elif notification_type == "push":
            self.notification = PushNotification()
        else:
            raise ValueError(f"Unknown type: {notification_type}")
  
    def send_alert(self, message: str):
        self.notification.send(message)


# What's wrong here?
# 1. Application class is TIGHTLY COUPLED to all notification classes
# 2. Adding SlackNotification requires MODIFYING Application class
# 3. Violates Open-Closed Principle (open for extension, closed for modification)
# 4. Client code has too much knowledge about concrete implementations
```

**This violates the Open-Closed Principle** - you can't add new notification types without modifying existing code.

---

## ✅ The Solution - Factory Method Pattern

The Factory Method pattern says:

> **"Define an interface for creating objects, but let subclasses decide which class to instantiate."**

In simpler terms: **Delegate object creation to subclasses.**

---

## 📝 Factory Method Pattern Structure

```
┌─────────────────────────────────────────────────────────────┐
│                         Creator                             │
│  (Abstract class with factory method)                       │
│                                                             │
│  + factory_method() -> Product  [ABSTRACT]                  │
│  + some_operation()              [Uses factory_method()]    │
└──────────────────┬──────────────────────────────────────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
┌────────▼────────┐  ┌───────▼─────────┐
│ ConcreteCreatorA│  │ ConcreteCreatorB│
│                 │  │                 │
│ + factory_method│  │ + factory_method│
│   -> ProductA   │  │   -> ProductB   │
└─────────────────┘  └─────────────────┘
         │                   │
         │ creates           │ creates
         ▼                   ▼
┌─────────────────┐  ┌─────────────────┐
│    ProductA     │  │    ProductB     │
└─────────────────┘  └─────────────────┘
```

---

## 💡 Complete Example - Notification System

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List

# ============================================================================
# PRODUCT INTERFACE - What the factory creates
# ============================================================================

class Notification(ABC):
    """
    PRODUCT INTERFACE - Defines what all notifications must do.
    This is what the factory method returns.
    """
  
    @abstractmethod
    def send(self, message: str, recipient: str) -> bool:
        """Send notification to recipient"""
        pass
  
    @abstractmethod
    def get_delivery_status(self) -> str:
        """Get delivery status"""
        pass


# ============================================================================
# CONCRETE PRODUCTS - Specific implementations
# ============================================================================

class EmailNotification(Notification):
    """CONCRETE PRODUCT - Email implementation"""
  
    def __init__(self):
        self.sent = False
  
    def send(self, message: str, recipient: str) -> bool:
        print(f"📧 Sending EMAIL to {recipient}")
        print(f"   Subject: Notification")
        print(f"   Body: {message}")
        self.sent = True
        return True
  
    def get_delivery_status(self) -> str:
        return "Email delivered" if self.sent else "Email pending"


class SMSNotification(Notification):
    """CONCRETE PRODUCT - SMS implementation"""
  
    def __init__(self):
        self.sent = False
  
    def send(self, message: str, recipient: str) -> bool:
        print(f"📱 Sending SMS to {recipient}")
        print(f"   Message: {message[:160]}")  # SMS limit
        self.sent = True
        return True
  
    def get_delivery_status(self) -> str:
        return "SMS delivered" if self.sent else "SMS pending"


class PushNotification(Notification):
    """CONCRETE PRODUCT - Push notification implementation"""
  
    def __init__(self):
        self.sent = False
  
    def send(self, message: str, recipient: str) -> bool:
        print(f"🔔 Sending PUSH notification to {recipient}")
        print(f"   Alert: {message}")
        self.sent = True
        return True
  
    def get_delivery_status(self) -> str:
        return "Push delivered" if self.sent else "Push pending"


class SlackNotification(Notification):
    """CONCRETE PRODUCT - Slack implementation (NEW - added without modifying existing code!)"""
  
    def __init__(self):
        self.sent = False
  
    def send(self, message: str, recipient: str) -> bool:
        print(f"💬 Sending SLACK message to #{recipient}")
        print(f"   Message: {message}")
        self.sent = True
        return True
  
    def get_delivery_status(self) -> str:
        return "Slack message delivered" if self.sent else "Slack pending"


# ============================================================================
# CREATOR (Abstract) - Defines the factory method
# ============================================================================

class NotificationService(ABC):
    """
    CREATOR - Abstract class that defines the factory method.
  
    This is the key: The factory method is ABSTRACT, so subclasses
    must implement it and decide which concrete Product to create.
    """
  
    @abstractmethod
    def create_notification(self) -> Notification:
        """
        FACTORY METHOD - Subclasses override this to create specific notifications.
        This is the method that "creates objects but lets subclasses decide which class."
        """
        pass
  
    def send_notification(self, message: str, recipient: str) -> bool:
        """
        TEMPLATE METHOD - Uses the factory method.
    
        This method doesn't know or care which concrete Notification it gets.
        It just calls create_notification() and uses the result.
        """
        # Call the factory method (implemented by subclass)
        notification = self.create_notification()
    
        # Use the product (works with any Notification subclass)
        success = notification.send(message, recipient)
        status = notification.get_delivery_status()
    
        print(f"   Status: {status}\n")
        return success
  
    def send_bulk_notifications(self, message: str, recipients: List[str]) -> None:
        """Send to multiple recipients"""
        print(f"📢 Sending bulk notifications to {len(recipients)} recipients")
        for recipient in recipients:
            self.send_notification(message, recipient)


# ============================================================================
# CONCRETE CREATORS - Implement the factory method
# ============================================================================

class EmailNotificationService(NotificationService):
    """CONCRETE CREATOR - Creates EmailNotification"""
  
    def create_notification(self) -> Notification:
        """Factory method implementation - returns EmailNotification"""
        return EmailNotification()


class SMSNotificationService(NotificationService):
    """CONCRETE CREATOR - Creates SMSNotification"""
  
    def create_notification(self) -> Notification:
        """Factory method implementation - returns SMSNotification"""
        return SMSNotification()


class PushNotificationService(NotificationService):
    """CONCRETE CREATOR - Creates PushNotification"""
  
    def create_notification(self) -> Notification:
        """Factory method implementation - returns PushNotification"""
        return PushNotification()


class SlackNotificationService(NotificationService):
    """CONCRETE CREATOR - Creates SlackNotification (NEW - no existing code modified!)"""
  
    def create_notification(self) -> Notification:
        """Factory method implementation - returns SlackNotification"""
        return SlackNotification()


# ============================================================================
# CLIENT CODE - Uses the factory
# ============================================================================

def main():
    print("=" * 80)
    print("FACTORY METHOD PATTERN - NOTIFICATION SYSTEM")
    print("=" * 80)
  
    # ========================================================================
    # Example 1: Using different notification services
    # ========================================================================
    print("\n1️⃣  SENDING NOTIFICATIONS VIA DIFFERENT SERVICES")
    print("-" * 80)
  
    # Client code works with NotificationService interface
    # It doesn't know or care about concrete implementations
  
    email_service = EmailNotificationService()
    email_service.send_notification(
        "Your order has been shipped!",
        "customer@example.com"
    )
  
    sms_service = SMSNotificationService()
    sms_service.send_notification(
        "Your verification code is: 123456",
        "+1-555-0123"
    )
  
    push_service = PushNotificationService()
    push_service.send_notification(
        "You have a new message!",
        "user_device_token_xyz"
    )
  
    slack_service = SlackNotificationService()
    slack_service.send_notification(
        "Deployment completed successfully!",
        "engineering"
    )
  
    # ========================================================================
    # Example 2: Runtime selection based on user preference
    # ========================================================================
    print("\n2️⃣  RUNTIME SELECTION - Based on user preference")
    print("-" * 80)
  
    def get_notification_service(preference: str) -> NotificationService:
        """
        Factory function that returns appropriate service.
        This is where runtime decision happens.
        """
        services = {
            "email": EmailNotificationService,
            "sms": SMSNotificationService,
            "push": PushNotificationService,
            "slack": SlackNotificationService
        }
    
        service_class = services.get(preference)
        if not service_class:
            raise ValueError(f"Unknown notification type: {preference}")
    
        return service_class()
  
    # Simulate user preferences
    user_preferences = ["email", "sms", "push", "slack"]
  
    for preference in user_preferences:
        service = get_notification_service(preference)
        service.send_notification(
            f"Testing {preference} notification",
            "test_recipient"
        )
  
    # ========================================================================
    # Example 3: Bulk notifications
    # ========================================================================
    print("\n3️⃣  BULK NOTIFICATIONS")
    print("-" * 80)
  
    recipients = ["user1@example.com", "user2@example.com", "user3@example.com"]
  
    email_service = EmailNotificationService()
    email_service.send_bulk_notifications(
        "Important system update scheduled for tonight",
        recipients
    )
  
    # ========================================================================
    # Example 4: Polymorphism in action
    # ========================================================================
    print("\n4️⃣  POLYMORPHISM - Same interface, different implementations")
    print("-" * 80)
  
    # All services implement the same interface
    services: List[NotificationService] = [
        EmailNotificationService(),
        SMSNotificationService(),
        PushNotificationService(),
        SlackNotificationService()
    ]
  
    # Client code treats them all the same way
    for service in services:
        service.send_notification(
            "This message goes through all channels",
            "recipient"
        )
  
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
FACTORY METHOD PATTERN - NOTIFICATION SYSTEM
================================================================================

1️⃣  SENDING NOTIFICATIONS VIA DIFFERENT SERVICES
--------------------------------------------------------------------------------
📧 Sending EMAIL to customer@example.com
   Subject: Notification
   Body: Your order has been shipped!
   Status: Email delivered

📱 Sending SMS to +1-555-0123
   Message: Your verification code is: 123456
   Status: SMS delivered

🔔 Sending PUSH notification to user_device_token_xyz
   Alert: You have a new message!
   Status: Push delivered

💬 Sending SLACK message to #engineering
   Message: Deployment completed successfully!
   Status: Slack message delivered

2️⃣  RUNTIME SELECTION - Based on user preference
--------------------------------------------------------------------------------
📧 Sending EMAIL to test_recipient
   Subject: Notification
   Body: Testing email notification
   Status: Email delivered

📱 Sending SMS to test_recipient
   Message: Testing sms notification
   Status: SMS delivered

🔔 Sending PUSH notification to test_recipient
   Alert: Testing push notification
   Status: Push delivered

💬 Sending SLACK message to #test_recipient
   Message: Testing slack notification
   Status: Slack message delivered

3️⃣  BULK NOTIFICATIONS
--------------------------------------------------------------------------------
📢 Sending bulk notifications to 3 recipients
📧 Sending EMAIL to user1@example.com
   Subject: Notification
   Body: Important system update scheduled for tonight
   Status: Email delivered

📧 Sending EMAIL to user2@example.com
   Subject: Notification
   Body: Important system update scheduled for tonight
   Status: Email delivered

📧 Sending EMAIL to user3@example.com
   Subject: Notification
   Body: Important system update scheduled for tonight
   Status: Email delivered

4️⃣  POLYMORPHISM - Same interface, different implementations
--------------------------------------------------------------------------------
📧 Sending EMAIL to recipient
   Subject: Notification
   Body: This message goes through all channels
   Status: Email delivered

📱 Sending SMS to recipient
   Message: This message goes through all channels
   Status: SMS delivered

🔔 Sending PUSH notification to recipient
   Alert: This message goes through all channels
   Status: Push delivered

💬 Sending SLACK message to #recipient
   Message: This message goes through all channels
   Status: Slack message delivered

================================================================================
✅ DEMONSTRATION COMPLETE
================================================================================
```

---

## 🎯 Key Concepts Explained

### 1. **Product (Notification)**

- The interface/abstract class that defines what the factory creates
- All concrete products implement this interface

### 2. **Concrete Products (EmailNotification, SMSNotification, etc.)**

- Specific implementations of the Product interface
- Each has its own behavior

### 3. **Creator (NotificationService)**

- Abstract class with the **factory method** (`create_notification()`)
- Has business logic that **uses** the product (e.g., `send_notification()`)
- **Doesn't know which concrete product it will get**

### 4. **Concrete Creators (EmailNotificationService, etc.)**

- Implement the factory method
- **Decide which concrete product to create**

### 5. **Client Code**

- Works with the Creator interface
- **Doesn't know about concrete products**
- Can switch implementations at runtime

---

## 💡 Why This is Better

### ✅ **Open-Closed Principle**

```python
# Adding SlackNotification:
# 1. Create SlackNotification class (new code)
# 2. Create SlackNotificationService class (new code)
# 3. NO MODIFICATION to existing code!

# Without Factory Method, you'd have to modify:
# - Application class
# - All if-elif chains
# - Risk breaking existing functionality
```

### ✅ **Loose Coupling**

```python
# Client code doesn't depend on concrete classes
service: NotificationService = get_service_from_config()
service.send_notification(message, recipient)
# Works with ANY NotificationService subclass!
```

### ✅ **Single Responsibility**

```python
# Each class has ONE job:
# - EmailNotification: Send emails
# - EmailNotificationService: Create EmailNotification objects
# - Client: Use notifications (doesn't care how they're created)
```

---

## 🤔 When to Use Factory Method

### ✅ **Use When:**

1. You don't know which exact class you need until runtime
2. You want to delegate object creation to subclasses
3. You need to add new types without modifying existing code
4. You have a family of related objects (notifications, payments, loggers, etc.)

### ❌ **Don't Use When:**

1. You only have one concrete class (overkill)
2. Object creation is simple (no complex logic)
3. You don't need runtime flexibility

---

## ❌ THE PROBLEM - Without Factory Method

```python
from abc import ABC, abstractmethod

# ============================================================================
# PRODUCTS (same in both approaches)
# ============================================================================

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailNotification(Notification):
    def send(self, message: str) -> None:
        print(f"📧 Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str) -> None:
        print(f"📱 SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str) -> None:
        print(f"🔔 Push: {message}")


# ============================================================================
# ❌ BAD APPROACH - Client creates objects directly
# ============================================================================

class Application:
    """
    PROBLEM: This class is TIGHTLY COUPLED to all notification types.
    It knows about EmailNotification, SMSNotification, PushNotification.
    """
  
    def __init__(self, notification_type: str):
        # ❌ PROBLEM 1: Client knows about ALL concrete classes
        # ❌ PROBLEM 2: if-elif chain will grow with every new type
        if notification_type == "email":
            self.notification = EmailNotification()
        elif notification_type == "sms":
            self.notification = SMSNotification()
        elif notification_type == "push":
            self.notification = PushNotification()
        else:
            raise ValueError(f"Unknown type: {notification_type}")
  
    def send_alert(self, message: str):
        self.notification.send(message)


# ============================================================================
# CLIENT CODE - Using the bad approach
# ============================================================================

def bad_example():
    print("=" * 80)
    print("❌ BAD APPROACH - Without Factory Method")
    print("=" * 80)
  
    # Works fine for existing types
    app1 = Application("email")
    app1.send_alert("Server is down!")
  
    app2 = Application("sms")
    app2.send_alert("Server is down!")
  
    print("\n⚠️  NOW LET'S ADD A NEW NOTIFICATION TYPE: SlackNotification")
    print("-" * 80)
  
    # First, create the new class
    class SlackNotification(Notification):
        def send(self, message: str) -> None:
            print(f"💬 Slack: {message}")
  
    # ❌ PROBLEM: We MUST modify Application class to support it!
    # We have to add another elif branch:
  
    class ApplicationModified:
        def __init__(self, notification_type: str):
            if notification_type == "email":
                self.notification = EmailNotification()
            elif notification_type == "sms":
                self.notification = SMSNotification()
            elif notification_type == "push":
                self.notification = PushNotification()
            elif notification_type == "slack":  # ❌ MODIFIED existing code!
                self.notification = SlackNotification()
            else:
                raise ValueError(f"Unknown type: {notification_type}")
      
        def send_alert(self, message: str):
            self.notification.send(message)
  
    app3 = ApplicationModified("slack")
    app3.send_alert("Server is down!")
  
    print("\n❌ PROBLEMS WITH THIS APPROACH:")
    print("   1. Had to MODIFY Application class (violates Open-Closed Principle)")
    print("   2. Application knows about ALL notification types (tight coupling)")
    print("   3. if-elif chain grows with every new type (maintenance nightmare)")
    print("   4. Can't add new types without touching existing code")
    print("   5. Risk of breaking existing functionality when adding new types")
    print("=" * 80)


bad_example()
```

---

## ✅ THE SOLUTION - With Factory Method

```python
from abc import ABC, abstractmethod

# ============================================================================
# PRODUCTS (same as before)
# ============================================================================

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailNotification(Notification):
    def send(self, message: str) -> None:
        print(f"📧 Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str) -> None:
        print(f"📱 SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str) -> None:
        print(f"🔔 Push: {message}")


# ============================================================================
# ✅ GOOD APPROACH - Factory Method Pattern
# ============================================================================

class Application(ABC):
    """
    SOLUTION: Application is now ABSTRACT.
    It doesn't know about concrete notification types.
    Subclasses decide which notification to create.
    """
  
    @abstractmethod
    def create_notification(self) -> Notification:
        """
        FACTORY METHOD - Subclasses implement this.
        Application doesn't know which concrete class will be returned.
        """
        pass
  
    def send_alert(self, message: str):
        """
        This method uses the factory method.
        It works with ANY Notification subclass.
        """
        # ✅ Application doesn't create the notification directly
        # ✅ It calls the factory method (implemented by subclass)
        notification = self.create_notification()
        notification.send(message)


# ============================================================================
# CONCRETE CREATORS - Each knows how to create ONE type
# ============================================================================

class EmailApplication(Application):
    """Creates EmailNotification"""
  
    def create_notification(self) -> Notification:
        return EmailNotification()


class SMSApplication(Application):
    """Creates SMSNotification"""
  
    def create_notification(self) -> Notification:
        return SMSNotification()


class PushApplication(Application):
    """Creates PushNotification"""
  
    def create_notification(self) -> Notification:
        return PushNotification()


# ============================================================================
# CLIENT CODE - Using the good approach
# ============================================================================

def good_example():
    print("\n" + "=" * 80)
    print("✅ GOOD APPROACH - With Factory Method")
    print("=" * 80)
  
    # Client works with Application interface
    app1: Application = EmailApplication()
    app1.send_alert("Server is down!")
  
    app2: Application = SMSApplication()
    app2.send_alert("Server is down!")
  
    app3: Application = PushApplication()
    app3.send_alert("Server is down!")
  
    print("\n✅ NOW LET'S ADD A NEW NOTIFICATION TYPE: SlackNotification")
    print("-" * 80)
  
    # Step 1: Create the new notification class
    class SlackNotification(Notification):
        def send(self, message: str) -> None:
            print(f"💬 Slack: {message}")
  
    # Step 2: Create a new Application subclass
    class SlackApplication(Application):
        def create_notification(self) -> Notification:
            return SlackNotification()
  
    # ✅ NO MODIFICATION to existing code!
    # ✅ Just added new classes
  
    app4: Application = SlackApplication()
    app4.send_alert("Server is down!")
  
    print("\n✅ BENEFITS OF THIS APPROACH:")
    print("   1. NO modification to existing Application class (Open-Closed Principle)")
    print("   2. Application doesn't know about concrete types (loose coupling)")
    print("   3. No if-elif chain (each subclass handles one type)")
    print("   4. Can add new types by just adding new classes")
    print("   5. Existing code is safe - no risk of breaking it")
    print("=" * 80)


good_example()
```

---

## 📊 Side-by-Side Comparison

Let me show you what happens when we add `SlackNotification`:

### ❌ **Without Factory Method** (Bad Approach)

```python
# BEFORE: Original Application class
class Application:
    def __init__(self, notification_type: str):
        if notification_type == "email":
            self.notification = EmailNotification()
        elif notification_type == "sms":
            self.notification = SMSNotification()
        elif notification_type == "push":
            self.notification = PushNotification()
        # ... rest of code

# AFTER: Modified Application class (MODIFIED EXISTING CODE!)
class Application:
    def __init__(self, notification_type: str):
        if notification_type == "email":
            self.notification = EmailNotification()
        elif notification_type == "sms":
            self.notification = SMSNotification()
        elif notification_type == "push":
            self.notification = PushNotification()
        elif notification_type == "slack":  # ❌ ADDED THIS LINE
            self.notification = SlackNotification()  # ❌ ADDED THIS LINE
        # ... rest of code

# ❌ You MODIFIED existing code
# ❌ Risk of introducing bugs
# ❌ Violates Open-Closed Principle
```

### ✅ **With Factory Method** (Good Approach)

```python
# BEFORE: Existing classes (UNCHANGED)
class Application(ABC):
    @abstractmethod
    def create_notification(self) -> Notification:
        pass
  
    def send_alert(self, message: str):
        notification = self.create_notification()
        notification.send(message)

class EmailApplication(Application):
    def create_notification(self) -> Notification:
        return EmailNotification()

class SMSApplication(Application):
    def create_notification(self) -> Notification:
        return SMSNotification()

# AFTER: Just ADD new classes (NO MODIFICATION to existing code!)
class SlackNotification(Notification):  # ✅ NEW class
    def send(self, message: str) -> None:
        print(f"💬 Slack: {message}")

class SlackApplication(Application):  # ✅ NEW class
    def create_notification(self) -> Notification:
        return SlackNotification()

# ✅ Existing code is UNTOUCHED
# ✅ No risk of breaking existing functionality
# ✅ Follows Open-Closed Principle
```

---

## 🎯 The Key Difference Visualized

```
❌ WITHOUT FACTORY METHOD:
┌────────────────────────────────────────────────────────────┐
│                      Application                           │
│  ┌────────────────────────────────────────────────────┐    │
│  │ if type == "email":                                │    │
│  │     notification = EmailNotification()             │    │
│  │ elif type == "sms":                                │    │
│  │     notification = SMSNotification()               │    │
│  │ elif type == "push":                               │    │
│  │     notification = PushNotification()              │    │
│  │ elif type == "slack":  ← MUST ADD THIS!            │    │
│  │     notification = SlackNotification()             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Application KNOWS about ALL notification types            │
│  Adding new type = MODIFYING this class                    │
└────────────────────────────────────────────────────────────┘


✅ WITH FACTORY METHOD:
┌────────────────────────────────────────────────────────────┐
│              Application (Abstract)                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ create_notification() -> Notification [ABSTRACT]   │    │
│  │                                                    │    │
│  │ send_alert(message):                               │    │
│  │     notification = self.create_notification()      │    │
│  │     notification.send(message)                     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Application DOESN'T know about concrete types             │
└────────────────────────────────────────────────────────────┘
         │
         ├──────────────┬──────────────┬──────────────┬──────────────┐
         │              │              │              │              │
┌────────▼────────┐ ┌───▼──────────┐ ┌──▼────────┐ ┌──▼─────────┐ ┌──▼─────────┐
│EmailApplication │ │SMSApplication│ │PushApp    │ │SlackApp    │ │ NEW types  │
│                 │ │              │ │           │ │ (NEW!)     │ │ (future)   │
│create() ->      │ │create() ->   │ │create() ->│ │create() -> │ │            │
│  Email          │ │  SMS         │ │  Push     │ │  Slack     │ │            │
└─────────────────┘ └──────────────┘ └───────────┘ └────────────┘ └────────────┘

Adding new type = ADDING new subclass (no modification!)
```

---

## 💡 Real-World Analogy

Think of it like a restaurant:

### ❌ **Without Factory Method:**

```
Customer (Application) says:
"I want pizza. Let me check the kitchen and make it myself."

Customer goes to kitchen and:
- Checks if ingredients for Margherita exist → makes Margherita
- Checks if ingredients for Pepperoni exist → makes Pepperoni
- Checks if ingredients for Hawaiian exist → makes Hawaiian

Problem: Customer knows TOO MUCH about the kitchen.
Adding new pizza = Customer must learn new recipe.
```

### ✅ **With Factory Method:**

```
Customer (Application) says:
"I want pizza. I'll ask the chef."

Customer tells chef (factory method):
"Make me a pizza" (doesn't specify which)

Chef (subclass) decides:
- ItalianChef → makes Margherita
- AmericanChef → makes Pepperoni
- HawaiianChef → makes Hawaiian

Benefit: Customer doesn't know HOW pizza is made.
Adding new pizza = Hire new chef (no change to customer).
```

---

## 📝 Complete Working Example

```python
from abc import ABC, abstractmethod

# Products
class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailNotification(Notification):
    def send(self, message: str) -> None:
        print(f"📧 Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str) -> None:
        print(f"📱 SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str) -> None:
        print(f"🔔 Push: {message}")

class SlackNotification(Notification):
    def send(self, message: str) -> None:
        print(f"💬 Slack: {message}")


# Creator (Abstract)
class Application(ABC):
    @abstractmethod
    def create_notification(self) -> Notification:
        pass
  
    def send_alert(self, message: str):
        notification = self.create_notification()
        notification.send(message)


# Concrete Creators
class EmailApplication(Application):
    def create_notification(self) -> Notification:
        return EmailNotification()

class SMSApplication(Application):
    def create_notification(self) -> Notification:
        return SMSNotification()

class PushApplication(Application):
    def create_notification(self) -> Notification:
        return PushNotification()

class SlackApplication(Application):  # NEW - no modification to existing code!
    def create_notification(self) -> Notification:
        return SlackNotification()


# Client Code
def main():
    print("=" * 80)
    print("FACTORY METHOD - SOLVING THE PROBLEM")
    print("=" * 80)
  
    # Client code works with Application interface
    # It doesn't know about concrete notification types
  
    apps = [
        EmailApplication(),
        SMSApplication(),
        PushApplication(),
        SlackApplication()  # NEW - just added to list, no other changes!
    ]
  
    for app in apps:
        app.send_alert("Critical system alert!")
  
    print("\n✅ Added SlackApplication without modifying ANY existing code!")
    print("=" * 80)


if __name__ == "__main__":
    main()
```

---

## 📤 Output

```
================================================================================
FACTORY METHOD - SOLVING THE PROBLEM
================================================================================
📧 Email: Critical system alert!
📱 SMS: Critical system alert!
🔔 Push: Critical system alert!
💬 Slack: Critical system alert!

✅ Added SlackApplication without modifying ANY existing code!
================================================================================
```

---

## 🎯 Summary: How Factory Method Solves the Problem

| **Problem**               | **How Factory Method Solves It**                       |
| ------------------------------- | ------------------------------------------------------------ |
| **Tight coupling**        | Application doesn't know about concrete classes              |
| **Modification required** | Add new subclass, don't modify existing code                 |
| **if-elif chain**         | Each subclass handles ONE type                               |
| **Violates Open-Closed**  | Open for extension (new subclasses), closed for modification |
| **Client knows too much** | Client only knows about abstract Application                 |

The key insight: **Instead of the client deciding which object to create, we delegate that decision to subclasses.**

---

## 🔍 Let's Compare: Inheritance vs Factory Method

### 1️⃣ **Simple Inheritance/Polymorphism** (What you're thinking of)

```python
from abc import ABC, abstractmethod

# ============================================================================
# SIMPLE INHERITANCE - Just polymorphism
# ============================================================================

class Animal(ABC):
    """Base class"""
  
    @abstractmethod
    def speak(self) -> str:
        pass


class Dog(Animal):
    """Subclass with specific implementation"""
  
    def speak(self) -> str:
        return "Bark!"


class Cat(Animal):
    """Subclass with specific implementation"""
  
    def speak(self) -> str:
        return "Meow!"


# CLIENT CODE - Simple polymorphism
def simple_inheritance_example():
    print("=" * 80)
    print("SIMPLE INHERITANCE - Just Polymorphism")
    print("=" * 80)
  
    # ❌ CLIENT creates objects directly
    dog = Dog()  # Client knows about Dog class
    cat = Cat()  # Client knows about Cat class
  
    # Polymorphism: Same interface, different behavior
    animals = [dog, cat]
  
    for animal in animals:
        print(animal.speak())
  
    print("\n⚠️  PROBLEM: Client code DIRECTLY creates Dog() and Cat()")
    print("   Client KNOWS about concrete classes!")
    print("=" * 80)


simple_inheritance_example()
```

**Output:**

```
================================================================================
SIMPLE INHERITANCE - Just Polymorphism
================================================================================
Bark!
Meow!

⚠️  PROBLEM: Client code DIRECTLY creates Dog() and Cat()
   Client KNOWS about concrete classes!
================================================================================
```

---

### 2️⃣ **Factory Method Pattern** (The actual pattern)

```python
from abc import ABC, abstractmethod

# ============================================================================
# FACTORY METHOD PATTERN - Inheritance + Factory
# ============================================================================

# Products (same as before)
class Animal(ABC):
    @abstractmethod
    def speak(self) -> str:
        pass


class Dog(Animal):
    def speak(self) -> str:
        return "Bark!"


class Cat(Animal):
    def speak(self) -> str:
        return "Meow!"


# ============================================================================
# THE KEY DIFFERENCE: CREATOR CLASSES
# ============================================================================

class AnimalShelter(ABC):
    """
    CREATOR - This is what makes it Factory Method!
  
    The shelter is responsible for CREATING animals.
    Client doesn't create animals directly.
    """
  
    @abstractmethod
    def create_animal(self) -> Animal:
        """FACTORY METHOD - Subclasses decide which animal to create"""
        pass
  
    def adopt_animal(self) -> str:
        """
        Business logic that USES the factory method.
        This method doesn't know which animal it will get.
        """
        # ✅ Shelter creates the animal (not the client!)
        animal = self.create_animal()
      
        # Use the animal
        sound = animal.speak()
        return f"You adopted an animal that says: {sound}"


class DogShelter(AnimalShelter):
    """Concrete creator - creates Dogs"""
  
    def create_animal(self) -> Animal:
        return Dog()


class CatShelter(AnimalShelter):
    """Concrete creator - creates Cats"""
  
    def create_animal(self) -> Animal:
        return Cat()


# CLIENT CODE - Factory Method
def factory_method_example():
    print("\n" + "=" * 80)
    print("FACTORY METHOD PATTERN - Inheritance + Factory")
    print("=" * 80)
  
    # ✅ CLIENT works with AnimalShelter interface
    # ✅ CLIENT doesn't know about Dog or Cat classes!
  
    dog_shelter: AnimalShelter = DogShelter()
    cat_shelter: AnimalShelter = CatShelter()
  
    print(dog_shelter.adopt_animal())
    print(cat_shelter.adopt_animal())
  
    print("\n✅ BENEFIT: Client doesn't create Dog() or Cat() directly!")
    print("   Client only knows about AnimalShelter interface!")
    print("=" * 80)


factory_method_example()
```

**Output:**

```
================================================================================
FACTORY METHOD PATTERN - Inheritance + Factory
================================================================================
You adopted an animal that says: Bark!
You adopted an animal that says: Meow!

✅ BENEFIT: Client doesn't create Dog() or Cat() directly!
   Client only knows about AnimalShelter interface!
================================================================================
```

---

## 🎯 The Critical Difference

Let me show you **side-by-side** what's different:

### ❌ **Simple Inheritance** (What you're thinking of)

```python
# CLIENT CODE
dog = Dog()        # ❌ Client creates Dog directly
cat = Cat()        # ❌ Client creates Cat directly

print(dog.speak()) # Polymorphism: calls Dog's speak()
print(cat.speak()) # Polymorphism: calls Cat's speak()

# PROBLEM: Client knows about Dog and Cat classes
# PROBLEM: Client is responsible for creating objects
```

### ✅ **Factory Method Pattern**

```python
# CLIENT CODE
shelter1 = DogShelter()  # ✅ Client creates a shelter
shelter2 = CatShelter()  # ✅ Client creates a shelter

# ✅ Shelter creates the animal (not the client!)
print(shelter1.adopt_animal())  # Shelter creates Dog internally
print(shelter2.adopt_animal())  # Shelter creates Cat internally

# BENEFIT: Client doesn't know about Dog or Cat classes
# BENEFIT: Object creation is delegated to the shelter
```

---

## 📊 Visual Comparison

### **Simple Inheritance:**

```
Client
  │
  ├─── creates ──→ Dog()
  │                 │
  │                 └─── calls speak() → "Bark!"
  │
  └─── creates ──→ Cat()
                    │
                    └─── calls speak() → "Meow!"

Client DIRECTLY creates Dog and Cat
Client KNOWS about concrete classes
```

### **Factory Method:**

```
Client
  │
  ├─── creates ──→ DogShelter
  │                 │
  │                 └─── create_animal() → Dog()
  │                                          │
  │                                          └─── speak() → "Bark!"
  │
  └─── creates ──→ CatShelter
                    │
                    └─── create_animal() → Cat()
                                             │
                                             └─── speak() → "Meow!"

Client creates SHELTERS (not animals)
Shelters create animals
Client DOESN'T know about Dog or Cat
```

---

## 🔑 The Key Insight

**Factory Method = Inheritance + One Extra Layer**

```python
# SIMPLE INHERITANCE:
# Client → Animal (Dog/Cat)
#
# Client creates animals directly

# FACTORY METHOD:
# Client → Creator (Shelter) → Animal (Dog/Cat)
#                    ↑
#              Factory Method
#
# Client creates creator
# Creator creates animals
```

---

## 💡 Real Example to Make it Clear

Let me show you a **real-world scenario** where the difference matters:

### Scenario: Pet Adoption System

```python
from abc import ABC, abstractmethod

# ============================================================================
# Products
# ============================================================================

class Animal(ABC):
    @abstractmethod
    def speak(self) -> str:
        pass
  
    @abstractmethod
    def get_care_instructions(self) -> str:
        pass


class Dog(Animal):
    def speak(self) -> str:
        return "Bark!"
  
    def get_care_instructions(self) -> str:
        return "Walk daily, feed twice a day, needs yard space"


class Cat(Animal):
    def speak(self) -> str:
        return "Meow!"
  
    def get_care_instructions(self) -> str:
        return "Clean litter box daily, feed twice a day, indoor pet"


class Rabbit(Animal):
    def speak(self) -> str:
        return "Squeak!"
  
    def get_care_instructions(self) -> str:
        return "Clean cage weekly, feed fresh vegetables, needs exercise"


# ============================================================================
# ❌ APPROACH 1: Simple Inheritance (What you're thinking of)
# ============================================================================

class PetAdoptionSystem_Bad:
    """
    BAD: Client code creates animals directly
    """
  
    def adopt_pet(self, pet_type: str) -> Animal:
        # ❌ System knows about ALL animal types
        # ❌ Adding new animal requires modifying this code
        if pet_type == "dog":
            return Dog()
        elif pet_type == "cat":
            return Cat()
        elif pet_type == "rabbit":
            return Rabbit()
        else:
            raise ValueError(f"Unknown pet type: {pet_type}")


def bad_approach():
    print("=" * 80)
    print("❌ BAD APPROACH - Simple Inheritance")
    print("=" * 80)
  
    system = PetAdoptionSystem_Bad()
  
    # Client uses the system
    dog = system.adopt_pet("dog")
    print(f"Adopted: {dog.speak()}")
    print(f"Care: {dog.get_care_instructions()}")
  
    print("\n⚠️  PROBLEM: PetAdoptionSystem knows about Dog, Cat, Rabbit")
    print("   Adding new animal = MODIFYING PetAdoptionSystem")
    print("=" * 80)


bad_approach()


# ============================================================================
# ✅ APPROACH 2: Factory Method Pattern
# ============================================================================

class AnimalShelter(ABC):
    """
    CREATOR - Abstract shelter
    Each shelter specializes in one type of animal
    """
  
    @abstractmethod
    def create_animal(self) -> Animal:
        """FACTORY METHOD - Subclasses decide which animal"""
        pass
  
    def adopt_animal(self) -> dict:
        """
        Business logic that uses the factory method.
        This is the same for ALL shelters.
        """
        # ✅ Shelter creates the animal (client doesn't!)
        animal = self.create_animal()
      
        # Perform adoption process (same for all animals)
        return {
            "animal": animal,
            "sound": animal.speak(),
            "care_instructions": animal.get_care_instructions(),
            "adoption_fee": self.get_adoption_fee(),
            "shelter_type": self.__class__.__name__
        }
  
    @abstractmethod
    def get_adoption_fee(self) -> float:
        """Each shelter can have different fees"""
        pass


class DogShelter(AnimalShelter):
    """Specializes in dogs"""
  
    def create_animal(self) -> Animal:
        return Dog()
  
    def get_adoption_fee(self) -> float:
        return 150.00


class CatShelter(AnimalShelter):
    """Specializes in cats"""
  
    def create_animal(self) -> Animal:
        return Cat()
  
    def get_adoption_fee(self) -> float:
        return 100.00


class RabbitShelter(AnimalShelter):
    """Specializes in rabbits"""
  
    def create_animal(self) -> Animal:
        return Rabbit()
  
    def get_adoption_fee(self) -> float:
        return 50.00


# NEW: Adding a new animal type (NO MODIFICATION to existing code!)
class Bird(Animal):
    def speak(self) -> str:
        return "Tweet!"
  
    def get_care_instructions(self) -> str:
        return "Clean cage daily, feed seeds, needs perch"


class BirdShelter(AnimalShelter):
    """NEW shelter - no modification to existing shelters!"""
  
    def create_animal(self) -> Animal:
        return Bird()
  
    def get_adoption_fee(self) -> float:
        return 75.00


def good_approach():
    print("\n" + "=" * 80)
    print("✅ GOOD APPROACH - Factory Method Pattern")
    print("=" * 80)
  
    # Client works with AnimalShelter interface
    # Client doesn't know about Dog, Cat, Rabbit, Bird classes!
  
    shelters = [
        DogShelter(),
        CatShelter(),
        RabbitShelter(),
        BirdShelter()  # NEW - just added to list!
    ]
  
    for shelter in shelters:
        adoption_info = shelter.adopt_animal()
        print(f"\n{adoption_info['shelter_type']}:")
        print(f"  Sound: {adoption_info['sound']}")
        print(f"  Care: {adoption_info['care_instructions']}")
        print(f"  Fee: ${adoption_info['adoption_fee']}")
  
    print("\n✅ BENEFIT: Added BirdShelter without modifying ANY existing code!")
    print("   Client doesn't know about Dog, Cat, Rabbit, or Bird classes!")
    print("=" * 80)


good_approach()
```

---

## 📤 Output

```
================================================================================
❌ BAD APPROACH - Simple Inheritance
================================================================================
Adopted: Bark!
Care: Walk daily, feed twice a day, needs yard space

⚠️  PROBLEM: PetAdoptionSystem knows about Dog, Cat, Rabbit
   Adding new animal = MODIFYING PetAdoptionSystem
================================================================================

================================================================================
✅ GOOD APPROACH - Factory Method Pattern
================================================================================

DogShelter:
  Sound: Bark!
  Care: Walk daily, feed twice a day, needs yard space
  Fee: $150.0

CatShelter:
  Sound: Meow!
  Care: Clean litter box daily, feed twice a day, indoor pet
  Fee: $100.0

RabbitShelter:
  Sound: Squeak!
  Care: Clean cage weekly, feed fresh vegetables, needs exercise
  Fee: $50.0

BirdShelter:
  Sound: Tweet!
  Care: Clean cage daily, feed seeds, needs perch
  Fee: $75.0

✅ BENEFIT: Added BirdShelter without modifying ANY existing code!
   Client doesn't know about Dog, Cat, Rabbit, or Bird classes!
================================================================================
```

---

## 🎯 Summary: The Difference

| **Aspect**              | **Simple Inheritance**                             | **Factory Method**                 |
| ----------------------------- | -------------------------------------------------------- | ---------------------------------------- |
| **What it is**          | Polymorphism (same interface, different implementations) | Polymorphism + Delegated object creation |
| **Client creates**      | `Dog()`, `Cat()` directly                            | `DogShelter()`, `CatShelter()`       |
| **Client knows about**  | Concrete animal classes                                  | Only shelter interface                   |
| **Who creates animals** | Client                                                   | Shelter (creator)                        |
| **Adding new type**     | Modify client code                                       | Add new shelter class                    |
| **Coupling**            | Client tightly coupled to animals                        | Client loosely coupled                   |

---

## 💡 When You Need Factory Method vs Simple Inheritance

### Use **Simple Inheritance** when:

```python
# You just need polymorphism
animals = [Dog(), Cat(), Rabbit()]  # Client creates directly
for animal in animals:
    print(animal.speak())  # Just using polymorphism
```

### Use **Factory Method** when:

```python
# You need to:
# 1. Hide object creation from client
# 2. Add new types without modifying existing code
# 3. Delegate creation responsibility to subclasses

shelters = [DogShelter(), CatShelter(), RabbitShelter()]
for shelter in shelters:
    adoption = shelter.adopt_animal()  # Shelter creates animal
    # Client doesn't know which animal type
```

---

**Does this clarify the difference?**

**Simple inheritance = Just polymorphism**
**Factory Method = Polymorphism + Delegated object creation**

The key is: **Who creates the objects?** In simple inheritance, the client does. In Factory Method, the creator (factory) does.
