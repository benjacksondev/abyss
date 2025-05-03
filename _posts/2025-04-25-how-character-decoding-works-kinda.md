---
title: "A journey into character decoding"
date: 2025-04-25
---

> Disclaimer:
This post isn’t a complete or technically rigorous explanation of UTF-8 decoding. It’s more of a “here’s how I started to wrap my head around it” kind of thing. The Lua code is intentionally simplified — it skips over edge cases like invalid byte sequences, overlong encodings, and error handling. If you're looking for a formal spec, check out [RFC 3629](https://datatracker.ietf.org/doc/html/rfc3629) or the [Unicode Standard](https://www.unicode.org/standard/standard.html).

This deeper dive into character encoding came about because I had an issue where Git claimed a change in an edited file where the before and after lines were identical in the GitHub UI, and even locally, comparing diff, there was no visible difference in my IDE.

Viewing the source code as hex showed a line feed at the end of the file, which was not showing in text editors. The original copy did not end with a line feed.

The line feed should be there, but the systems are not greatly tested, and I didn’t want to break something accidentally. It’s old code, and the system’s possibly fragile, so I ran truncate -s -1 file_name, removing the last byte from the file, and merged my changes without the added line feed.
This is what sent me down the rabbit hole of how character encoding/decoding works more than I understood already.

