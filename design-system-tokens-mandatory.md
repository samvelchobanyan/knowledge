---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [design-system, tokens, ds-colors, ds-spacing, ds-radius, ds-typography, theme, no-hardcoded-values]
related: [[layered-architecture]]
---

# Design system tokens are mandatory

## Rule

All visual values come from the design system tokens in `design_system/foundations/`:

- **`DSColors`** — palette (`black`, `white`, `blue`, `lime`, `orange`, `errorColor`, `transparent`, plus `gray5`..`gray90` opacity-based grayscale)
- **`DSSpacing`** — spacing scale (`xs` 4, `s` 2, `m` 16, `l` 24, `xl` 32, `xxl` 48)
- **`DSRadius`** — border-radius scale (`xs`, `s`, `m`, `l`, ...)
- **`DSTypography`** — text styles, accessed by name (`bodyL`, `subtitleM`, `h2`, `subtitleLSemi`, ...) — usually via context extensions in `design_system/ds_extensions.dart` (`context.subtitleM`, `context.h2`)

Feature code MUST NOT hardcode colors (`Color(0xFFABCDEF)`), pixel sizes (`EdgeInsets.all(13)`), or font styles (`TextStyle(fontSize: 14, ...)`). If a value isn't in the design system, raise it with the human before introducing a hardcoded one.

## Rationale

A design system that lives in code and a design system that lives in Figma diverge silently unless there's exactly one place each visual value can come from. Hardcoded values fragment ownership: a junior developer adds `Color(0xFF1A237E)` because it "looks right", a designer changes the brand blue, and that one widget stays at the old colour forever. Tokens make the design-to-code link traceable.

Tokens also enable theme variants (dark mode, brand variations) without touching feature code — the tokens change, the screens follow.

## Implications

- Every feature import list will include `package:<project>/design_system/foundations/ds_colors.dart` (and friends) — that's normal
- Use `context.<style>` extensions for typography (`context.bodyL?.copyWith(color: DSColors.gray60)`) — not `TextStyle(fontSize: 14, ...)` inline
- For colors with opacity, prefer `DSColors.gray60` (semantic grayscale) over `DSColors.black.withOpacity(0.6)` when a matching grayscale token exists
- New visual values get added to the appropriate token class, not hardcoded at the call site. If the design uses a new color/spacing, the token gets added first, then used.
- Higher-level components (in `design_system/components/`) are tokens at a higher level — reuse them instead of rebuilding common shapes (`DSNotificationIcon`, `DSWaveIcon`, etc.)

## When this principle does NOT apply

- Platform-specific values that aren't visual choices: `MediaQuery.of(context).padding.top` (safe area), `kToolbarHeight` (system standard), Material widget defaults exposed through `Theme.of(context)`.
- Animation curves and durations (`Curves.easeInOut`, `Duration(milliseconds: 300)`) — these aren't design tokens in this project; they live next to the animation they configure.
- One-off layout maths (`screen.height * 0.42` to leave room for controls) where the value is purely a relative ratio, not a design choice. If it becomes a recurring ratio, promote it to a token.
