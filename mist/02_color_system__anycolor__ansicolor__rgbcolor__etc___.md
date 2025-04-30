# Chapter 2: Color System (AnyColor, ANSIColor, RGBColor, etc.)

In [Chapter 1: Profile Struct](01_profile_struct_.md), we learned how the `Profile` struct helps `mist` understand what kinds of colors and styles your terminal supports. It acts like a translator, making sure your requests look right even on terminals with limited capabilities.

But how does `mist` actually represent the *colors themselves*? If you ask for "red", is that the same as asking for the specific hex code `#FF0000`? What about terminals that only have a basic set of colors? Let's explore `mist`'s color system.

## Why Different Color Types?

Imagine you have different ways to describe a color:

1.  **By Name (Basic):** "Red", "Green", "Blue". Simple, but limited.
2.  **By Code (Extended Palette):** Maybe you have a chart with 256 specific, numbered paint swatches. You can ask for "Color #196" (a bright red). More choice, but still a fixed set.
3.  **By Recipe (Full Custom):** You can specify the exact amount of Red, Green, and Blue light to mix, like "255 units of Red, 0 units of Green, 0 units of Blue". This gives you *millions* of possibilities ("True Color").

Terminals also understand colors in these different ways:

*   Some only know the basic named colors (**ANSI**).
*   Some know the 256-color palette (**ANSI256**).
*   Some can mix exact recipes (**True Color / RGB**).

`mist` needs a way to represent each of these color description methods internally.

## Meet the Color Types

`mist` defines specific structs for each way of representing color. You'll mostly interact with these indirectly through the `Style` and `Profile`, but it's good to know they exist!

*(These types are defined in `src/mist/color.mojo`)*

1.  **`ANSIColor`:** Represents one of the 16 basic ANSI colors (codes 0-15).
    *   Think: The small, 16-crayon box.
    *   Example: `ANSIColor(1)` represents standard red. `ANSIColor(9)` represents bright red.

    ```mojo
    import mist.color { ANSIColor }

    // Represents standard ANSI red (code 1)
    let basic_red = ANSIColor(1)
    print(basic_red) // Output: ANSIColor(1)

    // Represents bright ANSI blue (code 12)
    let bright_blue = ANSIColor(12)
    print(bright_blue) // Output: ANSIColor(12)
    ```
    These correspond to the classic 8 standard and 8 bright colors your terminal likely supports.

2.  **`ANSI256Color`:** Represents one of the 256 indexed colors (codes 0-255). The first 16 are the same as `ANSIColor`.
    *   Think: The larger, 256-crayon box.
    *   Example: `ANSI256Color(196)` represents a specific bright red from the 256-color palette. `ANSI256Color(21)` represents a specific shade of blue.

    ```mojo
    import mist.color { ANSI256Color }

    // Represents color index 196 (a bright red)
    let palette_red = ANSI256Color(196)
    print(palette_red) // Output: ANSI256Color(196)

    // Represents color index 21 (a dark blue)
    let palette_blue = ANSI256Color(21)
    print(palette_blue) // Output: ANSI256Color(21)
    ```
    This gives you access to a much wider, but still predefined, range of colors, including shades of grey.

3.  **`RGBColor`:** Represents a "True Color" using a 24-bit value, often written as a hexadecimal number (like `0xFF0000`).
    *   Think: Mixing your own paint from Red, Green, and Blue tubes. Millions of possibilities!
    *   Example: `RGBColor(0xFF0000)` represents pure, bright red. `RGBColor(0xC9A0DC)` is that specific purple from Chapter 1.

    ```mojo
    import mist.color { RGBColor }

    // Represents pure red using a hex code
    let true_red: UInt32 = 0xFF0000
    let rgb_red = RGBColor(true_red)
    print(rgb_red) // Output: RGBColor(16711680) (16711680 is the decimal for 0xFF0000)

    // Represents a specific purple
    let fancy_purple: UInt32 = 0xC9A0DC
    let rgb_purple = RGBColor(fancy_purple)
    print(rgb_purple) // Output: RGBColor(13214940)
    ```
    This provides the most color precision, but requires a terminal with True Color support (like those using the `TRUE_COLOR_PROFILE`).

4.  **`NoColor`:** A special type representing the absence of color, used for the `ASCII` profile.

## `AnyColor`: The Universal Holder

Now, imagine you have these different color types (`ANSIColor`, `ANSI256Color`, `RGBColor`, `NoColor`). How can `mist` handle them smoothly? What if a function needs to accept *any* kind of color?

That's where `AnyColor` comes in! It's like a universal container or wrapper that can hold *any one* of the specific color types.

*   Think: A special box that can perfectly fit *either* the small crayon box, *or* the big crayon box, *or* a custom-mixed paint can.

You usually won't create `AnyColor` directly. Instead, the [Profile Struct](01_profile_struct_.md) creates it for you when you ask for a color.

## How `Profile` Uses the Color System

Let's revisit the `Profile` from Chapter 1. Its job is to take whatever color you *request* and figure out the *best representation* the terminal can handle, returning it wrapped in an `AnyColor`.

Imagine you have a `Style` using the `ANSI_PROFILE` (which only understands the basic 16 `ANSIColor`s).

