# Chapter 3: Style Struct

Welcome back! In [Chapter 1: Profile Struct](01_profile_struct_.md), we learned how `mist` adapts to different terminal capabilities using the `Profile`. Then, in [Chapter 2: Color System (AnyColor, ANSIColor, RGBColor, etc.)](02_color_system__anycolor__ansicolor__rgbcolor__etc___.md), we explored how `mist` represents the various kinds of colors terminals can understand.

Now, it's time to put it all together! How do we actually *tell* `mist` we want our text to be, say, **bold and bright red**? This is where the `Style` struct comes in.

## What is the `Style` Struct?

Think of the `Style` struct like a painter's palette or a set of instructions for how text should look. You start with a blank `Style` and then add colors and effects one by one.

*   Want red text? Add the "red foreground" instruction.
*   Want it bold too? Add the "bold" instruction.
*   Want a blue background? Add the "blue background" instruction.

Each time you add an instruction, you essentially create a *new, updated* palette. The original palette remains unchanged. Finally, you use this combined set of instructions (your final palette) to "paint" or `render` your text onto the terminal canvas.

**The main job of `Style` is:**

1.  To hold a collection of desired visual attributes (like foreground color, background color, bold, italic, etc.).
2.  To use its associated [Profile Struct](01_profile_struct_.md) to ensure colors are handled correctly for the target terminal.
3.  To provide a `render()` method that takes your plain text and wraps it with the necessary ANSI escape codes to make it appear styled in the terminal.

## Creating a Style

You create a `Style` object like this:

```mojo
import mist

fn main():
    # Create a default Style.
    # Mist will try to detect your terminal's capabilities automatically.
    var my_style = mist.Style()

    # You can also be explicit about the terminal capabilities
    # For example, creating a style limited to basic 16 ANSI colors:
    var ansi_style = mist.Style(mist.ANSI_PROFILE)

    # Or a style that supports True Color (millions of colors):
    var true_color_style = mist.Style(mist.TRUE_COLOR_PROFILE)
```

This code creates `Style` objects. `my_style` will try to guess the best [Profile Struct](01_profile_struct_.md) for your terminal. `ansi_style` and `true_color_style` are explicitly told which color level to use.

## Adding Colors and Effects (Layer by Layer)

The core idea of `Style` is its **immutability**. When you apply a new attribute (like a color or effect), you don't change the *existing* `Style` object. Instead, you get a *brand new* `Style` object with the change applied. This makes it easy to build up styles step-by-step or reuse base styles.

Let's add some attributes:

```mojo
import mist

fn main():
    # Start with a basic style (auto-detects profile)
    let base_style = mist.Style()
    print("Base style:", base_style)

    # Add a foreground color (e.g., bright red 0xFF0000)
    # This creates a *new* style object. base_style is unchanged.
    let red_style = base_style.foreground(0xFF0000)
    print("Red style:", red_style)

    # Add bold effect to the red style.
    # This creates *another* new style object. red_style is unchanged.
    let red_bold_style = red_style.bold()
    print("Red and Bold style:", red_bold_style)

    # We could also chain the calls:
    let blue_italic_style = base_style.foreground(0x0000FF).italic()
    print("Blue and Italic style:", blue_italic_style)
```

**What's happening:**

1.  We start with `base_style`.
2.  `base_style.foreground(0xFF0000)` creates a *new* `Style` (`red_style`) that remembers "use foreground color 0xFF0000". `base_style` still has no attributes.
3.  `red_style.bold()` creates *another new* `Style` (`red_bold_style`) that remembers "use foreground color 0xFF0000" AND "use bold". `red_style` is unchanged.
4.  We can chain methods like `base_style.foreground(0x0000FF).italic()` because each method returns a new `Style` object that the next method can operate on.

`Style` has many methods like these:
*   `.foreground(color_value)`: Sets text color.
*   `.background(color_value)`: Sets background color.
*   `.bold()`: Makes text bold.
*   `.italic()`: Makes text italic (terminal support varies).
*   `.underline()`: Underlines text.
*   `.strikethrough()`: Draws a line through text.
*   ... and many more!