I watched a video here: [YouTube](https://www.youtube.com/watch?v=MijmeoH9LT4&t=73s)

And my understanding became this:

There was 7-bit ASCII, which was okay but limited the number of characters available. With more bits came more character sets. As with most things without a standard, things get pretty messy. Consequently, the Unicode Consortium put together a reasonable standard that was backwards compatible with ASCII. Then, after some iterations, UTF-8 was created as an implementation of Unicode.

I wanted to make sure I understood it in a little bit more low-level detail, and I’ve been learning Lua, so I decided to try decoding some ASCII in Lua. It was pretty easy and looked something like this.

```lua
-- Writes extended part of UTF-8 and original ASCII characters to file. UNCOMMENT TO CREATE THE FILE
-- local file = io.open("ascii", "w")
-- file:write("Hello, world!\n")
-- file:close()

local file = io.open("ascii", "r")
if file == nil then
  print("Unable to open file ".. arg[1])
  os.exit(-1)
end

local file_contents = file:read()
if file_contents == nil then os.exit() end

local ascii_map = {
  [72]  = "H",
  [101] = "e",
  [108] = "l",
  [44]  = ",",
  [32]  = " ",
  [119] = "w",
  [111] = "o",
  [114] = "r",
  [100] = "d",
  [33]  = "!",
  [12]  = "\n",
}

for i = 1, #file_contents do
  local byte = string.byte(file_contents, i)
  io.write(ascii_map[byte])
end
io.write(ascii_map[12]) -- write LF
```

Above, I created a table/map in Lua of the ASCII characters I would try to decode. 

I read in the file contents, `Hello World!` (all ASCII characters), access the map with each byte, writing them to stdout along the way; voila, it works.

Then, I wanted to extend this into a UTF-8 implementation and implement multibytes, which would handle multibytes.

How UTF-8 works.

So ASCII characters are 7-bit. So, as long as the leftmost bit of a byte starts with a zero, it is an ASCII character; otherwise, it's a multibyte character. If the first two bits of the first byte are a one, but the second bit is a zero, then it's a two-byte character. If the first two bits of the first byte are one but the third bit is a zero, then it is a two-byte char; if the first three bits of the first byte are one but the fourth bit is a zero, then it is a three-byte char. Finally, if the first four bits are a one, it is a four-byte char.

To implement this, I decided to start with a tree-type structure containing the extended parts. I could walk sequentially with each byte until I reached a character, print the character, and continue.

Which ended up like this:

```lua
-- Writes extended part of UTF-8 and original ASCII characters to file. UNCOMMENT TO CREATE THE FILE
--local file = io.open("utf8", "w")
--file:write("¢harmeleon")
--file:close()

if arg[1] == nil or arg[1] == "" or arg[1] == " " then
  print("Usage: " .. arg[0] .. "<file>")
  os.exit(-1)
end

local file = io.open(arg[1], "r")
if file == nil then
  print("Unable to open file ".. arg[1])
  os.exit(-1)
end

local file_contents = file:read()
if file_contents == nil then os.exit() end

-- UTF-8 map (ASCII + extended multibyte UTF-8)
local multi_byte_map = {
  -- ASCII characters (0x00 to 0x7F)
  [0x00] = "\0", [0x01] = "\1",   [0x02] = "\2",  [0x03] = "\3",
  [0x04] = "\4", [0x05] = "\5",   [0x06] = "\6",  [0x07] = "\a",
  [0x08] = "\b", [0x09] = "\t",   [0x0A] = "\n",  [0x0B] = "\v",
  [0x0C] = "\f", [0x0D] = "\r",   [0x0E] = "\14", [0x0F] = "\15",
  [0x10] = "\16", [0x11] = "\17", [0x12] = "\18", [0x13] = "\19",
  [0x14] = "\20", [0x15] = "\21", [0x16] = "\22", [0x17] = "\23",
  [0x18] = "\24", [0x19] = "\25", [0x1A] = "\26", [0x1B] = "\27",
  [0x1C] = "\28", [0x1D] = "\29", [0x1E] = "\30", [0x1F] = "\31",
  [0x20] = " ",   [0x21] = "!",   [0x22] = "\"",  [0x23] = "#",
  [0x24] = "$",   [0x25] = "%",   [0x26] = "&",   [0x27] = "'",
  [0x28] = "(",   [0x29] = ")",   [0x2A] = "*",   [0x2B] = "+",
  [0x2C] = ",",   [0x2D] = "-",   [0x2E] = ".",   [0x2F] = "/",
  [0x30] = "0",   [0x31] = "1",   [0x32] = "2",   [0x33] = "3",
  [0x34] = "4",   [0x35] = "5",   [0x36] = "6",   [0x37] = "7",
  [0x38] = "8",   [0x39] = "9",   [0x3A] = ":",   [0x3B] = ";",
  [0x3C] = "<",   [0x3D] = "=",   [0x3E] = ">",   [0x3F] = "?",
  [0x40] = "@",   [0x41] = "A",   [0x42] = "B",   [0x43] = "C",
  [0x44] = "D",   [0x45] = "E",   [0x46] = "F",   [0x47] = "G",
  [0x48] = "H",   [0x49] = "I",   [0x4A] = "J",   [0x4B] = "K",
  [0x4C] = "L",   [0x4D] = "M",   [0x4E] = "N",   [0x4F] = "O",
  [0x50] = "P",   [0x51] = "Q",   [0x52] = "R",   [0x53] = "S",
  [0x54] = "T",   [0x55] = "U",   [0x56] = "V",   [0x57] = "W",
  [0x58] = "X",   [0x59] = "Y",   [0x5A] = "Z",   [0x5B] = "[",
  [0x5C] = "\\",  [0x5D] = "]",   [0x5E] = "^",   [0x5F] = "_",
  [0x60] = "`",   [0x61] = "a",   [0x62] = "b",   [0x63] = "c",
  [0x64] = "d",   [0x65] = "e",   [0x66] = "f",   [0x67] = "g",
  [0x68] = "h",   [0x69] = "i",   [0x6A] = "j",   [0x6B] = "k",
  [0x6C] = "l",   [0x6D] = "m",   [0x6E] = "n",   [0x6F] = "o",
  [0x70] = "p",   [0x71] = "q",   [0x72] = "r",   [0x73] = "s",
  [0x74] = "t",   [0x75] = "u",   [0x76] = "v",   [0x77] = "w",
  [0x78] = "x",   [0x79] = "y",   [0x7A] = "z",   [0x7B] = "{",
  [0x7C] = "|",   [0x7D] = "}",   [0x7E] = "~",   [0x7F] = "\127", -- DEL

  -- 2-byte sequences 
  [0xC2] = {
    [0xA2] = "¢"
  },

  -- 3-byte sequences
  [0xD0] = {
    [0x9F] = "П", 
    [0xB8] = "и",
    [0xB2] = "в",
    [0xB5] = "е",
  },
  [0xD1] = {
    [0x80] = "р",
    [0x82] = "т",
  },

  [0xE0] = {
    [0xA4] = {
      [0xA8] = "न",
      [0xAE] = "म",
      [0xB8] = "स",
      [0x95] = "क",
      [0xBE] = "ा",
      [0xB0] = "र",
    },
    [0xA5] = {
      [0x8D] = "्"
    }
  },

  [0xE2] = {
    [0x9C] = {
      [0x94] = "✔"
    },
    [0x99] = {
      [0xAA] = "♪"
    }
  },

  [0xE3] = {
    [0x81] = {
      [0x93] = "こ",
      [0xAB] = "に",
      [0xA1] = "ち",
      [0xAF] = "は"
    },
    [0x82] = {
      [0x93] = "ん"
    }
  },

  [0xE4] = {
    [0xB8] = {
      [0xAD] = "中"
    },
    [0x96] = {
      [0x87] = "文"
    }
  },

  [0xD9] = {
    [0x85] = "م",
    [0x84] = "ل",
  },
  [0xD8] = {
    [0xB1] = "ر",
    [0xAD] = "ح",
    [0xA8] = "ب",
    [0xA7] = "ا",
  },

  [0xD7] = {
    [0xA9] = "ש",
    [0x9C] = "ל",
    [0x95] = "ו",
    [0x9D] = "ם"
  },

  -- 4-byte sequences
  [0xF0] = {
    [0x9F] = {
      [0x98] = {
        [0x81] = "😁"  -- emoji
      }
    }
  }
}

