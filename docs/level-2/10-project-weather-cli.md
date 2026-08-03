# 10 · Project — Weather CLI

The Level 2 capstone project: a command-line tool that fetches **real, live
weather data** from a public API and prints a short report for a city. It
deliberately pulls together everything this level covered — a `throws`-based
error model ([Module 3](03-error-handling.md)), `Codable` JSON parsing
([Module 7](07-json-codable.md)), a generic network client
([Module 5](05-generics.md)), organized as a proper Swift package
([Module 8](08-swift-package-manager.md)), with unit tests
([Module 6](06-testing-xctest.md)).

## The API — no key required

This project uses [Open-Meteo](https://open-meteo.com), a free weather API
that needs **no API key, no signup, and no authentication header** — you
send coordinates, it returns current conditions as JSON. That makes it ideal
for a learning project: nothing to configure, nothing to leak, nothing to
expire.

## What you'll build

A `weather-cli` executable that takes a city name, looks up its coordinates
from a small built-in catalog, calls Open-Meteo, and prints the current
temperature, wind speed, and conditions — with real error messages for an
unknown city or a failed network request, not a crash.

## Project layout

```text
WeatherCLI/
    Package.swift
    Sources/
        WeatherCLI/
            main.swift
            City.swift
            WeatherError.swift
            WeatherModels.swift
            WeatherCodeDescription.swift
            APIClient.swift
            WeatherService.swift
    Tests/
        WeatherCLITests/
            WeatherCLITests.swift
```

## `Package.swift`

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "WeatherCLI",
    platforms: [
        .macOS(.v12)   // needed for async/await URLSession APIs
    ],
    targets: [
        .executableTarget(
            name: "WeatherCLI"
        ),
        .testTarget(
            name: "WeatherCLITests",
            dependencies: ["WeatherCLI"]
        )
    ]
)
```

## The error model

Every failure mode gets its own case — an unknown city, a bad URL, a network
failure, a decoding failure, or a non-2xx server response — instead of one
generic "something went wrong":

```swift
// Sources/WeatherCLI/WeatherError.swift
import Foundation

enum WeatherError: Error {
    case unknownCity(String)
    case invalidURL
    case requestFailed(underlying: Error)
    case decodingFailed(underlying: Error)
    case serverError(statusCode: Int)
}

extension WeatherError: CustomStringConvertible {
    var description: String {
        switch self {
        case .unknownCity(let name):
            return "Unknown city \"\(name)\""
        case .invalidURL:
            return "Could not build a valid request URL"
        case .requestFailed(let underlying):
            return "Network request failed: \(underlying.localizedDescription)"
        case .decodingFailed(let underlying):
            return "Failed to parse weather data: \(underlying)"
        case .serverError(let statusCode):
            return "Server returned status code \(statusCode)"
        }
    }
}
```

`CustomStringConvertible` means `print("Error: \(error)")` shows the
friendly `description` instead of Swift's default enum dump — worth doing
for any error type a CLI tool will show directly to a user.

## City catalog and lookup

To keep the project self-contained (no second "find coordinates for a
city name" API to depend on), a small built-in catalog maps city names to
coordinates. `resolve(named:)` is where `throws` earns its keep — it's the
one place a user-facing failure (typo'd city name) turns into a specific,
catchable error:

```swift
// Sources/WeatherCLI/City.swift
struct City {
    let name: String
    let latitude: Double
    let longitude: Double
}

enum CityCatalog {
    static let all: [City] = [
        City(name: "New York", latitude: 40.7128, longitude: -74.0060),
        City(name: "London", latitude: 51.5072, longitude: -0.1276),
        City(name: "Tokyo", latitude: 35.6895, longitude: 139.6917),
        City(name: "Sydney", latitude: -33.8688, longitude: 151.2093),
        City(name: "Hyderabad", latitude: 17.3850, longitude: 78.4867)
    ]

    static func find(named name: String) -> City? {
        all.first { $0.name.lowercased() == name.lowercased() }
    }

