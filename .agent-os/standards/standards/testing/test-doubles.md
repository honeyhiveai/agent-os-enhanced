# Test Doubles - Universal Testing Pattern

**Timeless patterns for isolating code during testing.**

## What are Test Doubles?

Test doubles are objects that stand in for real dependencies during testing, allowing you to test code in isolation.

**Terminology by Gerard Meszaros (xUnit Test Patterns, 2007)**

**Key principle:** Test the code you're writing, not its dependencies.

---

## The Five Types of Test Doubles

```
Test Double (Generic Term)
    ├── Dummy
    ├── Stub
    ├── Spy
    ├── Mock
    └── Fake
```

---

## Type 1: Dummy

**Definition:** Objects passed around but never actually used. Typically just fulfill parameter lists.

**Purpose:** Satisfy required parameters when they're not relevant to the test.

### Example

```
// Real interface
interface Logger:
    def log(message)
    def error(message)

// Production code
class UserService:
    def __init__(self, database, logger):
        self.database = database
        self.logger = logger
    
    def create_user(self, name):
        user = self.database.save(User(name))
        self.logger.log(f"Created user: {name}")
        return user

// Test
def test_create_user():
    database = InMemoryDatabase()
    dummy_logger = DummyLogger()  # Never actually called in this test
    
    service = UserService(database, dummy_logger)
    user = service.create_user("Alice")
    
    assert user.name == "Alice"

class DummyLogger implements Logger:
    def log(self, message):
        pass  # Do nothing
    
    def error(self, message):
        pass  # Do nothing
```

**When to use:** Parameter is required but not relevant to the test.

---

## Type 2: Stub

**Definition:** Objects that return pre-configured responses to method calls.

**Purpose:** Control the test environment by providing predetermined data.

### Example

```
// Real interface
interface WeatherService:
    def get_temperature(city)

// Production code
class TravelRecommender:
    def __init__(self, weather_service):
        self.weather_service = weather_service
    
    def recommend_clothing(self, city):
        temp = self.weather_service.get_temperature(city)
        if temp < 10:
            return "Wear a coat"
        elif temp < 20:
            return "Wear a jacket"
        else:
            return "T-shirt is fine"

// Test with stub
def test_recommend_clothing_cold():
    stub_weather = StubWeatherService(temperature=5)  # Stub returns 5
    
    recommender = TravelRecommender(stub_weather)
    clothing = recommender.recommend_clothing("Paris")
    
    assert clothing == "Wear a coat"

class StubWeatherService implements WeatherService:
    def __init__(self, temperature):
        self.temperature = temperature
    
    def get_temperature(self, city):
        return self.temperature  # Always returns predetermined value
```

**When to use:** Need to control what dependencies return.

**Characteristics:**
- Returns hardcoded values
- Doesn't verify calls
- No logic, just returns data

---

## Type 3: Spy

**Definition:** Objects that record information about how they were called.

**Purpose:** Verify indirect outputs (that certain methods were called with certain arguments).

### Example

```
// Real interface
interface EmailService:
    def send_email(recipient, subject, body)

// Production code
class PasswordResetService:
    def __init__(self, email_service):
        self.email_service = email_service
    
    def reset_password(self, user):
        new_password = generate_random_password()
        user.password = new_password
        self.email_service.send_email(
            user.email,
            "Password Reset",
            f"Your new password is: {new_password}"
        )
        return True

// Test with spy
def test_reset_password_sends_email():
    spy_email = SpyEmailService()
    
    service = PasswordResetService(spy_email)
    user = User(email="alice@example.com")
    
    service.reset_password(user)
    
    # Verify the spy recorded the call
    assert spy_email.send_called == True
    assert spy_email.last_recipient == "alice@example.com"
    assert "Password Reset" in spy_email.last_subject

class SpyEmailService implements EmailService:
    def __init__(self):
        self.send_called = False
        self.last_recipient = None
        self.last_subject = None
        self.last_body = None
    
    def send_email(self, recipient, subject, body):
        self.send_called = True
        self.last_recipient = recipient
        self.last_subject = subject
        self.last_body = body
```