local function get_bytes_from_string(str, bytes, i)
  if i > string.len(str) then
    return bytes
  end

  local byte = string.byte(str, i)
  table.insert(bytes, byte)

  return get_bytes_from_string(str, bytes, i + 1)
end

local bytes = get_bytes_from_string(file_contents, {}, 1)

-- local function print_bytes(bytes, i)
--   if i > #bytes then return end
--   print(bytes[i])
--   print_bytes(bytes, i + 1)
-- end

-- print_bytes(bytes, 1)

local function map_bytes_as_utf8(bytes, map_to_return, bytes_of_char, i)
  if i > #bytes then
    return map_to_return
  end

  local curr_byte = bytes[i]

  table.insert(bytes_of_char, curr_byte)

  local how_many_bytes_in_char = 1

  if bytes_of_char[1] & 0xF0 == 0xF0 then
    how_many_bytes_in_char = 4
  elseif bytes_of_char[1] & 0xE0 == 0xE0 then
    how_many_bytes_in_char = 3
  elseif bytes_of_char[1] & 0xC0 == 0xC0 then
    how_many_bytes_in_char = 2
  end

  if #bytes_of_char < how_many_bytes_in_char then
    return map_bytes_as_utf8(bytes, map_to_return, bytes_of_char, i + 1)
  end

  table.insert(map_to_return, bytes_of_char)
  bytes_of_char = {}

  return map_bytes_as_utf8(bytes, map_to_return, bytes_of_char, i + 1)
end

local utf8_chars = map_bytes_as_utf8(bytes, {}, {}, 1)

local function gcnb(char, n)
  return char[n]
end

local function print_chars(chars, i)
  if i > #chars then return end
  local c = chars[i]

  if #chars[i] == 1 then
    io.write(multi_byte_map[gcnb(c, 1)])
  elseif #chars[i] == 2 then
    io.write(multi_byte_map[gcnb(c, 1)][gcnb(c, 2)])
  elseif #chars[i] == 3 then
    io.write(multi_byte_map[gcnb(c, 1)][gcnb(c, 2)][gcnb(c, 3)])
  elseif #chars[i] == 4 then
    io.write(multi_byte_map[gcnb(c, 1)][gcnb(c, 2)][gcnb(c, 3)][gcnb(c, 4)])
  end
  print_chars(chars, i + 1)
end

print_chars(utf8_chars, 1)
```

> Some refactoring occurred. Notably, I moved toward a more declarative programming approach using functional programming techniques.

0a

