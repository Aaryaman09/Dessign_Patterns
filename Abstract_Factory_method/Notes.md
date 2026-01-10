# Abstract Factory Pattern - Complete Notes

## 🎯 Core Concept

**Abstract Factory provides an interface for creating FAMILIES of related or dependent objects without specifying their concrete classes.**

**The Key Idea:**
> "I need to create multiple related objects that must work together, and I want to ensure they're all from the same family/variant."

**Formula:**
```
Abstract Factory = Multiple Factory Methods + Guaranteed Family Consistency
```

---

## 🔑 When to Use Abstract Factory

### ✅ **Use Abstract Factory When:**

| **Scenario** | **Example** | **Why Abstract Factory** |
|--------------|-------------|--------------------------|
| **1. Products must be used together** | UI components (Button + Checkbox + Input) must match | Guarantees consistency |
| **2. Multiple product families exist** | Windows UI vs macOS UI vs Linux UI | Each family is self-contained |
| **3. Family variants are mutually exclusive** | Can't mix Windows button with macOS checkbox | Prevents incompatible combinations |
| **4. Need to switch entire family at runtime** | Switch from light theme to dark theme | Change one factory, all products change |
| **5. System should be independent of product creation** | Client doesn't know concrete classes | Decouples client from implementations |
| **6. Related products have constraints** | Database connection + command + transaction must match DB type | Enforces constraints |

### ❌ **Don't Use Abstract Factory When:**

| **Scenario** | **Use Instead** | **Why** |
|--------------|-----------------|---------|
| Only ONE product type | Factory Method | Simpler, no need for families |
| Products are independent | Multiple simple factories | No relationship between products |
| Simple object creation | Simple factory function | Pattern is overkill |
| Complex step-by-step construction | Builder pattern | Better for complex objects |
| Products don't need to be consistent | Direct instantiation | No consistency requirement |

---

## 📊 Decision Matrix: When to Use What

```
┌─────────────────────────────────────────────────────────────┐
│              Do you need to create objects?                 │
└───────────────────────┬─────────────────────────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │                                │
        ▼                                ▼
┌───────────────┐              ┌─────────────────┐
│  ONE product  │              │ MULTIPLE related│
│     type?     │              │    products?    │
└───────┬───────┘              └────────┬────────┘
        │                               │
        │                               │
┌───────▼────────┐              ┌──────▼─────────┐
│ Have algorithm │              │ Must be from   │
│  that uses it? │              │ same family?   │
└───────┬────────┘              └────────┬───────┘
        │                                │
   ┌────┴─────┐                    ┌────┴─────┐
   │          │                    │          │
   ▼          ▼                    ▼          ▼
┌──────┐  ┌───────┐          ┌─────────┐  ┌────────┐
│ YES  │  │  NO   │          │   YES   │  │   NO   │
│  ↓   │  │   ↓   │          │    ↓    │  │    ↓   │
│Factory│ │Simple │          │Abstract │  │Multiple│
│Method │ │Factory│          │Factory  │  │Simple  │
│       │ │       │          │         │  │Factories│
└───────┘ └───────┘          └─────────┘  └────────┘
```

---

## 🏗️ Abstract Factory Structure

```
┌──────────────────────────────────────────────────────────────┐
│                   AbstractFactory                            │
│                                                              │
│  + create_product_a() -> AbstractProductA                   │
│  + create_product_b() -> AbstractProductB                   │
│  + create_product_c() -> AbstractProductC                   │
└────────────────────┬─────────────────────────────────────────┘
                     │
         ┌───────────┴────────────┐
         │                        │
┌────────▼─────────┐    ┌─────────▼────────┐
│ ConcreteFactory1 │    │ ConcreteFactory2 │
│  (Family 1)      │    │  (Family 2)      │
├──────────────────┤    ├──────────────────┤
│ create_a()       │    │ create_a()       │
│  -> Product_A1   │    │  -> Product_A2   │
│ create_b()       │    │ create_b()       │
│  -> Product_B1   │    │  -> Product_B2   │
│ create_c()       │    │ create_c()       │
│  -> Product_C1   │    │  -> Product_C2   │
└──────────────────┘    └──────────────────┘
         │                        │
         │                        │
    ┌────┴─────┬─────┐       ┌───┴──────┬─────┐
    │          │     │       │          │     │
┌───▼───┐  ┌──▼──┐ ┌▼──┐ ┌──▼───┐  ┌───▼─┐ ┌─▼──┐
│Prod_A1│  │Prod │ │Prod│ │Prod  │  │Prod │ │Prod│
│       │  │_B1  │ │_C1 │ │_A2   │  │_B2  │ │_C2 │
└───────┘  └─────┘ └────┘ └──────┘  └─────┘ └────┘
    └──────────┬──────────┘     └──────┬──────────┘
            FAMILY 1                 FAMILY 2
```

**Components:**

1. **AbstractFactory** - Interface declaring creation methods for each product type
2. **ConcreteFactory** - Implements creation methods, returns products from ONE family
3. **AbstractProduct** - Interface for each product type (ProductA, ProductB, ProductC)
4. **ConcreteProduct** - Specific implementations grouped by family
5. **Client** - Uses only AbstractFactory and AbstractProduct interfaces

---

## 📝 In-Depth Example: E-Commerce Payment System

### **Scenario:**

You're building an e-commerce platform that supports multiple payment gateways (Stripe, PayPal, Square). Each gateway requires:
- **PaymentProcessor** - Processes the payment
- **RefundHandler** - Handles refunds
- **WebhookValidator** - Validates webhook callbacks
- **FraudDetector** - Detects fraudulent transactions

