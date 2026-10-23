# IIT Academic Portal Design System

**Status:** Revised 2026-10-22 (UI refresh): Tailwind UI application idiom, a brand color scale around the unchanged IIT colors, Tailwind's gray and status scales, Heroicons, and a dark theme. Contrast values in Section 3 were computed for both themes; a hand-run visual review remains open as listed in Section 17.
**Scope:** Shared visual and interaction guidance for Admin, Student, Teacher, and Coordinator experiences.

## 1. Purpose and Authority

This document is the visual and interaction source of truth for the IIT Academic Portal. Feature specifications MUST reference these shared rules rather than restate them. Implementations MUST use the established tokens and component patterns. If this document conflicts with the [project constitution](../.specify/memory/constitution.md), the constitution takes precedence.

New screens follow the implementation contract in [Section 18](#18-building-new-screens): the shared class catalog, page recipes, status-badge mapping, and rules for new code.

This document defines presentation conventions only. It does not define academic business rules, role permissions, API behavior, or implementation tasks. It does not select a UI component library; follow the library already established by the application, if any.

## 2. Design Direction

The interface is a clean, modern, professional academic administration tool. Use the supplied TailwindAdmin template as a visual reference for its dashboard shell, compact information hierarchy, and restrained surfaces. Use the ReadymadeUI Tailwind component catalog as a reference for compact header and control geometry, not as a second component framework. Adapt useful patterns to Angular and the tokens below rather than copying a marketing navbar into the application shell. Favor clarity, scanability, predictable placement, and efficient common workflows over decoration. Use a restrained visual hierarchy, consistent spacing, and concise labels. All major patterns MUST work consistently across roles. Role-specific navigation and content may vary, but the visual language MUST NOT.

The look follows standard Tailwind UI application patterns: a white sidebar and header on a light gray canvas, surfaces separated by soft shadows and hairline rings rather than hard borders, `rounded-md` controls and `rounded-lg` panels, tinted status alerts and badges, and outline Heroicons beside navigation labels. The IIT brand is the accent (primary actions, the current navigation item, links, focus). The template's palette is not adopted; IIT brand and semantic color values in this document remain authoritative, and template colors, vendor stylesheets, and template-specific UI packages MUST NOT override them. Recreate approved visual patterns with the existing Angular and Tailwind CSS setup and the centralized tokens below.

The portal has a **light and a dark theme**. Both use the same semantic tokens; only their values differ (Section 3.2), so a page built from tokens needs no theme-specific styling.

## 3. Brand and Color System

### 3.1 Source of truth

The official IIT logo and approved IIT brand assets are the authority for brand identity. The project-confirmed palette for this design system is primary `#2E3192` and secondary `#4B4FB0`; no separate official accent color has been supplied, so the accent token is the brand blue at the shade that keeps text contrast in each theme (brand-600 light, brand-300 dark). White is used as the logo's contrast/negative-space color. Do not infer additional brand colors, recolor the logo, or use brand colors as substitutes for semantic status colors. If approved IIT guidance changes these values, update this document and the centralized tokens together.

All color values MUST be declared centrally as tokens. Components MUST consume semantic tokens and MUST NOT hard-code repeated color values. Add a new token only for a recurring semantic need and document it here.

### 3.2 Color roles

Colors come in two layers. **Scales** hold the raw values: the IIT brand scale `--color-brand-50` … `--color-brand-950` (600 = `#2E3192`, 500 = `#4B4FB0`; 50 `#EEF0FB`, 100 `#E0E3F6`, 200 `#C6CAEF`, 300 `#A2A7E3`, 400 `#7C80D3`, 700 `#252873`, 800 `#1F215E`, 900 `#1A1C52`, 950 `#10113A`), and Tailwind's own `gray`, `red`, `green`, `amber`, and `sky` scales (oklch values, unchanged), plus `white` and `black`. Tailwind's other color families are removed. **Semantic roles** point at the scales and are what pages use. The dark theme re-points only the roles.

| Category | Token | Meaning and use | Light | Dark |
|---|---|---|---|---|
| Brand | `--color-brand-primary` | Primary-action fill; white text on it is 10.66:1 (light) and 6.90:1 (dark) | brand-600 `#2E3192` | brand-500 `#4B4FB0` |
| Brand | `--color-brand-secondary` | Supporting brand emphasis | brand-500 | brand-400 |
| Brand | `--color-brand-accent` | Brand-colored text, links, the current nav item and tab: 10.66:1 on white; 7.75:1 on gray-900 | brand-600 | brand-300 `#A2A7E3` |
| Brand | `--color-on-brand` | Text and icons on a brand fill | white | white |
| Neutral | `--color-background` | Application canvas | gray-50 | gray-950 |
| Neutral | `--color-surface` | Panels, cards, sidebar, header | white | gray-900 |
| Neutral | `--color-surface-raised` | Menus and overlays | white | gray-800 |
| Neutral | `--color-text-primary` | Main text: 17.75:1 on white, 17.00:1 on gray-50; 16.13:1 on gray-900 | gray-900 | gray-100 |
| Neutral | `--color-text-secondary` | Labels and descriptions: 7.56:1 on white, 7.24:1 on gray-50; 6.82:1 on gray-900, 5.64:1 on gray-800 | gray-600 | gray-400 |
| Neutral | `--color-text-disabled` | Genuinely inactive content only (2.60:1 / 2.35:1) | gray-400 | gray-600 |
| Neutral | `--color-border` | Decorative outlines and the tab rule only | gray-200 | gray-700 |
| Neutral | `--color-control-border` | Essential control boundary: 4.84:1 on white; 3.67:1 on gray-900 | gray-500 | gray-500 |
| Neutral | `--color-divider` | Table-row and list separators | gray-200 | gray-800 |
| State | `--color-success` / `-subtle` | Successful outcomes; text 4.94:1 on white and 4.72:1 on its subtle tint; dark 8.37:1 on its tint | green-700 / green-50 | green-400 / 12% green-500 on gray-900 |
| State | `--color-warning` / `-subtle` | Caution; 5.05:1 on white, 4.87:1 on tint; dark 8.60:1 | amber-700 / amber-50 | amber-400 / 12% tint |
| State | `--color-error` / `-subtle` | Validation errors and failures; 6.42:1 on white, 5.88:1 on tint; dark 6.14:1 on gray-900, 5.41:1 on tint | red-700 / red-50 | red-400 / 12% tint |
| State | `--color-error-fill` / `-fill-hover` | Destructive button fill; white text 4.76:1 (hover 6.42:1) | red-600 / red-700 | red-600 / red-700 |
| State | `--color-info` / `-subtle` | Neutral information; 5.85:1 on white, 5.49:1 on tint; dark 6.96:1 | sky-700 / sky-50 | sky-400 / 12% tint |
| State | `--color-focus` | Keyboard focus outline: 10.66:1 on white; 8.79:1 on gray-950 | brand-600 | brand-300 |

Brand tokens identify IIT. Neutral tokens provide legible structure. State tokens communicate meaning and MUST remain distinguishable from one another and from ordinary brand emphasis. Status MUST also be communicated with text, iconography, or shape, never color alone. The disabled-text contrast exception applies only to genuinely inactive controls/content under WCAG; disabled styling MUST NOT be used for ordinary secondary or helper text. A low-contrast decorative border or ring MUST NOT be the only cue identifying a control; inputs keep the `--color-control-border` edge.

### 3.3 Interaction color states

Define hover, pressed/active, selected, focus, and disabled variants centrally: `--color-brand-primary-hover` (brand-700 light, brand-600 dark; white text 12.89:1 and 10.66:1), `--color-brand-primary-active` (brand-800 / brand-700), `--color-surface-hover` (gray-50 / gray-800), and `--color-surface-selected` (brand-50 light; 25% brand-500 over gray-900 dark). The brand accent is 9.38:1 on the light selected surface and 6.50:1 on the dark one. Do not create local component shades. Hover MUST NOT be the only indication of an available action. Disabled controls MUST have a clear disabled appearance and state semantics, while retaining sufficient legibility.

## 4. Typography

Use `DM Sans` as the primary Latin typeface and `Noto Sans Bengali` as the Bengali glyph fallback, followed by the system sans-serif fallback. When bundling font files, retain their license notices; do not copy or bundle operating-system fonts unless their terms permit it. The current frontend loads the approved font families through its existing font stylesheet link; no font package is added by this guidance.

| Role | Token | Guidance |
|---|---|---|
| Primary family | `--font-family-primary` | `DM Sans`, then `Noto Sans Bengali` for Bengali glyphs, then system sans-serif |
| Page title | `--font-size-page-title` | 1.5rem (24px); one clear title per page |
| Section heading | `--font-size-heading` | 1.125rem (18px); consistent hierarchy |
| Body | `--font-size-body` | 0.9375rem (15px) for a compact, readable work surface |
| Label/table | `--font-size-compact` | 0.875rem (14px); keep dense data easy to scan |
| Helper text | `--font-size-helper` | 0.8125rem (13px); remain readable and high contrast |

Use weights 400 for body text, 500 for labels and navigation, 600 for headings and emphasis, and 700 only for rare strong emphasis. Body line height is 1.5; headings use 1.2–1.35; compact table content uses at least 1.4. Buttons, labels, navigation, and tables MUST use the same family and weight conventions. Avoid all-caps labels, decorative type, and using font size alone to indicate meaning.

## 5. Layout and Spacing

### 5.1 Application layout

Use a stable application shell with a consistent compact header, persistent navigation region, main content area, and optional footer. At desktop sizes, use the shared 16rem (256px) sidebar and 4rem (64px) header dimensions; the account menu sits at the right of the header and the theme choice at the bottom of the sidebar, while the main content is the primary visual focus. Use a centered content container with a recommended maximum width of 80rem and fluid side gutters; wide data tables may use the available content width. Keep page titles, breadcrumbs, actions, and content aligned to the same grid. Every page opens with a `.page-header`: the title and its one-paragraph introduction (`.page-header-description`, secondary text, at most 48rem wide) on the left, and the page's actions (`.page-header-actions`, such as **Add teacher**) on the right from `sm:` up; below `sm:` the actions move under the introduction. A page without actions uses the same block with the title and introduction only. Breadcrumbs (`.breadcrumb`, compact secondary text) sit directly above it.

On desktop, persistent navigation may use a sidebar alongside the main area. Keep its width and collapsed behavior consistent throughout the application. On smaller screens, navigation MUST become a usable compact menu or drawer rather than compressing desktop navigation. Avoid nested cards and unnecessary columns.

Use `.full-container` to fill the available parent width; it does not remove a parent or child max-width constraint. Apply the shared content maximum where page content should remain constrained.

### 5.2 Spacing and grid

Use Tailwind's default spacing scale: a 4px step (`--spacing: 0.25rem`), so `p-1` is 4px, `p-4` 16px, `p-6` 24px, `p-8` 32px. Prefer these steps over one-off values. Use a simple 12-column or fluid grid only where it improves alignment; forms and reading content should use a single readable column by default.

Use 24px between form fields (`.field`) and between distinct page sections, and 8px between a label and its control. Cards and table containers pad 16px on narrow screens and 24px from `sm:` up. Page content pads 16px / 24px / 32px at the base, `sm:`, and `lg:` widths. Keep spacing consistent across modules. Do not use empty space to imply an interaction or state.

### 5.3 Breakpoints

Use shared breakpoints as layout transitions, not as device assumptions: `--breakpoint-sm: 40rem`, `--breakpoint-md: 48rem`, `--breakpoint-lg: 64rem`, and `--breakpoint-xl: 80rem`. Validate real content at each transition. Do not scale typography with viewport width.

## 6. Shared Component Guidelines

All components MUST have consistent dimensions, typography, spacing, focus treatment, and semantic tokens. Each component MUST expose clear default, hover, focus, active/selected, disabled, and error states where applicable. Interactive targets SHOULD be at least 44 by 44px on touch layouts. Components MUST preserve native keyboard and assistive-technology semantics.

| Component | Purpose, appearance, states, and usage rules |
|---|---|
| Buttons | Trigger actions. Use `.btn-sm` (32px desktop), `.btn` (40px with a mouse on wide screens, 44px on touch or narrow screens), or `.btn-lg` (48px) for hierarchy; small buttons retain a 44px minimum on touch/narrow layouts. Buttons are `rounded-md`, 14px semibold, with an extra-small shadow; an optional leading icon is 20px (16px in `.btn-sm`). Primary (`.btn-primary`, brand fill) is reserved for the main action; secondary (`.btn-secondary`) is a surface button with a light border and primary text; destructive uses the semantic error treatment and explicit wording: `.btn-danger` (filled `--color-error-fill`, white text, for the button that confirms the destructive action; 4.76:1) and `.btn-danger-outline` (surface with a translucent error border and `--color-error` text, for the button that starts it; 6.42:1 on white). The confirmation names the item ("Delete \"BSc\"?"). An icon-only control uses `.btn-icon` (44px square) and MUST carry an `aria-label`. Define hover, pressed, focus, disabled, and loading states centrally. Use links for navigation, not buttons styled as links. |
| Text inputs | Collect short text. Use a consistent border, height, padding, and label placement. Support default, focus, filled, disabled, read-only, and invalid states. Placeholder text is an example only and never replaces a label. |
| Password inputs | Collect secret text. Match text-input geometry; provide a clearly labeled visibility toggle if present. Do not expose values in helper content, logs, or error messages. Show authentication feedback without revealing whether an account exists unless policy explicitly permits it. |
| Select/dropdown | Choose from a finite set. Every dropdown is searchable: the shared `<app-select>` (an ARIA editable combobox with a listbox popup), never a native `<select>`. Its text field has the `.field-input` look with a chevron, shows the chosen label, and filters the list as the user types (case- and accent-insensitive, matching the label and any `keywords`, such as a configuration code, so "PGDIT" finds "Post Graduate Diploma in Information Technology"). Arrow keys, Home, and End move through the matches, Enter chooses (without submitting the form), Escape closes and restores the selection, a click chooses, and Tab or leaving the field closes it unchanged; "No matches" shows when nothing matches and the count is announced. The list opens below the field (at most 16rem tall, scrolling), with optional group headings (`group`, as for Upazilas and City corporations) and the chosen option in semibold brand accent. An `emptyLabel` adds the "" choice first ("Any", "All", "Select a degree") in secondary text and shows it as the placeholder. It is a form control (`formControlName`, `[formControl]`) holding a string, takes `invalid`, `describedBy`, and `required` for the field's error and help, and fires `(valueChange)` after the control changes. Options are 36px tall (44px on touch screens). |
| Checkbox | Select independent options or acknowledge a condition. Use for zero-or-more choices; pair with a clickable text label and visibly distinguish checked, mixed, disabled, and focus states. |
| Radio button | Choose exactly one option from a small group. Group under a visible legend; show checked, disabled, and focus states. Do not use for independent choices. |
| Toggle | Switch an immediate binary setting. Label both the setting and its current meaning; do not use for form choices that require a separate submission unless clearly presented as pending. Show on/off, disabled, and focus states. |
| Date picker | Enter or select a date. Use the same field structure and validation as other inputs. Display an unambiguous date format, support keyboard entry/navigation, and provide a text-entry path where practical. |
| Search | Find or filter a collection. Use a labeled field, clear action when non-empty, and optional submit affordance. Preserve the query during result interaction where practical; distinguish no matches from an empty collection. A list page's filter panel is a `.card` holding a `.filter-grid` (16px gaps; add `md:grid-cols-*` / `xl:grid-cols-*` for columns) of `field mb-0` controls, then a `.filter-actions` row with the primary Search or Apply filters button and a secondary Clear filters button of the same size. |
| Tables | Present comparable records for scanning. Use the shared `.data-table-container` for list views and `.data-table-scroll` inside an existing card. `.data-table-container` is a `rounded-lg` surface with the panel shadow. `.data-table` centralizes 14px text, a lightly tinted semibold header in primary text, 14px vertical and 12px horizontal cell padding (16px at the row ends), divider separators, and a restrained row-hover state. Preserve column alignment, headers, row actions, loading, empty, and responsive behavior as specified in Section 8. |
| Lists/list groups | Present related items with semantic `ul`/`ol` markup. Use `.list-group` with `.list-group-item`; add `.list-group-compact` for a 20rem maximum width or `.list-group-flush` for edge-to-edge separators. Use `.list-group-item-action` on a row link, `.list-group-icon` for a leading icon, and `.list-group-item-total` for a highlighted total. The group is a `rounded-lg` surface with the panel shadow; rows use 12px vertical and 16px horizontal padding, 14px text at 20px line height, and 8px row gaps (14px beside icons). Use `.list-marker` for a bulleted supporting list and `.list-checklist` for a check-mark list; their item spacing is 8px and 12px respectively. A `.badge-pill` can appear at the end of a row. Keep interactive rows keyboard accessible and at least 44px high; use a table for comparable multi-field records. |
| Infinite loading | For long collections, append bounded batches as the reader approaches the end. Preserve filters and sort, show loading/error/end states, and provide an accessible manual Load more fallback. Do not show numbered pages or an items-per-page selector. |
| Cards | Group closely related summary or dashboard content. Use `.card` for the surface and, for a template-style content wrapper, place `.card-body` as its direct child. A card is a `rounded-lg` surface with the panel shadow (a small shadow plus a hairline ring), padded 16px, or 24px from `sm:` up; `.card-body` carries the same padding and replaces the root card's. Use `.card-title` for 16px/24px semibold headings and `.card-subtitle` for 14px/20px secondary text. Links used as card titles are not underlined; preserve their brand color, hover feedback, and visible keyboard focus. Do not nest cards or make a non-interactive card look clickable. |
| Modal/dialog | Focus a short, interruptive task. Use a clear title, concise content, and explicit actions. Trap and restore focus correctly; close with Escape when safe. Do not use for routine page content. |
| Confirmation dialog | Confirm destructive or irreversible action. State the consequence plainly and name the affected item when possible. Make cancel easy to choose; label the destructive action explicitly and do not rely on color alone. |
| Alerts | Keep persistent or page-level feedback visible in context. `.alert` is a `rounded-md` panel with 16px padding, tinted with the state's `-subtle` color and ringed with its text color at 20%; the title (`.alert-title`) is semibold in the state color and the message is primary text. Provide success, information, warning, and error variants; the heading text names the state, so color is never the only cue. Optional dismissal only when safe. |
| Toast/notification | Report brief, non-blocking outcomes. Keep wording concise, avoid stacking excessive notifications, and allow enough time to read; critical errors require persistent in-page presentation instead. |
| Badges/status indicators | Summarize a short state. Use consistent shape, size, semantic color, and text label. `.badge` is a 24px-high `rounded-md` label with 13px medium text, tinted with the selected surface, brand-accent text, and a matching inset ring. `.badge-md` is larger (14px/20px semibold). `.badge-pill` is a fully rounded 12px pill; pair it with `.badge-pill-primary` for the IIT primary fill. State badges (`.badge-success`, `.badge-error`, `.badge-warning`, and `.badge-neutral`) use the state's subtle tint, text color, and a 25% ring, and meet 4.5:1 in both themes. Do not use a badge for long explanations or encode state by color alone. |
| Tabs | Switch among peer views within the same context. Keep labels short and active state obvious. Preserve keyboard arrow-key navigation and tab semantics; do not use tabs as a substitute for primary navigation. |
| Breadcrumbs | Show the current location within a hierarchy. Include only navigable ancestors and a clear current-page label; use consistent separators and hide or collapse gracefully on narrow screens. |
| Dropdown menus | Reveal a small set of related actions. Anchor to the trigger, group related items, support keyboard navigation and Escape, and use separators sparingly. Destructive actions remain clearly identified. |
| Tooltips | Explain a non-obvious control or truncated label. Keep content brief and supplementary; never put essential instructions or errors only in a tooltip. Support keyboard focus as well as pointer hover. |
| Loading indicators | Communicate that content is being retrieved or an action is in progress. Use a restrained spinner for short waits and stable skeletons for known content layouts. In the frontend, loading text is a `.loading-state` paragraph (14px secondary text after a `.spinner`), with `role="status"` where the change should be announced; at the end of an infinite list add `m-0`. Buttons show the spinner and a "…ing" label while they work and are disabled. Announce meaningful busy state accessibly and prevent accidental duplicate submission. |
| Empty states | Explain that a view has no content or results. Distinguish first-use/no data from no search matches; state the condition clearly and show an action only when one is relevant and authorized. In the frontend, use `.empty-state`: a centered panel with a 2px dashed border and 40px vertical padding, an optional 40px outline icon (`ng-icon`, disabled-text color), a 14px semibold `.empty-state-title`, one `.empty-state-text` sentence, and an optional button 24px below. It is used for empty lists, no matches, the role landing pages, and the access-denied and no-role pages. |
| Description lists | Show a record's facts as label/value pairs (`dl.description-list` with direct `dt`/`dd` children): a 12rem secondary-text label column beside the value from `sm:`, the label above the value below it, 14px text, and a divider between pairs. Use `.description-list-stacked` for long values such as message text, so every label sits above its value. |
| Error states | Explain what failed in user-facing terms, preserve entered data where safe, and offer a specific recovery action when available. Do not expose stack traces, SQL, or infrastructure details. |

## 7. Forms and Validation

Forms MUST be easy to scan and use a consistent single-column layout unless a short, related pair of fields benefits from two columns. Group related fields with headings and spacing, not nested panels. Place persistent labels above controls. The frontend uses the Tailwind UI stacked form:

- A form is a `.card` at most 48rem (`max-w-3xl`) wide. A card's section heading is a 16px semibold `h2` (or `legend`), with one sentence of secondary text under it.
- Each `.field` stacks the label (14px medium), the control, any error, then the help text (13px secondary), 8px apart, with 24px between fields. The help text stays before the control in the markup, linked by `aria-describedby`, so assistive technology hears it first; only the visual order changes.
- Related fields sit side by side in a `.form-grid`: two columns from `sm:` (add `md:grid-cols-3` and so on for more), 24px gaps both ways, and 24px below the grid; the grid owns the spacing, so its fields have no bottom margin. Examples: Staff ID and Phone; the student form's degree, batch, and rolls.
- A long form is split into sections, each a `fieldset.card` whose `legend` is the section heading (16px semibold, inside the card, with an optional one-sentence `p` description under it); do not nest cards. The last field or grid in a card has no bottom margin.
- Inputs and selects are full width, `rounded-md`, 14px text, 40px tall with a mouse on wide screens and 44px on touch or narrow screens (buttons follow the same rule).
- The buttons go in a `.form-actions` row: the secondary action (Cancel, Back) first and the primary action last, aligned right. Inside a card this row is the card's footer bar (tinted surface, divider above, flush with the card's edges); a form made of several cards ends with the row on the page, under a divider.

