# Strings: text, `bytes` and method syntax — M4 design (draft)

Status: **reviewed 2026-10-30**; §6 holds the decisions, the body is the design they settle.
**Superseded in part by D20 (2026-11-03)**: the method syntax and `\Phaenon\Str` (§4, and the rows
of §6 about methods, the unit of the methods, `Str` and `u()`) are gone. `string` and `bytes`, the
overlay, `(bytes)`, `.` on `bytes`, `P2530` and `P2305` stand. `\Phaenon\Bytes` keeps the checked
conversions — `text()`, `textLossy()`, `fromHex()`, `fromBase64()`, `hex()`, `base64()` — and the
rest is PHP's functions, which the overlay types for `bytes` (`strlen`, `substr`, `ord`,
`hash_equals`). Text by code points is `mb_*`; graphemes are `grapheme_*` or symfony/string's
`UnicodeString`. *Amended 2026-11-08 and 2026-11-09 (roadmap D24, D26): `\Phaenon\Bytes` is gone,
and the checked conversion is PHP's own `mb_check_encoding()` (§6).*
Roadmap D5 decided the mechanism (method-call syntax on primitives compiles to static calls,
strings stay primitive); this document fills in what the methods are, what `bytes` is, and where
Unicode draws its lines.

## 0. One paragraph

A Phaenon `string` is text: a well-formed UTF-8 sequence, which is what PHP strings are in
practice everywhere Tequila runs (UTF-8 source, UTF-8 databases, JSON on the wire). `bytes` is
the other thing PHP strings are used for — hashes, random bytes, binary protocol frames, file
contents — and the type system keeps the two apart while the runtime keeps one representation:
both erase to PHP `string`. Text gets one API, `\Phaenon\Str`, Unicode-correct by default and
reachable as methods (`$name->upper()`, `$path->startsWith('/')`); bytes get `\Phaenon\Bytes`.
`sprintf`, interpolation, `.` and every `str*`/`mb_*` function stay exactly as they are.

## 1. Goals and non-goals

### Goals

- **One string API with one argument order**, discoverable through method syntax, so nobody
  types `strpos($haystack, $needle)` and `in_array($needle, $haystack)` in the same hour again.
- **Unicode by default**: the API counts and slices text in a way that cannot cut a character in
  half, without `mb_` prefixes.
- **`bytes` for what is not text**, so a hash is never passed to `->upper()` and a password is
  never `strlen`-ed by accident.

### Non-goals (v1)

- A string *object*: value semantics, `===`, interpolation and `.` stay PHP's (D5).
- Changing PHP's string functions, encodings other than UTF-8, grapheme-cluster-correct
  *indexing* in the default API (§3 explains the cost and offers it explicitly).
- Localisation, collation, formatting (`\NumberFormatter`, `\IntlDateFormatter`): `intl` is
  callable, and that is enough for M4.

## 2. The types

| Phaenon | What it is | Erases to | Docblock |
|---|---|---|---|
| `string` | UTF-8 text | `string` | `string` |
| `bytes` | binary data of any length | `string` | `string` (PHPStan has no bytes; §5) |
| `non-empty-string`, `numeric-string`, `class-string<T>`, `literal-string` | refinements the checker infers and reads from docblocks, as in M3; `class<T>` is the native spelling of `class-string<T>` | `string` | the refinement |

- `string` and `bytes` are **distinct, unrelated types**: neither is a subtype of the other.
  Converting is explicit and free at runtime: `$text->bytes()` (`Str::bytes`, the identity)
  and `$data->text()` (`Bytes::text`, which **checks** UTF-8 validity and throws
  `\ValueError` otherwise — the one place a conversion costs a scan).
  *Since 2026-11-09 (§6, roadmap D26) the checked conversion is PHP's own test,
  `mb_check_encoding($data, 'UTF-8') ? $data : throw new \ValueError('not valid UTF-8')`:
  `mb_check_encoding()` narrows a `bytes` operand to `string&bytes` in its then-branch, a value
  that is both, accepted as either, and `bytes` again where that branch meets the untested value.
  From 2026-11-08 until then it was spelled `$data as string` and `$data is string` (D24). `Str`
  went with D20, `Bytes` with D24.*
