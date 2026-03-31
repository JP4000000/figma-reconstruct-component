---
name: reconstruct-component-figma
description: "Use ONLY when the user wants to CREATE new Figma components from a selected frame using Atomic Design — atoms, molecules, organisms built bottom-up directly on the Figma canvas. Do NOT use for code connect, linking to code, mapping components, publishing to a library, or generating code. Do NOT use for full page or screen generation. ONLY use when user says 'make this a component', 'build this as a component', 'turn this frame into a component', 'componentize this', 'build atoms from this', or 'reconstruct this as a component'."
disable-model-invocation: false
---

# Reconstruct Component from Selection

The user selects a **single Figma frame** and this skill **builds it as a proper Atomic Design component system** — atoms first, then molecules, then the organism. This skill CREATES new `figma.createComponent()` nodes. It does **not** do Code Connect, does **not** require published library components, and does **not** map components to code.

**Zero components is fine.** If the file has no existing components at all, this skill creates everything from scratch. The scan in Step 2 informs reuse — it never blocks progress.

**Bottom-up construction rule**: Always build in this order — Atoms → Molecules → Organism. Never build a higher-level component before its parts exist.

**Component-first rule**: Before creating any primitive, check whether a matching component already exists in the file. If it does, instantiate it. If not, create it.

**Variable-first rule**: Before applying any hardcoded value, check whether a variable or style already encodes it. If none exist, use hardcoded values and log them as token gaps in the annotation.

**MANDATORY**: Load [figma-use](../figma-use/SKILL.md) before any `use_figma` call.

**Always pass `skillNames: "figma-reconstruct-component"` when calling `use_figma`.**

---

## Skill Boundaries

- Use this skill when the user selects a frame and wants it **built as Figma components** using Atomic Design.
- **This skill is NOT Code Connect.** It does not connect components to code, does not require published library components, and does not use `search_design_system` to find published components. It builds new `figma.createComponent()` nodes.
- Valid selections: `FRAME`, `GROUP`, `COMPONENT`, `COMPONENT_SET` — any size up to 1920×1024px.
- If the selection exceeds 1920px wide or 1024px tall, warn the user — that's likely a full page. Ask them to select a discrete section (hero, nav, card, form, etc.).
- **Invalid selection types**: `IMAGE`, bare `RECTANGLE` with only an image fill, `VECTOR`, `SLICE`. Reject with a clear message.
- If the user wants to build a full screen, use [figma-generate-design](../figma-generate-design/SKILL.md).
- If the user wants to generate code from the result, use [figma-implement-design](../figma-implement-design/SKILL.md).

---

## Prerequisites

- Figma MCP server must be connected.
- Exactly one `FRAME`, `GROUP`, `COMPONENT`, or `COMPONENT_SET` must be selected.
- The frame should represent a single, discrete component — not a page layout.

---

## Required Workflow

Follow these steps in order. Do not skip steps.

---

### Step 1: Validate and Inspect the Selected Frame

Validate the selection is a component-sized frame, then read its entire node tree in one pass. This is the only step that reads the source — every subsequent step works from this data.