Mark required fields consistently and explain any non-obvious requirement near the relevant field or group.

Help text MUST appear adjacent to its field and remain distinct from validation feedback. Validate on the server as well as in the interface. Identify invalid fields in text, associate the message programmatically, and provide a concise summary when a form has multiple errors. Validation messages MUST explain how to correct the value and MUST NOT rely on color alone. Do not show an error before the user has had a reasonable opportunity to provide the value, unless the error is an immediate security or format requirement.

**Field error message:** place it directly below the control, in helper size, medium weight, and `--color-error`. Start it with a small error icon: a filled circle with an exclamation mark, 1em square, in the same error color. The icon sits on the first line of text, and wrapped lines align with the text, not the icon. The icon is decorative, so hide it from assistive technology; the message text carries the meaning. Do not use a bare "!" or other text symbol in its place. In the frontend, the shared `.field-error` class provides the icon.

Disabled fields communicate unavailable actions and are skipped by normal keyboard interaction; read-only fields remain readable and selectable. Do not use disabled styling for data that users should still copy. Keep submit and cancel actions together in a consistent location, with one visually primary submit action. Prevent duplicate submissions and show progress without shifting the form layout.

## 8. Tables and Data-Dense Views

Tables are for comparing records across consistent attributes. Use clear column headings, concise cell content, and a stable alignment rule: text is left-aligned, numeric values are right-aligned, and dates follow one documented format. Keep row heights consistent (recommended minimum 44px for interactive rows) and use subtle separators. Avoid heavy grid lines and excessive zebra striping.

