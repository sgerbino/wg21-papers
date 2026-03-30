# WG21 Paper Style Rules - Ambient Guardrails

Rules that must fire during writing, not just during audit.
For mechanical verification (citations, formatting, structure,
prose hygiene), invoke the Auditor:
`situation-room/tools/auditor.md`

## Formatting

1. Markdown files must be ASCII only - no Unicode characters
2. Diacritics in personal names must never be omitted -
   represent them using HTML character references. Prefer
   named character references (`&uuml;`) over numeric
   character references (`&#252;`),
   e.g. `Dietmar K&uuml;hl`, `Micha&lstrok; Dominiak`
3. Single dashes (`-`) for all dashes - no em-dashes, no double dashes
4. Human-written tone - not wordy, no AI-characteristic phrasing
5. No git operations - user handles all git manually
6. The `archive/` folder is restricted for human use only - do
   not read, write, or modify any file in that folder
7. WG21-appropriate style

## Front Matter

8. Every paper begins with a valid YAML front matter block
   delimited by `---`. Required and optional fields:

   ```yaml
   ---
   title: "Paper Title"
   document: D0000R0
   date: YYYY-MM-DD
   reply-to:
     - "Author Name <author@example.com>"
   audience: SG1, LEWG
   ---
   ```

   - `title` - required, double-quoted string
   - `document` - required, unquoted paper number
     (e.g. D4007R0)
   - `date` - required, unquoted ISO 8601 date
   - `reply-to` - required, YAML list of double-quoted
     strings in the form `"Name <email>"`
   - `audience` - required, unquoted comma-separated
     target audience (e.g. SG1, LEWG)
9. Do not add front matter fields that the Pandoc template
   (`tools/wg21.html5`) does not consume

## Language and Grammar

10. American English by default. If a paper already uses
    British/Irish English consistently (e.g. "organisation",
    "behaviour", "generalise"), preserve that convention -
    do not convert to American English. When in doubt, check
    the existing spelling in the file before editing.
    Papers where Mungo Gill is the sole author use Irish
    English
11. Follow the Chicago Manual of Style unless otherwise specified
12. Prefer "deeper" or "more thorough" over "full" for appendix
    cross-references. Understatement is better than hyperbole
13. Transitive verbs must not leave their object implicit when the
    referent is not immediately obvious. Supply the object explicitly.
    e.g. "forgetting to forward the allocator" not "forgetting to
    forward"
14. Avoid ambiguous prepositional attachment. When a prepositional
    phrase could modify either a nearby noun or the main verb,
    restructure to make the attachment clear.
    e.g. "The allocator - along with its state - must be
    discoverable" not "The allocator - with its state - must be
    discoverable" (where "with its state" could attach to
    "discoverable")
15. Do not use contractions (it's, they're, don't, etc.) in formal
    papers. Use the expanded form

## Code Examples

16. Consistent comment style within a code block - either all
    comments are sentences (capitalized, with terminating punctuation)
    or all are fragments (lowercase, no period). Do not mix
17. Consistent capitalization in groups of code comments that
    summarize arithmetic or data
18. Align trailing comment columns when consecutive lines have
    trailing comments
19. Fenced code blocks must not exceed 90 characters per
    line. This rule does not apply to mermaid blocks -
    never reformat mermaid source. When wrapping is
    needed, follow the conventions for the block's
    language:

    **C++** - break at a natural syntactic boundary and
    indent the continuation 4 spaces from the construct
    it belongs to:
    - Function signatures: break after the return type,
      or after the opening parenthesis
    - Template parameter lists: break after a comma,
      align with the first parameter or indent 4 spaces
    - Chained expressions (pipe `|`, `<<`, etc.): break
      before the operator, align operators vertically
    - Trailing comments: shorten the gap between code and
      comment rather than wrapping the code, but keep
      grouped comments aligned with each other (rule 18)
    - If a trailing comment still exceeds 90 characters
      after tightening the gap, wrap the comment text to
      a second line. The continuation `//` must align
      with the `//` on the first line. Never move a
      trailing comment above the code line - keep all
      comments in a contiguous block on the right

    **Prose in code fences** (markdown, plain text) -
    wrap at word boundaries with no continuation indent

## Proposed Wording Sections

20. Proposed wording uses Pandoc fenced divs and HTML
    `<ins>`/`<del>` elements styled by `tools/paperstyle.css`.
    Three div classes are available:

    - `:::wording` - unchanged spec text (gray box, gray
      left border). Inline code inherits neutral color
    - `:::wording-add` - purely additive text (green box,
      green left border, green text)
    - `:::wording-remove` - purely removed text (red box,
      red left border, red strikethrough text)

    For mixed changes (unchanged text with inline additions
    or deletions), wrap the block in `:::wording` and mark
    individual changes with `<ins>` (green underline) or
    `<del>` (red strikethrough).

    Do not use the dual-blockquote pattern (separate
    "Current text:" and "Proposed text:" blockquotes).
    Do not use separate "Current:" and "Proposed:" fenced
    code blocks for code changes. Always use the inline
    `<ins>`/`<del>` convention described here instead

21. Fenced div formatting rules:

    - No space between `:::` and the class name:
      `:::wording` not `::: wording`
    - Blank line after the opening `:::wording*` marker
    - Blank line before the closing `:::`
    - Do not use blockquote `>` syntax inside wording
      divs - use plain paragraphs. The div provides
      visual framing
    - Markdown formatting (`*italic*`, `` `code` ``)
      works normally inside wording divs

22. Code diffs in wording sections use raw HTML
    `<pre><code>` blocks (not fenced `` ``` `` blocks)
    inside a `:::wording` div, with `<ins>` and `<del>`
    for inline changes. Angle brackets in code must be
    escaped as `&lt;`/`&gt;`. Example:

    ```
    :::wording

    <pre><code><del>void</del><ins>coroutine_handle&lt;&gt;</ins> await_suspend(
        coroutine_handle&lt;Promise&gt;) noexcept
    {
        <del>start(state);</del>
        <ins>auto h = start(state);</ins>
        <ins>return h ? h : noop_coroutine();</ins>
    }</code></pre>

    :::
    ```

## Tone

23. Do not present options as predetermined conclusions. When
    recommending alternatives to a committee, present them as options
    to contemplate, not dictated outcomes
24. Avoid politically charged comparisons - do not invoke other
    contentious features as analogies unless the comparison is
    structurally precise. If the structures being compared are
    fundamentally different, the analogy will be perceived as
    political

## Abstract

25. Every abstract opens with a single sentence on its own line -
    the brutal summary. No citations, no paper numbers, no
    hedging. The sentence must be true, complete, and
    unsentimental. It is the sentence a reader remembers after
    forgetting everything else. Examples:
    - "Three deployed executor models were replaced by one that
      was never deployed."
    - "The committee voted that sender/receiver covers
      networking. No sender-based networking has shipped."
    For ask-papers, "This paper asks [specific request]" on its
    own line satisfies the rule - the ask IS the brutal summary.
    The rest of the abstract follows after a blank line