```js
const sel = figma.currentPage.selection;
if (!sel || sel.length === 0) return { error: "No selection. Please select a frame first." };
if (sel.length > 1) return { error: "Select exactly one frame to reconstruct." };

const node = sel[0];

// Type guard
const validTypes = ["FRAME", "GROUP", "COMPONENT", "COMPONENT_SET"];
if (!validTypes.includes(node.type)) {
  return {
    error: `Selection is a ${node.type}, which is not supported. Please select a FRAME, GROUP, COMPONENT, or COMPONENT_SET.`,
    hint: (node.type === "RECTANGLE" && node.fills?.some(f => f.type === "IMAGE"))
      ? "Looks like a raster image. Place it inside a Frame first, then select the frame."
      : undefined,
  };
}

// Page-size guard
if (node.width > 1920 || node.height > 1024) {
  return {
    warning: `The selected frame is ${Math.round(node.width)}×${Math.round(node.height)}px — this looks like a full page or screen, not a single component. Max supported size is 1920px wide and 1024px tall.`,
    hint: "Select a sub-section of the page (a card, nav bar, form section, etc.) and run the skill again.",
    nodeId: node.id,
  };
}

// Full tree read
function readNode(n) {
  const info = {
    id: n.id,
    name: n.name,
    type: n.type,
    width: Math.round(n.width),
    height: Math.round(n.height),
    x: Math.round(n.x),
    y: Math.round(n.y),
  };

  // Fills — detect image fills for media slot identification
  if ("fills" in n) {
    info.fills = n.fills;
    info.hasImageFill = n.fills?.some(f => f.type === "IMAGE");
  }
  if ("strokes" in n && n.strokes?.length) info.strokes = n.strokes;
  if ("effects" in n && n.effects?.length) info.effects = n.effects;

  // Style IDs — if already linked, carry forward without re-lookup
  if (n.fillStyleId) info.fillStyleId = n.fillStyleId;
  if (n.strokeStyleId) info.strokeStyleId = n.strokeStyleId;
  if (n.effectStyleId) info.effectStyleId = n.effectStyleId;

  // Corner radius
  if ("cornerRadius" in n && n.cornerRadius !== figma.mixed) info.cornerRadius = n.cornerRadius;

  // Auto layout
  if ("layoutMode" in n && n.layoutMode !== "NONE") {
    info.layoutMode = n.layoutMode;
    info.padding = { top: n.paddingTop, right: n.paddingRight, bottom: n.paddingBottom, left: n.paddingLeft };
    info.itemSpacing = n.itemSpacing;
    info.primaryAxisSizingMode = n.primaryAxisSizingMode;
    info.counterAxisSizingMode = n.counterAxisSizingMode;
    info.primaryAxisAlignItems = n.primaryAxisAlignItems;
    info.counterAxisAlignItems = n.counterAxisAlignItems;
  }
  if (n.layoutSizingHorizontal) info.layoutSizingHorizontal = n.layoutSizingHorizontal;
  if (n.layoutSizingVertical) info.layoutSizingVertical = n.layoutSizingVertical;

  // Existing variable bindings — carry forward as-is
  if (n.boundVariables && Object.keys(n.boundVariables).length > 0) {
    info.boundVariables = n.boundVariables;
  }

  // Text
  if (n.type === "TEXT") {
    info.characters = n.characters;
    info.fontSize = n.fontSize;
    info.fontName = n.fontName;
    info.fontWeight = n.fontWeight;
    info.letterSpacing = n.letterSpacing;
    info.lineHeight = n.lineHeight;
    info.textAlignHorizontal = n.textAlignHorizontal;
    if (n.textStyleId) info.textStyleId = n.textStyleId;
  }

  // Existing instances — capture key so we can re-use, not redraw
  if (n.type === "INSTANCE") {
    const mc = n.mainComponent;
    const cs = mc?.parent?.type === "COMPONENT_SET" ? mc.parent : null;
    info.instanceKey = cs?.key ?? mc?.key;
    info.instanceName = cs?.name ?? mc?.name;
    info.variantProperties = n.variantProperties ?? null;
  }

  if ("children" in n) info.children = n.children.map(readNode);
  return info;
}

return { nodeId: node.id, nodeType: node.type, nodeName: node.name, tree: readNode(node) };
```

---

### Step 2: Discover File Resources (Components + Variables)

**Run this before decomposing or building anything.** This step informs reuse — it never blocks progress. If nothing is found, that is expected and fine: proceed to build everything from scratch in Steps 4–6.

> **If the file has no components, no variables, and no styles — that is not an error.** Log `{ components: 0, variables: 0, styles: 0 }` and continue. The skill builds what doesn't exist.

#### 2a. Scan all pages for local components

```js
const originalPage = figma.currentPage;
const localComponents = [];

for (const page of figma.root.children) {
  await figma.setCurrentPageAsync(page);
  page.findAll(n => n.type === "COMPONENT" || n.type === "COMPONENT_SET").forEach(n => {
    localComponents.push({
      id: n.id,
      key: n.key,
      name: n.name,
      type: n.type,
      width: Math.round(n.width),
      height: Math.round(n.height),
      pageName: page.name,
    });
  });
}

await figma.setCurrentPageAsync(originalPage);

// Zero components is fine — log and continue
return {
  count: localComponents.length,
  components: localComponents,
  note: localComponents.length === 0 ? "No existing components found — all atoms will be created from scratch." : undefined,
};
```

#### 2b. Surface any linked library components from existing instances