- A string **literal** is `string`. There is no bytes literal: **`(bytes) $raw` retypes a
  string as binary data**, free at runtime (the cast disappears from the output), and is how
  binary that PHP hands over in a `string` enters the type; `Bytes::fromHex('0001')` /
  `fromBase64(...)` decode text into one. The functions that produce binary in PHP are typed
  `bytes` by the overlay: `random_bytes`, `hash(..., binary: true)`, `hash_hmac(..., binary:
  true)`, `md5`/`sha1(..., binary: true)`, `pack`, `hex2bin`, `sodium_*` (the common
  functions), `openssl_random_pseudo_bytes`, `MongoDB\BSON\Binary::getData`;
  `fread`/`file_get_contents` stay `string` (§6).
- Interpolation (`"Hello $name"`) requires `string` operands; a `bytes` operand is an error
  (`P2530`). **`.` joins two values of the same kind**: `string . string` is `string`,
  `bytes . bytes` is `bytes`, and `$buf .= $chunk` keeps a `bytes` variable, element or
  property `bytes` when `$chunk` is `bytes` (2026-11-01, §6); PHP's `.` is byte
  concatenation already, so it erases unchanged. Mixing the two — `bytes . string`,
  `string . bytes`, a `bytes` buffer started from a `''` literal — is `P2530`: a builder of
  binary data starts from `(bytes) ''`. Comparing `string` with `bytes` by `===` is an error
  too (`P2305`: both are PHP
  strings underneath and may even compare equal, the type confusion is the finding). PHP
  functions typed `string` accept a `string`; passing `bytes` to `strtoupper` is an error —
  call `Bytes::` methods or convert. **`(string) $data` is the unchecked way back**, for the
  PHP and vendor APIs that take binary in a `string` parameter (`Uuid::fromBinary`,
  `new Binary(...)`); `->text()` is the checked one.

## 3. Unicode: which unit?

Three units exist, and languages differ:

| Unit | PHP | Who uses it as the default |
|---|---|---|
| bytes | `strlen`, `substr`, `$s[$i]` | Go (`len`), C |
| code points | `mb_strlen`, `mb_substr` | Python, Rust (`chars()`), Java (nearly) |
| grapheme clusters | `grapheme_strlen`, `grapheme_substr` (ext-intl) | Swift (`count`), and nothing else, because it is slow |

Decision (2026-10-30): **code points are the unit of `string`.** `->length()`, `->slice()`, `->indexOf()`,
`->chars()` count code points (`mb_*`, which PHP ships everywhere and which never splits a
character). Grapheme clusters are explicit — `->graphemes()`, `->graphemeLength()` — for the
places that render text to a person (truncating a display name with an ellipsis, cursor
positions), and they need ext-intl. Bytes are what `->bytes()->length()` gives. Reasons: it is
the choice of every mainstream language that is not Swift; it is O(n) rather than the grapheme
API's O(n × intl); and it is what `mb_*` already does in every Tequila string function call
(`mb_strimwidth` is in `core`).

## 4. The API: `\Phaenon\Str` and `\Phaenon\Bytes`

*`Str` went with D20 (2026-11-03); `Bytes` with D24 (2026-11-08, §6): the checked conversion is
`mb_check_encoding()` (D26, 2026-11-09; `$b as string` until then), PHP's functions are the rest.*

Written in Phaenon (`stdlib/runtime/Str.phn`, `Bytes.phn`), read by the compiler as its own
declaration and compiled into `phaenon/runtime` (as for `Seq` and `Map`, collections.md §3).
Receiver first, operands next, callbacks last.