Sorting MUST be indicated on the active column and operable by keyboard. Filtering controls MUST be visibly associated with the table and distinguish active filters. Long results MUST preserve sort/filter state while appending bounded batches. Place row actions consistently, expose accessible names, and avoid making the entire row clickable when it contains independent controls.

**Infinite loading (one pattern for every long list):** load an initial bounded batch, then use the shared `appInfiniteScrollSentinel` inside the main scroll region to request and append the next batch as it approaches the viewport. Prefer stable cursor requests. If an existing endpoint only supports bounded page-number requests, request pages sequentially behind the scroll interaction; do not expose page numbers, totals, or an items-per-page selector. Keep existing rows visible and mark the list busy during a load. Ignore repeated load requests while one is in flight. Provide a keyboard-accessible **Load more** button as a fallback when automatic observation is unavailable, and show a concise loading status. A failed later batch MUST leave existing rows in place and offer retry; a failed first batch shows the normal list error state. Filter changes return to the initial batch. Distinguish an empty collection from no matching results, and announce when all matching records have loaded. Use stable ordering and bounded batch sizes so the client never loads the entire collection in one request.

**Audit timestamps:** audit history shows every date-time as `yyyy-MM-dd HH:mm:ss` followed by the label "UTC", inside a `time` element with the ISO value in `datetime` (the shared `app-utc-timestamp` component). Date-range filters on those screens are labeled "(UTC)" and sent as UTC. Outcomes appear as text in a badge, never as color alone. Audit results follow the infinite-loading pattern above; no numbered page control is shown.

