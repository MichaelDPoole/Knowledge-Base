# Chapter 5: Measurement Functions (get_width/get_height/get_dimensions)

In the [previous chapter](04_position_.md), we learned how to use `mog.Position` to align text within a fixed space. We can set a `Style` to have a specific `width` or `height`. But what if we *don't* know the exact size beforehand? What if we have some text, maybe with colors or **bold** styling, and we need to know how much space it *actually* takes up on the screen?

This might sound simple – just count the characters, right? Unfortunately, it's tricky in the terminal!

1.  **Invisible Codes:** Text styled with colors or effects (like bold) contains hidden "ANSI escape codes". These codes tell the terminal *how* to display the text, but they don't take up any visual space themselves. `len("red text")` is very different from `len("\x1b[31mred text\x1b[0m")`, even though they look the same size on screen.
2.  **Wide Characters:** Some characters, like emojis (✨) or Chinese/Japanese/Korean (CJK) characters (你好), often take up the space of *two* standard characters in the terminal. A simple character count won't accurately reflect the visual width.

So, how do we measure the *visible* size of text, ignoring the invisible codes and correctly accounting for wide characters? That's where `mog`'s measurement functions come in! Think of them as a special ruler designed specifically for terminal text.

## The Terminal Ruler: `get_width`, `get_height`, `get_dimensions`

`mog` provides three simple utility functions to measure strings accurately for terminal display, found in the `mog.size` module (though often available directly under `mog` depending on imports).

### 1. `mog.get_width()`: Measuring Horizontally

This function calculates the maximum visible width of a string in terminal cells. It intelligently skips over ANSI escape codes and counts wide characters (like emojis) as 2 cells wide. If the string has multiple lines, it returns the width of the *widest* line.

```mojo
import mog

fn main():
    let plain_text = "Hello"
    let styled_text = mog.Style().foreground(mog.Color(0xFF0000)).render("Hello") # Red "Hello"
    let emoji_text = "Hi ✨"
    let multi_line = "Line 1\nLonger Line 2"

    print("Width of '", plain_text, "': ", mog.get_width(plain_text))
    print("Width of '", styled_text, "': ", mog.get_width(styled_text)) # Note: Prints the raw styled string here
    print("Width of '", emoji_text, "': ", mog.get_width(emoji_text))
    print("Width of multi-line text:\n", multi_line, "\nWidth: ", mog.get_width(multi_line))

```

**Conceptual Output:**

```text
Width of ' Hello ':  5
Width of ' \x1b[38;2;255;0;0mHello\x1b[0m ':  5
Width of ' Hi ✨ ':  5
Width of multi-line text:
 Line 1
Longer Line 2
Width:  14
```

**Explanation:**

*   `mog.get_width("Hello")` is 5, as expected.
*   `mog.get_width(styled_text)` is also 5! It correctly ignored the red color codes (`\x1b[...m`).
*   `mog.get_width("Hi ✨")` is 5: "H" (1) + "i" (1) + " " (1) + "✨" (2) = 5.
*   `mog.get_width(multi_line)` is 14, because the second line ("Longer Line 2") is the widest, and its width is 14.

### 2. `mog.get_height()`: Measuring Vertically

This function calculates the visible height of a string in terminal cells. In most cases, this is simply the number of lines. It counts the newline characters (`\n`) and adds 1.

```mojo
import mog

fn main():
    let single_line = "Just one line."
    let multi_line = "Line 1\nLine 2\nLine 3"
    let trailing_newline = "Two lines\n" # Still effectively 2 lines of content

    print("Height of '", single_line, "': ", mog.get_height(single_line))
    print("Height of multi-line text:\n", multi_line, "\nHeight: ", mog.get_height(multi_line))
    print("Height of '", trailing_newline, "': ", mog.get_height(trailing_newline))

```

**Conceptual Output:**

```text
Height of ' Just one line. ':  1
Height of multi-line text:
 Line 1
Line 2
Line 3
Height:  3
Height of ' Two lines\n ':  2
```

**Explanation:**

*   `mog.get_height(single_line)` is 1.
*   `mog.get_height(multi_line)` is 3 (two `\n` characters + 1).
*   `mog.get_height(trailing_newline)` is 2 (one `\n` character + 1).

### 3. `mog.get_dimensions()`: Measuring Both

This function is a convenient way to get both the width and height at the same time. It returns a simple `mog.Dimensions` struct (which basically just holds a `width` and `height` field).

```mojo
import mog
from mog import Dimensions # Import the Dimensions struct

fn main():
    let text = "Box\nContent ✨"

    let dims: Dimensions = mog.get_dimensions(text)

    print("Text:\n", text)
    print("Width: ", dims.width)
    print("Height: ", dims.height)
```

**Conceptual Output:**

```text
Text:
 Box
Content ✨
Width:  10
Height:  2
```

**Explanation:**