    static func resolve(named name: String) throws -> City {
        guard let city = find(named: name) else {
            throw WeatherError.unknownCity(name)
        }
        return city
    }
}
```

## Modeling the JSON response with `Codable`

Open-Meteo's response uses `snake_case` keys — `temperature_2m`,
`wind_speed_10m` — so every model needs explicit `CodingKeys`, exactly the
trap [Module 7](07-json-codable.md) covers:

```swift
// Sources/WeatherCLI/WeatherModels.swift
struct WeatherResponse: Decodable {
    let latitude: Double
    let longitude: Double
    let currentUnits: CurrentUnits
    let current: CurrentWeather

    enum CodingKeys: String, CodingKey {
        case latitude
        case longitude
        case currentUnits = "current_units"
        case current
    }
}

struct CurrentUnits: Decodable {
    let temperature2m: String
    let windSpeed10m: String

    enum CodingKeys: String, CodingKey {
        case temperature2m = "temperature_2m"
        case windSpeed10m = "wind_speed_10m"
    }
}

struct CurrentWeather: Decodable {
    let time: String
    let temperature2m: Double
    let windSpeed10m: Double
    let weatherCode: Int

    enum CodingKeys: String, CodingKey {
        case time
        case temperature2m = "temperature_2m"
        case windSpeed10m = "wind_speed_10m"
        case weatherCode = "weather_code"
    }
}
```

A small helper turns Open-Meteo's numeric [WMO weather
codes](https://open-meteo.com/en/docs) into readable text:

```swift
// Sources/WeatherCLI/WeatherCodeDescription.swift
enum WeatherCodeDescription {
    static func describe(_ code: Int) -> String {
        switch code {
        case 0:
            return "Clear sky"
        case 1, 2, 3:
            return "Partly cloudy"
        case 45, 48:
            return "Fog"
        case 51, 53, 55:
            return "Drizzle"
        case 61, 63, 65:
            return "Rain"
        case 71, 73, 75:
            return "Snow"
        case 80, 81, 82:
            return "Rain showers"
        case 95, 96, 99:
            return "Thunderstorm"
        default:
            return "Unknown conditions"
        }
    }
}
```

## A generic API client

This is the module's generics payoff: `APIClient.fetch` doesn't know or
care that it's fetching weather data — it works for **any** `Decodable`
type, converting network and decoding failures into the project's own
`WeatherError` cases along the way:

```swift
// Sources/WeatherCLI/APIClient.swift
import Foundation

/// A tiny generic HTTP client: it knows how to fetch and decode ANY
/// Decodable type, not just weather data -- that's the point of making it generic.
struct APIClient {
    private let session: URLSession

    init(session: URLSession = .shared) {
        self.session = session
    }

    func fetch<T: Decodable>(_ type: T.Type, from url: URL) async throws -> T {
        let data: Data
        let response: URLResponse
        do {
            (data, response) = try await session.data(from: url)
        } catch {
            throw WeatherError.requestFailed(underlying: error)
        }

        if let httpResponse = response as? HTTPURLResponse,
           !(200...299).contains(httpResponse.statusCode) {
            throw WeatherError.serverError(statusCode: httpResponse.statusCode)
        }

        do {
            return try JSONDecoder().decode(T.self, from: data)
        } catch {
            throw WeatherError.decodingFailed(underlying: error)
        }
    }
}
```

## Wiring it together

`WeatherService` builds the actual request URL and calls the generic client
with a concrete type (`WeatherResponse.self`) — this is the moment `T` gets
filled in:

```swift
// Sources/WeatherCLI/WeatherService.swift
import Foundation

struct WeatherService {
    let client: APIClient

    init(client: APIClient = APIClient()) {
        self.client = client
    }

    func currentWeather(for city: City) async throws -> WeatherResponse {
        guard let url = Self.buildURL(for: city) else {
            throw WeatherError.invalidURL
        }
        return try await client.fetch(WeatherResponse.self, from: url)
    }