```js
const instances = figma.currentPage.findAll(n => n.type === "INSTANCE");
const libMap = new Map();
instances.forEach(inst => {
  const mc = inst.mainComponent;
  if (!mc) return;
  const cs = mc.parent?.type === "COMPONENT_SET" ? mc.parent : null;
  const key = cs?.key ?? mc.key;
  const name = cs?.name ?? mc.name;
  if (key && !libMap.has(key)) {
    libMap.set(key, { key, name, isSet: !!cs, sampleVariant: mc.name, width: Math.round(mc.width), height: Math.round(mc.height) });
  }
});

// Zero library components is fine — log and continue
const results = [...libMap.values()];
return {
  count: results.length,
  libraryComponents: results,
  note: results.length === 0 ? "No linked library instances found — proceeding without library reuse." : undefined,
};
```

#### 2c. Discover variables (colors, spacing, radii)

```js
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const vars = await figma.variables.getLocalVariablesAsync();

// Zero variables is fine — hardcoded fallbacks will be used and logged as token gaps
if (vars.length === 0) {
  return { colorVarMap: [], numberVarMap: [], collectionCount: 0, note: "No variables found — all values will be hardcoded and listed as token gaps in the annotation." };
}

const colorVarMap = [];
for (const v of vars) {
  if (v.resolvedType !== "COLOR") continue;
  const col = collections.find(c => c.variableIds.includes(v.id));
  if (!col) continue;
  const val = v.valuesByMode[col.defaultModeId];
  if (val && typeof val === "object" && "r" in val) {
    colorVarMap.push({ varId: v.id, name: v.name, r: val.r, g: val.g, b: val.b });
  }
}

const numberVarMap = vars
  .filter(v => v.resolvedType === "FLOAT")
  .map(v => {
    const col = collections.find(c => c.variableIds.includes(v.id));
    const val = col ? v.valuesByMode[col.defaultModeId] : null;
    return { varId: v.id, name: v.name, value: val };
  });

return { colorVarMap, numberVarMap, collectionCount: collections.length };
```

#### 2d. Discover text styles, paint styles, effect styles

```js
const textStyles = await figma.getLocalTextStylesAsync();
const paintStyles = await figma.getLocalPaintStylesAsync();
const effectStyles = await figma.getLocalEffectStylesAsync();

// Zero styles is fine — manual font loading and hardcoded fills will be used
return {
  textStyles: textStyles.map(s => ({ id: s.id, name: s.name, fontSize: s.fontSize, fontName: s.fontName })),
  paintStyles: paintStyles.map(s => ({ id: s.id, name: s.name })),
  effectStyles: effectStyles.map(s => ({ id: s.id, name: s.name })),
  note: textStyles.length === 0 ? "No text styles found — fonts will be loaded manually from source node data." : undefined,
};
```