```php
namespace Phaenon;

final class Str
{
    public static function length(string $s): int;                               // code points
    public static function isEmpty(string $s): bool;
    public static function upper(string $s): string;                             // mb_strtoupper
    public static function lower(string $s): string;
    public static function upperFirst(string $s): string;
    public static function trim(string $s, string $chars = " \t\n\r\0\x0B"): string;
    public static function trimStart(string $s, string $chars = …): string;
    public static function trimEnd(string $s, string $chars = …): string;
    public static function startsWith(string $s, string $prefix): bool;
    public static function endsWith(string $s, string $suffix): bool;
    public static function contains(string $s, string $needle): bool;
    public static function indexOf(string $s, string $needle, int $from = 0): ?int;   // null, never false
    public static function lastIndexOf(string $s, string $needle): ?int;
    public static function slice(string $s, int $start, ?int $length = null): string;  // negative from the end, as substr
    public static function split(string $s, string $separator, int $limit = PHP_INT_MAX): list<string>;   // non-empty-list
    public static function lines(string $s): list<string>;                        // \r\n, \n, \r
    public static function join(list<string> $parts, string $separator): string;  // implode with the order fixed
    public static function replace(string $s, string $search, string $replacement): string;
    public static function replaceAll(string $s, map<string, string> $pairs): string;
    public static function repeat(string $s, int $times): string;
    public static function padStart(string $s, int $length, string $pad = ' '): string;
    public static function padEnd(string $s, int $length, string $pad = ' '): string;
    public static function chars(string $s): list<string>;                        // code points
    public static function graphemes(string $s): list<string>;                    // ext-intl
    public static function graphemeLength(string $s): int;
    public static function truncate(string $s, int $graphemes, string $ellipsis = '…'): string;
    public static function matches(string $s, string $pattern): bool;             // preg_match === 1
    public static function bytes(string $s): bytes;
    public static function format(string $template, mixed ...$args): string;      // sprintf; or keep sprintf only — §6
}

final class Bytes
{
    public static function fromHex(string $hex): bytes;                          // ValueError on bad input
    public static function fromBase64(string $b64): bytes;                       // standard or URL-safe alphabet
    public static function length(bytes $b): int;
    public static function isEmpty(bytes $b): bool;
    public static function slice(bytes $b, int $start, ?int $length = null): bytes;
    public static function concat(bytes ...$parts): bytes;
    public static function hex(bytes $b): string;
    public static function base64(bytes $b, bool $urlSafe = false): string;
    public static function text(bytes $b): string;                               // validates UTF-8, throws ValueError
    public static function textLossy(bytes $b): string;                          // U+FFFD for invalid sequences
    public static function equals(bytes $a, bytes $b): bool;                     // hash_equals: constant time
    public static function at(bytes $b, int $i): int;                            // 0–255
}
```

Method syntax, as in collections.md §3.1: `$s->upper()` compiles to `Str::upper($s)` (with
`use Phaenon\Str;` imported by the compiler, roadmap D11) when the receiver's static type is
`string`, to `Bytes::` when it is `bytes`; a receiver typed `string|Foo` is an error to narrow
(`P2519`). A `\Stringable` receiver has the `Str` methods too: they take `string|\Stringable`
and convert inside (§6). Everything PHP has stays callable: `sprintf` (234 calls in `core`),
`implode`, `str_contains`, `preg_*`, `mb_*`.

## 5. Compilation and interop

- `bytes` erases to `string` in signatures and to `string` in docblocks: PHPStan has no notion
  of binary strings and `@param bytes` would be an unknown type. The distinction is Phaenon's
  own, like `map` vs `list` is PHPStan's own. A PHP caller sees `string`. That holds for the
  docblock tags a `.phn` writes by hand as well as for the ones the compiler generates
  (2026-11-01): `/** @return bytes */`, `@var list<bytes>`, `@param bytes $x` come out with
  `string` in the type, every other character of the comment as written. Left alone, the
  word would reach PHPStan as a class name, and with `use Phaenon\Bytes;` in the file it
  resolves to that class.
- The overlay types PHP's binary producers and consumers (`random_bytes(): bytes`,
  `bin2hex(bytes): string`, `hash(..., binary: true): bytes`, `base64_encode(bytes)`) so a `.phn`
  program gets `bytes` where it should without annotations. Where PHP's signature is the same
  function for both (`strlen`), the checker accepts both and the stdlib is the typed path.
- `phn migrate` cannot know which `string` parameters are bytes; everything migrates as `string`,
  and `bytes` is adopted by hand where the overlay's producers make the checker complain
  (`random_bytes(16)` assigned to a `string` property is the first finding a migration meets —
  open question §6 asks whether that is too noisy).