**Key Constraint:** All components for a transaction must be from the SAME gateway (can't mix Stripe processor with PayPal refund handler).

### **Complete Implementation:**

```python
from abc import ABC, abstractmethod
from typing import Dict, List
from datetime import datetime
from decimal import Decimal

# ============================================================================
# 1. ABSTRACT PRODUCTS - Interfaces for each component type
# ============================================================================

class PaymentProcessor(ABC):
    """Abstract product - Payment processor interface"""
    
    @abstractmethod
    def process_payment(self, amount: Decimal, currency: str, card_token: str) -> Dict:
        """Process a payment transaction"""
        pass
    
    @abstractmethod
    def get_transaction_status(self, transaction_id: str) -> str:
        """Get status of a transaction"""
        pass


class RefundHandler(ABC):
    """Abstract product - Refund handler interface"""
    
    @abstractmethod
    def process_refund(self, transaction_id: str, amount: Decimal) -> Dict:
        """Process a refund"""
        pass
    
    @abstractmethod
    def get_refund_status(self, refund_id: str) -> str:
        """Get status of a refund"""
        pass


class WebhookValidator(ABC):
    """Abstract product - Webhook validator interface"""
    
    @abstractmethod
    def validate_webhook(self, payload: str, signature: str) -> bool:
        """Validate webhook authenticity"""
        pass
    
    @abstractmethod
    def parse_webhook_event(self, payload: str) -> Dict:
        """Parse webhook payload"""
        pass


class FraudDetector(ABC):
    """Abstract product - Fraud detector interface"""
    
    @abstractmethod
    def analyze_transaction(self, transaction_data: Dict) -> Dict:
        """Analyze transaction for fraud"""
        pass
    
    @abstractmethod
    def get_risk_score(self, transaction_id: str) -> float:
        """Get fraud risk score (0.0 to 1.0)"""
        pass


# ============================================================================
# 2. CONCRETE PRODUCTS - Stripe Family
# ============================================================================

class StripePaymentProcessor(PaymentProcessor):
    """Concrete product - Stripe payment processor"""
    
    def __init__(self):
        self.api_key = "sk_test_stripe_key"
    
    def process_payment(self, amount: Decimal, currency: str, card_token: str) -> Dict:
        """Process payment via Stripe"""
        print(f"[Stripe] Processing ${amount} {currency}")
        print(f"[Stripe] Using API key: {self.api_key}")
        print(f"[Stripe] Card token: {card_token}")
        
        return {
            'transaction_id': 'stripe_txn_123456',
            'status': 'succeeded',
            'amount': amount,
            'currency': currency,
            'gateway': 'stripe',
            'timestamp': datetime.now().isoformat()
        }
    
    def get_transaction_status(self, transaction_id: str) -> str:
        """Get Stripe transaction status"""
        return f"[Stripe] Transaction {transaction_id}: succeeded"


class StripeRefundHandler(RefundHandler):
    """Concrete product - Stripe refund handler"""
    
    def __init__(self):
        self.api_key = "sk_test_stripe_key"
    
    def process_refund(self, transaction_id: str, amount: Decimal) -> Dict:
        """Process refund via Stripe"""
        print(f"[Stripe] Processing refund for {transaction_id}")
        print(f"[Stripe] Refund amount: ${amount}")
        
        return {
            'refund_id': 'stripe_ref_789012',
            'transaction_id': transaction_id,
            'status': 'succeeded',
            'amount': amount,
            'gateway': 'stripe',
            'timestamp': datetime.now().isoformat()
        }
    
    def get_refund_status(self, refund_id: str) -> str:
        """Get Stripe refund status"""
        return f"[Stripe] Refund {refund_id}: succeeded"


class StripeWebhookValidator(WebhookValidator):
    """Concrete product - Stripe webhook validator"""
    
    def __init__(self):
        self.webhook_secret = "whsec_stripe_secret"
    
    def validate_webhook(self, payload: str, signature: str) -> bool:
        """Validate Stripe webhook signature"""
        print(f"[Stripe] Validating webhook with secret: {self.webhook_secret}")
        print(f"[Stripe] Signature: {signature}")
        # In real implementation, would verify HMAC signature
        return True
    
    def parse_webhook_event(self, payload: str) -> Dict:
        """Parse Stripe webhook payload"""
        return {
            'event_type': 'payment.succeeded',
            'gateway': 'stripe',
            'data': payload
        }


class StripeFraudDetector(FraudDetector):
    """Concrete product - Stripe fraud detector (Radar)"""
    
    def analyze_transaction(self, transaction_data: Dict) -> Dict:
        """Analyze using Stripe Radar"""
        print(f"[Stripe Radar] Analyzing transaction: {transaction_data.get('transaction_id')}")
        
        return {
            'risk_score': 0.15,
            'risk_level': 'low',
            'gateway': 'stripe',
            'checks': {
                'cvc_check': 'pass',
                'address_check': 'pass',
                'velocity_check': 'pass'
            }
        }
    
    def get_risk_score(self, transaction_id: str) -> float:
        """Get Stripe Radar risk score"""
        return 0.15


# ============================================================================
# 3. CONCRETE PRODUCTS - PayPal Family
# ============================================================================

class PayPalPaymentProcessor(PaymentProcessor):
    """Concrete product - PayPal payment processor"""
    
    def __init__(self):
        self.client_id = "paypal_client_id"
        self.client_secret = "paypal_client_secret"
    
    def process_payment(self, amount: Decimal, currency: str, card_token: str) -> Dict:
        """Process payment via PayPal"""
        print(f"[PayPal] Processing ${amount} {currency}")
        print(f"[PayPal] Using client ID: {self.client_id}")
        print(f"[PayPal] Payment token: {card_token}")
        
        return {
            'transaction_id': 'paypal_txn_987654',
            'status': 'COMPLETED',
            'amount': amount,
            'currency': currency,
            'gateway': 'paypal',
            'timestamp': datetime.now().isoformat()
        }
    
    def get_transaction_status(self, transaction_id: str) -> str:
        """Get PayPal transaction status"""
        return f"[PayPal] Transaction {transaction_id}: COMPLETED"


class PayPalRefundHandler(RefundHandler):
    """Concrete product - PayPal refund handler"""
    
    def __init__(self):
        self.client_id = "paypal_client_id"
    
    def process_refund(self, transaction_id: str, amount: Decimal) -> Dict:
        """Process refund via PayPal"""
        print(f"[PayPal] Processing refund for {transaction_id}")
        print(f"[PayPal] Refund amount: ${amount}")
        
        return {
            'refund_id': 'paypal_ref_456789',
            'transaction_id': transaction_id,
            'status': 'COMPLETED',
            'amount': amount,
            'gateway': 'paypal',
            'timestamp': datetime.now().isoformat()
        }
    
    def get_refund_status(self, refund_id: str) -> str:
        """Get PayPal refund status"""
        return f"[PayPal] Refund {refund_id}: COMPLETED"


class PayPalWebhookValidator(WebhookValidator):
    """Concrete product - PayPal webhook validator"""
    
    def __init__(self):
        self.webhook_id = "paypal_webhook_id"
    
    def validate_webhook(self, payload: str, signature: str) -> bool:
        """Validate PayPal webhook signature"""
        print(f"[PayPal] Validating webhook with ID: {self.webhook_id}")
        print(f"[PayPal] Transmission signature: {signature}")
        # In real implementation, would verify PayPal signature
        return True
    
    def parse_webhook_event(self, payload: str) -> Dict:
        """Parse PayPal webhook payload"""
        return {
            'event_type': 'PAYMENT.CAPTURE.COMPLETED',
            'gateway': 'paypal',
            'data': payload
        }


class PayPalFraudDetector(FraudDetector):
    """Concrete product - PayPal fraud detector"""
    
    def analyze_transaction(self, transaction_data: Dict) -> Dict:
        """Analyze using PayPal fraud protection"""
        print(f"[PayPal Fraud Protection] Analyzing: {transaction_data.get('transaction_id')}")
        
        return {
            'risk_score': 0.25,
            'risk_level': 'normal',
            'gateway': 'paypal',
            'checks': {
                'seller_protection': 'eligible',
                'buyer_protection': 'eligible',
                'risk_assessment': 'normal'
            }
        }
    
    def get_risk_score(self, transaction_id: str) -> float:
        """Get PayPal risk score"""
        return 0.25


# ============================================================================
# 4. CONCRETE PRODUCTS - Square Family
# ============================================================================

class SquarePaymentProcessor(PaymentProcessor):
    """Concrete product - Square payment processor"""
    
    def __init__(self):
        self.access_token = "square_access_token"
        self.location_id = "square_location_id"
    
    def process_payment(self, amount: Decimal, currency: str, card_token: str) -> Dict:
        """Process payment via Square"""
        print(f"[Square] Processing ${amount} {currency}")
        print(f"[Square] Location: {self.location_id}")
        print(f"[Square] Card nonce: {card_token}")
        
        return {
            'transaction_id': 'square_txn_111222',
            'status': 'APPROVED',
            'amount': amount,
            'currency': currency,
            'gateway': 'square',
            'timestamp': datetime.now().isoformat()
        }
    
    def get_transaction_status(self, transaction_id: str) -> str:
        """Get Square transaction status"""
        return f"[Square] Transaction {transaction_id}: APPROVED"


class SquareRefundHandler(RefundHandler):
    """Concrete product - Square refund handler"""
    
    def __init__(self):
        self.access_token = "square_access_token"
    
    def process_refund(self, transaction_id: str, amount: Decimal) -> Dict:
        """Process refund via Square"""
        print(f"[Square] Processing refund for {transaction_id}")
        print(f"[Square] Refund amount: ${amount}")
        
        return {
            'refund_id': 'square_ref_333444',
            'transaction_id': transaction_id,
            'status': 'APPROVED',
            'amount': amount,
            'gateway': 'square',
            'timestamp': datetime.now().isoformat()
        }
    
    def get_refund_status(self, refund_id: str) -> str:
        """Get Square refund status"""
        return f"[Square] Refund {refund_id}: APPROVED"


class SquareWebhookValidator(WebhookValidator):
    """Concrete product - Square webhook validator"""
    
    def __init__(self):
        self.signature_key = "square_signature_key"
    
    def validate_webhook(self, payload: str, signature: str) -> bool:
        """Validate Square webhook signature"""
        print(f"[Square] Validating webhook with key: {self.signature_key}")
        print(f"[Square] Signature: {signature}")
        # In real implementation, would verify Square signature
        return True
    
    def parse_webhook_event(self, payload: str) -> Dict:
        """Parse Square webhook payload"""
        return {
            'event_type': 'payment.created',
            'gateway': 'square',
            'data': payload
        }


class SquareFraudDetector(FraudDetector):
    """Concrete product - Square fraud detector"""
    
    def analyze_transaction(self, transaction_data: Dict) -> Dict:
        """Analyze using Square fraud detection"""
        print(f"[Square Fraud Detection] Analyzing: {transaction_data.get('transaction_id')}")
        
        return {
            'risk_score': 0.10,
            'risk_level': 'very_low',
            'gateway': 'square',
            'checks': {
                'cvv_status': 'CVV_ACCEPTED',
                'avs_status': 'AVS_ACCEPTED',
                'risk_evaluation': 'NORMAL'
            }
        }
    
    def get_risk_score(self, transaction_id: str) -> float:
        """Get Square risk score"""
        return 0.10


# ============================================================================
# 5. ABSTRACT FACTORY - Interface for creating payment gateway families
# ============================================================================

class PaymentGatewayFactory(ABC):
    """
    Abstract Factory - Creates family of payment gateway components.
    
    Each concrete factory creates components for ONE gateway.
    All components from a factory are guaranteed to work together.
    """
    
    @abstractmethod
    def create_payment_processor(self) -> PaymentProcessor:
        """Create payment processor for this gateway"""
        pass
    
    @abstractmethod
    def create_refund_handler(self) -> RefundHandler:
        """Create refund handler for this gateway"""
        pass
    
    @abstractmethod
    def create_webhook_validator(self) -> WebhookValidator:
        """Create webhook validator for this gateway"""
        pass
    
    @abstractmethod
    def create_fraud_detector(self) -> FraudDetector:
        """Create fraud detector for this gateway"""
        pass
    
    def get_gateway_name(self) -> str:
        """Get gateway name (helper method)"""
        return self.__class__.__name__.replace('Factory', '')


# ============================================================================
# 6. CONCRETE FACTORIES - One for each payment gateway family
# ============================================================================

class StripeFactory(PaymentGatewayFactory):
    """
    Concrete Factory - Creates Stripe family of components.
    
    All components share:
    - Same API authentication (API key)
    - Same data formats
    - Same error handling
    - Guaranteed compatibility
    """
    
    def create_payment_processor(self) -> PaymentProcessor:
        return StripePaymentProcessor()
    
    def create_refund_handler(self) -> RefundHandler:
        return StripeRefundHandler()
    
    def create_webhook_validator(self) -> WebhookValidator:
        return StripeWebhookValidator()
    
    def create_fraud_detector(self) -> FraudDetector:
        return StripeFraudDetector()


class PayPalFactory(PaymentGatewayFactory):
    """
    Concrete Factory - Creates PayPal family of components.
    
    All components share:
    - Same OAuth authentication
    - Same REST API format
    - Same webhook structure
    - Guaranteed compatibility
    """
    
    def create_payment_processor(self) -> PaymentProcessor:
        return PayPalPaymentProcessor()
    
    def create_refund_handler(self) -> RefundHandler:
        return PayPalRefundHandler()
    
    def create_webhook_validator(self) -> WebhookValidator:
        return PayPalWebhookValidator()
    
    def create_fraud_detector(self) -> FraudDetector:
        return PayPalFraudDetector()


class SquareFactory(PaymentGatewayFactory):
    """
    Concrete Factory - Creates Square family of components.
    
    All components share:
    - Same access token authentication
    - Same location-based structure
    - Same webhook format
    - Guaranteed compatibility
    """
    
    def create_payment_processor(self) -> PaymentProcessor:
        return SquarePaymentProcessor()
    
    def create_refund_handler(self) -> RefundHandler:
        return SquareRefundHandler()
    
    def create_webhook_validator(self) -> WebhookValidator:
        return SquareWebhookValidator()
    
    def create_fraud_detector(self) -> FraudDetector:
        return SquareFraudDetector()


# ============================================================================
# 7. CLIENT CODE - Payment service that uses abstract factory
# ============================================================================

class PaymentService:
    """
    Client - Uses abstract factory to process payments.
    
    Key points:
    1. Doesn't know about concrete classes (StripePaymentProcessor, etc.)
    2. Works only with interfaces (PaymentProcessor, RefundHandler, etc.)
    3. Factory guarantees all components are compatible
    4. Can switch gateways by changing factory
    """
    
    def __init__(self, factory: PaymentGatewayFactory):
        """
        Initialize with a payment gateway factory.
        
        All components will be from the SAME gateway family.
        """
        self.factory = factory
        self.gateway_name = factory.get_gateway_name()
        
        # Create all components from the same family
        self.processor = factory.create_payment_processor()
        self.refund_handler = factory.create_refund_handler()
        self.webhook_validator = factory.create_webhook_validator()
        self.fraud_detector = factory.create_fraud_detector()
        
        print(f"\n{'='*70}")
        print(f"Payment Service initialized with: {self.gateway_name}")
        print(f"{'='*70}\n")
    
    def process_order(self, order_data: Dict) -> Dict:
        """
        Process a complete order with fraud detection.
        
        Uses multiple components from the same family:
        1. Fraud detection
        2. Payment processing
        """
        print(f"\n{'─'*70}")
        print(f"Processing Order: {order_data['order_id']}")
        print(f"{'─'*70}")
        
        # Step 1: Fraud detection
        print("\n[Step 1] Fraud Detection")
        fraud_result = self.fraud_detector.analyze_transaction({
            'transaction_id': order_data['order_id'],
            'amount': order_data['amount'],
            'customer_id': order_data['customer_id']
        })
        print(f"Risk Score: {fraud_result['risk_score']}")
        print(f"Risk Level: {fraud_result['risk_level']}")
        
        if fraud_result['risk_score'] > 0.7:
            return {
                'status': 'rejected',
                'reason': 'High fraud risk',
                'risk_score': fraud_result['risk_score']
            }
        
        # Step 2: Process payment
        print("\n[Step 2] Payment Processing")
        payment_result = self.processor.process_payment(
            amount=order_data['amount'],
            currency=order_data['currency'],
            card_token=order_data['card_token']
        )
        
        print(f"✅ Payment processed: {payment_result['transaction_id']}")
        
        return {
            'status': 'success',
            'transaction_id': payment_result['transaction_id'],
            'fraud_check': fraud_result,
            'payment': payment_result
        }
    
    def process_refund(self, transaction_id: str, amount: Decimal) -> Dict:
        """Process a refund"""
        print(f"\n{'─'*70}")
        print(f"Processing Refund")
        print(f"{'─'*70}")
        
        refund_result = self.refund_handler.process_refund(transaction_id, amount)
        print(f"✅ Refund processed: {refund_result['refund_id']}")
        
        return refund_result
    
    def handle_webhook(self, payload: str, signature: str) -> Dict:
        """Handle incoming webhook"""
        print(f"\n{'─'*70}")
        print(f"Handling Webhook")
        print(f"{'─'*70}")
        
        # Validate webhook
        is_valid = self.webhook_validator.validate_webhook(payload, signature)
        
        if not is_valid:
            return {'status': 'invalid', 'reason': 'Invalid signature'}
        
        # Parse event
        event = self.webhook_validator.parse_webhook_event(payload)
        print(f"✅ Webhook validated: {event['event_type']}")
        
        return {
            'status': 'processed',
            'event': event
        }


# ============================================================================
# 8. USAGE EXAMPLES
# ============================================================================

def main():
    print("="*80)
    print("ABSTRACT FACTORY PATTERN - PAYMENT GATEWAY SYSTEM")
    print("="*80)
    
    # ========================================================================
    # Example 1: Using Stripe gateway
    # ========================================================================
    print("\n" + "="*80)
    print("EXAMPLE 1: STRIPE PAYMENT GATEWAY")
    print("="*80)
    
    stripe_factory = StripeFactory()
    stripe_service = PaymentService(stripe_factory)
    
    # Process order
    order_result = stripe_service.process_order({
        'order_id': 'ORD-001',
        'amount': Decimal('99.99'),
        'currency': 'USD',
        'card_token': 'tok_visa',
        'customer_id': 'cust_123'
    })
    print(f"\nOrder Result: {order_result['status']}")
    
    # Process refund
    if order_result['status'] == 'success':
        refund_result = stripe_service.process_refund(
            order_result['transaction_id'],
            Decimal('99.99')
        )
    
    # Handle webhook
    webhook_result = stripe_service.handle_webhook(
        payload='{"event": "payment.succeeded"}',
        signature='stripe_sig_123'
    )
    
    # ========================================================================
    # Example 2: Using PayPal gateway
    # ========================================================================
    print("\n" + "="*80)
    print("EXAMPLE 2: PAYPAL PAYMENT GATEWAY")
    print("="*80)
    
    paypal_factory = PayPalFactory()
    paypal_service = PaymentService(paypal_factory)
    
    # Process order
    order_result = paypal_service.process_order({
        'order_id': 'ORD-002',
        'amount': Decimal('149.99'),
        'currency': 'USD',
        'card_token': 'paypal_token_456',
        'customer_id': 'cust_456'
    })
    print(f"\nOrder Result: {order_result['status']}")
    
    # ========================================================================
    # Example 3: Using Square gateway
    # ========================================================================
    print("\n" + "="*80)
    print("EXAMPLE 3: SQUARE PAYMENT GATEWAY")
    print("="*80)
    
    square_factory = SquareFactory()
    square_service = PaymentService(square_factory)
    
    # Process order
    order_result = square_service.process_order({
        'order_id': 'ORD-003',
        'amount': Decimal('199.99'),
        'currency': 'USD',
        'card_token': 'cnon_square_789',
        'customer_id': 'cust_789'
    })
    print(f"\nOrder Result: {order_result['status']}")
    
    # ========================================================================
    # Example 4: Runtime gateway selection
    # ========================================================================
    print("\n" + "="*80)
    print("EXAMPLE 4: RUNTIME GATEWAY SELECTION")
    print("="*80)
    
    def get_factory_for_merchant(merchant_preference: str) -> PaymentGatewayFactory:
        """Factory function - returns appropriate gateway factory"""
        factories = {
            'stripe': StripeFactory,
            'paypal': PayPalFactory,
            'square': SquareFactory
        }
        
        factory_class = factories.get(merchant_preference.lower())
        if not factory_class:
            raise ValueError(f"Unknown gateway: {merchant_preference}")
        
        return factory_class()
    
    # Merchant chooses their preferred gateway
    merchant_choice = 'stripe'
    factory = get_factory_for_merchant(merchant_choice)
    service = PaymentService(factory)
    
    print(f"Merchant selected: {merchant_choice}")
    order_result = service.process_order({
        'order_id': 'ORD-004',
        'amount': Decimal('299.99'),
        'currency': 'USD',
        'card_token': 'tok_mastercard',
        'customer_id': 'cust_999'
    })
    
    # ========================================================================
    # Example 5: Demonstrating consistency guarantee
    # ========================================================================
    print("\n" + "="*80)
    print("EXAMPLE 5: CONSISTENCY GUARANTEE")
    print("="*80)
    
    print("\n✅ WITH Abstract Factory:")
    print("   All components are guaranteed to be from same gateway")
    print("   StripeFactory → Stripe processor + Stripe refund + Stripe webhook")
    print("   PayPalFactory → PayPal processor + PayPal refund + PayPal webhook")
    print("   Components work together seamlessly!")
    
    print("\n❌ WITHOUT Abstract Factory:")
    print("   Could accidentally mix:")
    print("   Stripe processor + PayPal refund handler + Square webhook validator")
    print("   Result: Runtime errors, authentication failures, data format mismatches!")
    
    # ========================================================================
    # Example 6: Switching gateways (Open-Closed Principle)
    # ========================================================================
    print("\n" + "="*80)
    print("EXAMPLE 6: EASY GATEWAY SWITCHING")
    print("="*80)
    
    print("\nProcessing same order through different gateways:")
    
    order_data = {
        'order_id': 'ORD-005',
        'amount': Decimal('399.99'),
        'currency': 'USD',
        'card_token': 'tok_test',
        'customer_id': 'cust_test'
    }
    
    gateways = [
        ('Stripe', StripeFactory()),
        ('PayPal', PayPalFactory()),
        ('Square', SquareFactory())
    ]
    
    for name, factory in gateways:
        print(f"\n--- Using {name} ---")
        service = PaymentService(factory)
        result = service.process_order(order_data)
        print(f"Result: {result['status']}")
    
    print("\n" + "="*80)
    print("✅ DEMONSTRATION COMPLETE")
    print("="*80)
    
    # ========================================================================
    # Summary of benefits
    # ========================================================================
    print("\n" + "="*80)
    print("BENEFITS DEMONSTRATED:")
    print("="*80)
    print("""
1. ✅ Guaranteed Consistency
   - All components from same gateway family
   - No mixing Stripe processor with PayPal refund handler
   
2. ✅ Easy Gateway Switching
   - Change factory = change entire gateway
   - Client code unchanged
   
3. ✅ Decoupling
   - Client doesn't know concrete classes
   - Works only with interfaces
   
4. ✅ Open-Closed Principle
   - Add new gateway = add new factory + products
   - No modification to existing code
   
5. ✅ Single Responsibility
   - Each factory creates one family
   - Each product has one responsibility
   
6. ✅ Type Safety
   - Factory methods return correct types
   - Compile-time checking (with type hints)
    """)


if __name__ == "__main__":
    main()
```

---

## 📤 Expected Output (Abbreviated)

```
================================================================================
ABSTRACT FACTORY PATTERN - PAYMENT GATEWAY SYSTEM
================================================================================

================================================================================
EXAMPLE 1: STRIPE PAYMENT GATEWAY
================================================================================

======================================================================
Payment Service initialized with: Stripe
======================================================================

──────────────────────────────────────────────────────────────────────
Processing Order: ORD-001
──────────────────────────────────────────────────────────────────────

[Step 1] Fraud Detection
[Stripe Radar] Analyzing transaction: ORD-001
Risk Score: 0.15
Risk Level: low

[Step 2] Payment Processing
[Stripe] Processing $99.99 USD
[Stripe] Using API key: sk_test_stripe_key
[Stripe] Card token: tok_visa
✅ Payment processed: stripe_txn_123456

Order Result: success

──────────────────────────────────────────────────────────────────────
Processing Refund
──────────────────────────────────────────────────────────────────────
[Stripe] Processing refund for stripe_txn_123456
[Stripe] Refund amount: $99.99
✅ Refund processed: stripe_ref_789012

──────────────────────────────────────────────────────────────────────
Handling Webhook
──────────────────────────────────────────────────────────────────────
[Stripe] Validating webhook with secret: whsec_stripe_secret
[Stripe] Signature: stripe_sig_123
✅ Webhook validated: payment.succeeded

[Similar output for PayPal and Square examples...]

================================================================================
BENEFITS DEMONSTRATED:
================================================================================

1. ✅ Guaranteed Consistency
2. ✅ Easy Gateway Switching
3. ✅ Decoupling
4. ✅ Open-Closed Principle
5. ✅ Single Responsibility
6. ✅ Type Safety
```

---

## 🔄 Factory Method vs Abstract Factory - Comprehensive Comparison

### **1. Structural Difference**

```python
# ============================================================================
# FACTORY METHOD - Creates ONE product
# ============================================================================

class DocumentCreator(ABC):
    """Factory Method - ONE product type"""
    
    @abstractmethod
    def create_document(self) -> Document:
        """Creates ONE product"""
        pass
    
    def generate_report(self, title: str, data: str) -> str:
        """Template method - uses the ONE product"""
        doc = self.create_document()  # ← ONE product
        # ... algorithm that uses document
        return f"{title}: {doc.create_content()}"

class PDFCreator(DocumentCreator):
    def create_document(self) -> Document:
        return PDFDocument()  # ← Returns ONE product

# Usage:
creator = PDFCreator()
report = creator.generate_report("Sales", "Data")


# ============================================================================
# ABSTRACT FACTORY - Creates FAMILY of products
# ============================================================================

class DocumentFactory(ABC):
    """Abstract Factory - FAMILY of products"""
    
    @abstractmethod
    def create_document(self) -> Document:
        """Create document"""
        pass
    
    @abstractmethod
    def create_template(self) -> Template:
        """Create template"""
        pass
    
    @abstractmethod
    def create_stylesheet(self) -> Stylesheet:
        """Create stylesheet"""
        pass
    
    @abstractmethod
    def create_exporter(self) -> Exporter:
        """Create exporter"""
        pass

class PDFFactory(DocumentFactory):
    """Creates PDF family"""
    
    def create_document(self) -> Document:
        return PDFDocument()  # ← Product 1
    
    def create_template(self) -> Template:
        return PDFTemplate()  # ← Product 2
    
    def create_stylesheet(self) -> Stylesheet:
        return PDFStylesheet()  # ← Product 3
    
    def create_exporter(self) -> Exporter:
        return PDFExporter()  # ← Product 4

# Usage:
factory = PDFFactory()
doc = factory.create_document()      # Product 1
template = factory.create_template()  # Product 2
style = factory.create_stylesheet()   # Product 3
exporter = factory.create_exporter()  # Product 4
# All products work together (same family)
```

### **2. Inheritance vs Composition**

```python
# ============================================================================
# FACTORY METHOD - Uses INHERITANCE
# ============================================================================

class ReportGenerator(ABC):
    """Uses inheritance - subclass decides product"""
    
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    def generate(self) -> str:
        doc = self.create_document()
        return doc.create_content()

class PDFReportGenerator(ReportGenerator):
    """Subclass - inherits from creator"""
    def create_document(self) -> Document:
        return PDFDocument()

# Client creates subclass:
generator = PDFReportGenerator()  # ← Inheritance
report = generator.generate()


# ============================================================================
# ABSTRACT FACTORY - Uses COMPOSITION
# ============================================================================

class ReportGenerator:
    """Uses composition - factory creates products"""
    
    def __init__(self, factory: DocumentFactory):
        """Inject factory - composition"""
        self.factory = factory
    
    def generate(self) -> str:
        doc = self.factory.create_document()
        template = self.factory.create_template()
        style = self.factory.create_stylesheet()
        # Use all products together
        return f"{template.apply(doc, style)}"

# Client creates factory and injects:
factory = PDFFactory()
generator = ReportGenerator(factory)  # ← Composition
report = generator.generate()
```

### **3. Template Method Requirement**

```python
# ============================================================================
# FACTORY METHOD - REQUIRES template method
# ============================================================================

class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        """Factory method"""
        pass
    
    def generate_report(self) -> str:
        """TEMPLATE METHOD - required for Factory Method pattern"""
        doc = self.create_document()
        # Algorithm that uses document
        header = self._create_header()
        content = doc.create_content()
        footer = self._create_footer()
        return f"{header}\n{content}\n{footer}"

# Without template method, it's just an abstract interface, NOT Factory Method!


# ============================================================================
# ABSTRACT FACTORY - NO template method required
# ============================================================================

class UIFactory(ABC):
    """Just creation methods - no template method needed"""
    
    @abstractmethod
    def create_button(self) -> Button:
        pass
    
    @abstractmethod
    def create_checkbox(self) -> Checkbox:
        pass
    
    @abstractmethod
    def create_input(self) -> Input:
        pass

# Client uses products directly:
factory = WindowsFactory()
button = factory.create_button()
checkbox = factory.create_checkbox()
# No template method needed
```

### **4. Purpose and Use Cases**

```python
# ============================================================================
# FACTORY METHOD - Share algorithm, vary product
# ============================================================================

# Use when:
# - You have an ALGORITHM that uses a product
# - The algorithm is the SAME for all creators
# - Only the PRODUCT varies

class DataProcessor(ABC):
    """Same algorithm, different parser"""
    
    @abstractmethod
    def create_parser(self) -> Parser:
        """Varies"""
        pass
    
    def process(self, data: str) -> dict:
        """SAME algorithm for all processors"""
        parser = self.create_parser()  # ← Varies
        
        # Fixed algorithm:
        parsed = parser.parse(data)
        validated = self._validate(parsed)
        transformed = self._transform(validated)
        return transformed

class JSONProcessor(DataProcessor):
    def create_parser(self) -> Parser:
        return JSONParser()  # ← Only this varies

class XMLProcessor(DataProcessor):
    def create_parser(self) -> Parser:
        return XMLParser()  # ← Only this varies


# ============================================================================
# ABSTRACT FACTORY - Guarantee product family consistency
# ============================================================================

# Use when:
# - You have MULTIPLE related products
# - Products must be from SAME family
# - Need to prevent mixing incompatible products

class Application:
    """Needs consistent UI components"""
    
    def __init__(self, factory: UIFactory):
        # All from SAME family (guaranteed):
        self.button = factory.create_button()
        self.checkbox = factory.create_checkbox()
        self.input = factory.create_input()
        # All Windows OR all macOS OR all Linux
    
    def render(self):
        # All components have consistent look and feel
        self.button.paint()
        self.checkbox.paint()
        self.input.paint()
```

### **5. Comparison Table**

| **Aspect** | **Factory Method** | **Abstract Factory** |
|------------|-------------------|----------------------|
| **Creates** | ONE product type | FAMILY of related products |
| **Mechanism** | Inheritance (subclass) | Composition (factory object) |
| **Template Method** | Required (that's the point!) | Not required |
| **Primary Goal** | Share algorithm, vary product | Guarantee family consistency |
| **Client Code** | Subclasses creator | Uses factory object |
| **Adding Product Type** | Easy (new subclass) | Hard (modify all factories) |
| **Adding Family** | N/A (only one product) | Easy (new factory class) |
| **Complexity** | Simpler | More complex |
| **Example** | `DocumentCreator.generate_report()` creates and uses ONE document | `UIFactory` creates Button + Checkbox + Input (all from same platform) |

### **6. When to Use Which**

```python
# ============================================================================
# Use FACTORY METHOD when:
# ============================================================================

# ✅ Scenario 1: Algorithm uses product
class ReportGenerator(ABC):
    @abstractmethod
    def create_exporter(self) -> Exporter:
        pass
    
    def export_data(self, data: dict) -> bytes:
        """Algorithm that uses exporter"""
        exporter = self.create_exporter()
        # ... complex export algorithm
        return exporter.export(data)

# ✅ Scenario 2: Template method with hooks
class GameCharacter(ABC):
    @abstractmethod
    def create_weapon(self) -> Weapon:
        pass
    
    def attack(self, target):
        """Template method"""
        weapon = self.create_weapon()
        damage = weapon.calculate_damage()
        target.take_damage(damage)


# ============================================================================
# Use ABSTRACT FACTORY when:
# ============================================================================

# ✅ Scenario 1: Multiple related products
class UIFactory(ABC):
    def create_button(self) -> Button: pass
    def create_checkbox(self) -> Checkbox: pass
    def create_input(self) -> Input: pass
    # All must be from same family

# ✅ Scenario 2: Products must work together
class DatabaseFactory(ABC):
    def create_connection(self) -> Connection: pass
    def create_command(self) -> Command: pass
    def create_transaction(self) -> Transaction: pass
    # All must be for same database type

# ✅ Scenario 3: Prevent mixing incompatible products
class ThemeFactory(ABC):
    def create_colors(self) -> ColorScheme: pass
    def create_fonts(self) -> FontFamily: pass
    def create_icons(self) -> IconSet: pass
    # All must match theme (can't mix light colors with dark icons)
```

### **7. Can They Be Combined?**

**YES! Abstract Factory can use Factory Methods internally:**

```python
class UIFactory(ABC):
    """Abstract Factory using Factory Methods"""
    
    # Factory Methods (abstract):
    @abstractmethod
    def create_button(self) -> Button:
        pass
    
    @abstractmethod
    def create_checkbox(self) -> Checkbox:
        pass
    
    # Template Method (uses factory methods):
    def create_form(self, fields: List[str]) -> Form:
        """Template method that uses factory methods"""
        form = Form()
        
        for field in fields:
            # Uses factory methods:
            label = self.create_label(field)
            input_field = self.create_input()
            button = self.create_button()
            
            form.add_row(label, input_field, button)
        
        return form

class WindowsFactory(UIFactory):
    def create_button(self) -> Button:
        return WindowsButton()
    
    def create_checkbox(self) -> Checkbox:
        return WindowsCheckbox()
    
    def create_label(self, text: str) -> Label:
        return WindowsLabel(text)
    
    def create_input(self) -> Input:
        return WindowsInput()

# This combines:
# - Abstract Factory (creates family of UI components)
# - Factory Method (create_form is template method using factory methods)
```

---

## 🎓 Key Takeaways

### **1. Abstract Factory in One Sentence**

> "Abstract Factory creates families of related objects, guaranteeing they're all from the same variant/family."

### **2. The Three Guarantees**

1. **Consistency** - All products from same family
2. **Compatibility** - Products work together seamlessly
3. **Decoupling** - Client doesn't know concrete classes

### **3. When to Use (Decision Tree)**

```
Do you need to create objects?
│
├─ ONE product type?
│  └─ Use Factory Method (if you have template method)
│     or Simple Factory (if you don't)
│
└─ MULTIPLE related products?
   │
   ├─ Must be from same family?
   │  └─ YES → Abstract Factory ✅
   │
   └─ Independent products?
      └─ NO → Multiple simple factories
```

### **4. Real-World Mental Models**

**Abstract Factory is like:**
- **Car manufacturer** - Toyota factory makes Toyota engine + Toyota transmission + Toyota wheels (all compatible)
- **Restaurant kitchen** - Italian kitchen makes Italian appetizer + Italian main + Italian dessert (consistent cuisine)
- **Theme park** - Disneyland creates Disney rides + Disney shows + Disney merchandise (consistent brand)

**Factory Method is like:**
- **Assembly line** - Same assembly process, different product at the end
- **Template document** - Same structure, different content
- **Recipe** - Same cooking steps, different ingredient

### **5. Common Pitfalls**

❌ **Using Abstract Factory for one product** → Use Factory Method instead
❌ **Mixing products from different families** → Defeats the purpose
❌ **Adding new product types frequently** → Abstract Factory makes this hard
❌ **No relationship between products** → Don't need Abstract Factory

---

## 💡 Final Comparison Example

```python
# ============================================================================
# SAME DOMAIN - Different patterns
# ============================================================================

# FACTORY METHOD - Document generation
class DocumentCreator(ABC):
    """Creates ONE document, has algorithm"""
    
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    def generate_report(self, data: str) -> str:
        """Template method - algorithm"""
        doc = self.create_document()
        header = self._create_header()
        content = doc.format(data)
        return f"{header}\n{content}"

# ABSTRACT FACTORY - Document ecosystem
class DocumentFactory(ABC):
    """Creates FAMILY of document-related objects"""
    
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    @abstractmethod
    def create_template(self) -> Template:
        pass
    
    @abstractmethod
    def create_stylesheet(self) -> Stylesheet:
        pass
    
    @abstractmethod
    def create_exporter(self) -> Exporter:
        pass

# Different purposes:
# - Factory Method: Share report generation algorithm
# - Abstract Factory: Ensure document, template, stylesheet, exporter all match
```

---

**You now have a complete understanding of Abstract Factory and how it differs from Factory Method!** 🎯