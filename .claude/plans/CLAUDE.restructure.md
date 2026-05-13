# Website Restructuring Plan: Single-Page to Multi-Page

## Current State

The website is currently a **single-page application** with all content in `index.html`. Navigation uses anchor links (`#section-id`) to scroll to sections.

### Current Sections in `index.html`:
| Section ID | Description | Nav Location |
|------------|-------------|--------------|
| `#hero` | Conference banner (title, location, dates) | - |
| `#information` | Theme description + About TSWIM | Home |
| `#important-dates` | Timeline of key dates | Top nav |
| `#call-for-papers` | Paper types, session formats, why submit | Top nav |
| `#submission` | Submit button (TBA) | Within Call for Papers |
| `#registration` | Register button (TBA) | Top nav |
| `#keynote-speakers` | 3 keynote speakers with photos | Dropdown |
| `#international-advisors` | Hidden advisors (TBA) | - |
| `#program` | Program schedule (TBA) | Dropdown |
| `#committee` | 6 committee categories | Dropdown |
| `#travel-information` | Address, map, parking | Dropdown |
| `#contact` | Email contact box | Dropdown |
| `#sponsors` | 4 sponsor logos | Top nav |

---

## Proposed Multi-Page Structure

### Option A: Section-Based Pages (Recommended)
Each major section becomes its own page:

```
index.html              → Home (Hero + Theme/Description only)
important-dates.html    → Important Dates
call-for-papers.html    → Call for Papers + Submission
registration.html       → Registration
keynote-speakers.html   → Keynote Speakers
program.html            → Program
committee.html          → Committee
travel.html             → Travel Information
contact.html            → Contact
sponsors.html           → Sponsors
```

### Option B: Consolidated Pages
Group related content into fewer pages:

```
index.html              → Home (Hero + Theme/Description)
submission.html         → Important Dates + Call for Papers + Registration
speakers.html           → Keynote Speakers + International Advisors
program.html            → Program Schedule
about.html              → Committee + Contact + Sponsors
travel.html             → Travel Information
```

---

## Questions to Discuss

### 1. Page Granularity
**Which option do you prefer?**
- **Option A**: More pages, each focused on one topic (easier navigation, cleaner URLs)
- **Option B**: Fewer pages, grouped by theme (less clicking, but longer pages)
- **Custom**: Different grouping?

### 2. Home Page Content
What should remain on the front page besides the hero banner?

**Currently planned to keep:**
- Hero section (conference name, location, dates)
- Theme description ("Innovating with Care...")
- About TSWIM callout

**Should we also include:**
- [ ] Important Dates timeline (for quick visibility)?
- [ ] Quick links / "Featured" cards to key pages?
- [ ] Keynote speaker preview (just names/photos)?
- [ ] Sponsors section (often kept on home page)?

### 3. Navigation Structure
**Current dropdown menu "General Information"** contains:
- Keynote Speakers, Program, Committee, Travel, Contact

**Options:**
- Keep dropdown structure but link to pages
- Flatten all items to top-level nav
- Different grouping?

### 4. International Advisors
Currently hidden with `d-none`. Should this:
- Become a separate page?
- Be merged into Keynote Speakers page?
- Remain hidden until content is ready?

### 5. Shared Components
How to handle repeated elements (header, footer, CSS):
- **Option A**: Duplicate in each HTML file (simpler, but maintenance overhead)
- **Option B**: Use a static site generator (Jekyll, Hugo, etc.)
- **Option C**: JavaScript includes (less ideal for SEO)

For GitHub Pages, Option A (duplication) is simplest unless you want to set up Jekyll.

---

## Technical Considerations

### File Structure (Proposed)
```
/
├── index.html              # Home page
├── important-dates.html
├── call-for-papers.html
├── registration.html
├── keynote-speakers.html
├── program.html
├── committee.html
├── travel.html
├── contact.html
├── sponsors.html
├── assets/
│   ├── css/main.css
│   ├── js/main.js
│   └── img/...
```

### Navigation Updates Required
- Change all `href="#section"` to `href="page.html"`
- Update active state logic in `main.js`
- Ensure mobile menu works correctly

### CSS Considerations
- Some section-specific styles may need adjustment
- Hero section might need variants for inner pages (smaller banner?)
- Consider a consistent "page header" style for inner pages

### Cache Busting
- Increment CSS/JS versions after restructuring
- May want to add `?v=2.0.0` to mark major restructure

---

## Implementation Steps (Draft)

1. **Create page template** with shared header/footer
2. **Extract each section** into its own page
3. **Update navigation** links throughout
4. **Update CSS** for multi-page layout
5. **Test all internal links**
6. **Test responsive design** on all new pages
7. **Update CLAUDE.md** with new structure

---

## Open Items / Your Feedback Needed

Please review and let me know:

1. **Option A or B** for page structure (or custom)?
2. **What additional content** should stay on home page?
3. **Navigation structure** preference?
4. **Shared components approach** (duplicate vs. build tool)?
5. Any sections you want to **combine or separate differently**?

Once we align on these decisions, I'll update this plan with the final structure before implementation.