Loading MUST preserve the table's expected geometry. When another batch is loading, keep current rows on screen, dimmed and marked `aria-busy`; replacing the table with a spinner shrinks the page and throws the scroll position back to the top. Keep the manual **Load more** fallback operable and do not move focus during automatic loads. Empty data and no matching results MUST have distinct messages. On smaller screens, prioritize important columns and provide a usable horizontal-scroll region or an explicitly designed compact representation; do not silently clip critical values or actions. Keep table headers available to assistive technology and provide captions or an equivalent accessible name.

**Record history:** an edit screen for a stored record shows a read-only "Record history" card below the form with two rows, "Created" and "Last modified", each giving the UTC timestamp (the shared `app-utc-timestamp`) followed by "by" and the account's email. When no account made the change it says "System", and when the account has since been removed it says "(account removed)". It is not shown when creating a record. Any screen for a new kind of record should show the same card.

**Admin maintenance lists and forms** (used by Configuration and Teachers): a list defaults to active records with a status filter, search, and the shared infinite-loading behavior; an inactive record is labeled "Inactive" in text and, in a selector, "(inactive)". A form shows required fields with persistent labels and field-level errors, moves focus to the first invalid field, and keeps what was typed. Deactivation is the Active checkbox. Deleting asks for explicit confirmation in the page and states the consequence plainly (the record disappears from every list and cannot be restored from the portal; the data is kept in the database); when the service refuses because the record is in use, the screen explains why and offers "Deactivate instead". A selector that depends on another (an upazila's district depends on its division) is disabled-by-prompt ("Choose a division first") until the parent is chosen, and changing the parent clears the child.

**Staff management** follows the same list, form, and record-history patterns. Because staff details are checked by a person, every save
needs an attestation: an unticked checkbox above the save button ("I have manually verified ... against the institution's records") with
help text saying the confirmation is recorded with the change and no document is kept. It is required on every save and cleared after
each one, its error sits under it like any field error, and the portal never shows the details as "verified". A profile whose account was
deactivated or lost its role, or a deactivated staff member, is shown with an info alert and its fields disabled. Adding a staff member
takes an email address rather than an account picker, and the directory's success notice says what happened (new account and setup email
sent; new account but the setup email could not be requested, with where to reissue it; or linked to an existing Admin account). The
directory shows staff state as an "Active"/"Inactive" badge and offers "Deactivate…" (small danger outline) per active row; it asks for
confirmation in the row, like removing a grade band: a short question naming the person and the consequence (account deactivated,
sessions end, profile and history kept), a danger "Yes, deactivate", and a secondary "Cancel". A refusal from the service (the signed-in
Admin, the last active Admin) is shown as an error in that row.

**Whole-set editors** (the Grade scale) are used when a rule spans records, so the records are saved together: one table of editable rows with a visually hidden label per input (for example "Letter grade, band 3"), every column heading left-aligned over its input (the input sits at the start of the cell), the number inside a numeric input right-aligned, an "Add" button below the table, and a small danger-outline "Remove…" button per row whose accessible name includes the row number. "Remove…" asks for confirmation in the row itself: the button is replaced by a short question naming the row ("Remove A? It is deleted when you save the scale.", or "It has not been saved yet." for a row added on the screen) with a danger "Yes, remove" and a secondary "Cancel"; focus moves to "Yes, remove", Cancel returns it to "Remove…", and after removing it moves to the next row's "Remove…" (or "Add" when none is left). Removing a row only takes effect on save, and the screen says so. One primary "Save" and a secondary "Discard changes" (which reloads) sit below. The same checks the service makes run before sending; rule violations across rows (a gap or overlap) are listed in an error summary at the top, naming the affected values, and each is also shown on the row field where it starts. A refused save keeps every edit, and the summary states that nothing was changed. A "Record history" card shows the latest change to any row.

**Teacher management** follows the same list, form, and record-history patterns, with these additions. Account state is always text in a badge ("Active", "Pending setup", "Deactivated"), never color alone, and a pending or deactivated teacher also gets an info alert on the edit screen explaining it. A teacher is never deleted: the destructive action is "Deactivate teacher…" (danger outline), which asks for in-page confirmation, states that sessions end and sign-in stops while the record and history stay, and shows the service's reason (for example the last active Admin) without changing anything. Roles are a separate card with its own save: required roles are shown checked and disabled, system-managed roles (Coordinator) are described in text rather than offered as controls, and the default-role choice lists only the roles that would be assigned.