    static func buildURL(for city: City) -> URL? {
        var components = URLComponents(string: "https://api.open-meteo.com/v1/forecast")
        components?.queryItems = [
            URLQueryItem(name: "latitude", value: "\(city.latitude)"),
            URLQueryItem(name: "longitude", value: "\(city.longitude)"),
            URLQueryItem(name: "current", value: "temperature_2m,wind_speed_10m,weather_code"),
            URLQueryItem(name: "temperature_unit", value: "fahrenheit")
        ]
        return components?.url
    }
}
```

## `main.swift`

`main.swift` stays intentionally thin — it parses arguments, calls into the
real logic, and turns any thrown `WeatherError` into a clean message. All
the actual behavior lives in testable files, not here:

```swift
// Sources/WeatherCLI/main.swift
import Foundation

let arguments = CommandLine.arguments
let availableCities = CityCatalog.all.map { $0.name }.joined(separator: ", ")

guard arguments.count > 1 else {
    print("Usage: weather-cli <city name>")
    print("Available cities: \(availableCities)")
    exit(1)
}

let cityName = arguments.dropFirst().joined(separator: " ")
let service = WeatherService()

do {
    let city = try CityCatalog.resolve(named: cityName)
    let weather = try await service.currentWeather(for: city)
    let conditions = WeatherCodeDescription.describe(weather.current.weatherCode)

    print("Weather in \(city.name):")
    print("  Temperature: \(weather.current.temperature2m)\(weather.currentUnits.temperature2m)")
    print("  Wind speed: \(weather.current.windSpeed10m) \(weather.currentUnits.windSpeed10m)")
    print("  Conditions: \(conditions)")
} catch let error as WeatherError {
    print("Error: \(error)")
    print("Available cities: \(availableCities)")
    exit(1)
} catch {
    print("Unexpected error: \(error)")
    exit(1)
}
```

Top-level code in `main.swift` is allowed to use `await` directly — no
`@main` struct or explicit `Task` wrapper needed for a simple CLI like this
one.

## Tests

Notice none of these tests touch the network — they check the *pure logic*
(city lookup, error mapping, code-to-text conversion, JSON decoding of a
fixed sample, URL construction) since a unit test that depends on a live
network call is slow and flaky by nature. That's also *why* `CityCatalog`,
`WeatherCodeDescription`, and the `CodingKeys`-based models were written as
small, independent pieces in the first place — each one is trivially
testable in isolation:

```swift
// Tests/WeatherCLITests/WeatherCLITests.swift
import XCTest
@testable import WeatherCLI

final class CityCatalogTests: XCTestCase {
    func testFindKnownCityIsCaseInsensitive() {
        let city = CityCatalog.find(named: "hyderabad")
        XCTAssertEqual(city?.name, "Hyderabad")
    }

    func testFindUnknownCityReturnsNil() {
        XCTAssertNil(CityCatalog.find(named: "Atlantis"))
    }
}

final class WeatherCodeDescriptionTests: XCTestCase {
    func testClearSky() {
        XCTAssertEqual(WeatherCodeDescription.describe(0), "Clear sky")
    }

    func testThunderstormGroup() {
        XCTAssertEqual(WeatherCodeDescription.describe(95), "Thunderstorm")
        XCTAssertEqual(WeatherCodeDescription.describe(99), "Thunderstorm")
    }

    func testUnknownCodeFallsBack() {
        XCTAssertEqual(WeatherCodeDescription.describe(-1), "Unknown conditions")
    }
}

final class WeatherModelsDecodingTests: XCTestCase {
    func testDecodesSampleResponse() throws {
        let json = """
        {
            "latitude": 40.71,
            "longitude": -74.0,
            "current_units": { "temperature_2m": "°F", "wind_speed_10m": "km/h" },
            "current": {
                "time": "2026-01-01T00:00",
                "temperature_2m": 72.5,
                "wind_speed_10m": 10.2,
                "weather_code": 3
            }
        }
        """.data(using: .utf8)!

        let response = try JSONDecoder().decode(WeatherResponse.self, from: json)
        XCTAssertEqual(response.current.temperature2m, 72.5)
        XCTAssertEqual(response.currentUnits.temperature2m, "°F")
        XCTAssertEqual(response.current.weatherCode, 3)
    }
}

