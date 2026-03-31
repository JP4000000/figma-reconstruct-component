# reconstruct-component-figma

A skill that takes a selected Figma frame and rebuilds it as a proper **Atomic Design component system** — directly on the Figma canvas. Atoms first, then molecules, then the organism. No Code Connect. No published library required.

---

## What It Does

You select a single frame in Figma. Claude analyzes its structure, decomposes it into atomic parts, and builds each level as real `figma.createComponent()` nodes — bottom-up.

- **Atoms** — indivisible elements: buttons, icons, badges, avatars, inputs
- **Molecules** — small groups of atoms: search bars, form fields, nav items
- **Organism** — the final assembled component matching your selected frame

Along the way it:
- Reuses existing local or library components where they match
- Binds variables and styles where they exist, logs gaps where they don't
- Replaces image fills with properly labeled media placeholder slots
- Runs WCAG contrast checks on all text/background pairs
- Places a living annotation on the canvas documenting everything it built

---

## Atomic Design — Quick Reference

| Level | Prefix | Built From | Examples |
|---|---|---|---|
| Atom | `Atom/` | Primitives only | Button, Icon, Badge, Avatar, Input |
| Molecule | `Molecule/` | Atom instances | SearchBar, FormField, NavItem |
| Organism | `Organism/` | Molecules + Atoms | ProductCard, NavBar, ArticleRow |

---

## Prerequisites

- A Figma file open with at least one frame selected
- The **Figma MCP server** connected to Claude (see below)
- A Figma personal access token — get one from **Figma → Account Settings → Personal Access Tokens**

> This skill uses the **Figma MCP server**, not a Figma plugin. Claude reads and writes directly to your Figma file through the MCP connection — no plugin panel or Figma Community install required.

---

## Connecting the Figma MCP Server

### Using Claude.ai (browser)

1. Go to **Claude.ai → Settings → Integrations**
2. Find Figma and click **Connect**
3. Authorize with your Figma account
4. The Figma MCP server will be available in all your Claude conversations

### Using Claude Code (terminal)

Add the Figma MCP server to your Claude config file (`~/.claude.json` or `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "figma": {
      "url": "https://mcp.figma.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_FIGMA_ACCESS_TOKEN"
      }
    }
  }
}
```

Replace `YOUR_FIGMA_ACCESS_TOKEN` with your personal access token. Store it in `.component-contracts` (gitignored) and reference it from there — never hardcode it in the config directly.

---

## Installation

1. **Clone the repo**

```bash
git clone https://github.com/your-username/figma-reconstruct-component.git
cd figma-reconstruct-component
```

2. **Store your Figma access token**

Create a `.component-contracts` file in the root (gitignored — never commit this):

```
FIGMA_ACCESS_TOKEN=your_token_here
```

3. **Add the skill to Claude**

Place the `SKILL.md` file in your Claude skills directory:

```
/mnt/skills/public/figma-reconstruct-component/SKILL.md
```

---

## How to Trigger It

In Claude, with a Figma frame selected, use any of these phrases:

- `"Make this a component"`
- `"Build this as a component"`
- `"Turn this frame into a component"`
- `"Componentize this"`
- `"Build atoms from this"`
- `"Reconstruct this as a component"`

Claude will validate your selection, present a component manifest for confirmation, then build everything bottom-up on the canvas.

---

## What to Select

| Supported | Not Supported |
|---|---|
| `FRAME` | `IMAGE` |
| `GROUP` | `VECTOR` |
| `COMPONENT` | `SLICE` |
| `COMPONENT_SET` | Bare `RECTANGLE` with image fill only |

**Size limit:** frames larger than 1920×1024px are likely full pages — select a discrete section instead (a card, nav bar, form, hero, etc.).

---

## What Gets Built

For a frame like a `ProductCard`, Claude will produce:

```
Atom/Avatar         ← new ✦
Atom/Badge          ← reused from file ✓
Atom/Button         ← reused from library ✓
Atom/Icon           ← new ✦
Molecule/UserMeta   ← new ✦ (Avatar + name)
Molecule/ActionRow  ← new ✦ (Button + Icon)
Organism/ProductCard ← final assembled component
_Annotation / ProductCard ← canvas documentation
```

---

## Skill Boundaries

This skill **only** builds Figma components. It does not:

- Connect components to code (that's Code Connect — use `figma-implement-design` for that)
- Generate full page layouts (use `figma-generate-design` for that)
- Require a published component library

---

## Contributing

Contributions are welcome! Here's how to get started:

1. Fork the repo
2. Create a branch: `git checkout -b feature/your-improvement`
3. Make your changes to `SKILL.md`
4. Test against at least two different frame types (simple and complex)
5. Open a pull request with a clear description of what changed and why

### Guidelines

- Keep the skill boundary clear — this skill builds components, nothing else
- Follow the existing step numbering and structure in `SKILL.md`
- If adding new error handling cases, add them to the error table at the bottom of the skill
- Do not commit `.component-contracts`, `scripts/`, or any generated artifacts

---

## License

MIT