**Notification administration** (history, templates, reminder setting) follows the audit-history pattern for its history list (filters, newest first, infinite loading with a "Load more" button as the keyboard and no-observer fallback, rows kept when a later batch fails) and the maintenance-form pattern for templates and the setting. A notification's status is always written out ("Sent", "Failed", "Uncertain"...), with the badge color only reinforcing it, and "Sent" is explained as accepted by the delivery service, never as delivered or read. Content that is never stored (secure-link messages) is shown as an explicit "Content withheld" note, not as an empty area. A preview is labelled as made with sample data, states that nothing was sent or saved, shows a secure message's link only as a visibly nonfunctional placeholder, and disappears as soon as the wording is edited. A control that policy fixes (the enabled switch of a secure template) is shown checked and disabled with the reason beside it. A resend asks for explicit in-page confirmation, uses one idempotency key per confirmation, and shows the service's refusal in plain language. The three screens share a tab strip (History, Templates, Reminder setting): see **Tabs** in section 9. **Emails** are sent in one fixed IIT layout: the approved logo (alt text "IIT, University of Dhaka", loaded from a public HTTPS address) on white with a brand-primary rule, the subject as the heading in brand primary, the message in 16px primary text, a link on its own line as a brand-primary button with the address repeated as text, and a small secondary-text footer, in a table-based single column of at most 600px with inline styles for mail clients. Wording stays plain text and the layout is not editable; the logo address, brand color, portal name, footer, and button label are deployment settings (the brand color must stay dark enough for the white button text, 4.5:1).

## 9. Navigation

Navigation MUST make the user's current location and available destinations understandable. Use a consistent top header for the application identity, with a persistent sidebar on wide screens where the application structure warrants it. Each sidebar link (`.nav-link`) is a 44px, `rounded-md`, 14px semibold row with a 24px outline Heroicon before its label (decorative, `aria-hidden`): Dashboard `home`, Accounts `users`, Teachers `academic-cap`, Staff `briefcase`, Students `user-group`, Configuration `cog-6-tooth`, Notifications `bell`, Audit history `clipboard-document-list`. The current item (`aria-current="page"`) gets the selected surface and brand-accent text and icon, so it is marked by more than color alone (fill and weight). Nested items MUST have clear parent/child hierarchy and expansion state.

**Account dropdown:** the compact profile trigger sits at the right edge of the header (a `section` named "Your account"). It greets the signed-in person: "Welcome, <name>" (`.account-dropdown-label`, with the session's `displayName`, which is the active role's profile name, another profile name, or the email; shortened with an ellipsis and given in full as a `title` tooltip), right-aligned; for an account with more than one role, the active role's name (for example "Admin") sits under it (`.account-dropdown-role`, secondary text) so a multi-role user can always tell which role they are acting in, while for a single-role account, which has nothing to switch, the role is not shown anywhere and is only screen-reader text on the trigger ("Role: Student"). Then come a round avatar with up to two initials (first and last word of the name, titles such as "Dr" skipped; the first letter of an email) and a chevron. Only if the session sends no name does it read just "Welcome", with the role's first letter. Below `sm:` the trigger shows the avatar and chevron only; the greeting and role are its screen-reader text, and the open menu starts with the greeting (`.account-dropdown-greeting`). The menu opens downward, right-aligned, 16rem wide (never wider than the screen less 2rem). Keep the trigger at least 44px high and keyboard operable.
- For accounts with multiple roles, the menu starts with the available roles as one labelled radio group headed "Active role", then a divider. Changing a radio switches the active role immediately. A single-role account has no role section: its menu holds only My Profile and Sign out.
- **My Profile** is a link row (`a.account-dropdown-item`) to the shared `/profile` page, available to every role, marked with `aria-current="page"` while it is open; choosing it closes the menu and, on mobile, the drawer.
- Place **Sign out** as a visually compact, full-width secondary button inside the menu, while retaining a 44px minimum hit area.
- Close the menu on Escape and outside click. Escape returns focus to the trigger. Preserve disclosure semantics; do not add a menu library solely for this pattern.

On wide screens the shell fills the viewport and only the main content scrolls, so the header account menu and the sidebar theme choice are always visible. The main region is positioned (`relative`) so visually hidden text stays inside it; otherwise it stretches the page and adds a second, empty scrollbar. The account menu stays in the header on every screen size, so **Sign out** is always one tap from any page. The header holds the application identity (the approved IIT logo at 36px tall, original aspect ratio, beside the portal name in semibold primary text; below `sm:` the name is hidden and the logo, whose text alternative names IIT, stands alone, so the header fits a 320px screen) , on mobile, an icon-only menu button (three-bar / close icon) named "Open menu" or "Close menu" with `aria-expanded`, and the account menu at its right edge. The **theme switch** sits at the bottom of the sidebar (in the menu drawer on mobile), below a divider: a "Theme" heading over a segmented group (`.theme-switch`, `role="group"` labelled by that heading) of three toggle buttons (`.theme-option`), each a 16px icon and a text label: Light (sun), Dark (moon), and System (desktop), with `aria-pressed`; 36px tall, 44px on touch screens. The pressed option shows on a raised surface in the brand accent. System follows the operating-system preference and is the default; the choice is kept per browser. In the dark theme the logo sits on a small white tile so its brand blue keeps its contrast.

**Tabs:** use a tab strip (`.tabs` and `.tab`) to move between sibling pages of one section that each have their own address, for example the notification screens. Each tab is a real link inside a labelled `nav` and `ul`, so back, bookmarks, deep links, and lazy loading keep working. The current tab is marked with `aria-current="page"`, shown by a brand-accent underline and semibold brand-accent text (never color alone), and stays current on that tab's own sub-pages. Do not use `role="tablist"` or `role="tab"` for page links: that pattern is for panels within one page and needs roving focus and arrow keys. Tabs are keyboard-focusable links with the standard focus ring, at least 44px tall on touch screens, and wrap onto more lines on narrow screens. Do not use buttons for navigation between pages.

Role-specific navigation may expose different destinations, but must use the same component, spacing, active-state, and naming conventions. The server remains authoritative for authorization; navigation visibility is not a security boundary. Use breadcrumbs for deep hierarchies. On mobile, use an accessible menu or drawer with a clear open/close control, sensible focus behavior, and enough touch space. Avoid duplicating the same destination in multiple navigation regions without a clear reason.

## 10. Authentication UI

Login, forgot-password, and password-reset screens MUST be focused, uncluttered, and visibly part of the IIT Academic Portal. Use the approved logo asset at its correct aspect ratio and the approved brand token; do not add unapproved decorative brand colors. Place the form in a clear content region with a prominent heading, persistent labels, readable help/error text, and one primary action.

**Brand header:** every focused authentication screen (sign-in, forgot password, reset password, and page not found) opens with the same brand header:
- The approved IIT logo (the "IIT" lettermark above a "University of Dhaka" bar, in brand primary on a white background), 80px tall at its original 600×327 aspect ratio. Never stretch or recolor it.
- The text alternative "IIT, University of Dhaka".
- "IIT Academic Portal" set in the heading size, semibold, in primary text.
- Both centered horizontally in the card, with 24px below the header. In the dark theme the logo sits on a white rounded tile.

The card is the standard `.card`, widened to 28rem with 40px padding and `rounded-xl` corners from `sm:` up. A theme toggle (an icon-only button named "Use the dark theme" / "Use the light theme") sits in the page's top-right corner, because these screens have no header.

The page heading, form, and actions stay left-aligned for scanning. Implement the header once as a shared component rather than repeating it on each page.

**Password fields:** every password input (sign-in, reset password, account setup, and their confirmations) has a show/hide button inside its right edge: an eye icon that becomes an eye with a slash while the password is shown. It is a real 44px button that keeps the shared focus ring, is named for what it will do ("Show password" / "Hide password", or "Show new password" and so on), reports its state with `aria-pressed`, and never submits the form. The password starts hidden every time the screen opens. In the frontend, wrap the input and `<app-password-toggle>` in `.password-field`.

