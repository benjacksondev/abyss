---
title: "A journey into character decoding"
date: 2025-04-25
---

> Disclaimer:
This post isn’t a complete or technically rigorous explanation of UTF-8 decoding. It’s more of a “here’s how I started to wrap my head around it” kind of thing. The Lua code is intentionally simplified — it skips over edge cases like invalid byte sequences, overlong encodings, and error handling. If you're looking for a formal spec, check out [RFC 3629](https://datatracker.ietf.org/doc/html/rfc3629) or the [Unicode Standard](https://www.unicode.org/standard/standard.html).

This series of events came about because of a hidden character in a Pull Request. Essentially the two files appeared identical in GitHub and there was no visible difference in my IDE.

Viewing the source code as hex showed the line feed which was not showing in text editors. The original copy did not end with a line feed. 

Really, the line feed should be there, but I didn't want to get hassled in code review, so I ran `truncate -s -1 file_name`, removing the last byte from the file.

This prompted me to look into how character encoding works.

I watched a video here: [YouTube](https://www.youtube.com/watch?v=MijmeoH9LT4&t=73s)

And my understanding became this: 

There was 7-bit ASCII, this was ok, but limited amount of characters available. With more bits, came more character sets. As with most things without a standard, things get pretty messy. Consequently, the Unicode Consortium put together a reasonable standard, which was backwards compatible with ASCII. Then UTF-8 was created as an implementation of Unicode, which is simply put, somewhat elegant.

So, then I wanted to make sure I understood it in a little bit more low-level detail, and I've been learning lua, so I decided I would try and decode some ASCII in lua. It was pretty easy and looked something like this.

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

Above, I created a table/map in lua of the ASCII characters I was going to try and decode. 
I read in the file contents `Hello World!` (all ASCII characters), access the map with each byte,
 writing them to stdout along the way, and voila, it works.

Then I wanted to extend this into UTF-8 and implement a second part, which would handle multi-bytes. This is where things started to get interesting, it's the elegant part I mentioned earlier. How UTF-8 works.

So ASCII characters are 7 bit. So we say, as long as a byte starts with a zero it is an ASCII character, otherwise, it's the extended part and its a multi-byte character.

> Each byte of a multi-byte begins with an active bit, i.e. the left-most bit of the byte is on/1/active. By peeking the left-most bit of the next byte we can tell if the multi-byte continues or ends.

To implement this, I decided to start with a tree type structure containing the extended parts which I could walk sequentially with each byte until I reach a character, print the character, and continue.

Which ended up like this:

```lua
-- Writes extended part of UTF-8 and original ASCII characters to file. UNCOMMENT TO CREATE THE FILE
--local file = io.open("utf8", "w")
--file:write("¢harmele")
--file:close()

local file = io.open("utf8", "r")
if file == nil then
  print("Unable to open file ".. arg[1])
  os.exit(-1)
end

local file_ctents = file:read()
if file_ctents == nil then os.exit() end

local multi_byte_map = {
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
  [104] = "h",
  [97]  = "a",
  [109] = "m",
  [110] = "n",
  [194] = {
    [162] = "¢",
  },
}

-- Recursively walk over list (table) of bytes to get multi byte char
local functi get_char(bytes, map)
  if type(map) == "string" then return map end
  local first_byte = bytes[1]
  table.remove(bytes, 1)
  return get_char(
    bytes,
    map[first_byte]
  )
end

local multi_byte_char = {}
for i = 1, #file_ctents do
  local hex = string.byte(file_ctents, i)
  local left_most_bit = (hex >> 7) & 1
  if left_most_bit == 1 then
    table.insert(multi_byte_char, hex)

    local peek = string.byte(file_ctents, i + 1)
    local peeked_left_most_bit = (peek >> 7) & 1

    if peeked_left_most_bit == 0 then
      io.write(get_char(multi_byte_char, multi_byte_map))
      multi_byte_char = {}
    end
  else
    io.write(get_char({hex}, multi_byte_map))
  end
end
io.write(multi_byte_map[12]) -- write LF

```

As we go over the bytes sequentially inside the for loop, if the left most bit of the current byte is 1 we know it is a multi-byte char, We start appending it to a list (table). We know we have reached the end of a multi-byte char by peeking the next byte. We then take the list of multi-byte chars and pass it to a recursive functi which will walk over the multi-byte map until it reaches a character. We write the character to stdout and then we can ctinue to the next char.

> Notes: 
> - There is likely a bug if the last character is a multi-byte character as when it tries to peek the left most bit of the next character the next character is non existent.
> - I only implemented the characters I was going to decode, rather than the full set too, maybe there is more bugs here as well.

0a

