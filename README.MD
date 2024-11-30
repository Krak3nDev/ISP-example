# Interface Segregation Principle (ISP) in Practice: A Practical Example

## Interface Clients: Who Are They?

A **client** is the code that uses an interface.

**Key Points:**
- Clients define the requirements for the interface.
- Client dependencies are limited to the interface's methods.
- Different clients may need different parts of the interface.

As Martin Fowler noted, **"Abstractions should always be defined by their clients."** This implies:
- Interface methods are determined by clients.
- Interface implementations are adapted to client requests, not the other way around.
- Interface design is focused on usage rather than internal implementation.

**Example of Clients** in a user management system:
- **Authentication Service** only requires a method to retrieve a user by ID.
- **Registration Service** only needs a method to add users.
- **Administrative Panel** uses functionality to retrieve a list of all users.

Thus, the interface should be thoughtfully designed to **accommodate the individual needs of all its clients.**

## Interface Segregation Principle (ISP)

The **Interface Segregation Principle (ISP)** states: "Clients should not be forced to depend on interfaces they do not use."

This principle entails:
- Interfaces should be small and specialized.
- Each interface should serve a specific purpose.
- Clients should only see and use the methods they need.

## The Problem: Violating ISP in Practice

Let's consider an example of a user system. Initially, we created one large interface:

```python
from typing import Protocol
from uuid import UUID
from decimal import Decimal
from dataclasses import dataclass

@dataclass
class User:
    id: UUID
    balance: Decimal

class UserGateway(Protocol):
    async def add(self, user: User) -> None: ...
    async def get(self, user_id: int) -> User: ...
    async def get_all(self) -> list[User]: ...
```

### SQL Implementation

```python
class DBUserGateway(UserGateway):
    async def get(self, user_id: int) -> User:
        print(f"Fetching user {user_id} from SQL")
        return User(id=user_id, balance=Decimal('100.00'))

    async def add(self, user: User) -> None:
        print(f"Adding user {user.id} with balance {user.balance} to SQL")

    async def get_all(self) -> list[User]:
        print("Fetching all users from SQL")
        return [
            User(id=UUID("12345678-1234-5678-1234-567812345678"), balance=Decimal('100.00')),
            User(id=UUID("87654321-4321-8765-4321-876543218765"), balance=Decimal('200.00')),
        ]
```

### The Problem Arises

When caching with Redis was needed, we encountered a problem:

```python
class RedisUserGateway(UserGateway):
    async def get(self, user_id: int) -> User:
        print(f"Fetching user {user_id} from Redis")
        return User(id=user_id, balance=Decimal('100.00'))

    async def add(self, user: User) -> None:
        print(f"Adding user {user.id} with balance {user.balance} to Redis")

    async def get_all(self) -> list[User]:
        # The method is not needed for the cache, but we have to implement it
        raise NotImplementedError("get_all is not supported in Redis")
```

### Problems with This Approach:

1. The Redis implementation is forced to implement an unnecessary method.
2. Clients depend on methods they do not need.
3. Testing becomes complicated due to the need to mock unused methods.

## Solution: Applying ISP

Let's split the large interface into specialized ones:

```python
class UserByIdGetter(Protocol):
    async def get(self, user_id: int) -> User: ...

class UserAdder(Protocol):
    async def add(self, user: User) -> None: ...

class UserLister(Protocol):
    async def get_all(self) -> list[User]: ...
```

### New Implementations

The SQL implementation uses all interfaces:

```python
class DBUserGateway(UserByIdGetter, UserAdder, UserLister):
    async def get(self, user_id: UUID) -> User:
        print(f"Fetching user {user_id} from SQL")
        return User(id=user_id, balance=Decimal('100.00'))

    async def add(self, user: User) -> None:
        print(f"Adding user {user.id} with balance {user.balance} to SQL")

    async def get_all(self) -> list[User]:
        print("Fetching all users from SQL")
        return [
            User(id=UUID("12345678-1234-5678-1234-567812345678"), balance=Decimal('100.00')),
            User(id=UUID("87654321-4321-8765-4321-876543218765"), balance=Decimal('200.00')),
        ]
```

The Redis implementation uses only the necessary interfaces:

```python
class RedisUserGateway(UserByIdGetter, UserAdder):
    async def get(self, user_id: int) -> User:
        print(f"Fetching user {user_id} from Redis")
        return User(id=user_id, balance=Decimal('100.00'))

    async def add(self, user: User) -> None:
        print(f"Adding user {user.id} with balance {user.balance} to Redis")
```

## Advantages of the Resulting Solution

1. **Flexibility**: Clients depend only on the methods they need.
2. **Ease of Testing**: No need to mock unused methods.
3. **Clarity of Intent**: Each interface has a clear responsibility.
4. **Ease of Extension**: New implementations can use only the interfaces they need.

## Conclusion

Adhering to the ISP allows for creating more flexible and maintainable systems. Key points:
- Interfaces should be shaped based on client needs.
- It's better to have many small, specialized interfaces.
- Implementations should be able to use only the interfaces they need.

## References

1. [The Interface Segregation Principle by Robert C. Martin](https://www.webcitation.org/6AL2qqIGg?url=http://www.objectmentor.com/resources/articles/isp.pdf)  
   An article on the Interface Segregation Principle that emphasizes the importance of designing client-focused interfaces.

2. [Separated Interface by Martin Fowler](https://martinfowler.com/eaaCatalog/separatedInterface.html)  
   Discussion of the concept of separated interfaces, where the interface is defined separately from the implementation, allowing clients to focus only on the methods they need.

3. [Interface Implementation Pair by Martin Fowler](https://martinfowler.com/bliki/InterfaceImplementationPair.html)  
   An article on how to properly design interfaces, avoiding the creation of unnecessary "interface-implementation" pairs and focusing on client needs.