**Favicon and app icons:** derive them from the approved logo, without recoloring it or changing its proportions.
- **Browser tab** (`favicon.ico`, 16, 32, and 48px): the "IIT" lettermark only, cropped above the "University of Dhaka" bar because the bar text cannot be read at these sizes. Center it at 92% of the width on a white rounded square (corner radius 18%), which keeps it legible on light and dark browser chrome.
- **Home screen** (`apple-touch-icon.png`, 180px): the full logo at 84% of the width on a square white tile. The platform applies its own corner mask.

Regenerate both files whenever the logo changes.

Show loading and validation states without moving controls unexpectedly. Authentication errors MUST be actionable and MUST NOT expose secrets or sensitive account-existence details. Password visibility controls, if provided, require accessible labels and state announcements. If a user can choose or switch among roles, present only the roles authorized for that user and clearly identify the active role; do not imply that a UI selection grants permission.

## 11. Dashboard Patterns

Dashboards MUST begin with a clear page heading and prioritize the most relevant summary information. Use summary cards only for concise, comparable values; avoid decorative charts or dense card grids without a clear reading purpose. Recent activity, notifications, and quick actions should each have clear headings and predictable placement. Use the same card, typography, and spacing rules as other pages.

For a template-informed dashboard, use Tailwind's generated `grid grid-cols-12 gap-4` utilities with responsive `col-span-*` utilities for widget placement. Use `.full-container` only to fill its parent; it does not remove a max-width constraint. Build summary widgets from `.card` and its direct `.card-body`, with `.card-title`, `.card-subtitle`, and `.badge-md` or `.badge-pill` as needed. Follow the reusable list and badge recipes in Section 6. Replace template palette classes such as `text-primary`, `bg-white`, or `text-dark` with the IIT color tokens in Section 3. Do not assume the template's ApexCharts or Tabler icon classes are available: icons are Heroicons (Section 16), and adding a chart package requires separate approval.

Provide loading, empty, and error states for each dashboard region. Until a role's dashboard content is delivered, its landing page shows the module name as the page header and an `.empty-state` ("Nothing here yet"). Do not define role-specific dashboard content in this document. Avoid duplicate summaries that compete with the primary content and avoid placing cards inside cards.

## 12. Feedback and System States

| State | Presentation rule |
|---|---|
| Success | Confirm what completed in plain language. Use the success token and an icon or label; use a toast only for non-critical confirmation. |
| Error | State what failed and the next useful action. Keep important errors visible; never expose internal diagnostics. |
| Warning | Explain the risk or condition that needs attention. Do not present routine information as a warning. |
| Information | Provide neutral context without implying success or failure. |
| Loading | Identify the region or action in progress; keep layout stable and prevent repeated submission when applicable. |
| Empty | State why there is no content, distinguishing no data from no matches; show a relevant next step only if one exists. |
| Unauthorized | Use a clear access-denied message without disclosing restricted data; provide a safe navigation or support path where appropriate. In the frontend the access-denied and no-role pages are `.empty-state` panels with a lock or user icon, the page title as the `h1`, and (access denied) a primary button home. |
| Not found | Explain that the requested page or record could not be found and offer a safe return path. |

Messages MUST be concise, specific, respectful, and actionable. Announce dynamic updates accessibly. Consistent state meaning MUST be preserved across pages and roles.

## 13. Accessibility

All user-facing experiences MUST meet WCAG 2.2 AA as the project accessibility target. At minimum:

- Normal text MUST achieve 4.5:1 contrast; large text and meaningful non-text UI boundaries MUST achieve 3:1 where WCAG requires it. Verify actual approved colors in combinations, not in isolation.
- All functionality MUST be keyboard operable, with logical focus order, visible focus, no keyboard traps, and focus restoration after dialogs.
- Inputs MUST have persistent programmatic labels. Related controls MUST use semantic grouping and legends.
- Errors MUST identify the affected field or region in text, be associated with controls, and explain a correction when possible.
- Buttons and links MUST have meaningful accessible names; icon-only controls require an accessible label and tooltip where useful.
- Color MUST NOT be the sole way to communicate status, selection, required fields, or errors.
- Use semantic headings, landmarks, table headers, and appropriate status/live-region announcements. Respect reduced-motion preferences and do not use motion as the only state cue.
- Both themes MUST meet these contrast rules; the values in Section 3.2 are computed for each. The dark theme is applied before the first paint so a page never flashes light, and form controls use the matching `color-scheme`.
- Icons are decorative next to a text label (`aria-hidden`); an icon-only control MUST have an accessible name.
- Do not remove browser zoom capability. Content MUST remain usable at 200% zoom and reflow without two-dimensional scrolling except where the content itself, such as a data table, requires it.

## 14. Responsive Behavior

| Viewport | Layout behavior |
|---|---|
| Desktop (`>= 64rem`) | Use the full application shell and persistent navigation where applicable. Keep content within the shared maximum width; use available width for data tables. |
| Tablet (`48rem–63.99rem`) | Reduce gutters, allow suitable content grids to collapse, and use a compact or collapsible navigation region. Keep primary actions visible. |
| Mobile (`< 48rem`) | Use a single-column content flow and compact navigation. Stack form actions when needed, preserve readable labels, and make controls touch-friendly. Adapt data tables deliberately rather than shrinking text. |

At all sizes, prevent overlap, clipped labels, and unexpected horizontal page scrolling. Content order MUST remain meaningful when columns stack. Test long names, translated or wrapped labels, validation messages, and dense records at supported sizes. Do not hide essential actions or information solely to fit a viewport.

## 15. Centralized Design Tokens

Implement tokens in one shared theme source and expose them as CSS custom properties. Components MUST consume semantic tokens, not palette values directly. The following token contract records the currently specified values; preserve the semantic token names when mapping them into the frontend's shared theme.

The values are centralized here to prevent local component overrides and repeated hard-coded values.

**Frontend implementation:** `iit-academic-portal/src/styles/tokens.css` declares these tokens, with the same names, as the Tailwind CSS v4 theme. Each token is therefore also a utility (`bg-brand-primary`, `p-4`, `rounded-md`, `md:`). Tailwind's other color families are removed, so only the scales and roles in this document compile. Change a value here and in that file together.