**Summary after Step 2:** You now have four maps (may all be empty — that's fine). Use them for reuse lookups in Steps 4–6. Never stop or ask the user to publish components — this skill builds what isn't there.

---

### Step 3: Decompose the Frame into Atomic Parts

Walk the tree from Step 1 and classify every meaningful sub-element into its atomic level. Build a **component manifest** — the list of parts this component is made of.

#### Classification rules

Work **bottom-up**: classify leaf nodes first, then their parents.

| Level        | Criteria                                                                                                     |
|--------------|--------------------------------------------------------------------------------------------------------------|
| **Atom**     | A single indivisible element: one text node, one icon, one shape, one input field, one badge, one avatar     |
| **Molecule** | 2–5 atoms functioning as a unit: label + input, icon + label, avatar + name + role                           |
| **Organism** | Multiple molecules and/or atoms forming a self-contained section: card, nav bar, form, article row           |

#### Identify atoms needed

For each leaf or near-leaf node in the tree, ask:
1. Is it already an `INSTANCE`? → Record its `instanceKey`. **No creation needed.**
2. Does its name or structure match a local/library component? → Record the match key. **No creation needed.**
3. Is it a raw frame/group acting as a discrete reusable element? → **Atom candidate** — needs creation.

Candidate atom signals:
- A small frame (< 200px in both dimensions) containing only one or two children
- A frame named like a component: "Button", "Icon", "Badge", "Tag", "Avatar", "Chip", "Toggle"
- An icon-sized frame (16–48px square) with a vector or image fill
- A text node with a visible background shape (badge, label, tag)
- Any interactive-looking region (button shape, input field outline)

#### Output the manifest

Produce a structured manifest before any creation:

```
Manifest for "ProductCard":
  Atomic level: Organism

  Atoms needed:
    - Atom/Avatar        (48×48, rounded — exists as local component ✓)
    - Atom/Badge         (auto, pill shape — NOT found, must create)
    - Atom/Button        (auto × 36, filled — exists in library ✓)
    - Atom/Icon          (24×24, star — NOT found, must create)

  Molecules needed:
    - Molecule/UserMeta  (Avatar + name + role row — NOT found, must create)
    - Molecule/ActionRow (Button + Icon row — NOT found, must create)

  Slots:
    - Slot/Media         (320×200, image fill → placeholder)
    - Slot/Content       (text body area)

  Token gaps (no variable match):
    - #1A73E8 (primary blue) → no variable found
    - 12px spacing → no variable found
```

Present this manifest to the user for confirmation before building. If the user confirms, proceed.

---

### Step 4: Build Atoms (Bottom-Up, First)

For each atom in the manifest marked **"must create"**, build it as a standalone component **before** building molecules or the organism. Work one atom per `use_figma` call. Return all created IDs.

#### 4a. Check once more before creating

```js
// Final check — search by name in case it was missed
const existing = figma.currentPage.findOne(n =>
  (n.type === "COMPONENT" || n.type === "COMPONENT_SET") &&
  n.name.toLowerCase().includes("badge") // use atom name
);
if (existing) return { alreadyExists: true, id: existing.id, key: existing.key };
```

#### 4b. Create the atom component

```js
const atom = figma.createComponent();
atom.name = "Atom/Badge"; // Atom/ prefix always
atom.layoutMode = "HORIZONTAL";
atom.paddingTop = 4; atom.paddingBottom = 4;
atom.paddingLeft = 8; atom.paddingRight = 8;
atom.itemSpacing = 4;
atom.primaryAxisSizingMode = "HUG";
atom.counterAxisSizingMode = "HUG";
atom.cornerRadius = 99; // pill

// Apply fill — variable-first
// (use applyFill helper from Step 5b, binding variable if found)
atom.fills = [{ type: "SOLID", color: { r: 0.10, g: 0.45, b: 0.96 } }];

// Add text label
await figma.loadFontAsync({ family: "Inter", style: "Medium" });
const label = figma.createText();
label.characters = "Label";
label.fontSize = 12;
label.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
atom.appendChild(label);

// Position away from existing content — find rightmost node
const allNodes = figma.currentPage.children;
const rightmost = allNodes.reduce((max, n) => Math.max(max, n.x + n.width), 0);
atom.x = rightmost + 80;
atom.y = 100;
figma.currentPage.appendChild(atom);

// Expose text as a component property
atom.addComponentProperty("label", "TEXT", "Label");

return { createdNodeIds: [atom.id], key: atom.key, name: atom.name };
```

Repeat this pattern for each missing atom, adjusting shape, padding, and fills to match the source frame's design. Each atom is:
- A `figma.createComponent()` (not a frame)
- Named `Atom/<Name>`
- Built with auto layout
- Has its text/content exposed as component properties
- Variable-bound for colors and spacing where matches exist

Record every created atom's `key` in the manifest — molecules and the organism will import them by key.

---

### Step 5: Build Molecules (Compose from Atoms)

For each molecule in the manifest marked **"must create"**, build it by composing atom instances. One molecule per `use_figma` call.

```js
// Import atoms by key (local or library)
const avatarComp = await figma.importComponentByKeyAsync("ATOM_AVATAR_KEY");
const badgeComp  = await figma.importComponentByKeyAsync("ATOM_BADGE_KEY");

const molecule = figma.createComponent();
molecule.name = "Molecule/UserMeta";
molecule.layoutMode = "HORIZONTAL";
molecule.itemSpacing = 8;
molecule.paddingTop = molecule.paddingBottom = 0;
molecule.paddingLeft = molecule.paddingRight = 0;
molecule.primaryAxisSizingMode = "HUG";
molecule.counterAxisSizingMode = "HUG";
molecule.counterAxisAlignItems = "CENTER";
molecule.fills = [];

// Compose from atoms
const avatarInst = avatarComp.createInstance();
molecule.appendChild(avatarInst);
// FILL sizing must be set AFTER appendChild
avatarInst.layoutSizingHorizontal = "FIXED";

const nameText = figma.createText();
await figma.loadFontAsync({ family: "Inter", style: "SemiBold" });
nameText.characters = "User Name";
nameText.fontSize = 14;
molecule.appendChild(nameText);
nameText.layoutSizingHorizontal = "HUG";

// Position away from canvas content
const rightmost = figma.currentPage.children.reduce((max, n) => Math.max(max, n.x + n.width), 0);
molecule.x = rightmost + 80;
molecule.y = 100;
figma.currentPage.appendChild(molecule);

return { createdNodeIds: [molecule.id], key: molecule.key, name: molecule.name };
```

Rules for molecules:
- Always named `Molecule/<Name>`
- Always composed from atom **instances** — never raw primitives (unless the atom is truly unique and not worth extracting)
- Auto layout always on
- Media slots (image fills detected in Step 3) become `Slot/Media` placeholder frames inside the molecule if the molecule contains a media region

---

### Step 6: Build the Organism (Final Component)

With all atoms and molecules resolved (existing or freshly created), assemble the organism. This is the component that corresponds to the user's selected frame.

```js
const organism = figma.createComponent();
organism.name = "Organism/ProductCard"; // derive from selected frame name
organism.layoutMode = "VERTICAL";
organism.paddingTop = 16; organism.paddingBottom = 16;
organism.paddingLeft = 16; organism.paddingRight = 16;
organism.itemSpacing = 12;
organism.primaryAxisSizingMode = "HUG";
organism.counterAxisSizingMode = "FIXED";
organism.resize(sourceWidth, organism.height); // match source width
organism.cornerRadius = 12; // match source

// Position to the right of source frame (don't overlap it)
const sourceNode = figma.getNodeById(sourceNodeId);
organism.x = sourceNode.x + sourceNode.width + 120;
organism.y = sourceNode.y;
figma.currentPage.appendChild(organism);

// --- Compose from resolved molecules and atoms ---

// 1. Media slot (top of card)
const mediaSlot = figma.createFrame();
mediaSlot.name = "Slot/Media";
mediaSlot.resize(sourceWidth - 32, 200); // match source proportions
mediaSlot.fills = [{ type: "SOLID", color: { r: 0.38, g: 0.38, b: 0.38 } }]; // Gray 500
mediaSlot.cornerRadius = 8;
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
const mediaLabel = figma.createText();
mediaLabel.characters = "□ Media";
mediaLabel.fontSize = 12;
mediaLabel.fills = [{ type: "SOLID", color: { r: 0.90, g: 0.90, b: 0.90 } }];
mediaSlot.layoutMode = "VERTICAL";
mediaSlot.primaryAxisAlignItems = "CENTER";
mediaSlot.counterAxisAlignItems = "CENTER";
mediaSlot.appendChild(mediaLabel);
organism.appendChild(mediaSlot);
mediaSlot.layoutSizingHorizontal = "FILL";

// 2. Molecule instances
const userMetaComp = await figma.importComponentByKeyAsync("MOLECULE_USERMETA_KEY");
const userMetaInst = userMetaComp.createInstance();
organism.appendChild(userMetaInst);
userMetaInst.layoutSizingHorizontal = "FILL";

const actionRowComp = await figma.importComponentByKeyAsync("MOLECULE_ACTIONROW_KEY");
const actionInst = actionRowComp.createInstance();
organism.appendChild(actionInst);
actionInst.layoutSizingHorizontal = "FILL";

return { createdNodeIds: [organism.id], key: organism.key, name: organism.name };
```

Rules for the organism:
- Named `Organism/<Name>` (derived from the selected frame's name)
- Composed **only** from molecule and atom instances — no raw primitives except `Slot/*` placeholder frames for media
- Dimensions match the source frame (width exact; height HUG unless source is fixed)
- Positioned to the **right** of the source frame, not overlapping it

---

### Step 7: Apply Variables and Tokens

After the component structure is assembled, bind variables and styles. One pass per `use_figma` call.

#### Variable-first color application

```js
function findColorVar(r, g, b, colorVarMap, tol = 0.02) {
  return colorVarMap.find(v =>
    Math.abs(v.r - r) < tol && Math.abs(v.g - g) < tol && Math.abs(v.b - b) < tol
  );
}

async function applyFill(node, r, g, b, opacity = 1, colorVarMap) {
  const match = findColorVar(r, g, b, colorVarMap);
  if (match) {
    const variable = await figma.variables.getVariableByIdAsync(match.varId);
    const boundPaint = figma.variables.setBoundVariableForPaint(
      { type: "SOLID", color: { r, g, b }, opacity },
      "color",
      variable
    );
    node.fills = [boundPaint]; // returns NEW paint — must reassign
  } else {
    node.fills = [{ type: "SOLID", color: { r, g, b }, opacity }];
    // log as unmatched token for the annotation
  }
}
```

#### Spacing variable binding

```js
// Bind padding/gap to number variables when matched
const gapVar = numberVarMap.find(v => v.value === node.itemSpacing);
if (gapVar) {
  const variable = await figma.variables.getVariableByIdAsync(gapVar.varId);
  node.setBoundVariable("itemSpacing", variable);
}
```

#### Carry forward existing bindings from source

If a source node already had `boundVariables`, re-apply those same variable IDs to the corresponding new nodes — no lookup needed:

```js
// Source node had: boundVariables: { fills: [{ type: "VARIABLE_ALIAS", id: "VariableID:123:456" }] }
const sourceVar = await figma.variables.getVariableByIdAsync("VariableID:123:456");
const boundPaint = figma.variables.setBoundVariableForPaint(
  { type: "SOLID", color: { r, g, b } },
  "color",
  sourceVar
);
newNode.fills = [boundPaint];
```

Track every color/spacing value that **had no variable match** — these become the "token gaps" section of the annotation.

---

### Step 8: Enforce WCAG AA Compliance

Run this audit on the completed organism. **Do not skip.**

#### Contrast ratios (WCAG 2.1 AA)
- Normal text (< 18pt regular, < 14pt bold): ≥ **4.5:1**
- Large text (≥ 18pt regular or ≥ 14pt bold): ≥ **3:1**
- UI components and icons: ≥ **3:1** against adjacent background

```js
function luminance({ r, g, b }) {
  const lin = c => c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
  return 0.2126 * lin(r) + 0.7152 * lin(g) + 0.0722 * lin(b);
}
function contrast(c1, c2) {
  const [l, d] = [luminance(c1), luminance(c2)].sort((a, b) => b - a);
  return (l + 0.05) / (d + 0.05);
}

function auditTree(node, parentBg = { r: 1, g: 1, b: 1 }) {
  const results = [];
  const bg = node.fills?.[0]?.type === "SOLID" ? node.fills[0].color : parentBg;
  if (node.type === "TEXT") {
    const fg = node.fills?.[0]?.color;
    if (fg) {
      const ratio = contrast(fg, bg);
      const isLarge = node.fontSize >= 18 || (node.fontSize >= 14 && node.fontWeight >= 700);
      const required = isLarge ? 3.0 : 4.5;
      results.push({ id: node.id, name: node.name, ratio: Math.round(ratio * 100) / 100, required, pass: ratio >= required });
    }
  }
  if ("children" in node) node.children.forEach(c => results.push(...auditTree(c, bg)));
  return results;
}

const audits = auditTree(figma.getNodeById(organismId));
return { total: audits.length, failures: audits.filter(a => !a.pass) };
```

Fix any failures by adjusting text color toward black or white until the ratio passes. Also check:
- **Touch targets**: interactive elements ≥ 44×44px
- **Text size**: no body text below 12px
- **Non-color cues**: state (error, success) must have icon or label, not color alone

---

### Step 9: Generate the Component Annotation

Place a styled annotation frame to the **right** of the new organism documenting its full atomic composition. This serves as inline documentation on the Figma canvas.

```js
const organism = figma.getNodeById(organismId);

// Position annotation to the right of the organism with a gap
const annotX = organism.x + organism.width + 48;
const annotY = organism.y;

// Outer annotation frame
const annot = figma.createFrame();
annot.name = `_Annotation / ${organism.name}`;
annot.layoutMode = "VERTICAL";
annot.itemSpacing = 16;
annot.paddingTop = 24; annot.paddingBottom = 24;
annot.paddingLeft = 24; annot.paddingRight = 24;
annot.primaryAxisSizingMode = "HUG";
annot.counterAxisSizingMode = "FIXED";
annot.resize(320, annot.height);
annot.fills = [{ type: "SOLID", color: { r: 0.97, g: 0.97, b: 1.0 } }]; // soft lavender-white
annot.strokes = [{ type: "SOLID", color: { r: 0.78, g: 0.78, b: 0.98 } }];
annot.strokeWeight = 1.5;
annot.cornerRadius = 12;
annot.x = annotX;
annot.y = annotY;
figma.currentPage.appendChild(annot);

await figma.loadFontAsync({ family: "Inter", style: "Bold" });
await figma.loadFontAsync({ family: "Inter", style: "SemiBold" });
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Medium" });

// Helper: create a text node and append it
function addText(parent, content, size, style, color = { r: 0.1, g: 0.1, b: 0.1 }) {
  const t = figma.createText();
  t.fontName = { family: "Inter", style };
  t.characters = content;
  t.fontSize = size;
  t.fills = [{ type: "SOLID", color }];
  parent.appendChild(t);
  t.layoutSizingHorizontal = "FILL";
  return t;
}

// Helper: pill badge for atomic level
function addBadge(parent, label, bgColor) {
  const badge = figma.createFrame();
  badge.layoutMode = "HORIZONTAL";
  badge.paddingTop = badge.paddingBottom = 3;
  badge.paddingLeft = badge.paddingRight = 10;
  badge.primaryAxisSizingMode = "HUG";
  badge.counterAxisSizingMode = "HUG";
  badge.cornerRadius = 99;
  badge.fills = [{ type: "SOLID", color: bgColor }];
  const t = figma.createText();
  t.fontName = { family: "Inter", style: "Medium" };
  t.characters = label;
  t.fontSize = 11;
  t.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
  badge.appendChild(t);
  parent.appendChild(badge);
  return badge;
}

// --- Section: Component name + level ---
addText(annot, organism.name.replace("Organism/", "").replace("Molecule/", "").replace("Atom/", ""), 18, "Bold");
addBadge(annot, "Organism", { r: 0.38, g: 0.27, b: 0.90 }); // purple for organism

// Divider (thin frame)
const div1 = figma.createFrame();
div1.resize(272, 1);
div1.fills = [{ type: "SOLID", color: { r: 0.85, g: 0.85, b: 0.95 } }];
annot.appendChild(div1);

// --- Section: Molecules ---
if (moleculeList.length > 0) {
  addText(annot, "Molecules", 12, "SemiBold", { r: 0.4, g: 0.4, b: 0.55 });
  for (const mol of moleculeList) {
    addText(annot, `  · ${mol.name}`, 13, "Regular");
    for (const atom of mol.atoms) {
      addText(annot, `      ↳ ${atom}`, 12, "Regular", { r: 0.5, g: 0.5, b: 0.5 });
    }
  }
}

// --- Section: Atoms ---
if (atomList.length > 0) {
  addText(annot, "Atoms", 12, "SemiBold", { r: 0.4, g: 0.4, b: 0.55 });
  for (const atom of atomList) {
    const status = atom.wasCreated ? "  · " + atom.name + "  ✦ new" : "  · " + atom.name;
    addText(annot, status, 13, "Regular");
  }
}

// --- Section: Slots ---
if (slotList.length > 0) {
  addText(annot, "Slots", 12, "SemiBold", { r: 0.4, g: 0.4, b: 0.55 });
  for (const slot of slotList) {
    addText(annot, `  · ${slot.name}  (${slot.type})`, 13, "Regular");
  }
}

// --- Section: Token gaps ---
if (tokenGaps.length > 0) {
  addText(annot, "Token gaps — no variable found", 12, "SemiBold", { r: 0.85, g: 0.35, b: 0.15 });
  for (const gap of tokenGaps) {
    addText(annot, `  · ${gap.property}: ${gap.value}`, 12, "Regular", { r: 0.55, g: 0.35, b: 0.15 });
  }
}

// --- Section: WCAG ---
const wcagStatus = wcagFailures.length === 0 ? "✓ All contrast checks pass" : `⚠ ${wcagFailures.length} contrast issue(s) — see canvas`;
addText(annot, wcagStatus, 12, "Regular", wcagFailures.length === 0 ? { r: 0.15, g: 0.6, b: 0.3 } : { r: 0.8, g: 0.2, b: 0.2 });

return { annotationId: annot.id, createdNodeIds: [annot.id] };
```

The annotation stays on the canvas as living documentation. Name it `_Annotation / <ComponentName>` so it sorts separately from design nodes.

---

### Step 10: Validate

Take a screenshot and do a final structure check before reporting back to the user.

```js
const organism = figma.getNodeById(organismId);
return {
  name: organism.name,
  atomicLevel: "Organism",
  width: organism.width,
  height: organism.height,
  layoutMode: organism.layoutMode,
  directChildren: organism.children.length,
  componentProperties: Object.keys(organism.componentPropertyDefinitions || {}),
  annotationId: annotationNodeId,
};
```

Use `get_screenshot` targeting the organism ID to visually confirm it matches the source frame.

Report to the user:
- Component name + Atomic level
- Atoms created (new ✦) vs. reused from file/library
- Molecules created vs. reused
- Variables bound vs. token gaps (with hex values for gaps)
- WCAG result (pass count / fix count)
- Annotation placed ✓

---

## Atomic Design Reference

| Level        | Name prefix     | Built from                        | Examples                              |
|--------------|-----------------|-----------------------------------|---------------------------------------|
| **Atom**     | `Atom/`         | Primitives only                   | Button, Icon, Badge, Tag, Avatar, Input |
| **Molecule** | `Molecule/`     | Atom instances                    | SearchBar, FormField, NavItem, CardHeader |
| **Organism** | `Organism/`     | Molecule + Atom instances         | ProductCard, NavBar, ArticleRow, Form  |
| **Template** | `Template/`     | Organism instances (layout shell) | PageShell, DashboardGrid              |

---

## Slot Naming Conventions

| Slot type      | Frame name        | Example                    |
|----------------|-------------------|----------------------------|
| Image / media  | `Slot/Media`      | `Slot/Media - Hero`        |
| Video          | `Slot/Video`      | `Slot/Video - Preview`     |
| Icon           | `Slot/Icon`       | `Slot/Icon - Leading`      |
| Text / content | `Slot/Content`    | `Slot/Content - Body`      |
| Avatar         | `Slot/Avatar`     | `Slot/Avatar - User`       |

---

## Media Placeholder Spec

| Property        | Value                                              |
|-----------------|----------------------------------------------------|
| Fill color      | `#616161` — Gray 500 (`r: 0.38 g: 0.38 b: 0.38`) |
| Label text      | `□ Media` (image) or `▶ Video`                    |
| Label color     | `#E5E5E5`                                          |
| Label font      | Inter Regular, 12px                                |
| Corner radius   | Match source radius, minimum 4px                   |
| Contrast ratio  | ≥ 4.5:1 always (label on fill)                    |

---

## Design Fidelity Rules

### Color — priority order
1. **Source had `boundVariables`** → carry the same variable ID forward unchanged.
2. **Matched variable by RGB value** (within 2% tolerance) → bind via `setBoundVariableForPaint`.
3. **Matched paint style** → apply via `fillStyleId`.
4. **Hardcoded raw color** → log as a token gap in the annotation.

### Typography — priority order
1. **Source had `textStyleId`** → carry it forward unchanged.
2. **Matched local/linked text style** (family + size + weight) → apply via `textStyleId`.
3. **Manual construction** → `loadFontAsync` + set properties individually.

### Components — priority order
1. **Source child is already an `INSTANCE`** → re-instantiate by `instanceKey`.
2. **Local component name match** → `importComponentByKeyAsync`.
3. **Linked library match** (surfaced from canvas instances) → `importComponentByKeyAsync`.
4. **Build from primitives** → only for Atoms with no match. Always make it a `figma.createComponent()`.

---

## Error Handling

| Situation                                               | Response                                                                                              |
|---------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| No selection                                            | "Select a single frame first."                                                                        |
| Multiple nodes selected                                 | "Select exactly one frame."                                                                           |
| Selection is an IMAGE, VECTOR, SLICE, or TEXT           | Return error with type; suggest placing inside a Frame first                                          |
| Selection is a bare RECTANGLE with only an image fill   | "Looks like a raster image — place it inside a Frame, then select the frame."                        |
| Selection is page-sized (> 1920px wide or > 1024px tall)| Warn; ask user to select a discrete section (hero, nav, card, form, etc.)                            |
| Selection is already a COMPONENT / COMPONENT_SET        | Confirm before overwriting; offer to create a new variant instead                                     |
| Selection is a library INSTANCE                         | Report source component name/key; offer to wrap it in a new organism                                  |
| **No components found in the file**                     | **Expected and fine. Log it, proceed to create all atoms from scratch in Step 4.**                    |
| **"No published components" message appears**           | **Ignore it. This skill does not use Code Connect or published components. Keep going.**              |
| No variables in file                                    | Proceed with hardcoded values; list all as token gaps in the annotation                               |
| No text styles in file                                  | Load fonts manually from source node `fontName` data; continue                                        |
| Atom already exists under a different name              | Ask user to confirm match before reusing — show both names                                            |
| Component import fails (library not connected)          | Fall back to creating the atom from primitives; warn user                                             |
| Font unavailable                                        | Fall back to Inter at same size/weight; note the substitution                                         |
| Contrast cannot be fixed automatically                  | Flag in annotation with failing ratio; suggest accessible alternative                                 |
| Node too complex (> 200 children)                       | Split into multiple `use_figma` calls, one atomic level at a time                                     |
