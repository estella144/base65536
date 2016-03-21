# base65536

[Base64](https://en.wikipedia.org/wiki/Base64) is used to encode arbitrary binary data as "plain" text using a small, extremely safe repertoire of 64 (well, 65) characters. Base64 remains highly suited to text systems where the range of characters available is very small -- i.e., anything still constrained to plain ASCII. Base64 encodes 6 bits, or 3/4 of an octet, per character.

However, now that Unicode rules the world, the range of characters which can be considered "safe" in this way is, in many situations, significantly wider. Base65536 applies the same basic principle to a carefully-chosen repertoire of 65,536 (well, 65,792) Unicode code points, encoding 16 bits, or 2 octets, per character. This allows up to 280 octets of binary data to fit in a Tweet.

In theory, this project could have been a one-liner. In practice, naively taking each pair of bytes and smooshing them together to make a single code point is a bad way to do this because you end up with:

* Control characters
* Whitespace
* Unpaired surrogate pairs
* Normalization corruption
* No way to tell whether the final byte in the sequence was there in the original or not

For details of how these code points were chosen and why they are thought to be safe, [see the sibling project `base65536gen`](https://github.com/ferno/base65536gen).

## Installation

```bash
npm install base65536
```

## Usage

```js
var base65536 = require('base65536');

var buf = new Buffer("hello world"); // 11 bytes

var str = base65536.encode(buf); 
console.log(str); // 6 code points, "驨ꍬ啯𒁷ꍲᕤ"

var buf2 = base65536.decode(str);
console.log(buf.equals(buf2)); // true
```

## API

### base65536.encode(buf)

Encodes a [`Buffer`](https://nodejs.org/api/buffer.html#buffer_new_buffer_str_encoding) and returns a Base65536 `String`, suitable for passing safely through almost any "Unicode-clean" text-handling API. This string contains no special characters and is immune to Unicode normalization. The string encodes two bytes per code point.

#### Note

While you might expect that the `length` of the resulting string is half the `length` of the original buffer, this is only true when counting *Unicode code points*. In JavaScript, a string's `length` property reports not the number of code points but the number of *16-bit code units* in the string. For characters outside of the Basic Multilingual Plane, a [surrogate pair of 16-bit code units](https://en.wikipedia.org/wiki/UTF-16) is used to represent each code point. `base65536` makes extensive use of these characters.

As a worked example:

```js
var buf = new Buffer([255, 255]);    // two bytes
var str = base65536.encode(buf);     // "𨗿", one code point, U+285FF
console.log(str.length);             // 2, two 16-bit code units
console.log(str.charCodeAt(0));      // 55393 = 0xD861
console.log(str.charCodeAt(1));      // 56831 = 0xDDFF
console.log(str === "\uD861\uDDFF"); // true
```

### base65536.decode(str)

Decodes a Base65536 `String` and returns a `Buffer` containing the original binary data.

This function is currently very strict, with no tolerance for whitespace or other unexpected characters. An `Error` is thrown if the supplied string is not a valid Base65536 text, or if there is a "final byte" code point in the middle of the string.

## More examples

```js
var hash = md5("");                 // "d41d8cd98f00b204e9800998ecf8427e", 32 hex digits
var buf = new Buffer(hash, "hex");  // <Buffer d4 1d ... 7e>
console.log(base65536.encode(buf)); // "勔𥾌㒏㢲𠛩𡸉𧻬𠑂", 8 chars
```

```js
var uuid = "8eb44f6c-2505-4446-aa57-22d6897c9922";   // 32 hex digits
var buf = new Buffer(uuid.replace(/-/g, ""), "hex"); // <Buffer 8e b4 ... 22>
console.log(base65546.encode(buf));                  // "𣪎ꍏ㤥筄貪𥰢𠊉垙", 8 chars
```

## Why?

Erm.

I wanted people to be able to share [HATETRIS](http://qntm.org/files/hatetris/hatetris.html) replays via Twitter.

Twitter supports tweets of up to 140 characters. "[Tweet length is measured by the number of codepoints in the NFC normalized version of the text.](https://dev.twitter.com/overview/api/counting-characters)"

HATETRIS has four buttons: left, right, down and rotate. A single move in HATERIS therefore encodes two bits of information. At present, replays are encoded as hexadecimal and spaced for legibility/selectability. Although a game of HATETRIS may extend for an arbitrary number of keystrokes (simply press rotate forever), in general, the longer the game goes on, the higher one's score.

The world record HATETRIS replay (30 points) is 1,440 keystrokes = 2,880 bits long. At present, HATETRIS replays are encoded as hexadecimal, with each hexadecimal digit encoding 4 bits = 2 keystrokes, and spaces added for clarity/legibility, then presented as text, like so:

> C02A AAAA AAAB 00AA AAAA AC08 AAAA AAC2 AAAA AAAA C2AA AAAA AEAA AAAA AA56
> AAAA AAAA B55A AAAA AA96 AAAA AAAA D5AA AAAA A9AA AAAA AAB5 AAAA AAAA AAAA
> AAAA DAAA AAAA 9756 AAAA AA8A AAAA AAAB AAAA AAAB 5AAA AAAB 56AA AAAA AAAA
> A82A AAAA B00A AAAA A6D6 AB55 6AAA AAA9 4AAA AAA6 AAAA AD56 AAAA B56A AAAA
> 032A AAAA A65B F00A AAAA AA6E EFC0 2AAA AAAA EB00 AAAA AAA8 0AAA AAAA 802A
> AAAA AA54 AAAA AAA1 AAAA AAA0 AAAA AAA0 0AAA AAAA C02A AAAA B002 AAAA B00A
> AAAC 2AAA AAB0 AAAA AEAA AAA9 5AAA AAA9 D5AA AAA5 AAAA AAB5 6AAA A6AA AAAB
> 5AAA AAAA AAAA DAAA AAD5 56AA AA2A AAAA BAAA AAD6 AAAB 56AA AAAA 82AA AC02
> AAA7 B5AA D556 AAAA 52AA A6AA B55A AB56 AA80 FCAA AAA5 583F 0AAA A9BB BF00
> AAAA AE80 32AA AA82 FAAA A802 AAAA 96AA AA1A AAA8 2AAA A00A AAAB 00AA AB00
> AAB0 AAAB 0AAB AAA9 5AAA AD56 AA5A AAB5 6AAC 02A9 AAAB 5AAA AAAD AAB5 5AA2
> AAAE AA0A AAB2 AAD5 6AB5 AA02 AAA0 0AAA B55A AD6A BAAC 2AAB 0AA0 C2AA C02A

That's 899 characters including spaces, or 720 characters if the spaces were removed. Were the hexadecimal characters converted to binary, I would have 360 bytes, and were the binary expressed in Base64, I would have 480 characters.

Using elementary run-length encoding, with two bits of keystroke and two bits of run length, I get down to 2040 bits. That's 255 bytes, which is still 340 characters of Base64. But in Base65536 this is 128 code points! Much better.

> 𤇃𢊻𤄻嶜𤄋𤇁𡊻𤄛𤆬𠲻𤆻𠆜𢮻𤆻ꊌ𢪻𤆻邌𤆻𤊻𤅋𤲥𣾻𤄋𥆸𣊻𤅛ꊌ𤆻𤆱炼綻
> 𤋅𤅴薹𣪻𣊻𣽻𤇆𤚢𣺻赈𤇣綹𤻈𤇣𤾺𤇃悺𢦻𤂻𤅠㢹𣾻𤄛𤆓𤦹𤊻𤄰炜傼𤞻𢊻𣲻
> 𣺻ꉌ邹𡊻𣹫𤅋𤇅𣾻𤇄𓎜𠚻𤊻𢊻𤉛𤅫𤂑𤃃𡉌𤵛𣹛𤁐𢉋𡉻𡡫𤇠𠞗𤇡𡊄𡒌𣼻燉𣼋
> 𦄘炸邹㢸𠞻𠦻𡊻𣈻𡈻𣈛𡈛ꊺ𠆼𤂅𣻆𣫃𤮺𤊻𡉋㽻𣺬𣈛𡈋𤭻𤂲𣈻𤭻𤊼𢈛儛𡈛ᔺ

This [fits comfortably in a Tweet](https://twitter.com/qntm/status/673523018224791552), with an extravagant 12 characters left over for your comment.

And of course, the worse you are at HATETRIS, the shorter your replay is, and the more room you have for invective.

## Efficiency

Given 1MB of input, Base65536 returns 1.79MB of UTF-8 output on average, 1.57MB of UTF-16 or 2.00MB of UTF-32.

Compare Base64, which would return 1.33MB of UTF-8, 2.67MB of UTF-16 or 5.33MB of UTF-32.

## Unicode has 1,114,112 code points, most of which we aren't using. Can we go further?

To encode one additional bit per character, or 140 additional bits (37.5 additional octets) per Tweet, we need to *double* the number of code points we use.

[`base65536gen`](https://github.com/ferno/base65536gen) returns only 92,240 safe characters from the "Letter, Other" [General Category](https://en.wikipedia.org/wiki/Unicode_character_property#General_Category). Modifying it to add other safe General Categories (all the Letter, Number and Symbol GCs) yields only 101,064 safe characters. We really need 131,072, and even then the gain would be marginal (17 bits per code point instead of 16).

## License

MIT

## In other languages

* [Python](https://github.com/Parkayun/base65536)
* [Go](https://github.com/Nightbug/go-base65536)
* [Ruby](https://github.com/Nightbug/base65536-ruby)