# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: structs,
enums, optionals, closures, collections, and control flow.

## What you'll build

A single-file command-line to-do app that:

- Adds tasks
- Lists all tasks, marking done ones
- Marks a task done by its number
- Deletes a task by its number
- Persists everything to a plain-text file between runs

## Project layout

```text
todo/
    main.swift
```

Everything lives in one file for this project — Swift Package Manager and
multi-file module organization are covered in
[Level 2, Module 8](../level-2/08-swift-package-manager.md).

## `main.swift` — the data model

```swift
// main.swift
import Foundation

struct Task {
    let title: String
    var isDone: Bool

    // Serialize to a single line for file storage: "done|title"
    func toStorageLine() -> String {
        "\(isDone ? "1" : "0")|\(title)"
    }

    // Parse a stored line back into a Task. Returns nil for malformed lines.
    static func fromStorageLine(_ line: String) -> Task? {
        let parts = line.split(separator: "|", maxSplits: 1, omittingEmptySubsequences: false)
        guard parts.count == 2, let flag = parts.first else {
            return nil
        }
        return Task(title: String(parts[1]), isDone: flag == "1")
    }

    func display(index: Int) -> String {
        let box = isDone ? "[x]" : "[ ]"
        return "\(index). \(box) \(title)"
    }
}
```

## Storage — reading and writing the task file

```swift
enum StorageError: Error {
    case readFailed(String)
    case writeFailed(String)
}

struct TaskStorage {
    let fileURL: URL

    init(fileName: String) {
        fileURL = URL(fileURLWithPath: fileName)
    }

    func load() -> [Task] {
        guard FileManager.default.fileExists(atPath: fileURL.path) else {
            return []   // no file yet -- start empty
        }
        guard let contents = try? String(contentsOf: fileURL, encoding: .utf8) else {
            print("Warning: could not read tasks file")
            return []
        }
        return contents
            .split(separator: "\n", omittingEmptySubsequences: true)
            .compactMap { Task.fromStorageLine(String($0)) }
    }

    func save(_ tasks: [Task]) {
        let contents = tasks.map { $0.toStorageLine() }.joined(separator: "\n")
        do {
            try contents.write(to: fileURL, atomically: true, encoding: .utf8)
        } catch {
            print("Error: could not save tasks: \(error.localizedDescription)")
        }
    }
}
```

`compactMap` drops any `nil` results from `Task.fromStorageLine`, so a
corrupted line is skipped instead of crashing the whole load.

## Command handling

```swift
enum Command {
    case add(title: String)
    case list
    case done(index: Int)
    case delete(index: Int)
    case unknown

    static func parse(_ args: [String]) -> Command {
        guard let first = args.first else { return .unknown }

        switch first {
        case "add":
            let title = args.dropFirst().joined(separator: " ")
            return title.isEmpty ? .unknown : .add(title: title)
        case "list":
            return .list
        case "done":
            guard args.count == 2, let index = Int(args[1]) else { return .unknown }
            return .done(index: index)
        case "delete":
            guard args.count == 2, let index = Int(args[1]) else { return .unknown }
            return .delete(index: index)
        default:
            return .unknown
        }
    }
}

func printUsage() {
    print("Usage: todo [add <title> | list | done <n> | delete <n>]")
}
```

## Putting it together

```swift
let storage = TaskStorage(fileName: "tasks.txt")
var tasks = storage.load()

// CommandLine.arguments[0] is the program name -- drop it
let arguments = Array(CommandLine.arguments.dropFirst())
let command = Command.parse(arguments)

switch command {
case .add(let title):
    tasks.append(Task(title: title, isDone: false))
    storage.save(tasks)
    print("Added: \(title)")

case .list:
    if tasks.isEmpty {
        print("No tasks yet.")
    } else {
        for (offset, task) in tasks.enumerated() {
            print(task.display(index: offset + 1))
        }
    }

case .done(let index):
    guard index >= 1, index <= tasks.count else {
        print("No task number \(index)")
        break
    }
    tasks[index - 1].isDone = true
    storage.save(tasks)
    print("Marked done: \(tasks[index - 1].title)")

case .delete(let index):
    guard index >= 1, index <= tasks.count else {
        print("No task number \(index)")
        break
    }
    let removed = tasks.remove(at: index - 1)
    storage.save(tasks)
    print("Deleted: \(removed.title)")

case .unknown:
    printUsage()
}
```

## Running it

```bash
swiftc main.swift -o todo

./todo add "Buy groceries"
# Added: Buy groceries

./todo add "Write Swift lesson"
./todo list
# 1. [ ] Buy groceries
# 2. [ ] Write Swift lesson

./todo done 1
# Marked done: Buy groceries

./todo list
# 1. [x] Buy groceries
# 2. [ ] Write Swift lesson

./todo delete 2
./todo list
# 1. [x] Buy groceries
```

Each run reloads `tasks.txt` from disk, so tasks persist across separate
invocations of the program — an enum-driven command parser, optionals for
safe parsing, and a struct-based model come together into a small but
realistic CLI tool.

## Stretch goals

- Add an `edit <n> <new title>` command that replaces a task's title.
- Sort `list` output so incomplete tasks show before done ones.
- Switch storage from the custom `done|title` format to real JSON — you'll
  use `Codable` for this in
  [Level 2, Module 7](../level-2/07-json-codable.md).
- Add a `clear` command that removes all completed tasks at once, using
  `filter`.

Completing this project means you're ready for **Level 2 · Intermediate**.