*   `mog.get_dimensions(text)` calculates the width of the widest line ("Content ✨" is 1 + 1 + 1 + 1 + 1 + 1 + 1 + 1 + 2 = 10) and the height (1 newline + 1 = 2).
*   It returns a `Dimensions` object, and we access its fields using `.width` and `.height`.

## Why Measure?

Knowing the *real* size of terminal text is essential for:

*   **Layout:** If you want to place two pieces of text side-by-side using [Layout Functions (join_horizontal/join_vertical)](06_layout_functions__join_horizontal_join_vertical__.md), you need to know how wide each piece is to align them correctly.
*   **Dynamic Sizing:** You might want to draw a border around some text using `Style.border()`. Without measuring the text first, how would you know how big the box needs to be? (Though `Style.render()` often handles this measurement internally).
*   **Building Components:** When creating more complex UI elements like a [Table](07_table_.md), the component needs to measure the content of each cell to determine the correct column widths and row heights.

## How Measurement Works (Under the Hood)

Measuring terminal text correctly involves understanding how terminals process character streams.

1.  **Scanning the String:** The function iterates through the input string, rune by rune (Mojo's equivalent of characters, handling Unicode correctly).
2.  **ANSI State Machine:** It uses a state machine (provided by the underlying `mist` library) to detect ANSI escape sequences. When it encounters the start of an escape sequence (like `\x1b[`), it enters a state where it consumes characters belonging to that sequence until it reaches the end marker (usually a letter like `m`). All characters consumed while in this "ANSI sequence state" are ignored for width calculation.
3.  **Rune Width Check:** For any rune that is *not* part of an ANSI sequence, the function asks a specialized Unicode library (like `mist.transform.ansi.printable_rune_width`) "How many terminal cells wide is this specific rune?". This library knows that 'A' is 1 cell, but '✨' or '你' are typically 2 cells.
4.  **Summing Width:** For `get_width`, it keeps track of the current line's width by summing the cell widths of the printable runes. When it encounters a newline (`\n`), it compares the current line's width to the maximum width found so far, updates the maximum if necessary, and resets the current line's width to 0. After processing the whole string, it returns the overall maximum width found.
5.  **Counting Height:** For `get_height`, it simply counts the number of newline (`\n`) characters encountered and adds 1 to the final count.

Essentially, `get_width` acts like a tiny terminal emulator, parsing the string to figure out what would actually be displayed and how much space it would take.

## Code Dive: `src/mog/size.mojo`

Let's peek at the functions themselves. They are quite straightforward as they delegate the hard work to `mist`.

```mojo
# From: src/mog/size.mojo
import mist.transform.ansi
from mist.transform.traits import AsStringSlice
from mog._properties import Dimensions

# Calculates width of the widest line, ignoring ANSI, counting wide chars
fn get_width[T: AsStringSlice](text: T) -> Int:
    var width = 0
    # Iterate through each line of the input string
    for line in text.as_string_slice().splitlines():
        # Use mist's function to get the printable width of this line
        var w = ansi.printable_rune_width(line[])
        # Keep track of the maximum width found so far
        if w > width:
            width = w
    return width

# Calculates height by counting newlines
fn get_height[T: AsStringSlice](text: T) -> Int:
    # Count '\n' characters and add 1 for the number of lines
    return text.as_string_slice().count(NEWLINE) + 1

# Gets both width and height
fn get_dimensions[T: AsStringSlice](text: T) -> Dimensions:
    # Call the other two functions and return a Dimensions struct
    return Dimensions(width=get_width(text), height=get_height(text))

```

**Explanation:**

*   The functions are generic (`[T: AsStringSlice]`) meaning they can accept various string-like types.
*   `get_width` iterates lines and calls `mist.transform.ansi.printable_rune_width()` on each line to get its visible width, tracking the maximum.
*   `get_height` simply uses the built-in `count()` method to find newlines.
*   `get_dimensions` is a wrapper that calls `get_width` and `get_height` and bundles the results into a `Dimensions` struct.

The core magic for width calculation lies within `mist.transform.ansi.printable_rune_width`, which handles both skipping ANSI codes and checking Unicode character widths.

## Conclusion

You've learned about `mog`'s essential measurement tools:

*   `get_width()`: Measures the maximum visible **width** of a string, ignoring ANSI codes and handling wide characters correctly.
*   `get_height()`: Measures the visible **height** of a string (usually the number of lines).
*   `get_dimensions()`: A convenient function to get both width and height.

These functions are like a special ruler for your terminal text, providing the accurate measurements needed for precise layout and component building.

Speaking of layout, now that we know how to style text ([Style](02_style_.md)), create borders ([Border](03_border_.md)), align content ([Position](04_position_.md)), and measure the results, how do we combine multiple styled elements together? In the next chapter, we'll explore the [Layout Functions (join_horizontal/join_vertical)](06_layout_functions__join_horizontal_join_vertical__.md) that let us arrange our styled text blocks side-by-side or one above the other.

[Next Chapter: Layout Functions (join_horizontal/join_vertical)](06_layout_functions__join_horizontal_join_vertical__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)