```css
/* Abridged; see tokens.css for every scale step. Confirmed brand values: primary #2E3192; secondary #4B4FB0. */
@theme static {
  --color-*: initial;                     /* only the families below exist */
  --color-white: #fff;
  --color-black: #000;
  --color-brand-50: #eef0fb;  /* … */  --color-brand-500: #4b4fb0;  --color-brand-600: #2e3192;  /* … */  --color-brand-950: #10113a;
  /* Tailwind's gray, red, green, amber, and sky scales (oklch, unchanged). */

  /* Semantic roles (light). */
  --color-brand-primary: var(--color-brand-600);
  --color-brand-primary-hover: var(--color-brand-700);
  --color-brand-primary-active: var(--color-brand-800);
  --color-brand-secondary: var(--color-brand-500);
  --color-brand-accent: var(--color-brand-600);
  --color-on-brand: var(--color-white);
  --color-background: var(--color-gray-50);
  --color-surface: var(--color-white);
  --color-surface-raised: var(--color-white);
  --color-surface-hover: var(--color-gray-50);
  --color-surface-selected: var(--color-brand-50);
  --color-text-primary: var(--color-gray-900);
  --color-text-secondary: var(--color-gray-600);
  --color-text-disabled: var(--color-gray-400);
  --color-border: var(--color-gray-200);
  --color-control-border: var(--color-gray-500);
  --color-divider: var(--color-gray-200);
  --color-success: var(--color-green-700);   --color-success-subtle: var(--color-green-50);
  --color-warning: var(--color-amber-700);   --color-warning-subtle: var(--color-amber-50);
  --color-error: var(--color-red-700);       --color-error-subtle: var(--color-red-50);
  --color-error-fill: var(--color-red-600);  --color-error-fill-hover: var(--color-red-700);
  --color-info: var(--color-sky-700);        --color-info-subtle: var(--color-sky-50);
  --color-focus: var(--color-brand-600);

  /* Typography */
  --font-family-primary: "DM Sans", "Noto Sans Bengali", sans-serif;
  --font-weight-regular: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;

  /* Spacing, shape, elevation, breakpoints: Tailwind defaults */
  --spacing: 0.25rem;
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-xl: 0.75rem;
  --color-surface-ring: rgb(0 0 0 / 0.05);   /* dark: rgb(255 255 255 / 0.1) */
  --shadow-panel: 0 0 0 1px var(--color-surface-ring), 0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1);
  --shadow-overlay: 0 0 0 1px var(--color-surface-ring), 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
  --breakpoint-sm: 40rem;
  --breakpoint-md: 48rem;
  --breakpoint-lg: 64rem;
  --breakpoint-xl: 80rem;
}

/* Dark theme: <html class="dark"> re-points the roles (values in Section 3.2). */
@custom-variant dark (&:where(.dark, .dark *));
:root.dark { color-scheme: dark; --color-background: var(--color-gray-950); /* … */ }

:root {
  --font-size-page-title: 1.5rem;
  --font-size-heading: 1.125rem;
  --font-size-body: 0.9375rem;
  --font-size-compact: 0.875rem;
  --font-size-helper: 0.8125rem;
  --line-height-heading: 1.25;
  --line-height-body: 1.5;
  --border-width: 1px;
  --control-height-sm: 2rem;
  --control-height-md: 2.5rem;
  --control-height-lg: 2.75rem;
  --content-max-width: 80rem;
  --sidebar-width: 16rem;
  --header-height: 4rem;
}
```

Use Tailwind's shape scale: `rounded-md` (6px) for controls, buttons, alerts, and badges; `rounded-lg` (8px) for cards, tables, and list groups; `rounded-xl` (12px) only for the authentication card. Elevation has three steps: `shadow-xs` (buttons and inputs), `shadow-panel` (cards, tables, list groups), and `shadow-overlay` (menus). The panel and overlay shadows include a hairline ring in `--color-surface-ring` (5% black; 10% white in the dark theme), so surfaces stay distinct in both themes without a separate border. Keep token scales stable; do not add near-duplicate values for one-off component styling. Document and review any token changes because they affect all modules.

**Theme switching:** `ThemeService` (`src/app/core/theme/theme.ts`) holds the Light / Dark / System choice, stores an explicit choice in `localStorage` under `iit-theme` (System clears it), and toggles `dark` on `<html>`; an inline script in `index.html` applies the same rule before the first paint. Components needing a different value in the dark theme beyond the semantic roles may use the `dark:` variant, sparingly.

## 16. UI Consistency Rules

- Do not introduce arbitrary or unapproved brand colors; use the centralized semantic tokens.
- Reuse established components and patterns before creating a new visual pattern; Section 18 lists them and gives a recipe for each page type.
- Use DM Sans and the shared sizes/radii through centralized tokens; do not copy template CSS or add its vendor packages.
- Icons come from Heroicons (outline, 24px) through `@ng-icons/core` and `@ng-icons/heroicons` 34.0.0 (approved in the 2026-10-22 UI refresh; the last release supporting Angular 21). Register only the icons a component uses with `provideIcons`, size them with `--ng-icon__size`, and color them with `--ng-icon__color` or `currentColor`. Do not add another icon set.
- Keep spacing, typography, control sizing, border, and focus treatments consistent across roles.
- Maintain a clear button hierarchy with one primary action per context where practical.
- Present labels, validation, errors, loading, empty, and success states consistently.
- Preserve table sorting, filtering, infinite-loading behavior, and responsive conventions across modules.
- Make new components follow this document and contribute reusable patterns rather than local variants.
- Do not add decoration that reduces scanability or obscures academic information.
- Review responsive and accessibility behavior alongside visual appearance.
- Update this document when an approved shared pattern or token changes; feature specifications MUST NOT quietly redefine the shared visual system.

## 17. Open Design Inputs

Complete these validation items against the implementation and supported viewports:

1. The 2026-10-22 UI refresh checked the Admin screens in a browser (headless Chrome against the running API and seeded data) at 1280px in both themes and at 375px in the light theme: the shell, the sign-in page, the dashboard, every list, the account, teacher, staff, admission, degree, and audit-event detail and edit screens, notifications, templates, the reminder setting, the grade scale, configuration, and access denied. There were no console errors and no horizontal page overflow. Still to check: keyboard focus visibility on every control, 375px in the dark theme, 768px, 200% zoom, Bengali text, and the Student, Teacher, and Coordinator landing pages signed in as those roles (they use the same component as the Admin dashboard).
2. Contrast values in Section 3.2 are computed from the token values, not measured on screen.

## 18. Building New Screens

This section is the implementation contract for every new screen and feature. It lists the shared classes and components that exist, the page recipes to copy, and the rules a change must follow. A feature that needs a pattern not listed here adds it to `src/styles.css` (or a shared component) **and** to this section in the same change; it does not style it locally.

### 18.1 Shared class and component catalog

All classes live in `iit-academic-portal/src/styles.css` (`@layer components`); colors, sizes, and shadows come from `src/styles/tokens.css`.

