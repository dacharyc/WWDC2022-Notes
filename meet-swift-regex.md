# Meet Swift Regex

Presenters:
- Michael Ilseman, Swift Standard Library

## Processing Strings

Strings are a collection, so you get access to collection-related tools:
- Element-based: `map`, `filter`, `split`
- Low-level index-based: `index(after: )`, `firstIndex(of: )`, slicing subscript

We could try splitting on whitespace, but that doesn't help with the tabs:

```swift
let transaction = "DEBIT     03/05/2022    Doug's Dugout Dogs         $33.27"

let fragments = transaction.split(whereSeparator: \.isWhitespace)
// ["DEBIT", "03/05/2022", "Doug\'s", "Dugout", "Dogs", "$33.27"]
```

We could try dropping down to low-level index manipulation, but it's a lot of code and hard to do it right:

```swift
var slice = transaction[...]

// Extract a field, advancing `slice` to the start of the next field
func extractField() -> Substring {
  let endIdx = {
    var start = slice.startIndex
    while true {
      // Position of next whitespace (including tabs)
      guard let spaceIdx = slice[start...].firstIndex(where: \.isWhitespace) else {
        return slice.endIndex
      }

      // Tab suffices
      if slice[spaceIdx] == "\t" {
        return spaceIdx
      }

      // Otherwise check for a second whitespace character
      let afterSpaceIdx = slice.index(after: spaceIdx)
      if afterSpaceIdx == slice.endIndex || slice[afterSpaceIdx].isWhitespace {
        return spaceIdx
      }

      // Skip over the single space and try again
      start = afterSpaceIdx
    }
  }()
  defer { slice = slice[endIdx...].drop(while: \.isWhitespace) }
  return slice[..<endIdx]
}

let kind = extractField()
let date = try Date(String(extractField()), strategy:  Date.FormatStyle(date: .numeric))
let account = extractField()
let amount = try Decimal(String(extractField()), format: .currency(code: "USD"))
```

Regular expressions provide a solution.

Regex is a struct, generic over its output, which is the result of applying it, including captures.

```swift
struct Regex<Output>
```

You can create one using a regex literal containing regex syntax between slash delimiters.

```swift
// Regex literals
let digits = /\d+/
// digits: Regex<Substring>
```

Regex can be created at runtime from a string containing the same regex syntax. This throws an error at runtime if the input contains invalid syntax.

```swift
// Run-time construction
let runtimeString = #"\d+"#
let digits = try Regex(runtimeString)
// digits: Regex<AnyRegexOutput>
```

The same regex can be written using a declarative and well-structured regex builder:

```swift
// Regex builders
let digits = OneOrMore(.digit)
// digits: Regex<Substring>
```

Swift regex is different:
- Literals are concise; builders give structure. Literals can be used within builders to find the perfect balance.
- Date formats need a real parser. Text representations for dates are very complicated. Swift lets you interweve real parsers as parts of regex.
- The world is Unicode, not just ASCII. Swift regex does the Unicode without compromising expressivity.
- Swift regex provides predictable execution and surfaces controls prominently. They aren't hidden away or behind a syntax that is difficult to reason about.

Let's create a regex builder:

```swift
// CREDIT    03/02/2022    Payroll from employer         $200.23
// CREDIT    03/03/2022    Suspect A                     $2,000,000.00
// DEBIT     03/03/2022    Ted's Pet Rock Sanctuary      $2,000,000.00
// DEBIT     03/05/2022    Doug's Dugout Dogs            $33.27

import RegexBuilder

let fieldSeparator = /\s{2,}|\t/
let transactionMatcher = Regex {
  /CREDIT|DEBIT/
  fieldSeparator
  One(.date(.numeric, locale: Locale(identifier: "en_US"), timeZone: .gmt))
  fieldSeparator
  OneOrMore {
    NegativeLookahead { fieldSeparator }
    CharacterClass.any
  }
  fieldSeparator
  One(.localizedCurrency(code: "USD").locale(Locale(identifier: "en_US")))
}
```

To extract some of the data from the parsed lines, we can use captures:

```swift
let fieldSeparator = /\s{2,}|\t/
let transactionMatcher = Regex {
  Capture { /CREDIT|DEBIT/ }
  fieldSeparator

  Capture { One(.date(.numeric, locale: Locale(identifier: "en_US"), timeZone: .gmt)) }
  fieldSeparator

  Capture {
    OneOrMore {
      NegativeLookahead { fieldSeparator }
      CharacterClass.any
    }
  }
  fieldSeparator
  Capture { One(.localizedCurrency(code: "USD").locale(Locale(identifier: "en_US"))) }
}
// transactionMatcher: Regex<(Substring, Substring, Date, Substring, Decimal)>
```

We can use named captures to make things easier to reason abaout:

```swift
let regex = #/
  (?<date>     \d{2} / \d{2} / \d{4})
  (?<middle>   \P{currencySymbol}+)
  (?<currency> \p{currencySymbol})
/#
// Regex<(Substring, date: Substring, middle: Substring, currency: Substring)>
```

A regex declares an algorithm over some model of String. Swift's String presents multiple models for working with Unicode. This string, representing a love story for the ages, contains 3 characters. 

```swift
let aZombieLoveStory = "🧟‍♀️💖🧠"
// Characters: 🧟‍♀️, 💖, 🧠
```

These characters are complex entities formally called Unicode extended grapheme clusters. A single Character is composed of one or more Unicode scalar values. 

```swift
aZombieLoveStory.unicodeScalars
// Unicode scalar values: U+1F9DF, U+200D, U+2640, U+FE0F, U+1F496, U+1F9E0
```

String provides a UnicodeScalarView to access this lower-level representation of its contents. This enables advanced usage as well as compatibility with other systems.

For an example of Unicode processing:

```swift
switch ("🧟‍♀️💖🧠", "The Brain Cafe\u{301}") {
case (/.\N{SPARKLING HEART}./, /.*café/.ignoresCase()):
  print("Oh no! 🧟‍♀️💖🧠, but 🧠💖☕️!")
default:
  print("No conflicts found")
}
```

You can perform complex scalar processing if needed:

```swift
let input = "Oh no! 🧟‍♀️💖🧠, but 🧠💖☕️!"

input.firstMatch(of: /.\N{SPARKLING HEART}./)
// 🧟‍♀️💖🧠

input.firstMatch(of: /.\N{SPARKLING HEART}./.matchingSemantics(.unicodeScalar))
// ️💖🧠
```

We can use `TryCapture` to pass a matched field to our closure and test for a match. We capture the match or return nil.

```swift
// CREDIT    <proprietary>      <redacted>        200.23        A1B34EFF     ...
let fieldSeparator = /\s{2,}|\t/
let field = OneOrMore {
  NegativeLookahead { fieldSeparator }
  CharacterClass.any
}
let transactionMatcher = Regex {
  Capture { /CREDIT|DEBIT/ }
  fieldSeparator

  TryCapture(field) { timestamp ~= $0 ? $0 : nil }
  fieldSeparator

  TryCapture(field) { details ~= $0 ? $0 : nil }
  fieldSeparator

  // ...
}
```

We can put things in a local or global scope. To solve the issue of backtracking and retrying, we can put our field separator in a Local scope:

```swift
let fieldSeparator = Local { /\s{2,}|\t/ }
```

Global backtracking, the regex default, is great for search and fuzzy matching. Local is better for matching a token. It contains a search space.

Local is known elsewhere as an atomic non-capturing group.

## Related Sessions

- Swift Regex: Beyond the basics