It also includes convenient shortcuts for the 16 standard ANSI colors:
*   `.red()`: Sets foreground to bright red (ANSI 9).
*   `.blue()`: Sets foreground to bright blue (ANSI 12).
*   `.green_background()`: Sets background to bright green (ANSI 10).
*   ... etc.

These shortcuts are equivalent to using `.foreground()` or `.background()` with the corresponding ANSI color codes (0-15).

```mojo
import mist

fn main():
    let style = mist.Style()

    # These two are equivalent ways to get bright red text:
    let style1 = style.foreground(9)  # ANSI Bright Red code is 9
    let style2 = style.red()          # Shortcut method

    print("Style 1:", style1)
    print("Style 2:", style2)

    # These two are equivalent ways to get a standard blue background:
    let style3 = style.background(4)  # ANSI Standard Blue code is 4
    let style4 = style.navy_background() # Shortcut uses the 'navy' name

    print("Style 3:", style3)
    print("Style 4:", style4)

```

## Rendering: Applying the Style to Text

Okay, we've built our palette (`Style` object) with all the desired colors and effects. How do we apply it to our text? We use the `.render()` method.

```mojo
import mist

fn main():
    let text = "Hello, styled world!"

    # 1. Create a style: Bright Magenta Foreground, Bold, Underlined
    let my_style = mist.Style().magenta().bold().underline()

    # 2. Render the text using this style
    let styled_text: String = my_style.render(text)

    # 3. Print the result
    print(styled_text)
```

**What happens when you print `styled_text`?**

You won't see the literal string `"\x1b[95;1;4mHello, styled world!\x1b[0m"` (unless your terminal is very basic). Instead, your terminal interprets these special codes:

*   `\x1b[`: This is the Control Sequence Introducer (CSI), telling the terminal "styling instructions coming!"
*   `95`: Code for bright magenta foreground.
*   `;1`: Code for bold.
*   `;4`: Code for underline.
*   `m`: Marks the end of the styling instructions.
*   `Hello, styled world!`: Your actual text.
*   `\x1b[0m`: This resets *all* styling back to the default, so text after this isn't accidentally magenta, bold, and underlined.

Your terminal reads these codes and displays "Hello, styled world!" in bright magenta, bolded, and underlined text. The `Style` struct did the work of figuring out the correct codes (`95;1;4`) based on the methods you called (`.magenta().bold().underline()`).

## Under the Hood: How Styling Works

Let's trace the journey when you call a styling method and then `render()`:

1.  **You call a method:** e.g., `style.foreground(0xFF0000)`
2.  **`Style` consults `Profile`:** The `Style` object asks its associated [Profile Struct](01_profile_struct_.md), "Hey, the user wants color `0xFF0000`. What's the best way to represent this for the terminal you support?".
3.  **`Profile` determines `AnyColor`:** The `Profile` might convert `0xFF0000`. If the profile is `TRUE_COLOR`, it keeps it as an `RGBColor`. If it's `ANSI`, it converts it to the closest basic `ANSIColor` (like standard red, code `1`). It wraps the result in an [AnyColor](02_color_system__anycolor__ansicolor__rgbcolor__etc___.md) container.
4.  **`Style` gets the sequence:** `Style` asks the returned `AnyColor` for its specific ANSI code sequence (e.g., `31` for standard red foreground, or `38;2;255;0;0` for true color red foreground).
5.  **`Style` stores the sequence:** The `Style` object creates a *new* `Style` instance, copying the old sequences and adding this new sequence string (e.g., "31") to its internal list of styles.
6.  **You call `.render(text)`:** The final `Style` object takes all the stored sequence strings (e.g., "31", "1" for bold) joins them with semicolons, wraps them in `CSI` (`\x1b[`) and `m`, adds your text, and finally appends the reset sequence (`\x1b[0m`).

