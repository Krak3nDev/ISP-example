# Принцип разделения интерфейсов (ISP) на практическом примере

## Клиенты интерфейса: кто они?

**Клиент** — это код, использующий интерфейс.

Ключевые моменты:
- Именно клиенты формируют требования к интерфейсу.
- Зависимость клиентов ограничивается методами интерфейса.
- Разным клиентам могут быть нужны различные части интерфейса.

Как отмечал Мартин Фаулер: **"абстракции всегда должны определяться их клиентами"**. Это подразумевает:
- Методы интерфейса определяются клиентами.
- Реализации интерфейсов адаптируются под запросы клиентов, а не наоборот.
- Проектирование интерфейсов ориентировано на их использование, а не на внутреннюю реализацию.

Пример **клиентов** из системы управления пользователями:
- **Сервис аутентификации** требует только метода получения пользователя по ID.
- **Сервис регистрации** нуждается исключительно в методе добавления пользователей.
- **Административная панель** использует функциональность для получения списка всех пользователей.

Таким образом, интерфейс должен быть продуман так, чтобы **учитывать индивидуальные потребности всех его клиентов.**

## Принцип разделения интерфейсов (ISP)

**Interface Segregation Principle (ISP)** гласит: "Клиенты не должны зависеть от интерфейсов, которые они не используют."

Этот принцип предполагает:
- Интерфейсы должны быть маленькими и специализированными
- Каждый интерфейс должен служить конкретной цели
- Клиенты должны видеть и использовать только нужные им методы

## Проблема: нарушение ISP на практике

Рассмотрим пример системы с пользователями. Изначально мы создали один большой интерфейс:

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

### SQL реализация

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

### Возникновение проблемы

Когда потребовалось добавить кеширование в Redis, мы столкнулись с проблемой:

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

### Проблемы такого подхода:

1. Redis-реализация вынуждена реализовывать ненужный метод
2. Клиенты зависят от методов, которые им не нужны
3. Тестирование усложняется необходимостью мокать неиспользуемые методы

## Решение: применение ISP

Разделим большой интерфейс на специализированные:

```python
class UserByIdGetter(Protocol):
    async def get(self, user_id: int) -> User: ...

class UserAdder(Protocol):
    async def add(self, user: User) -> None: ...

class UserLister(Protocol):
    async def get_all(self) -> list[User]: ...
```

### Новые реализации

SQL реализация использует все интерфейсы:

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

Redis реализация использует только нужные интерфейсы:

```python
class RedisUserGateway(UserByIdGetter, UserAdder):
    async def get(self, user_id: int) -> User:
        print(f"Fetching user {user_id} from Redis")
        return User(id=user_id, balance=Decimal('100.00'))

    async def add(self, user: User) -> None:
        print(f"Adding user {user.id} with balance {user.balance} to Redis")
```

## Преимущества полученного решения

1. **Гибкость**: клиенты зависят только от нужных им методов
2. **Простота тестирования**: нет необходимости мокать неиспользуемые методы
3. **Ясность намерений**: каждый интерфейс имеет четкую ответственность
4. **Легкость расширения**: новые реализации могут использовать только нужные интерфейсы

## Заключение

Следование принципу ISP позволяет создавать более гибкие и поддерживаемые системы. Ключевые моменты:
- Интерфейсы должны формироваться исходя из потребностей клиентов
- Лучше иметь много маленьких специализированных интерфейсов
- Реализации должны иметь возможность использовать только нужные им интерфейсы

## Источники

1. [The Interface Segregation Principle by Robert C. Martin](https://www.webcitation.org/6AL2qqIGg?url=http://www.objectmentor.com/resources/articles/isp.pdf)  
   Статья о принципе разделения интерфейсов, которая подчеркивает важность проектирования интерфейсов, ориентированных на конкретных клиентов.

2. [Separated Interface by Martin Fowler](https://martinfowler.com/eaaCatalog/separatedInterface.html)  
   Обсуждение концепции разделенных интерфейсов, где интерфейс определяется отдельно от реализации, что позволяет клиентам сосредотачиваться только на нужных им методах.

3. [Interface Implementation Pair by Martin Fowler](https://martinfowler.com/bliki/InterfaceImplementationPair.html)  
   Статья о том, как правильно проектировать интерфейсы, избегая создания ненужных пар "интерфейс-реализация" и ориентируясь на потребности клиентов.
