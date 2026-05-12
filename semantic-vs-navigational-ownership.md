---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [navigation, go-router, stateful-shell-route, branches, user-flow, information-architecture, back-stack]
related: [[layered-architecture]], [[adr-004-go-router-stateful-shell]]
---

# Semantic vs. navigational ownership

## Rule

Navigation structure follows the user's flow — *where they came from and where the back button should take them* — not the content's domain category.

When deciding which tab/branch should "own" a screen, ask: from which existing screen do users naturally arrive here? That screen's branch is the right home, even if the new screen's content technically belongs to a different domain.

## Rationale

Information architecture and navigation architecture are different things. A news article is content-wise news, but if users open it from the Home feed, it belongs to the Home branch — because pressing back should return to Home, not jump to an out-of-context "News" tab. Treating navigation as a mirror of content taxonomy produces apps where the back button keeps surprising users.

The user's mental model is *"I tapped here, so back goes there."* The wiki side of this principle: when introducing a new screen, ask the user-flow question first, the domain-category question only if user flow is ambiguous.

## Implications

In `go_sport`, three `StatefulShellBranch`es own routes by user-flow ancestry, not by content category:

- **Home branch** (root path `/`) — owns the Home dashboard plus **news routes** nested as `path: 'news'` and `path: 'news/:id'` under `/`. The URLs come out as `/news` and `/news/:id`, but the routes are nested inside the Home branch's route tree, so back-navigation stays in Home. News is its own content domain, but users enter it from Home's feed.
- **Music branch** (`/music`) — owns the music dashboard plus all library screens: `/music/myfavorites`, `/music/myplaylists`, `/music/myalbums`, `/music/myartists`, `/music/myepisodes`, `/music/myprograms`. Plus `/music/playlist/:id`, `/music/album/:id`, `/music/artist/:id`, `/music/program/:id`. Library screens are *not* in a separate "Library" tab — users get there from Music, so back returns to Music.
- **Radio branch** (`/radio`) — owns the radio screen plus `/radio/schedule`.

When adding a new screen:

- Identify the screen(s) from which users will reach it
- Find the branch those source screens are in
- Nest the new route under that branch — even if the new content belongs to a different content category

Global routes (outside any branch) are reserved for cross-tab flows: auth screens (`/login`, `/registration-*`, `/confirm-*`), profile (`/profile`), expired-guest (`/expired-guest`), password reset (`/forgot-password`, `/check-email`, `/new-password`, `/password-changed`). These don't have a "home tab" — they interrupt whatever the user was doing.

## When this principle does NOT apply

- Truly cross-cutting flows that must work identically from every tab (settings, profile, sign-in/sign-out, error pages) → global route, not under any branch.
- Apps with a single navigation stack (no branches, no tabs) — the principle is about which *branch* owns a route, and degenerates when there's only one branch.
- When user-flow analysis genuinely points to two source branches with roughly equal traffic — that's a sign the screen should be a global route accessible from both, not duplicated.