- **Decided 2026-10-31: `bytes` into a `string` parameter that PHP code declares takes an explicit
  `(string)`.** Vendor code and the php-src stubs say `string` because PHP has no other word
  (`Uuid::fromBinary(string $uid)`, `MessageId::fromBinary(string $bytes)`,
  `assertStringContainsString()`); `tequila/core` met twelve such calls, eight of them
  `Uuid::fromBinary($binary->getData())`. The alternative, reading a `string` declared in a
  PHP-mode file as text-or-binary so `bytes` passes without a cast, was weighed and not taken:
  the conversion is a boundary worth seeing in the code, and `bytes` values are few enough for
  the cast to be bearable. The overlay keeps widening the binary-safe PHP functions to
  `string|bytes` so the cast is written only where a PHP API genuinely takes binary as text.

## 6. Decisions (reviewed 2026-10-30)

| Question | Decision | Why |
|---|---|---|
| `bytes` in M4 | yes, the type and the overlay ship with M4 | later is the moment nobody adds a type |
| Unit of `string` | code points by default (`mb_*`); graphemes explicit (`->graphemes()`, `->graphemeLength()`, `->truncate()`), `->bytes()->length()` for bytes | every mainstream language but Swift; never splits a character; O(n) without `intl` |
| Names | `\Phaenon\Str` (`String` is reserved) and `\Phaenon\Bytes`, singular like `Seq` and `Map` | collections.md §7 |
| `Str::format` | not added; `sprintf` stays (234 calls in `core`) | a wrapper that changes nothing is noise |
| `file_get_contents`, `fread` | `string`; `(bytes) file_get_contents(…)` for binary files | text files are the overwhelming case |
| `indexOf` | `?int`, never `false` | the stdlib is where PHP's `false` returns end |
| `\Stringable` and `string` | **no implicit conversion in argument position**: a `\Stringable` passed where `string` is declared is the error PHP's strict mode makes it. Interpolation and `.` call `__toString()` as PHP does. The `Str` methods accept `string\|\Stringable` and convert inside, so `$id->startsWith('u-')` works on a value object. The other direction (a string where `\Stringable` is expected) does not exist without boxing and is not added | PHP's own asymmetry; an inserted `->__toString()` would be code the programmer did not write (D9) and the implicit conversion strict mode exists to forbid |
| Symfony-String-like objects (`u()`) | not needed: `$s->upper()` is `u($s)->upper()` without the object | the method syntax exists because Phaenon can put a method on a string, which PHP cannot |
| Building a `bytes` value (2026-10-30) | the `(bytes)` cast, Phaenon-only syntax erased from the output; `Bytes::of()` dropped | the stdlib itself needs a primitive (`return $raw;` from `string` to `bytes` is the type error the type exists for); a cast is zero-cost where a static call is not, and `(string)` is its unchecked mirror |
| `(string) $bytes` | allowed, unchecked, typed `string` | PHP's own cast; vendor APIs take binary in `string` parameters and need an escape hatch that is visible in the code |
| `$bytes === $string` | `P2305` although both are PHP strings | the comparison is a type confusion even when it is true at runtime |
| `bytes` where PHP's function serves text and binary alike (`strlen`, `substr`, `bin2hex`, `base64_encode`, `hash_equals`, `strcmp`, `ord`, `fwrite`, …) | the overlay widens the parameter to `string\|bytes`; `substr` returns what it was given | one function, two uses; the stdlib is the typed path |
| `.` and `.=` on `bytes` (2026-11-01) | `bytes . bytes` is `bytes`; `.=` with a `bytes` operand keeps a `bytes` variable, array element or property `bytes`. `bytes . string` and `string . bytes` stay `P2530`, interpolation stays text-only, a string literal is still `string` (a binary buffer starts from `(bytes) ''`) | PHP's `.` is byte concatenation already, so the operator erases unchanged; what the type exists to catch is binary mixed into text, not binary joined to binary. Without it, every builder of frames, archives or zip entries became a `list<bytes>` and `Bytes::concat()` (tq-cli's dogfood). Python draws the same line: `b'a' + b'b'` works, `b'a' + 'a'` is a `TypeError` |
| `\Phaenon\Bytes` (2026-11-08) | *(The `is` / `as` spelling is superseded by the next row, 2026-11-09.)* Removed (roadmap D24, generics.md §11.4). The checked conversion to text is `$b as string` on a `bytes` operand: `\mb_check_encoding($b, 'UTF-8') ? $b : throw new \ValueError('not valid UTF-8')`, `as?` gives `null`, `is string` the test alone, under the operand rule of §11.4; `(string) $b` stays the unchecked mirror. In the then-branch of `is string` the operand is `string&bytes`, the intersection: a subtype of both, accepted wherever either is, printed so in messages and erased to `string`; the join with the untested value is `bytes` by plain subtyping, and an assignment ends it as any assignment does. An `as` whose operand can fail in two ways (`?bytes`, a tuple with a `bytes` member) throws one `\TypeError` naming what was expected, "with valid UTF-8" included; `\ValueError` is thrown only where the encoding is the sole way to fail. On a `string\|bytes` operand the test is `P3011`: PHP cannot tell them apart. The other methods are PHP's own functions, typed for `bytes` by the overlay: `hex2bin()` returns `bytes\|false`; `base64_decode()` keeps PHP's `string\|false`, since base64 carries JSON, PEM and secrets as often as binary (typed `bytes` it put `as string` on six sites in core and 28 in tq-cli), and `(bytes) base64_decode(…)` retypes the binary case; `bin2hex()`, `base64_encode()`, `mb_scrub()` (the lossy conversion), `hash_equals()`, `strlen()`, `substr()`, `\ord($b[$i])`, `.`. | A compiler-written call into a package the programmer did not choose is what D24 ends, and every method was a one-line wrapper. The test a `bytes` value needs before it is text is a runtime type test, which is what `as` means. `mb_check_encoding` over `preg_match('//u')`: 11 against 66–82 ns on a 12 KB string under PHP 8.5, which caches a string's validity, and it is the idiom PHP developers know; `ext-mbstring` is the one extension the lowering assumes, and Symfony's polyfill provides it where it is missing. |
| The checked conversion to text (2026-11-09) | PHP's own test (roadmap D26, generics.md §11.5): `mb_check_encoding($b, 'UTF-8')` narrows a `bytes` operand to `string&bytes` in its then-branch, the intersection of the row above, and leaves it `bytes` in the else-branch. The idiom is `mb_check_encoding($b, 'UTF-8') ? $b : throw new \ValueError('not valid UTF-8')`, typed `string&bytes`; `… ? $b : null` is the nullable form, and `mb_scrub()` stays the lossy one. The encoding is a literal naming UTF-8 (`'UTF-8'`, in any case); any other argument narrows nothing. On a `string\|bytes` operand the then-branch is `string`. The compiler writes nothing of its own: `ext-mbstring` is needed where the source calls it, as for any `mb_*` function. `base64_decode()`, `hex2bin()` and `(string) $b` stay as the row above has them. | `as string` was a second spelling of this ternary. `mb_check_encoding()` is what PHP developers write to validate UTF-8, and what the lowering wrote; the row above measured it against `preg_match('//u')`. A `bytes` value becoming text is a type the checker learns from a test, as `is_int()` teaches it `int`, so all it needs is this one reading of a PHP call. |

## 7. Checker and emitter work (once reviewed)

- Lattice: `bytes` as a primitive disjoint from `string`; literal typing; `P2530` for
  interpolation on `bytes` and for `.` between `bytes` and text.
- Method resolution on `string` / `bytes` receivers → `Str::` / `Bytes::` (shared with
  collections.md §3.1).
- Overlay: the binary producers and consumers; `Str`/`Bytes` stubs generated from the
  runtime package.
- Runtime package: the two classes with tests, UTF-8 validation path measured.
- Goldens: checker `strings.phn`, emitter `strings.phn` → `.php` under PHPStan max.
- Dogfood: `tequila/core` with `collections.md`.
- 2026-10-30: lattice, `(bytes)`, `P2530`, `P2305` on bytes vs text, the overlay
  (`stdlib/overlay/bytes.php`, `mongodb.php`), `Bytes.phn` and `Str::bytes`, erasure to `string`
  in signatures and docblocks, goldens (`internal/checker/testdata/bytes.phn`, emit `bytes.phn`
  under PHPStan max) — done. Later the same day: `Str` methods on a `\Stringable` receiver,
  `?->` on a string / bytes receiver (collections.md §3.1), `phn stdlib` writing the compiled
  classes for the runtime package.
