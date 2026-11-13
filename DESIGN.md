# Design System: Monochrome Minimalist Ledger

**Archetype**: Swiss High-Contrast Monochrome / Technical Ledger  
**Mode**: Strict Light Mode Only  
**Philosophy**: Stark, functional, ultra-minimalist, editorial precision. Zero superfluous color, zero synthetic gradients, zero generic AI slop.

---

## 1. Color Palette (Monochrome Only)

- **Canvas (Background)**: `#ffffff` (Pure Stark White) / `#fafafa` (Neutral Surface)
- **Cards / Containers**: `#ffffff` with hairline `#e5e5e5` borders
- **Ink Primary**: `#0a0a0a` (Deep Carbon Black)
- **Ink Secondary**: `#404040` (Neutral Slate)
- **Ink Muted / Captions**: `#737373` (Mid Neutral)
- **Borders & Dividers**: `#e5e5e5` (Hairline Neutral) / `#d4d4d4` (Active Border)
- **Action Solid**: `#000000` (Pure Black, white text `#ffffff`)
- **Action Hover**: `#262626` (Off-Black)

---

## 2. Typography

- **Display & Headings**: Clean sans-serif / geometric modern typography (`system-ui, -apple-system, sans-serif`) with bold weights and tight tracking (`tracking-tight`).
- **Body Text**: High legibility sans-serif with generous line-height (`leading-relaxed`).
- **Data, Figures, ISBNs, & Codes**: Monospace (`font-mono`) with `tabular-nums` for strict column alignment in financial and inventory ledgers.

---

## 3. Component Standards

- **Buttons**:
  - Primary: `bg-black text-white hover:bg-neutral-800 text-xs font-mono uppercase tracking-wider rounded-sm`
  - Secondary / Outline: `bg-white text-black border border-neutral-300 hover:border-black text-xs font-mono uppercase tracking-wider rounded-sm`
- **Badges & Tags**:
  - Minimal geometric pills or brackets (`[CRITICAL]`, `[HIGH]`, `[OPEN]`).
  - Critical: `bg-black text-white font-mono text-[10px]`
  - Normal / Info: `border border-neutral-300 text-neutral-800 font-mono text-[10px]`
- **Ledger Cards**:
  - Border: `1px solid #e5e5e5`
  - Corner radius: `rounded-lg` or `rounded-sm` (crisp, architectural)
  - Shadow: None or ultra-fine `shadow-xs` (no heavy blur)
