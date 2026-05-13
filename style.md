# TSWIM 2026 Website Style Guide

## Color Palette

| Role | Variable | Value | Description |
|---|---|---|---|
| Background | `--background-color` | `#ffffff` | Main page background |
| Default text | `--default-color` | `#000000` | Body text |
| Heading | `--heading-color` | `#0b5394` | All headings and titles |
| Accent | `--accent-color` | `#f6b26b` | Buttons, links, hover states, underlines |
| Surface | `--surface-color` | `#f3f6f4` | Card backgrounds, alternating sections |
| Contrast | `--contrast-color` | `#ffffff` | Text on dark/accent backgrounds |

### Dark Background Preset (`.dark-background`)

Used by: hero section, footer.

| Variable | Value |
|---|---|
| `--background-color` | `#073763` (navy blue) |
| `--surface-color` | `#0b5394` |
| `--default-color` | `#ffffff` |
| `--heading-color` | `#ffffff` |

### Light Background Preset (`.light-background`)

Used by: program, committee, sponsors sections.

| Variable | Value |
|---|---|
| `--background-color` | `#f3f6f4` (light grey-green) |
| `--surface-color` | `#ffffff` |

### Additional Colors

| Context | Value | Description |
|---|---|---|
| Scrolled header bg | `#0e1d34` | Very dark navy on scroll |
| Button fill | `#f4a65f` | `.btn-accent` solid buttons |
| Button fill hover | `#ee9a4c` | Darker orange on hover |
| Price text | `#d4822e` | Burnt orange for pricing |
| Card soft bg | `#fffbf6` | `.note-soft` warm white |

### Nav Colors

| Variable | Value |
|---|---|
| `--nav-color` | `#ffffff` (links on dark header) |
| `--nav-hover-color` | `#f6b26b` (orange on hover/active) |
| `--nav-mobile-background-color` | `#ffffff` |
| `--nav-dropdown-background-color` | `#ffffff` |
| `--nav-dropdown-color` | `#000000` |
| `--nav-dropdown-hover-color` | `#f6b26b` |

**Theme summary:** Navy blue + warm orange/gold on a clean white base. Conveys trust (blue) and warmth (orange) -- fitting for an academic conference.

## Typography

| Role | Font Family | CSS Variable |
|---|---|---|
| Body | **Roboto** | `--default-font` |
| Headings | **Poppins** | `--heading-font` |
| Navigation | **Poppins** | `--nav-font` |

Both loaded from Google Fonts with full weight range (100-900).

### Font Sizes

| Element | Size | Weight |
|---|---|---|
| Hero title (h2) | 48px | 700 |
| Hero subtitle (p) | 24px | -- |
| Section title (h2) | 32px | 700 |
| Nav links (desktop) | 15px | 400 |
| Nav links (mobile) | 17px | 500 |
| Footer | 14px | -- |

## Layout

- **Framework:** Bootstrap 5 (grid, utilities, responsive breakpoints)
- **Single-page layout** with anchor navigation
- **Container:** `container-fluid container-xl` in the header; `container` in content sections
- **Sections** alternate white and light grey-green backgrounds
- **Section padding:** `40px 0`
- **Scroll margin top:** `100px` (desktop), `66px` (< 1200px)

### Responsive Breakpoints

| Breakpoint | Behavior |
|---|---|
| >= 1200px | Desktop nav (horizontal menu) |
| < 1200px | Mobile nav (hamburger + overlay) |
| >= 992px | Committee two-column grid |
| < 960px | Buttons go full-width (`.btn-mobile-full`) |

## Components

### Header

- Fixed top, transparent initially
- On scroll: shrinks padding (20px -> 10px), gains box-shadow, bg turns `#0e1d34`
- Logo shrinks from 80px to 40px on scroll
- Active nav link: orange underline animation + color change

### Hero Section

- Min height: `55vh`
- Full-bleed background image with 70% navy overlay (`color-mix(in srgb, var(--background-color), transparent 30%)`)
- Centered text content (z-index above overlay)

### Cards

- Border radius: `12-14px`
- Box shadow: `0 2px 10px rgba(0,0,0,0.06)`
- Invitee/speaker cards: flex row with 140x140 rounded image + info column

### Buttons

- **Accent fill** (`.btn-accent`): `#f4a65f` bg, white text, rounded pill
- **Accent outline** (`.btn-accent-outline`): transparent bg, `#f4a65f` 2px border, orange text
- Hover: slightly darker orange (`#ee9a4c`)

### Section Titles

- Centered, 32px bold Poppins
- 50px-wide, 3px-tall orange underline bar (centered via `::after`)

### Timeline (Important Dates)

- Vertical timeline with Bootstrap Icon markers
- Each item: icon + heading + date text

### Preloader

- Full-screen overlay with expanding concentric orange rings animation

### Scroll-to-Top

- Fixed bottom-right, 40x40px, accent orange bg, 4px border-radius

## Icons

- **Bootstrap Icons** (`bootstrap-icons.css`) -- deferred load for performance
- Used in: nav dropdowns, timeline markers, session format headers, contact, buttons

## Files

| File | Purpose |
|---|---|
| `assets/css/main.css` | All global styles and component styles |
| `assets/css/schedule.css` | Program schedule table (scoped under `.program-schedule`) |
| `assets/js/main.js` | Scroll behavior, nav toggling, preloader |
| `assets/vendor/bootstrap/` | Bootstrap 5 CSS + JS |
