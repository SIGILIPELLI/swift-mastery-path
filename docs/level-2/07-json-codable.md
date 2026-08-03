# 07 · Working with JSON

Nearly every real app talks to the outside world through JSON — API
responses, config files, saved state. Swift's `Codable` protocol turns that
into a compile-time-checked, mostly automatic process: define a type that
mirrors the JSON shape, and the compiler generates the encoding/decoding
logic for you.

## `Codable` is `Encodable` + `Decodable`

```swift
import Foundation

struct Product: Codable {
    let name: String
    let price: Double
    let inStock: Bool
}
```

Because every stored property is itself `Codable` (`String`, `Double`, and
`Bool` all conform out of the box), the compiler synthesizes conformance
automatically — no extra code required. `Codable` is shorthand for
conforming to both `Encodable` (struct → JSON) and `Decodable` (JSON →
struct).

## Decoding JSON

```swift
let jsonData = """
{
    "name": "Wireless Mouse",
    "price": 24.99,
    "inStock": true
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let product = try! decoder.decode(Product.self, from: jsonData)
print(product.name, product.price, product.inStock)
// Wireless Mouse 24.99 true
```

`decoder.decode(Product.self, from:)` needs the *type* to decode into
(`Product.self`) because the same decoder works for any `Decodable` type —
it has no other way to know what shape to expect.

## Encoding to JSON

```swift
let newProduct = Product(name: "USB Cable", price: 8.5, inStock: false)

let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted   // optional, for human-readable output
let encodedData = try! encoder.encode(newProduct)
print(String(data: encodedData, encoding: .utf8)!)
// {
//   "name" : "USB Cable",
//   "price" : 8.5,
//   "inStock" : false
// }
```

## Decoding arrays and nested types

Codable composes naturally — a `Codable` struct containing another
`Codable` struct, or an array of them, decodes with no extra work:

```swift
struct Address: Codable {
    let city: String
    let zip: String
}

struct Customer: Codable {
    let name: String
    let address: Address        // nested Codable type
    let orderIds: [Int]         // array of a simple Codable type
}

let customerJSON = """
{
    "name": "Priya",
    "address": { "city": "Austin", "zip": "78701" },
    "orderIds": [101, 102, 103]
}
""".data(using: .utf8)!

let customer = try! JSONDecoder().decode(Customer.self, from: customerJSON)
print(customer.address.city)   // Austin
print(customer.orderIds)        // [101, 102, 103]
```

## The most common trap: mismatched key names

JSON keys are frequently `snake_case`, while Swift convention is
`camelCase`. Decoding fails at runtime, not compile time, if the names don't
line up — this is one of the single most common Codable bugs:

```swift
struct BrokenUser: Codable {
    let firstName: String   // JSON actually has "first_name"
}

let mismatchedJSON = """
{ "first_name": "Sam" }
""".data(using: .utf8)!

// try! JSONDecoder().decode(BrokenUser.self, from: mismatchedJSON)
// fatal error: 'try!' expression unexpectedly raised an error:
// keyNotFound(CodingKeys.firstName, ... "No value associated with key firstName")
```

### Fix 1: automatic key conversion

```swift
struct User: Codable {
    let firstName: String
}

let snakeCaseDecoder = JSONDecoder()
snakeCaseDecoder.keyDecodingStrategy = .convertFromSnakeCase   // first_name -> firstName automatically
let user = try! snakeCaseDecoder.decode(User.self, from: mismatchedJSON)
print(user.firstName)   // Sam
```

### Fix 2: explicit `CodingKeys`

Use this when key names don't follow a simple pattern, or you only want to
rename *some* fields:

```swift
struct Employee: Codable {
    let fullName: String
    let yearsOfService: Int

    enum CodingKeys: String, CodingKey {
        case fullName = "full_name"
        case yearsOfService = "tenure_years"
    }
}

let employeeJSON = """
{ "full_name": "Jordan Lee", "tenure_years": 4 }
""".data(using: .utf8)!

let employee = try! JSONDecoder().decode(Employee.self, from: employeeJSON)
print(employee.fullName, employee.yearsOfService)   // Jordan Lee 4
```

Once you write a `CodingKeys` enum, **every** stored property needs a case
in it (or must be handled in a custom `init(from:)`) — a property left out
of `CodingKeys` will fail to decode, since the compiler no longer
synthesizes the mapping automatically once you've taken it over.

## Optional fields and missing keys

A `Decodable` property typed as `Optional` is allowed to be missing from the
JSON entirely — `Codable` treats a missing key the same as an explicit `null`
for optional properties:

```swift
struct Profile: Codable {
    let username: String
    let bio: String?   // may be absent from the JSON
}

let noBioJSON = """
{ "username": "dev123" }
""".data(using: .utf8)!

let profile = try! JSONDecoder().decode(Profile.self, from: noBioJSON)
print(profile.bio ?? "no bio set")   // no bio set
```

## Handling decode failures properly (no `try!` in real code)

Every earlier example used `try!` to keep things short, but production code
should use real error handling — exactly the tools from
[Module 3](03-error-handling.md):

```swift
func decodeProduct(from data: Data) -> Product? {
    do {
        return try JSONDecoder().decode(Product.self, from: data)
    } catch {
        print("Decode failed: \(error)")
        return nil
    }
}

let badData = "not json at all".data(using: .utf8)!
print(decodeProduct(from: badData) ?? "decoding failed")
// Decode failed: dataCorrupted(...)
// decoding failed
```

## Cheat sheet

| Task | Tool |
|------|------|
| Struct ↔ JSON automatically | Conform to `Codable` |
| JSON → struct | `JSONDecoder().decode(Type.self, from: data)` |
| Struct → JSON | `JSONEncoder().encode(value)` |
| snake_case JSON keys | `decoder.keyDecodingStrategy = .convertFromSnakeCase` |
| Custom/partial key renaming | `enum CodingKeys: String, CodingKey` |
| Optional/missing JSON field | Make the Swift property `Optional` |
| Pretty-printed output | `encoder.outputFormatting = .prettyPrinted` |

## Exercise

Model this JSON with a `Codable` struct (use `CodingKeys` for the mismatched
names):

```json
{
    "movie_title": "Arrival",
    "release_year": 2016,
    "rating": 8.0,
    "genres": ["Drama", "Sci-Fi"]
}
```

Decode it into your struct and print each field. Then construct a second
instance in code, encode it with `.prettyPrinted`, and print the resulting
JSON string. Finally, deliberately remove the `"rating"` key from a copy of
the JSON, make `rating` an `Optional<Double>` in your struct, and confirm it
still decodes successfully with `rating` equal to `nil`.
