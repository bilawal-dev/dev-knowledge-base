# JavaScript Array and Object Comparison with JSON.stringify

A quick reference for deep equality comparison of arrays and objects in JavaScript.

---

## 1. The Problem

In JavaScript, comparing arrays or objects directly doesn't work as expected:

```javascript
const arr1 = ['option1', 'option2'];
const arr2 = ['option1', 'option2'];

arr1 === arr2  // false (compares references, not values)
arr1 == arr2   // false (same issue)
```

Even though both arrays have identical content, they're different objects in memory, so the comparison returns `false`.

---

## 2. The Solution

`JSON.stringify` converts the array/object to a string representation:

```javascript
JSON.stringify(arr1)  // '["option1","option2"]'
JSON.stringify(arr2)  // '["option1","option2"]'

JSON.stringify(arr1) === JSON.stringify(arr2)  // true (string comparison)
```

Now we're comparing strings, which works correctly for equality.

---

## 3. Practical Example

Detecting if an options array changed in an activity logger:

```javascript
if (JSON.stringify(diff.before.options) !== JSON.stringify(diff.after.options)) {
    return 'Updated options';
}
```

This detects if the options array changed - whether items were added, removed, or reordered.

---

## 4. Limitations

| Limitation | Description |
|------------|-------------|
| Order matters | `['a', 'b']` vs `['b', 'a']` are considered different |
| Performance | Not ideal for very large objects |
| Circular references | Will throw an error |

---

## 5. When to Use

For small arrays of strings or simple objects, `JSON.stringify` is a simple and effective approach for deep equality comparison.

For complex scenarios, consider using:
* Lodash's `_.isEqual()`
* A custom deep comparison function
* Libraries like `fast-deep-equal`

---

This document serves as a quick reference for array/object comparison patterns in JavaScript.
