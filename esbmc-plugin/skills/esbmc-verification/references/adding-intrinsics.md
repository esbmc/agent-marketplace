# Adding Verification Intrinsics to Code

## Process

### Step 1: Analyze the Code

Read the code and identify:
- **Function inputs**: Parameters that should become symbolic
- **External inputs**: User input, file reads, network data
- **Array accesses**: Indices that need bounds assumptions
- **Pointer operations**: Allocations, dereferences, frees
- **Arithmetic operations**: Potential overflow points
- **Critical properties**: What should always be true (postconditions)
- **Implicit assumptions**: What the code assumes about inputs

### Step 2: Add Symbolic Inputs (for test harnesses)

Replace concrete inputs with non-deterministic values:

```c
// Before: concrete test
int test_sort() {
    int arr[5] = {3, 1, 4, 1, 5};
    sort(arr, 5);
}

// After: symbolic verification
int test_sort() {
    int arr[5];
    for (int i = 0; i < 5; i++) {
        arr[i] = __ESBMC_nondet_int();
    }
    sort(arr, 5);
}
```

### Step 3: Add Preconditions (Assumptions)

Constrain inputs to valid ranges:

```c
void process(int *arr, int size, int index) {
    __ESBMC_assume(arr != NULL);
    __ESBMC_assume(size > 0 && size <= MAX_SIZE);
    __ESBMC_assume(index >= 0 && index < size);
    // Original code follows...
}
```

### Step 4: Add Postconditions (Assertions)

Specify what must be true after operations:

```c
int abs_value(int x) {
    int result = (x < 0) ? -x : x;
    __ESBMC_assert(result >= 0, "Absolute value is non-negative");
    return result;
}
```

### Step 5: Add Safety Assertions

Insert checks at critical points:

```c
// Before array access
__ESBMC_assert(idx >= 0 && idx < size, "Array index in bounds");
arr[idx] = value;

// Before pointer dereference
__ESBMC_assert(ptr != NULL, "Pointer not null");
*ptr = value;

// After allocation
ptr = malloc(size);
__ESBMC_assert(ptr != NULL, "Allocation succeeded");

// After arithmetic (if overflow checking needed)
int sum = a + b;
__ESBMC_assert(sum >= a, "No overflow in addition");  // For positive values
```

### Step 6: Add Loop Invariants (for k-induction)

For unbounded verification, add invariants:

```c
int sum = 0;
for (int i = 0; i < n; i++) {
    __ESBMC_assert(i >= 0 && i <= n, "Loop index invariant");
    __ESBMC_assert(sum >= 0, "Running sum invariant");
    sum += arr[i];
}
```

## Guidelines

1. **Be conservative with assumptions**: Only assume what is truly necessary
2. **Be liberal with assertions**: Assert expected properties at critical points
3. **Document constraints**: Add comments explaining why assumptions exist
4. **Constrain symbolic values realistically**: Use domain knowledge for bounds
5. **Don't over-constrain**: Too many assumptions make verification trivial
6. **Preserve original behavior**: Intrinsics should not change program semantics

## Common Verification Patterns

### Safe Array Access
```c
#define SIZE 100
int arr[SIZE];
int idx = __ESBMC_nondet_int();
__ESBMC_assume(idx >= 0 && idx < SIZE);
arr[idx] = value;  // Verified safe
```

### Safe Pointer Handling
```c
int *ptr = malloc(sizeof(int));
if (ptr != NULL) {
    *ptr = value;
    // ... use ptr
    free(ptr);
    ptr = NULL;  // Prevent use-after-free
}
```

### Safe Integer Arithmetic
```c
#include <limits.h>
int a = __ESBMC_nondet_int();
int b = __ESBMC_nondet_int();
__ESBMC_assume(a >= 0 && a <= 10000);
__ESBMC_assume(b >= 0 && b <= 10000);
int sum = a + b;  // Cannot overflow
```

### Safe Division
```c
int a = __ESBMC_nondet_int();
int b = __ESBMC_nondet_int();
__ESBMC_assume(b != 0);  // Prevent division by zero
int result = a / b;
```

## Python Patterns

### Symbolic Inputs
```python
from esbmc import nondet_int, assume

x: int = nondet_int()
assume(x > 0 and x < 100)
assert x > 0
assert x < 100
```

### Postcondition Checking
```python
def compute(x: float) -> float:
    return x

x = nondet_float()
__ESBMC_assume(x >= 0)
result = compute(x)
assert result >= 0, "Result non-negative"
```

### List Verification
```python
size: int = nondet_int()
__ESBMC_assume(size > 0 and size <= 10)
lst: list[int] = []
for _ in range(size):
    val: int = nondet_int()
    __ESBMC_assume(val > -100 and val < 100)
    lst.append(val)
```

### Loop Invariant

```python
from typing import List
from esbmc import nondet_int, __ESBMC_assume

def two_sum(nums: List[int], n: int, target: int) -> List[int]:
    i: int = 0
    while i < n:
        assert i >= 0 and i < n, "INV outer: i in [0, n)"
        j: int = i + 1
        while j < n:
            assert j > i and j < n, "INV inner: i < j < n"
            if nums[i] + nums[j] == target:
                return [i, j]
            j = j + 1
        i = i + 1
    return []

def test_two_sum() -> None:
    n: int = nondet_int()
    __ESBMC_assume(n >= 2 and n <= 4)
    a0: int = nondet_int()
    a1: int = nondet_int()
    a2: int = nondet_int()
    a3: int = nondet_int()
    __ESBMC_assume(a0 >= -10 and a0 <= 10)
    __ESBMC_assume(a1 >= -10 and a1 <= 10)
    __ESBMC_assume(a2 >= -10 and a2 <= 10)
    __ESBMC_assume(a3 >= -10 and a3 <= 10)
    nums: List[int] = [a0, a1, a2, a3]
    target: int = nondet_int()
    __ESBMC_assume(target >= -20 and target <= 20)
    result: List[int] = two_sum(nums, n, target)
    assert len(result) == 0 or len(result) == 2, "Result length is 0 or 2"
    if len(result) == 2:
        idx_i: int = result[0]
        idx_j: int = result[1]
        assert idx_i >= 0 and idx_i < n, "First index in bounds"
        assert idx_j >= 0 and idx_j < n, "Second index in bounds"
        assert idx_i != idx_j, "Indices are distinct"
        assert nums[idx_i] + nums[idx_j] == target, "Sum equals target"
        assert idx_i < idx_j, "Indices returned in ascending order"
    found: bool = False
    p: int = 0
    while p < n:
        assert p >= 0 and p < n, "INV completeness outer: p in [0, n)"
        q: int = p + 1
        while q < n:
            assert q > p and q < n, "INV completeness inner: p < q < n"
            if nums[p] + nums[q] == target:
                found = True
            q = q + 1
        p = p + 1
    if found:
        assert len(result) == 2, "Solution exists but was not returned"

test_two_sum()
```

## C++ Patterns

### Class Invariant Verification
```cpp
class BoundedCounter {
    int value;
    int max_val;
public:
    BoundedCounter(int max) : value(0), max_val(max) {
        __ESBMC_assert(max > 0, "Max must be positive");
    }
    void increment() {
        if (value < max_val) value++;
        __ESBMC_assert(value >= 0 && value <= max_val, "Invariant");
    }
};
```

### STL Container Verification
```cpp
#include <vector>
std::vector<int> v;
int n = __ESBMC_nondet_int();
__ESBMC_assume(n > 0 && n <= 10);
for (int i = 0; i < n; i++) {
    v.push_back(__ESBMC_nondet_int());
}
assert(v.size() == (size_t)n);
```