**When to use:** Need to verify that methods were called with correct arguments.

**Characteristics:**
- Records method calls
- Allows verification after the fact
- Manual assertions

---

## Type 4: Mock

**Definition:** Objects pre-programmed with expectations about calls they should receive.

**Purpose:** Verify that code interacts with dependencies correctly, with built-in verification.

### Example

```
// Production code
class OrderService:
    def __init__(self, payment_service, inventory_service):
        self.payment_service = payment_service
        self.inventory_service = inventory_service
    
    def place_order(self, order):
        # Must charge payment first
        self.payment_service.charge(order.total)
        
        # Then reduce inventory
        for item in order.items:
            self.inventory_service.reduce_stock(item.product_id, item.quantity)
        
        return True

// Test with mock
def test_place_order():
    mock_payment = MockPaymentService()
    mock_inventory = MockInventoryService()
    
    # Set expectations
    mock_payment.expect_charge(100.00)
    mock_inventory.expect_reduce_stock("product-123", 2)
    
    service = OrderService(mock_payment, mock_inventory)
    order = Order(total=100.00, items=[Item("product-123", 2)])
    
    service.place_order(order)
    
    # Verify expectations were met
    mock_payment.verify()  # Throws if expectations not met
    mock_inventory.verify()

class MockPaymentService:
    def __init__(self):
        self.expected_charges = []
        self.actual_charges = []
    
    def expect_charge(self, amount):
        self.expected_charges.append(amount)
    
    def charge(self, amount):
        self.actual_charges.append(amount)
    
    def verify(self):
        assert self.expected_charges == self.actual_charges
```

**When to use:** Need to verify complex interactions with strict expectations.

**Characteristics:**
- Pre-programmed with expectations
- Fails test if expectations not met
- Built-in verification

**Difference from Spy:**
- **Spy:** Records calls, you verify manually
- **Mock:** Has expectations, verifies automatically

---

## Type 5: Fake

**Definition:** Objects with working implementations, but simplified (e.g., in-memory database instead of real database).

**Purpose:** Replace expensive or complex dependencies with lightweight alternatives.

### Example

```
// Real interface
interface Database:
    def save(entity)
    def find_by_id(id)
    def find_all()
    def delete(id)

// Production implementation (real database)
class PostgreSQLDatabase implements Database:
    def save(self, entity):
        # Complex SQL logic, network calls, transactions
        pass
    
    def find_by_id(self, id):
        # SQL queries, connection pooling
        pass

// Fake implementation (in-memory)
class InMemoryDatabase implements Database:
    def __init__(self):
        self.entities = {}
        self.next_id = 1
    
    def save(self, entity):
        entity.id = self.next_id
        self.entities[self.next_id] = entity
        self.next_id += 1
        return entity
    
    def find_by_id(self, id):
        return self.entities.get(id)
    
    def find_all(self):
        return list(self.entities.values())
    
    def delete(self, id):
        if id in self.entities:
            del self.entities[id]

// Test with fake
def test_user_repository():
    fake_db = InMemoryDatabase()
    repository = UserRepository(fake_db)
    
    # Create user
    user = User(name="Alice")
    saved = repository.save(user)
    assert saved.id is not None
    
    # Find user
    found = repository.find_by_id(saved.id)
    assert found.name == "Alice"
    
    # Delete user
    repository.delete(saved.id)
    assert repository.find_by_id(saved.id) is None
```

**When to use:** Dependency is too slow or complex for tests, but you need realistic behavior.

**Characteristics:**
- Has real logic (not just hardcoded responses)
- Simplified implementation
- Often reusable across many tests

---

## Comparison Matrix