```mermaid
sequenceDiagram
    participant User
    participant MyStyle as Style
    participant MyProfile as Profile
    participant ColorSystem as AnyColor/etc.
    participant Terminal

    User->>MyStyle: foreground(0xFF0000)
    MyStyle->>MyProfile: What color should I use for 0xFF0000?
    MyProfile->>ColorSystem: Convert 0xFF0000 based on profile (e.g., to ANSI Red=1)
    ColorSystem-->>MyProfile: Use ANSIColor(1)
    MyProfile-->>MyStyle: Use ANSIColor(1) wrapped in AnyColor
    MyStyle->>ColorSystem: Get sequence for ANSIColor(1) (foreground)
    ColorSystem-->>MyStyle: Sequence is "31"
    MyStyle->>MyStyle: Create new Style, store "31"

    User->>MyStyle: bold()
    MyStyle->>MyStyle: Create new Style, store "1"

    User->>MyStyle: render("Hi")
    MyStyle->>Terminal: Send "\x1b[31;1mHi\x1b[0m"
    Terminal-->>User: Displays bold red "Hi"
```

Let's look at simplified code concepts from `src/mist/style.mojo`:

```mojo
// Simplified concept from src/mist/style.mojo

struct Style:
    var styles: List[String] # Stores sequences like "31", "1"
    var profile: Profile

    # Simplified concept for adding a foreground color
    fn foreground(self, color_value: UInt32) -> Style:
        # 1. Ask profile to convert the input color
        let actual_color: AnyColor = self.profile.color(color_value)

        # If profile is ASCII, actual_color might be NoColor, handle that...
        if actual_color.isa[NoColor]():
            return self.copy() # No change needed

        # 2. Get the ANSI sequence string for this color (foreground)
        let sequence: String = actual_color.sequence[False]() # False means foreground

        # 3. Create a *new* Style with the sequence added
        var new_style = self.copy()
        new_style.styles.append(sequence)
        return new_style

    # Simplified concept for adding bold
    fn bold(self) -> Style:
        # Bold has a fixed sequence "1" (defined in SGR)
        let sequence = SGR.BOLD # SGR.BOLD is "1"

        # Create a *new* Style with the sequence added
        var new_style = self.copy()
        new_style.styles.append(sequence)
        return new_style

    # Simplified concept for rendering
    fn render[T: Writable, //](self, text: T) -> String:
        # Handle ASCII profile (no styling) or no styles applied
        if self.profile == Profile.ASCII or len(self.styles) == 0:
            return String(text) # Just return plain text

        # Join all stored sequences with ';'
        let joined_styles = ";".join(self.styles)

        # Construct the final string: CSI + styles + m + text + RESET
        return String(CSI, joined_styles, "m", text, RESET_STYLE)
```

This shows how `Style` uses its `profile` to get the right color representation via `AnyColor`, gets the corresponding ANSI sequence string from the color object, stores these strings, and finally combines them with your text inside the `CSI...m` wrapper and adds a reset code when you call `render()`.

## Conclusion

You've now learned about the `Style` struct, the central tool in `mist` for defining how your text should look!

*   It acts like a **painter's palette**, letting you layer colors and effects.
*   It's **immutable**: methods like `.foreground()` or `.bold()` return *new* `Style` objects.
*   It uses the [Profile Struct](01_profile_struct_.md) to intelligently handle colors for different terminals.
*   It relies on the [Color System (AnyColor, ANSIColor, RGBColor, etc.)](02_color_system__anycolor__ansicolor__rgbcolor__etc___.md) to get the right ANSI sequences.
*   The `.render()` method applies the combined style to your text, producing the final string with ANSI escape codes.

With `Profile`, the Color System, and `Style`, you have the core tools to add rich formatting to your terminal output. But `mist` can do more than just color text – it can also help reshape it!

**Next Up:** [Chapter 4: Text Transformation (transform modules)](04_text_transformation__transform_modules__.md) - Learn how to indent, wrap, truncate, and otherwise manipulate blocks of text.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)