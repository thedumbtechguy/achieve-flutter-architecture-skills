# Achieve Flutter Skills

Claude Code skills for building Flutter apps following the Achieve architecture — battle-tested across 6 years of production fintech, 30+ features, 80+ models, 200+ API endpoints.

## Skills

### [achieve-flutter-architecture](achieve-flutter-architecture/SKILL.md)

The core architecture skill. Teaches the pattern: **Model -> Repository -> Screen -> Route.** Screens don't catch errors, manage loading spinners, or know auth tokens exist. The architecture handles all of it.

Key patterns:
- **Repository + Caching Mixin** — 3-tier caching (persistent, secure, ephemeral) with automatic error wrapping
- **DataPage** — Base class for screens. Implement 3 methods, get loading/error/refresh for free. Auto-reloads on focus via FocusDetector
- **OperationRunnerState** — User-triggered actions with automatic overlay, error dialogs, and analytics
- **OverlayManager** — App-level loading overlay that survives screen disposal
- **EventBus** — Typed pub/sub for cross-feature side effects (PageReloaded, auth state, maintenance mode)
- **GetIt** — Service locator DI, accessible outside the widget tree

Files:
- `SKILL.md` — Patterns, principles, feature checklist, common mistakes
- `achieve-flutter-reference.md` — Complete code examples for every pattern (14 sections)

## Setup

Symlink into Claude Code's skills directory:

```bash
ln -s /path/to/achieve-flutter-skills/achieve-flutter-architecture ~/.claude/skills/achieve-flutter-architecture
```

The skill will be available in all Claude Code sessions.

## Usage

The skill triggers automatically when Claude Code detects Flutter architecture work. You can also reference it explicitly:

```
Follow the achieve-flutter-architecture skill when building this feature.
```

## Origin

Architecture developed at [achieve by Petra](https://theachieveapp.com). See the companion blog post: [A Flutter Architecture for Production Apps](https://thedumbtechguy.hashnode.dev/a-flutter-architecture-for-production-apps).
