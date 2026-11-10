# Markdown Plugin Specification

## Heading attributes

### Overview
Heading attributes allow metadata (IDs, classes, and key-value pairs) to be attached to Markdown headings, following the [python-markdown attr_list specification](https://python-markdown.github.io/extensions/attr_list/). Example: `# Header {#custom-id .class-name key="value"}`.

### Syntax and Attribute Rules
1. **Reference Syntax**: Attribute syntax must conform to standard `python-markdown` `attr_list` (`{#id .class key=value}`). This explicitly includes support for the optional leading colon (for example `{: #id .class}`) and bare attributes (for example `{checked}`). Both double quotes (`"`) and single quotes (`'`) are supported for attribute values.
2. **Terminal Attribute Block & Parsing Direction**: The parser scans the heading text from right to left and recognizes the final terminal `{...}` block only if it is the final non-whitespace content on the line after trailing ATX hashes have been removed. 
   - **ATX Trailing Hashes**: Any optional ATX trailing hashes (for example `# Heading #`) MUST be stripped from the end of the line before scanning for the attribute block. Thus, `# Heading {#id} #` correctly resolves `{#id}` as the terminal block.
   - **Terminal-Block Rule**: The attribute block MUST be the final non-whitespace content on the line. If any non-whitespace text appears after the closing `}` (for example `# Header {#id} text`), the block is non-terminal and the entire heading is treated as literal text.
   - **Rightmost-Block Precedence**: The parser MUST NOT fall back to an earlier valid block when the rightmost candidate is malformed or unclosed. If the parser locates a terminal `}` but cannot find a matching unescaped `{` before the start of the heading, or the candidate fails `attr_list` validation, parsing aborts and the full heading is preserved verbatim. For example, given `# Header {#id} {bad`, the parser rejects `{bad`, aborts, and preserves the whole line as literal text.
   - **Quote-Aware Boundary Detection**: The scan MUST be quote-aware. Braces inside quoted values (for example `"a {b}"`) are ignored for boundary detection, and all content until the closing quote is treated as part of the value.
   - **Escaped Braces Boundary**: The scan MUST differentiate between literal escapes and block boundaries. Outside of quoted values, escaped curly braces (`\{` and `\}`) MUST be treated as literal text and do not open or close an attribute block. A valid terminal attribute block MUST begin with an unescaped `{` and end with an unescaped `}`.
3. **Formatting Pipeline Order**: The formatter MUST process headings in this strict sequence:
   - *Step 1 (Extraction)*: Strip trailing ATX hashes, then scan from the end of the line for the final terminal `{...}` block. If no valid terminal block is found, or the rightmost candidate is malformed or unclosed, treat the entire heading as verbatim text and exit parsing.
   - *Step 2 (Empty Check)*: Remove the extracted `{...}` block. Trim external whitespace from the remaining heading text. If no heading text remains, abort formatting and restore the original verbatim text.
   - *Step 3 (Normalization)*: Only if Step 2 succeeds (heading text exists), apply whitespace normalization to the valid attribute block.
4. **Empty Headings**: Headings containing no non-whitespace text (for example `# `, `#   {#id}`, or `##\t{#id}`) cannot contain heading attributes. If an attribute block exists without trimmed heading text, it is treated as plain text.
5. **Malformed Attribute Blocks**: Unparseable or malformed attribute blocks (for example `{}`, `{#}`, `{.}`, `{key=}`, or an unterminated quoted value) that do not conform to valid `attr_list` syntax MUST be treated as plain text and preserved verbatim.
6. **Leading Space Normalization**: A single space MUST precede the opening curly brace `{` in the formatted output. Input headings without a space before `{` (for example `# Heading{#id}`) are valid input and will be normalized to include a single space (`# Heading {#id}`).
7. **Whitespace Normalization**: Multiple consecutive spaces or tabs between individual attributes are collapsed to a single space. Any whitespace immediately after the opening `{` or before the closing `}` is removed entirely (for example `{ #id }` normalizes to `{#id}`).
8. **Quoted Attribute Values**: Attribute values enclosed in single or double quotes (for example `key="value with spaces"`, `title='a {b}'`) preserve all internal whitespace, braces, and escaped characters as-is. Inside quoted values, the backslash `\` is the only valid escape character. It escapes the quote character currently in use (for example `\"` inside double quotes, `\'` inside single quotes) and the backslash itself (`\\`). All other backslash sequences (for example `\n`, `\{`) are treated as literal string characters. An escaped quote does not terminate the attribute value. If a quoted value is unterminated (for example the closing quote is missing), the entire attribute block is considered malformed and MUST be treated as verbatim plain text.

### Line Wrapping & Reversibility Rules (textWrap)

The formatting behavior depends strictly on the heading syntax (ATX vs Setext) to guarantee **Round-trip reversibility** (formatting the file must not alter its parsed AST structure).

1. **ATX Headings (`#` through `######`)**:
   - ATX headings MUST NOT be wrapped across multiple lines under any circumstances, even if line length exceeds `lineWidth`.
   - Both heading text and attribute blocks `{ ... }` remain on a single line to preserve valid Markdown AST syntax upon re-parsing.

2. **Setext Headings (Level 1 `===` and Level 2 `---`)**:
   - When `heading.kind: setext` and `textWrap: always`, Level 1 and Level 2 heading text MUST wrap across multiple lines if the line length exceeds the configured `lineWidth`.
   - **Attribute Attachment on Wrap**: When a Setext heading wraps across multiple lines, the attribute block `{ ... }` MUST be attached to the end of the final (last) wrapped line of heading text.
   - **Underline Size**: The underline (`===` or `---`) MUST be placed on the line immediately following the heading text/attributes. Its length MUST equal the string/code-point length of the longest line in the wrapped block, explicitly including the attached attribute block. Visual or display width (e.g., full-width CJK characters) is explicitly ignored in favor of strict code-point counts to ensure consistent cross-platform generation.
   - Level 3+ headings do not exist in Setext syntax and MUST NOT wrap.

3. **Attribute Block Atomicity**:
   - The attribute block `{ ... }` is strictly atomic. The formatter MUST NOT introduce line breaks inside the `{}` block.
   - When wrapping Setext headings, the attribute block MUST remain on the same line as the final word of the heading text. It cannot be orphaned onto a new line by itself, even if the combined length of the final text word and the attribute block exceeds `lineWidth`.

### Examples

#### Example 1: ATX Heading (No Wrapping Allowed)
##### Input
# This heading is long enough that it should wrap onto multiple lines {#my-id .my-class}

##### Output
# This heading is long enough that it should wrap onto multiple lines {#my-id .my-class}

#### Example 2: Setext Level 1 Wrapping & Underline Sizing (lineWidth: 40, textWrap: always)
##### Input
This heading is long enough that it should wrap onto multiple lines {#my-id .my-class}
===

##### Output
This heading is long enough that it
should wrap onto multiple lines {#my-id .my-class}
==================================================

#### Example 3: Literal Braces & Quoted Value Boundaries
##### Input
# Header {literal} {#my-id key="a {b}"}

##### Output
# Header {literal} {#my-id key="a {b}"}

#### Example 4: Invalid Empty Heading Attributes
##### Input
#    {#my-id}

##### Output
#    {#my-id}

#### Example 5: Malformed Attribute Blocks Preserved Verbatim
##### Input
# Header {} {#} {key=}

##### Output
# Header {} {#} {key=}

#### Example 6: Multiple Attribute Blocks and Precedence
##### Input
# Header {#first-id} {.valid-class} {unclosed-bad

##### Output
# Header {#first-id} {.valid-class} {unclosed-bad

#### Example 7: ATX Trailing Hashes After Attribute Block
##### Input
# Header {#my-id} ##

##### Output
# Header {#my-id} ##

#### Example 8: Multiple Valid Attribute Blocks (Only Terminal Block is Extracted)
##### Input
# Header {#first} {.second}

##### Output
# Header {#first} {.second}

#### Example 9: Missing Opening Brace
##### Input
# Header }

##### Output
# Header }

#### Example 10: Escaped Braces
##### Input
# Header \{#id}

##### Output
# Header \{#id}

#### Example 11: Optional Colon and Bare Attributes
##### Input
# Header {: #my-id checked}

##### Output
# Header {: #my-id checked}