| Type | Returns Data | Records Calls | Verifies Expectations | Has Logic | Use Case |
|------|-------------|---------------|----------------------|-----------|----------|
| **Dummy** | ❌ | ❌ | ❌ | ❌ | Fill parameter lists |
| **Stub** | ✅ | ❌ | ❌ | Minimal | Control input data |
| **Spy** | ✅ | ✅ | ❌ (manual) | Minimal | Verify calls manually |
| **Mock** | ✅ | ✅ | ✅ (automatic) | Minimal | Verify complex interactions |
| **Fake** | ✅ | ❌ | ❌ | ✅ Simplified | Replace slow dependencies |

---

## When to Use Each Type

### Use Dummy when:
- Parameter is required but not used in test
- Example: Logger passed but test doesn't log anything

### Use Stub when:
- Need to control what dependency returns
- Testing different scenarios (error cases, edge cases)
- Example: API returns 404, 500, timeout

### Use Spy when:
- Need to verify method was called
- Need to check arguments passed
- Manual verification is fine
- Example: Verify email was sent with correct recipient

### Use Mock when:
- Need to verify complex call sequences
- Need to verify calls were made in specific order
- Want automatic verification
- Example: Verify payment charged before inventory reduced

### Use Fake when:
- Dependency is too slow (database, network)
- Need realistic behavior (not just return values)
- Multiple tests can reuse the same fake
- Example: In-memory database for repository tests

---

## Anti-Patterns

### Anti-Pattern 1: Mocking Everything
❌ Mocking every dependency, even simple ones.

**Problem:** Tests become brittle, coupled to implementation details.

**Solution:** Only mock external dependencies (database, network, file system). Use real objects for simple classes.

### Anti-Pattern 2: Stubbing Private Methods
❌ Stubbing private/internal methods of the class under test.

**Problem:** Tests are coupled to implementation, not behavior.

**Solution:** Only stub dependencies, not internals. If you need to stub internal methods, consider refactoring.

### Anti-Pattern 3: Over-Specifying Mocks
❌ Mock has expectations for every single method call.

**Problem:** Tests are brittle, break on any refactoring.

**Solution:** Only verify what matters. Use spies for loose verification, mocks for critical interactions only.

### Anti-Pattern 4: Testing the Mock
❌ Test verifies mock behavior instead of production code behavior.

**Problem:** Test isn't testing anything real.

**Solution:** Ensure tests verify actual business logic, not just that mocks were called.

---

## Test Double Frameworks (Language-Specific)

Most languages have test double frameworks:

- **Python:** `unittest.mock`, `pytest-mock`, `doubles`
- **JavaScript:** `sinon.js`, `jest.mock`
- **Java:** `Mockito`, `EasyMock`, `JMock`
- **Go:** `gomock`, `testify/mock`
- **C#:** `Moq`, `NSubstitute`, `FakeItEasy`
- **Rust:** `mockall`, `mockito`

**See language-specific guides for concrete examples.**

---

## Best Practices

### 1. Prefer Fakes for Complex Dependencies
If you can build a simple in-memory fake, it's often better than mocks.

**Benefits:**
- Reusable across many tests
- More realistic behavior
- Less brittle

### 2. Use Stubs for Data Control
When you need to control input data (error cases, edge cases), stubs are perfect.

### 3. Use Spies for Loose Verification
When you need to verify calls were made but don't care about exact sequence, spies work well.

### 4. Use Mocks for Critical Interactions
When order matters or interactions are complex, mocks provide strong verification.

### 5. Don't Mock What You Don't Own
Avoid mocking third-party libraries directly. Create an adapter/wrapper and mock that instead.

---

## Language-Specific Implementation

**This document covers universal concepts. For language-specific implementations:**
- See `.agent-os/standards/development/python-testing.md` (Python: `unittest.mock.Mock`, `MagicMock`)
- See `.agent-os/standards/development/go-testing.md` (Go: interfaces, table tests)
- See `.agent-os/standards/development/js-testing.md` (JavaScript: `sinon`, `jest.mock`)
- Etc.

---

**Test doubles are essential for isolated, fast unit tests. Choose the right type for your needs: Dummy for unused parameters, Stub for data control, Spy for loose verification, Mock for strict verification, and Fake for realistic lightweight alternatives.**