final class WeatherServiceURLTests: XCTestCase {
    func testBuildURLIncludesCoordinates() {
        let city = City(name: "Testville", latitude: 1.5, longitude: -2.5)
        let url = WeatherService.buildURL(for: city)
        XCTAssertNotNil(url)
        XCTAssertTrue(url!.absoluteString.contains("latitude=1.5"))
        XCTAssertTrue(url!.absoluteString.contains("longitude=-2.5"))
    }
}

final class ResolveCityTests: XCTestCase {
    func testResolveKnownCitySucceeds() throws {
        let city = try CityCatalog.resolve(named: "London")
        XCTAssertEqual(city.name, "London")
    }

    func testResolveUnknownCityThrows() {
        XCTAssertThrowsError(try CityCatalog.resolve(named: "Nowhere")) { error in
            guard case WeatherError.unknownCity(let name) = error else {
                XCTFail("Expected .unknownCity, got \(error)")
                return
            }
            XCTAssertEqual(name, "Nowhere")
        }
    }
}
```

## Running it

```bash
swift build
swift run WeatherCLI Hyderabad
```

```text
Weather in Hyderabad:
  Temperature: 83.3°F
  Wind speed: 14.6 km/h
  Conditions: Partly cloudy
```

Since this hits a live weather API, your actual numbers will differ every
time you run it — that's the point, it's real data, not a fixture. An
unknown city produces a clean error instead of a crash:

```bash
swift run WeatherCLI Atlantis
```

```text
Error: Unknown city "Atlantis"
Available cities: New York, London, Tokyo, Sydney, Hyderabad
```

And with no arguments at all:

```bash
swift run WeatherCLI
```

```text
Usage: weather-cli <city name>
Available cities: New York, London, Tokyo, Sydney, Hyderabad
```

Run the test suite with:

```bash
swift test
```

```text
Test Suite 'All tests' started at ...
Test Case 'CityCatalogTests.testFindKnownCityIsCaseInsensitive' passed.
Test Case 'CityCatalogTests.testFindUnknownCityReturnsNil' passed.
Test Case 'WeatherCodeDescriptionTests.testClearSky' passed.
Test Case 'WeatherCodeDescriptionTests.testThunderstormGroup' passed.
Test Case 'WeatherCodeDescriptionTests.testUnknownCodeFallsBack' passed.
Test Case 'WeatherModelsDecodingTests.testDecodesSampleResponse' passed.
Test Case 'WeatherServiceURLTests.testBuildURLIncludesCoordinates' passed.
Test Case 'ResolveCityTests.testResolveKnownCitySucceeds' passed.
Test Case 'ResolveCityTests.testResolveUnknownCityThrows' passed.
Executed 9 tests, with 0 failures
```

## Stretch goals

- Add a `--units metric` flag that switches `temperature_unit` to Celsius
  and reports wind speed as-is (already metric from the API).
- Add a real geocoding step using Open-Meteo's free [Geocoding
  API](https://open-meteo.com/en/docs/geocoding-api) instead of the fixed
  `CityCatalog`, so any city name in the world works, not just five.
- Cache the last successful response for a city to disk (reusing the
  file-persistence pattern from the [Level 1 project](../level-1/10-project-todo-cli.md))
  and fall back to it if the network request fails.
- Add a `forecast` subcommand using Open-Meteo's `daily` parameters to show
  a 3-day outlook instead of just current conditions.
- Make `APIClient.fetch` genuinely reusable by pointing it at a second,
  unrelated API (anything returning JSON) and confirming the same generic
  method decodes a completely different `Decodable` type with no changes.

Completing this project means you're ready for **Level 3 · Advanced**.
