# Add a `trim()` method to `String`

* Proposal: SE-0TBD
* Author(s): [Andrew McDonald](https://github.com/armcknight)
* Status: TBD

<!---
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
* -->

## Introduction

This proposal adds a new `trim()` method to the standard library. It removes leading and trailing whitespaces using the Regex and Unicode notion of whitespace.

*This proposal was first discussed on the Swift Evolution list in the 
[Surveying how Swift evolves](https://forums.swift.org/t/surveying-how-swift-evolves/12726) thread. 

## Motivation

Surveying Swift utility libraries on GitHub revealed many interesting trends. String trimming was by far the most popular third party customization for Swift.

These are the top 10 function declarations from extensions on String, in their canonical form from swiftc:

1. 24 `trim() -> String`
2. 13 `substring(from: Int) -> String`
3. 12 `substring(to: Int) -> String`
4. 11 `isValidEmail() -> Bool`
5. 10 `trimmed() -> String`
6. 10 `toBool() -> Bool?`
7. 10 `height(withConstrainedWidth width: CGFloat, font: UIFont) -> CGFloat`
8. 9 `trim()`
9. 9 `toDouble() -> Double?`
10. 9 `isNumber() -> Bool`

The count for the first of these, 24, is misleading, as trimming also appears in the #5 and #8 spots, the latter being a mutating variation. It appears in many forms further down the list, with 84 methods in just this sample:

* 24 `trim() -> String`
* 10 `trimmed() -> String`
* 9 `trim()`
* 3 `trimPhoneNumberString() -> String`
* 3 `trimNewLine() -> String`
* 3 `trimForNewLineCharacterSet() -> String`
* 2 `trimmedRight(characterSet set: NSCharacterSet = default) -> String`
* 2 `trimmedLeft(characterSet set: NSCharacterSet = default) -> String`
* 1 `trimmingWhitespacesAndNewlines() -> String`
* 1 `trimmedStart(characterSet set: CharacterSet = default) -> String`
* 1 `trimmedRight() -> String`
* 1 `trimmedLeft() -> String`
* 1 `trimmedEnd(characterSet set: CharacterSet = default) -> String`
* 1 `trimWhitespace() -> String`
* 1 `trimPrefix(prefix: String)`
* 1 `trimInside() -> String`
* 1 `trimDuplicates() -> String`
* 1 `trim(trim: String) -> String`
* 1 `trim(_ characters: String) -> String`
* 1 `trim(_ characterSet: CharacterSet) -> <>`
* 1 `stringByTrimmingTailCharactersInSet(_ set: CharacterSet) -> String`
* 1 `sk4TrimSpaceNL() -> String`
* 1 `sk4TrimSpace() -> String`
* 1 `sk4Trim(str: String) -> String`
* 1 `sk4Trim(charSet: NSCharacterSet) -> String`
* 1 `prefixTrimmed(prefix: String) -> String`
* 1 `omTrim()`
* 1 `m_trimmed() -> String`
* 1 `m_trim()`
* 1 `jjs_trimWhitespaceAndNewline() -> String`
* 1 `jjs_trimWhitespace() -> String`
* 1 `jjs_trimNewline() -> String`
* 1 `jjs_emptyOrStringAndTrim(str: String?) -> String`
* 1 `hyb_trimRight(trimNewline: Bool = default) -> String`
* 1 `hyb_trimLeft(trimNewline: Bool = default) -> String`
* 1 `hyb_trim(trimNewline: Bool = default) -> String`

Lots of people are solving the same problem the same way, a function that is sufficiently universal to justify inclusion in the Standard Library in Swift 5.

## Detailed Design

This implementation uses `NSString`'s categorization of newlines and
white spaces, specifically Unicode General Category Z*, U+000A ~ U+000D, and U+0085.

```swift
extension String {
  /// Returns a new string made by removing whitespace from
  /// both ends of the source string. Whitespace characters
  /// are defined as Unicode General Category Z*,
  /// U+000A ~ U+000D, and U+0085.
  ///
  /// - Returns: A string trimmed of its whitespace on both
  ///   the leading and trailing text.
  public mutating func trimmed() -> String {
    return self.trimmingCharacters(in: .whitespacesAndNewlines)
  }

  /// Trims a string in-place by removing whitespace from
  /// both ends of the source string. Whitespace characters
  /// are defined as Unicode General Category Z*,
  /// U+000A ~ U+000D, and U+0085.
  ///
  /// - Returns: A string trimmed of its whitespace on both
  ///   the leading and trailing text.
  @discardableResult
  public mutating func trim() -> String {
    self = self.trimmed()
    return self
  }
}
```

## Alternatives Considered

Not adopting this proposal.

## Source compatibility

This proposal is strictly additive.

## Effect on ABI stability

This proposal does not affect ABI stability.

## Effect on API resilience

This proposal does not affect ABI resilience.