```mojo
import mist { Style, ANSI_PROFILE }
import mist.color { AnyColor, ANSIColor, RGBColor }

fn main():
    let style_ansi = Style(ANSI_PROFILE)

    // Request 1: A basic ANSI color code (Red = 1)
    let color1: AnyColor = style_ansi.profile.color(1)
    print("Requested 1:", color1)
    // Output: Requested 1: AnyColor(value=ANSIColor(1))
    // Profile sees 1 is < 16, knows ANSI can handle it. Returns AnyColor(ANSIColor(1)).

    // Request 2: A 256-color code (Dark Orange = 208)
    let color2: AnyColor = style_ansi.profile.color(208)
    print("Requested 208:", color2)
    // Output: Requested 208: AnyColor(value=ANSIColor(11))
    // Profile sees 208 is > 15, knows ANSI *can't* handle it directly.
    // It converts 208 to the *closest* basic ANSI color (Yellow = 11).
    // Returns AnyColor(ANSIColor(11)).

    // Request 3: A True Color hex code (Fancy Purple = 0xC9A0DC)
    let fancy_purple: UInt32 = 0xC9A0DC
    let color3: AnyColor = style_ansi.profile.color(fancy_purple)
    print("Requested 0xC9A0DC:", color3)
    // Output: Requested 0xC9A0DC: AnyColor(value=ANSIColor(13))
    // Profile sees a hex code, knows ANSI *can't* handle it directly.
    // It converts 0xC9A0DC to the *closest* basic ANSI color (Magenta = 13).
    // Returns AnyColor(ANSIColor(13)).
```

Notice how `style_ansi.profile.color()` always returns an `AnyColor`. Inside that `AnyColor` is the *actual* color type (`ANSIColor` in this case, because the profile is `ANSI`) that the profile determined is appropriate.

If the profile was `TRUE_COLOR_PROFILE`, requesting `0xC9A0DC` would result in `AnyColor(value=RGBColor(13214940))`. The `Profile` intelligently chooses the best representation and conversion path.

## Under the Hood: `Profile.color()` Logic

How does the `Profile` decide which color type to create and whether to convert? Let's peek at the simplified logic inside the `Profile.color()` method (found in `src/mist/profile.mojo`):

```mojo
// Simplified concept from src/mist/profile.mojo

struct Profile:
    var _value: Int # TRUE_COLOR, ANSI256, ANSI, or ASCII

    fn color(self, value: UInt32) -> AnyColor:
        # 1. Handle ASCII Profile (No Colors)
        if self._value == ASCII:
            return NoColor() # Return the special NoColor type

        # 2. Handle Basic ANSI Color Input (0-15)
        if value < 16:
            # All color profiles (ANSI, ANSI256, TRUE_COLOR) support basic ANSI.
            return ANSIColor(value.cast[DType.uint8]())

        # 3. Handle ANSI 256 Color Input (16-255)
        elif value < 256:
            if self._value == ANSI:
                # If profile is only ANSI, convert down to the closest basic ANSI color.
                let converted_ansi: UInt8 = ansi256_to_ansi(value.cast[DType.uint8]())
                return ANSIColor(converted_ansi)
            else:
                # ANSI256 and TRUE_COLOR profiles can handle this directly.
                return ANSI256Color(value.cast[DType.uint8]())

        # 4. Handle True Color Input (>= 256, assumed Hex Code)
        else: # value >= 256
             if self._value == TRUE_COLOR:
                 # True color profile handles this directly.
                 return RGBColor(value)
             elif self._value == ANSI256:
                 # ANSI256 profile: Convert True Color down to closest 256-color.
                 # (Uses functions like hex_to_rgb and hex_to_ansi256)
                 let rgb_tuple = hex_to_rgb(value)
                 let hue_color = hue.Color(rgb_tuple)
                 let converted_256: UInt8 = hex_to_ansi256(hue_color)
                 return ANSI256Color(converted_256)
             else: # Must be ANSI profile
                 # ANSI profile: Convert True Color *way* down to closest 16-color.
                 # (May involve converting hex -> 256 -> 16)
                 let rgb_tuple = hex_to_rgb(value)
                 let hue_color = hue.Color(rgb_tuple)
                 let converted_256: UInt8 = hex_to_ansi256(hue_color)
                 let converted_ansi: UInt8 = ansi256_to_ansi(converted_256)
                 return ANSIColor(converted_ansi)

```

This logic ensures that:
*   You can provide color input as simple ANSI codes (0-15), extended codes (16-255), or full hex codes.
*   The `Profile` checks its own capability level.
*   It converts the input color *down* if necessary, using helper functions like `ansi256_to_ansi` and `hex_to_ansi256` (which internally use color math from `src/mist/_hue.mojo` and color tables from `src/mist/_ansi_colors.mojo`).
*   It always returns the result wrapped in an `AnyColor` containing the final, profile-appropriate color type (`NoColor`, `ANSIColor`, `ANSI256Color`, or `RGBColor`).

The `Style` then takes this `AnyColor` and uses its `sequence()` method (defined in `src/mist/color.mojo`) to generate the correct ANSI escape code string to send to the terminal.

## Conclusion

You've now seen how `mist` represents colors internally using different types:
*   `ANSIColor` for the basic 16.
*   `ANSI256Color` for the 256-color palette.
*   `RGBColor` for True Color hex codes.
*   `NoColor` for terminals without color support.

And you learned about `AnyColor`, the flexible container that holds any of these specific types.

Crucially, you saw how the [Profile Struct](01_profile_struct_.md) uses this color system. When you request a color, the `Profile` intelligently converts it to the best representation the terminal supports (e.g., downgrading an `RGBColor` to an `ANSIColor` if needed) and returns it neatly packaged in an `AnyColor`.

With `Profile` handling terminal capabilities and the Color System handling color representation and conversion, we now have all the pieces to actually *apply* styles to text!

**Next Up:** [Chapter 3: Style Struct](03_style_struct_.md) - Let's put it all together and make some colorful text!

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)