# 07 · Memory Management

Swift manages memory for class instances with Automatic Reference Counting
(ARC): every strong reference bumps a retain count, and an instance is
deallocated the moment its count hits zero. This is invisible almost all of
the time — until two objects reference each other and neither count ever
reaches zero. This module covers `weak`, `unowned`, and the closure-capture
version of the same problem.

## The retain cycle

Two classes holding strong references to each other never deallocate,
because each one's retain count is kept above zero by the other:

```swift
final class Owner {
    let name: String
    var pet: Pet?   // if this were strong AND Pet.owner were strong, cycle
    init(name: String) { self.name = name; print("Owner \(name) init") }
    deinit { print("Owner \(name) deinit") }
}
```

## `weak`: breaking the cycle, allowing nil

`weak` references don't increase the retain count, and Swift automatically
sets them to `nil` when the referenced object is deallocated — which is why
`weak` properties must always be `Optional var`:

```swift
final class Pet {
    let name: String
    weak var owner: Owner?
    init(name: String) { self.name = name; print("Pet \(name) init") }
    deinit { print("Pet \(name) deinit") }
}

func weakDemo() {
    let owner = Owner(name: "Sam")
    let pet = Pet(name: "Rex")
    owner.pet = pet
    pet.owner = owner
    print("Both created, no retain cycle because owner.pet<->pet.owner uses weak")
}
weakDemo()
print("--- after weakDemo scope ---")
```

Output:

```
Owner Sam init
Pet Rex init
Both created, no retain cycle because owner.pet<->pet.owner uses weak
Owner Sam deinit
Pet Rex deinit
--- after weakDemo scope ---
```

Both `deinit`s fire right when `weakDemo()` returns and its local
`owner`/`pet` variables go out of scope — `owner.pet`'s strong reference to
`pet`, and `pet.owner`'s weak (non-counting) reference back, don't keep each
other alive.

## `unowned`: like `weak`, but non-optional and unsafe if wrong

`unowned` is for a reference that should *never* legitimately be `nil`
while it's in use — a `CreditCard` doesn't outlive its `Customer`, so it
holds `customer` as `unowned` instead of `weak`, avoiding the need to
unwrap an optional on every access:

```swift
final class CreditCard {
    let number: String
    unowned let customer: Customer
    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
        print("CreditCard \(number) init")
    }
    deinit { print("CreditCard \(number) deinit") }
}

final class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) { self.name = name; print("Customer \(name) init") }
    deinit { print("Customer \(name) deinit") }
}

func unownedDemo() {
    let customer = Customer(name: "Mia")
    customer.card = CreditCard(number: "1234", customer: customer)
    print("Customer and card created, unowned avoids retain cycle")
}
unownedDemo()
print("--- after unownedDemo scope ---")
```

Output:

```
Customer Mia init
CreditCard 1234 init
Customer and card created, unowned avoids retain cycle
Customer Mia deinit
CreditCard 1234 deinit
--- after unownedDemo scope ---
```

If a `CreditCard` somehow outlived its `Customer` and something accessed
`card.customer`, the program would crash — `unowned` trades the safety of
optional-unwrapping for the convenience of non-optional access, so only use
it when the lifetime relationship truly guarantees the referenced object is
still alive.

## Closures capture strongly by default

A closure stored on a class, referencing that same class inside its body,
is exactly as much a retain cycle as two objects pointing at each other —
just less visible:

```swift
final class Cache {
    var onEvict: (() -> Void)?
    let label: String
    init(label: String) { self.label = label }
    deinit { print("Cache \(label) deinit") }
}

func closureCaptureDemo() {
    let cache = Cache(label: "leaky")
    cache.onEvict = {
        print("Evicting \(cache.label)")   // strong capture of `cache` -> retain cycle
    }
    cache.onEvict = nil   // breaking it manually here avoids a real leak in this demo
    print("closure cleared to avoid the cycle")
}
closureCaptureDemo()
```

Without the manual `cache.onEvict = nil`, `cache` would never deallocate:
`cache` strongly holds the closure via `onEvict`, and the closure strongly
holds `cache` by capturing it — a two-node cycle exactly like `Owner`/`Pet`
would have been without `weak`.

## `[weak self]`: the standard fix

The idiomatic fix for a closure held by `self` and capturing `self` is a
weak capture list, unwrapped at the top of the closure:

```swift
final class Downloader {
    var completion: (() -> Void)?
    let id: Int
    init(id: Int) { self.id = id }
    deinit { print("Downloader \(id) deinit") }

    func start() {
        completion = { [weak self] in
            guard let self else { return }
            print("Download \(self.id) finished")
        }
    }
}

let downloader = Downloader(id: 7)
downloader.start()
downloader.completion?()
```

Full run of every section above (`swift memory-management.swift`):

```
Owner Sam init
Pet Rex init
Both created, no retain cycle because owner.pet<->pet.owner uses weak
Owner Sam deinit
Pet Rex deinit
--- after weakDemo scope ---
Customer Mia init
CreditCard 1234 init
Customer and card created, unowned avoids retain cycle
Customer Mia deinit
CreditCard 1234 deinit
--- after unownedDemo scope ---
closure cleared to avoid the cycle
Cache leaky deinit
Download 7 finished
Downloader 7 deinit
--- done ---
```

Note `Cache leaky deinit` prints right after clearing `onEvict` — proof
the manual break worked; had `onEvict` kept its strong-capturing closure,
that line would never appear.

## Swift-specific traps

- **`weak` requires `Optional var`** — you cannot declare a `weak let` or a
  non-optional `weak` property; the compiler enforces both because a weak
  reference must be nil-able and assignable to `nil` automatically.
- **`unowned` crashes (not just misbehaves) if the referenced object is
  already deallocated** — reach for `weak` instead whenever the lifetime
  guarantee isn't airtight.
- **`guard let self else { return }` inside `[weak self]` needs Swift 5.7+**
  (the shorthand unwrap) — older code spells this `guard let self = self
  else { return }`; both work, but mixing styles inconsistently across a
  codebase is a readability trap for reviewers expecting one or the other.
- **Structs and enums never participate in retain cycles** — ARC only
  tracks class instances (and closures, which are reference types under the
  hood); switching a type from `class` to `struct` where possible sidesteps
  this entire category of bug.

## Cheat sheet

| Tool | Nilable? | Use when |
|------|----------|----------|
| Strong (default) | No | The normal case; the owner should keep the referenced object alive |
| `weak` | Yes, auto-nils | A back-reference where the referenced object might be deallocated first |
| `unowned` | No, non-optional | A back-reference guaranteed to outlive the referencer |
| `[weak self]` in a closure | Yes | A closure stored by `self` that also references `self` |

## Exercise

Model a `Parent`/`Child` pair where `Parent` holds `var children: [Child]`
(strong, an array can own many children safely) and each `Child` holds
`weak var parent: Parent?`. Create a `Parent` with two `Child` instances
inside a function, print a message confirming both `Child.deinit`s and the
`Parent.deinit` fire when the function returns, and explain in a comment
why using `unowned` instead of `weak` for `Child.parent` would be riskier
here specifically.
