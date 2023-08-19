## What is DRY

DRY stands for "Don't Repeat Yourself". It's a principle that roughly says that we should avoid duplication.

Most developers intuitively understand that repeating the same block of code multiple times in our code base is a bad idea, but what exactly is the problem?

## Problems

- **Bugs and Inconsistencies**

  There's a risk that someone will accidentally change the logic in one place but forget to update the same logic elsewhere

- **Making changes will take longer**

  It will take a bit longer to type out changes because we have to make them in multiple places. However that's a small issue compared to how much longer
  it will take us to reason about what changes to make in the first place. We're likely to spend time:

  - tracking down all the places where the logic is used
  - understanding the current implementation for each one, since they may be slightly different
  - figuring out the right change in each case

## What is duplication

We can see that these problems apply more widely than just when lines of code are repeated. As a simple example, we could implement that exact same functionality twice but without any lines of code in common. But we can still hit the problems above even if each piece of functionality is only implemented once.

The more general principle is

!!! note ""

    We should avoid depending on the same choice in multiple different places

Here "choice" means any decision about the functionality our software supports or how that functionality is implemented. These choices don't have to have been made by us, for instance they could come from a product team or an upstream system.

If we did have code that depends on particular choice scattered throughout our system, then if that choice were to change we would hit the same problems described above.

To give a couple of examples:

#### Example 1

- We need to be able to quickly find items in a list of names, so we **choose** to keep the list sorted.
- In one part of the codebase we have a function that inserts items into the list, ensuring the list remains sorted.
- In another part of the codebase we have a function the finds an item in the list, assuming that the list is already sorted.

#### Example 2

- We need to store data about a customers order. We **choose** to store it in a csv file with the column order being (item, price, quantity)
- In one part of the codebase we write the csv file with column order (item, price, quantity)
- In another part of the codebase we read the csv file, assuming the column order is (item, price, quantity)

In both examples there is no duplication of the functionality, but there are multiple places where the logic relies on the same underlying choice about how the data is structured.

In both cases the solution is to move the two functions next to each other in the code base. Although there is still some duplication, the associated problems are greatly reduced because they are contained to a single file rather than the whole codebase.

Often we won't be able to completely remove duplication but we can still manage it by limiting it to a specific part of the codebase. This idea ties into other principles such as "Encapsulation" and "Separation of Concerns", which will be covered in other articles.

## Tradeoffs

### Generalization complexity

Sometimes each use case of the same logic is different enough that extracting the shared part is quite difficult. It might create more complexity to implement the general case rather than having a different implementation for each case.

### Over-generalization

Sometimes code is only "coincidentally" that same rather than logically representing the same thing. For example you may have two types, one that represents the user input for an order, and one that represents an order as it is stored in the database. They might currently have all the same fields, however logically they represent different things and will likely deviate overtime.

### Difficulty of updating shared functionality

If multiple functions call the same function, and then any changes to that API for that shared function will require updating all of the callers. This might be reasonable if the code is all in a single repository, however if the function is part of a shared library then this will be more difficult.

## Code Examples

### Repeated lines of code

This is a simple case where two functions implement very similar logic. The solution is to pull out the common part of the logic into a shared function.

```python title="With duplication"

def total_book_value(orders: List[Order]):
    return sum(
        order.price * order.quantity
        for order in orders
        if order.category == "book"
    )


def total_grocery_value(orders: List[Order]):
    return sum(
        order.price * order.quantity
        for order in orders
        if order.category == "grocery"
    )
```

```python title="Fixed"

def total_value_by_category(orders: List[Order], category: str) -> int:
    return sum(
        order.price * order.quantity
        for order in orders
        if order.category == category
    )


def total_book_value(orders: List[Order]) -> int:
    return total_value_by_category(orders, "book")


def total_grocery_value(orders: List[Order]) -> int:
    return total_value_by_category(orders, "grocery")


```

### Repeated constants

The problem here is that two different functions assume `book` is spelt in a certain way, i.e. `book` rather than `Book`. In general these constants are called "Magic strings" and they can be replaced by named constants. In this case there are multiple categories so we could also use an enum.

```python title="With duplication"
def total_book_value(orders: List[Order]) -> int:
    return total_value_by_category(orders, "book")


def create_book_order(price: int, quantity: int) -> Order:
    return Order(price, quantity, "book")

```

```python title="Fixed"

class Category(enum.Enum):
    BOOK = "book"
    GROCERY = "grocery"


def total_book_value(orders: List[Order]) -> int:
    return total_value_by_category(orders, Category.BOOK)


def create_book_order(price: int, quantity: int) -> Order:
    return Order(price, quantity, Category.BOOK)


```

### Duplicated storage format dependency

With duplication:

```python title="save_orders.py"

def save_orders(orders: List[Order]) -> None:
    with open("orders.csv", "w") as f:
        writer = csv.writer(f)
        for order in orders:
            writer.writerow((order.item, order.price, order.quantity))
```

```python title="total_value.py"

def total_value() -> int:
    total = 0
    with open("orders.csv", "r") as f:
        reader = csv.reader(f)
        for row in reader:
            item, price, quantity = row
            total += price * quantity
    return total

```

Fixed:

```python title="order_storage.py"

ORDER_FILENAME = "order.csv"

def save_orders(orders: List[Order]) -> None:
    with open(ORDER_FILENAME, "w") as f:
        writer = csv.writer(f)
        for order in orders:
            writer.writerow((order.item, order.price, order.quantity))


def read_order() -> List[Order]:
    results: List[Order] = []
    with open(ORDER_FILENAME, "r") as f:
        reader = csv.reader(f)
        for row in reader:
            item, price, quantity = row
            results.append(Order(item, price, quantity))
    return results
```

```python title="total_value.py"

def total_value() -> int
    orders = read_orders()
    return sum(order.price * order.quantity for order in orders)


```
