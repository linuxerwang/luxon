# Ad-hoc parser

## Consider alternatives

You generally shouldn't use Luxon to parse arbitrarily formatted date strings:

 1. If the string was generated by a computer for programmatic access, use a standard format like ISO 8601. Then you can parse it using Luxon's `DateTime.fromISO` method.
 2. If the string is typed out by a human, it may not conform to the format you specify when asking Luxon to parse it. Luxon is not a natural language tool and it's quite strict about the format matching the string exactly.
 
Sometimes, though, you get a string from some legacy system in some terrible ad-hoc format and you need to parse it.

## fromString

See [fromString](../class/src/datetime.js~DateTime.html#static-method-fromString) for the method signature. A brief example:

```js
DateTime.fromString('May 25 1982', 'LLLL dd yyyy');
```

## Intl

Luxon supports parsing internationalized strings:

```js
DateTime.fromString('mai 25 1982', 'LLLL dd yyyy', { locale: 'fr' });
```

Note, however, that Luxon derives the list of strings that can match, say, "LLLL" (and their meaning) by introspecting the environment's Intl implementation. Thus the exact strings may in some cases be environment-specific.

## Limitations

Not every token supported by `DateTime#toFormat` is supported in the parser. For example, there's no `ZZZZ` or `ZZZZZ` tokens. This is for a few reasons:

 * Luxon relies on natively-available functionality that only provides the mapping in one way. We can ask what the named offset is and get "Eastern Standard Time" but not ask what "Eastern Standard Time" is most likely to mean.
 * Some things are ambiguous. There are several Eastern Standard Times in different countries and Luxon has no way to know which one you mean without additional information (such as that the zone is America/New_York) that would make EST superfluous anyway. Similarly, the single-letter month and weekday formats (EEEEE) that are useful in displaying calendars graphically can't be parsed because ambiguity.
 * Luxon doesn't yet support parsing the macro tokens it provides for formatting. This may eventually be addressed.

## Debugging

There are two kinds of things that can go wrong when parsing a string: a) you make a mistake with the tokens or b) the information parsed from the string does not correspond to a valid date.

Luxon provides a method called [fromStringExplain](class/src/datetime.js~DateTime.html#static-method-fromStringExplain). It takes the same arguments as `fromString` but returns a map of information about the parse that can be useful in debugging.

For example, here the code is using "MMMM" where "MMM" was needed. You can see the regex Luxon uses and see that it didn't match anything:

```js
> DateTime.fromStringExplain("Aug 6 1982", "MMMM d yyyy")

{ input: 'Aug 6 1982',
  tokens:
   [ { literal: false, val: 'MMMM' },
     { literal: false, val: ' ' },
     { literal: false, val: 'd' },
     { literal: false, val: ' ' },
     { literal: false, val: 'yyyy' } ],
  regex: '(January|February|March|April|May|June|July|August|September|October|November|December)( )(\\d\\d?)( )(\\d{4})',
  matches: {},
  result: {},
  zone: null }
```

If you parse something and get an invalid date, the debugging steps are slightly different. Here, we're attempting to parse August 32nd, which doesn't exist:

```js
var d = DateTime.fromString("August 32 1982", "MMMM d yyyy")
d.isValid //=> false
d.invalidReason //=> 'day out of range'
```

For more on validity and how to debug it, see [validity](validity.html). You may find more comprehensive tips there. But as it applies specifically to `fromString`, again try `fromStringExplain`:

```js
> DateTime.fromStringExplain("August 32 1982", "MMMM d yyyy")

{ input: 'August 32 1982',
  tokens:
   [ { literal: false, val: 'MMMM' },
     { literal: false, val: ' ' },
     { literal: false, val: 'd' },
     { literal: false, val: ' ' },
     { literal: false, val: 'yyyy' } ],
  regex: '(January|February|March|April|May|June|July|August|September|October|November|December)( )(\\d\\d?)( )(\\d{4})',
  matches: { M: 8, d: 32, y: 1982 },
  result: { month: 8, day: 32, year: 1982 },
  zone: null }
```

Because Luxon was able to parse the string without difficulty, the output is a lot richer. And you can see that the "day" field is set to 32. Combined with the "out of range" explanation above, that should clear up the situation.

## Table of tokens

(Examples below given for 2014-08-06T13:07:04.054 considered as a local time in America/New_York).

| Standlone token | Format token | Description | Example |
| --- | --- | --- | --- |
| S | | millisecond, no padding | 54 |
| SSS | | millisecond, padded to 3 | 054 |
| s | | second, no padding | 4 |
| ss | | second, padded to 2 padding | 04 |
| m | | minute, no padding | 7 |
| mm | | minute, padded to 2 | 07 |
| h | | hour in 12-hour time, no padding | 1 |
| hh | | hour in 12-hour time, padded to 2 | 01 |
| H | | hour in 24-hour time, padded to 2 | 9 |
| HH | | hour in 24-hour time, padded to 2 | 13 |
| Z | | narrow offset | +5 |
| ZZ | | short offset | +05:00 |
| ZZZ | | techie offset | +0500 |
| z | | IANA zone | America/New_York |
| a | | meridiem | AM |
| d | | day of the month, no padding | 6 |
| dd | | day of the month, padded to 2 | 06 |
| E | c | day of the week, as number from 1-7 (Monday is 1, Sunday is 7) | 3 |
| EEE | ccc | day of the week, as an abbreviate localized string | Wed |
| EEEE | cccc | day of the week, as an unabbreviated localized string | Wednesday |
| M | L | month as an unpadded number | 8 |
| MM | LL | month as an padded number | 08 |
| MMM | LLL | month as an abbreviated localized string | Aug |
| MMMM | LLLL | month as an unabbreviated localized string | August |
| y | | year, unpadded | 2014 |
| yy | | two-digit year | 14 |
| yyyy | | four-digit year | 2014 |
| G | | abbreviated localized era | AD
| GG | | unabbreviated localized era | Anno Domini
| GGGGG | | one-letter localized era | A
| kk | | ISO week year, unpadded | 17 
| kkkk | | ISO week year, padded to 4 | 2014
| W | | ISO week number, unpadded | 32
| WW | | ISO week number, padded to 2 | 32
| o | | ordinal (day of year), unpadded | 218
| ooo | | ordinal (day of year), padded to 3 | 218
| D | | localized numeric date | 9/4/2017 
| DD | | localized date with abbreviated month | Aug 6, 2014
| DDD | | localized date with full month | August 6, 2014
| DDDD | | localized date with full month and weekday | Wednesday, August 6, 2014