| Area | Use | Do not |
|---|---|---|
| Page | `.page-header` containing `div.min-w-0` (with the `h1` and `p.page-header-description`) and `.page-header-actions`; `nav.breadcrumb` (`aria-label="Breadcrumb"`) above it on any page below a section; `.section-title` for a heading outside a card | Put a page's primary action anywhere but `.page-header-actions` (list pages) or `.form-actions` (forms) |
| Buttons | `.btn` with one of `.btn-primary`, `.btn-secondary`, `.btn-danger`, `.btn-danger-outline`; sizes `.btn-sm` and `.btn-lg`; `.btn-block` for full width (signed-out screens); `.btn-icon` for an icon-only control (with `aria-label`); `.spinner` and an "…ing" label while busy, with the button disabled | Use more than one primary button in a form or panel; use a button for navigation (use a link styled `.btn`) |
| Fields | `.field` containing `label.field-label` (or `legend.field-label` in a `fieldset.field`), `p.field-help` (before the control in markup, linked by `aria-describedby`), the control with `.field-input` (`input`, `textarea`) or a dropdown (`<app-select inputId="…" formControlName="…" emptyLabel="…" [invalid]="…" [describedBy]="…" (valueChange)="…">` with one `<app-option [value]="…" [label]="…" [keywords]="option.code" group="…" />` per choice; the label's `for` is the `inputId`), and `p.field-error`; `label.choice` for a checkbox or radio option card; `.password-field` with `<app-password-toggle>` | Put help text after the control in markup, use a placeholder instead of a label, or use a native `<select>` |
| Form layout | `form.card.max-w-3xl` (one card), or `form.max-w-3xl` containing `fieldset.card` sections with a `legend`; `.form-grid` (add `md:grid-cols-3` and so on) for side-by-side fields; `.form-actions` last | Nest cards; set field widths with arbitrary values |
| Filters | `form.card.mb-6` containing a `.filter-grid` (with `md:` or `xl:grid-cols-*`) of `.field.mb-0`, then `.filter-actions` (primary Search or Apply filters, then secondary Clear filters, both the default size) | Make Clear filters a different size from Search |
| Tables | `.data-table-container` around `table.data-table` with a `min-w-[NNrem]` (narrow screens then scroll inside the container); `caption.sr-only`; `th` with `scope`; the identifying column as `th scope="row"` holding the record link; `.data-table-scroll` for a table inside a card | Hide columns on small screens; let the page itself scroll sideways |
| Long lists | `.infinite-scroll-footer` containing a `div.infinite-scroll-sentinel` with `appInfiniteScrollSentinel`, then an error alert, a `.loading-state.m-0`, or a "Load more …" secondary button | Numbered pages (audit history is the one exception) |
| Details | `dl.description-list` with direct `dt`/`dd` pairs; `.description-list-stacked` for long values; a "Record history" card (Section 8) on edit screens | Build label/value grids by hand |
| Surfaces | `.card` (optional `.card-body`, `.card-title`, `.card-subtitle`); the `.list-group` family | Add borders or shadows to a card locally |
| Feedback | `.alert` with `.alert-success`, `.alert-info`, `.alert-warning`, or `.alert-error`, containing `strong.alert-title` and a `p`; `role="status"` for outcomes, `role="alert"` for failures, `role="note"` for standing guidance. `.empty-state` with an optional `ng-icon`, `.empty-state-title`, `.empty-state-text`, and an optional button. `.loading-state` | Show an error by color alone; use a toast |
| Status | `.badge` (neutral brand tint), `.badge-success`, `.badge-error`, `.badge-warning`, `.badge-neutral`, `.badge-md`, `.badge-pill`; mapping in Section 18.3 | Invent a new status color |
| Navigation | `.tabs` and `.tab` for sibling pages; `.nav-link` in the sidebar (shell only) | Add a second sidebar or a nested menu without updating Section 9 |
| Shell and signed-out screens | Shell only: `.skip-link`, `.account-dropdown*` (header account menu), `.account-role-*`, `.theme-switch` and `.theme-option` (sidebar theme choice). Signed-out screens: `main.auth-page` containing `section.auth-card.card` with `<app-auth-brand>`, `<app-theme-toggle>` in the page corner, and `p.auth-footer` | Reuse shell classes in feature pages |
| Shared components | `<app-select>` and `<app-option>` for every dropdown, `<app-utc-timestamp>` for every UTC time, `<app-password-toggle>`, `<app-auth-brand>`, `<app-theme-toggle>`, the `appInfiniteScrollSentinel` directive, `ThemeService` | Re-implement any of these locally |

### 18.2 Page recipes

**List page** (Accounts, Teachers, Staff, Students, configuration catalogs):

```html
<nav aria-label="Breadcrumb" class="breadcrumb">…</nav>
<div class="page-header">
  <div class="min-w-0">
    <h1>Teachers</h1>
    <p class="page-header-description">One sentence on what this list is for.</p>
  </div>
  <div class="page-header-actions"><a class="btn btn-primary" routerLink="new">Add teacher</a></div>
</div>
<!-- optional notice after a save: div.alert.alert-success role="status" -->
<form class="card mb-6">
  <div class="filter-grid md:grid-cols-3"><div class="field mb-0">…</div></div>
  <div class="filter-actions">
    <button type="submit" class="btn btn-primary">Search</button>
    <button type="button" class="btn btn-secondary">Clear filters</button>
  </div>
</form>
<section aria-labelledby="results-title">
  <h2 id="results-title" class="sr-only">Teacher results</h2>
  <!-- first load: p.loading-state
       failure: div.alert.alert-error with a "Try again" button
       nothing: div.empty-state ("No teachers yet" or "No teachers match")
       rows: div.data-table-container, then div.infinite-scroll-footer -->
</section>
```

**Create and edit page:** breadcrumb, then the `h1` (a `.page-header` when it has a description), then any standing alert, then the form.

- A short form is one `form.card.max-w-3xl` ending in `.form-actions`: the Cancel link first, the primary submit last.
- A long form is `form.max-w-3xl` with one `fieldset.card` per section (a `legend`, an optional description `p`, and `.form-grid`s), and a page-level `.form-actions` after the last section.
- A part that saves on its own (Roles, Email address) is a further `section.card.mt-6.max-w-3xl` with an `h2`, a description `p`, its own form, and its own `.form-actions`.
- On edit screens the "Record history" card follows the forms, and the destructive card ("Deactivate this …", "Delete this …") comes last.

**Detail section:** `section.card` containing an `h2`, then `dl.description-list`.

**Destructive action:** a `.btn-danger-outline` labelled with an ellipsis ("Deactivate teacher…") opens an in-place confirmation, not a dialog: a `role="group"` labelled by a `p` that names the item and the consequence, then a `.btn-danger` ("Yes, deactivate") and a `.btn-secondary` ("Cancel"). When a button carries extra screen-reader text, wrap the visible label in one `span` (`<span>Remove<span class="sr-only"> band 3</span>…</span>`) so the button's gap does not split the label.

**Shared page** (one page for every role, such as My Profile at `/profile`): a route directly under the shell, listed in `SHARED_PAGES` (`core/auth/auth.models.ts`) so a role switch stays on it, and a component that reloads when `AuthSession.activeRole()` changes, clearing the previous role's view first. It follows the detail-section recipe: `.page-header`, then `section.card.max-w-3xl` cards with a `dl.description-list` each; a value that is not recorded reads "Not recorded" in secondary text, and a missing record is an `.empty-state`.

**Placeholder page** (a module not yet delivered): a `.page-header` with the module name, then an `.empty-state` with an outline icon.

### 18.3 Status badge mapping

| Badge | Meaning | Current uses |
|---|---|---|
| `.badge-success` | Active, complete, or succeeded | Account Active, profile active, template enabled, notification Sent, audit success, requirement Met |
| `.badge-error` | Ended, refused, or failed | Account Deactivated, staff without the Admin role, notification Failed or Uncertain, audit failure, requirement Not met |
| `.badge` (default) | Waiting or neutral | Pending setup, notification Pending or Retrying, the active-role badge |
| `.badge-warning`, `.badge-neutral` | Needs attention soon; inactive but not an error | Not used yet; for new states such as Overdue or Draft |

The badge text always names the state.

### 18.4 Rules for new code

- **Colors:** templates use semantic role utilities only (`text-text-secondary`, `bg-surface`, `text-error`, `bg-brand-primary`, and so on). Scale classes (`gray-*`, `brand-*`) and hex values belong in `tokens.css`. Use `dark:` only where a role cannot express the difference (the logo tile, a pressed theme button), and check both themes.
- **Sizes:** use Tailwind's scale (`p-4`, `gap-6`, `max-w-3xl`, `max-w-lg`). Arbitrary values are allowed only for a table's `min-w-[…]`.
- **Widths:** forms, detail cards, and their alerts are `max-w-3xl`; lists, tables, and filter panels use the full content width.
- **Icons:** Heroicons outline only, through `NgIcon`, with `provideIcons` in the component's `viewProviders` listing only the icons it uses. Icons beside text are decorative; icon-only controls need an `aria-label`. Size and color icons with `--ng-icon__size` and `--ng-icon__color`.
- **Dropdowns:** always `<app-select>` with `<app-option>`s; for configuration records pass the code as `[keywords]` so it can be searched. In tests, use the helpers in `src/testing/select-testing.ts` (`chooseOption`, `optionLabels`, `optionValues`, `optionGroups`, `selectedValue`), addressed by the field's `#inputId`; the options stay in the page (hidden) while the list is closed.
- **Numbers:** right-align numeric table columns and numeric inputs in tables (`text-right`).
- **Times:** show every stored time with `<app-utc-timestamp>`.
- **States:** every data region handles loading, error (with a retry where one exists), empty, and no matches.
- **Bundle:** the initial bundle has a 500 kB warning budget. Lazy-load new feature routes (`loadComponent`) and register icons per component.
- **Verification:** before finishing a UI change, run `npm test` and `npm run build`, and look at the changed screens at 375px and 1280px in both themes.
