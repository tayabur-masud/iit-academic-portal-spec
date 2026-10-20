# IIT Academic Portal Design System

**Status:** IIT brand colors are unchanged. Shared typography, geometry, and shell dimensions are aligned to the supplied admin-template reference; implementation-time contrast and responsive validation remain open as listed in Section 17.
**Scope:** Shared visual and interaction guidance for Admin, Student, Teacher, and Coordinator experiences.

## 1. Purpose and Authority

This document is the visual and interaction source of truth for the IIT Academic Portal. Feature specifications MUST reference these shared rules rather than restate them. Implementations MUST use the established tokens and component patterns. If this document conflicts with the [project constitution](../.specify/memory/constitution.md), the constitution takes precedence.

This document defines presentation conventions only. It does not define academic business rules, role permissions, API behavior, or implementation tasks. It does not select a UI component library; follow the library already established by the application, if any.

## 2. Design Direction

The interface is a clean, modern, professional academic administration tool. Use the supplied TailwindAdmin template as a visual reference for its dashboard shell, compact information hierarchy, and restrained surfaces. Use the ReadymadeUI Tailwind component catalog as a reference for compact header and control geometry, not as a second component framework. Adapt useful patterns to Angular and the tokens below rather than copying a marketing navbar into the application shell. Favor clarity, scanability, predictable placement, and efficient common workflows over decoration. Use a restrained visual hierarchy, consistent spacing, and concise labels. All major patterns MUST work consistently across roles. Role-specific navigation and content may vary, but the visual language MUST NOT.

The template's palette is not adopted. IIT brand and semantic color values in this document remain authoritative; template colors, vendor stylesheets, and template-specific UI packages MUST NOT override them. Recreate approved visual patterns with the existing Angular and Tailwind CSS setup and the centralized tokens below.

## 3. Brand and Color System

### 3.1 Source of truth

The official IIT logo and approved IIT brand assets are the authority for brand identity. The project-confirmed palette for this design system is primary `#2E3192` and secondary `#4B4FB0`; no separate official accent color has been supplied, so the accent token aliases the primary. White is used as the logo's contrast/negative-space color. Do not infer additional brand colors, recolor the logo, or use brand colors as substitutes for semantic status colors. If approved IIT guidance changes these values, update this document and the centralized tokens together.

All color values MUST be declared centrally as tokens. Components MUST consume semantic tokens and MUST NOT hard-code repeated color values. Add a new token only for a recurring semantic need and document it here.

### 3.2 Color roles

| Category | Token | Meaning and use | Value |
|---|---|---|---|
| Brand | `--color-brand-primary` | IIT logo blue; primary actions and selected navigation emphasis; 10.66:1 against white | `#2E3192` |
| Brand | `--color-brand-secondary` | Supporting brand emphasis; 6.90:1 against white (AA normal text, below AAA) | `#4B4FB0` |
| Brand | `--color-brand-accent` | No separate official accent is established; alias the primary token | `var(--color-brand-primary)` |
| Neutral | `--color-background` | Application canvas | `#F8F9FC` |
| Neutral | `--color-surface` | Standard panels, menus, and cards | `#FFFFFF` |
| Neutral | `--color-surface-raised` | Dialogs and overlays, separated from surfaces by shadow | `#FFFFFF` |
| Neutral | `--color-text-primary` | Main content text; 17.74:1 on white and 16.85:1 on background | `#111827` |
| Neutral | `--color-text-secondary` | Supporting labels and descriptions; 7.56:1 on white and 7.18:1 on background | `#4B5563` |
| Neutral | `--color-text-disabled` | Genuinely inactive content only; 2.54:1 on white and not for ordinary helper text | `#9CA3AF` |
| Neutral | `--color-border` | Decorative card/container outlines only; 1.47:1 on white | `#D1D5DB` |
| Neutral | `--color-control-border` | Essential control boundary; contrast is 4.83:1 on white, 4.59:1 on background, 4.37:1 on hover, and 4.22:1 on selected surface | `#6B7280` |
| Neutral | `--color-divider` | Subtle table-row and list separators | `#E5E7EB` |
| State | `--color-success` | Confirmed successful outcomes; 5.02:1 on white | `#15803D` |
| State | `--color-warning` | Caution or action requiring attention; 5.02:1 on white | `#B45309` |
| State | `--color-error` | Validation errors and failed outcomes; 6.47:1 on white | `#B91C1C` |
| State | `--color-info` | Neutral informational messages; 5.93:1 on white | `#0369A1` |
| State | `--color-focus` | Visible keyboard focus indicator; 10.66:1 on white | `#2E3192` |

Brand tokens identify IIT. Neutral tokens provide legible structure. State tokens communicate meaning and MUST remain distinguishable from one another and from ordinary brand emphasis. Status MUST also be communicated with text, iconography, or shape, never color alone. The disabled-text contrast exception applies only to genuinely inactive controls/content under WCAG; disabled styling MUST NOT be used for ordinary secondary or helper text. A low-contrast decorative border MUST NOT be the only cue identifying a control.

### 3.3 Interaction color states

Define hover, pressed/active, selected, focus, and disabled variants centrally. The supplied values are `--color-brand-primary-hover: #252873`, `--color-brand-primary-active: #1A1C52`, `--color-surface-hover: #F2F3FA`, and `--color-surface-selected: #EEEFF8`. Their contrast against white is 12.89:1, 15.77:1, and (for the selected surface) 9.31:1 for brand primary, 15.49:1 for primary text, and 6.60:1 for secondary text. Do not create local component shades. Hover MUST NOT be the only indication of an available action. Disabled controls MUST have a clear disabled appearance and state semantics, while retaining sufficient legibility.

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

Use a stable application shell with a consistent compact header, persistent navigation region, main content area, and optional footer. At desktop sizes, use the shared 15rem (240px) sidebar and 4rem (64px) header dimensions; the account section remains at the bottom of the sidebar, while the main content is the primary visual focus. Use a centered content container with a recommended maximum width of 80rem and fluid side gutters; wide data tables may use the available content width. Keep page titles, breadcrumbs, actions, and content aligned to the same grid. A list page header has two rows: the title on the left and the primary action (such as **Add teacher**) on the right, vertically centered on one row (the action keeps its size with `shrink-0` and moves below the title only when the screen is too narrow for both); then the introduction on its own line across the full content width, so its right edge lines up with the action's.

On desktop, persistent navigation may use a sidebar alongside the main area. Keep its width and collapsed behavior consistent throughout the application. On smaller screens, navigation MUST become a usable compact menu or drawer rather than compressing desktop navigation. Avoid nested cards and unnecessary columns.

Use `.full-container` to fill the available parent width; it does not remove a parent or child max-width constraint. Apply the shared content maximum where page content should remain constrained.

### 5.2 Spacing and grid

Use a 4px base spacing unit and the shared scale below. Prefer these values over one-off margins. Use a simple 12-column or fluid grid only where it improves alignment; forms and reading content should use a single readable column by default.

| Token | Value |
|---|---:|
| `--spacing-1` | 4px |
| `--spacing-2` | 8px |
| `--spacing-3` | 12px |
| `--spacing-4` | 16px |
| `--spacing-6` | 20px |
| `--spacing-8` | 24px |
| `--spacing-12` | 32px |
| `--spacing-16` | 40px |

Use 12–16px between related form fields, 16–24px between distinct page sections, and 6–8px between a label and its control. Keep spacing consistent across modules. Do not use empty space to imply an interaction or state.

### 5.3 Breakpoints

Use shared breakpoints as layout transitions, not as device assumptions: `--breakpoint-sm: 40rem`, `--breakpoint-md: 48rem`, `--breakpoint-lg: 64rem`, and `--breakpoint-xl: 80rem`. Validate real content at each transition. Do not scale typography with viewport width.

## 6. Shared Component Guidelines

All components MUST have consistent dimensions, typography, spacing, focus treatment, and semantic tokens. Each component MUST expose clear default, hover, focus, active/selected, disabled, and error states where applicable. Interactive targets SHOULD be at least 44 by 44px on touch layouts. Components MUST preserve native keyboard and assistive-technology semantics.

| Component | Purpose, appearance, states, and usage rules |
|---|---|
| Buttons | Trigger actions. Use `.btn-sm` (32px desktop), `.btn` (44px default), or `.btn-lg` (48px) for hierarchy; small buttons retain a 44px minimum on touch/narrow layouts. Use a restrained corner radius and clear text. Primary is reserved for the main action; secondary is for alternatives; destructive uses the semantic error treatment and explicit wording: in the frontend, `.btn-danger` (filled `--color-error`, white text, for the button that confirms the destructive action) and `.btn-danger-outline` (white with an `--color-error` border and text, for the button that starts it). Both keep white-on-error and error-on-white contrast of at least 6.47:1, and the confirmation names the item ("Delete \"BSc\"?"). Define hover, pressed, focus, disabled, and loading states centrally. Use links for navigation, not buttons styled as links. |
| Text inputs | Collect short text. Use a consistent border, height, padding, and label placement. Support default, focus, filled, disabled, read-only, and invalid states. Placeholder text is an example only and never replaces a label. |
| Password inputs | Collect secret text. Match text-input geometry; provide a clearly labeled visibility toggle if present. Do not expose values in helper content, logs, or error messages. Show authentication feedback without revealing whether an account exists unless policy explicitly permits it. |
| Select/dropdown | Choose from a finite set. All native selects use the shared `.field-input` height, border, focus/error treatment, spacing, and token-colored caret. Keep the native option popup and keyboard behavior; use a searchable alternative only when list size warrants it. |
| Checkbox | Select independent options or acknowledge a condition. Use for zero-or-more choices; pair with a clickable text label and visibly distinguish checked, mixed, disabled, and focus states. |
| Radio button | Choose exactly one option from a small group. Group under a visible legend; show checked, disabled, and focus states. Do not use for independent choices. |
| Toggle | Switch an immediate binary setting. Label both the setting and its current meaning; do not use for form choices that require a separate submission unless clearly presented as pending. Show on/off, disabled, and focus states. |
| Date picker | Enter or select a date. Use the same field structure and validation as other inputs. Display an unambiguous date format, support keyboard entry/navigation, and provide a text-entry path where practical. |
| Search | Find or filter a collection. Use a labeled field, clear action when non-empty, and optional submit affordance. Preserve the query during result interaction where practical; distinguish no matches from an empty collection. Filter panels use a 12px grid gap between controls and actions. |
| Tables | Present comparable records for scanning. Use the shared `.data-table-container` for list views and `.data-table-scroll` inside an existing card. `.data-table` centralizes 14px text, a 44px lightly tinted semibold header, 44px minimum body rows, 12px horizontal cell padding, subtle separators, and a restrained row-hover state. Preserve column alignment, headers, row actions, loading, empty, and responsive behavior as specified in Section 8. |
| Lists/list groups | Present related items with semantic `ul`/`ol` markup. Use `.list-group` with `.list-group-item`; add `.list-group-compact` for a 20rem maximum width or `.list-group-flush` for edge-to-edge separators. Use `.list-group-item-action` on a row link, `.list-group-icon` for a leading icon, and `.list-group-item-total` for a highlighted total. Match template geometry: 8px group radius, 12px vertical and 16px horizontal row padding, 14px text at 20px line height, and 8px row gaps (14px beside icons). Use `.list-marker` for a bulleted supporting list and `.list-checklist` for a check-mark list; their item spacing is 8px and 12px respectively. A `.badge-pill` can appear at the end of a row. Keep interactive rows keyboard accessible and at least 44px high; use a table for comparable multi-field records. |
| Infinite loading | For long collections, append bounded batches as the reader approaches the end. Preserve filters and sort, show loading/error/end states, and provide an accessible manual Load more fallback. Do not show numbered pages or an items-per-page selector. |
| Cards | Group closely related summary or dashboard content. Use `.card` for the surface and, for a template-style content wrapper, place `.card-body` as its direct child. The root uses 16px padding; `.card-body` uses 20px and replaces the root card's padding. Use `.card-title` for 16px/24px semibold headings and `.card-subtitle` for 14px/20px secondary text. Links used as card titles are not underlined; preserve their brand color, hover feedback, and visible keyboard focus. Keep a simple surface, subtle border, and restrained radius. Do not nest cards or make a non-interactive card look clickable. |
| Modal/dialog | Focus a short, interruptive task. Use a clear title, concise content, and explicit actions. Trap and restore focus correctly; close with Escape when safe. Do not use for routine page content. |
| Confirmation dialog | Confirm destructive or irreversible action. State the consequence plainly and name the affected item when possible. Make cancel easy to choose; label the destructive action explicitly and do not rely on color alone. |
| Alerts | Keep persistent or page-level feedback visible in context. Use a semantic icon, concise heading/message, and optional dismissal only when safe. Provide success, information, warning, and error variants. |
| Toast/notification | Report brief, non-blocking outcomes. Keep wording concise, avoid stacking excessive notifications, and allow enough time to read; critical errors require persistent in-page presentation instead. |
| Badges/status indicators | Summarize a short state. Use consistent shape, size, semantic color, and text label. `.badge-md` provides the template's 7px radius, 14px/20px semibold text, and 4px vertical/12px horizontal padding. `.badge-pill` provides a 24px-high pill with 12px text and 4px/8px padding; pair it with `.badge-pill-primary` for the IIT primary fill. Do not use a badge for long explanations or encode state by color alone. Use IIT semantic colors. Outcome badges (`badge-success`, `badge-error`) use a white surface with a border, because the semantic text colors do not reach 4.5:1 on the tinted default badge. |
| Tabs | Switch among peer views within the same context. Keep labels short and active state obvious. Preserve keyboard arrow-key navigation and tab semantics; do not use tabs as a substitute for primary navigation. |
| Breadcrumbs | Show the current location within a hierarchy. Include only navigable ancestors and a clear current-page label; use consistent separators and hide or collapse gracefully on narrow screens. |
| Dropdown menus | Reveal a small set of related actions. Anchor to the trigger, group related items, support keyboard navigation and Escape, and use separators sparingly. Destructive actions remain clearly identified. |
| Tooltips | Explain a non-obvious control or truncated label. Keep content brief and supplementary; never put essential instructions or errors only in a tooltip. Support keyboard focus as well as pointer hover. |
| Loading indicators | Communicate that content is being retrieved or an action is in progress. Use a restrained spinner for short waits and stable skeletons for known content layouts. Announce meaningful busy state accessibly and prevent accidental duplicate submission. |
| Empty states | Explain that a view has no content or results. Distinguish first-use/no data from no search matches; state the condition clearly and show an action only when one is relevant and authorized. |
| Error states | Explain what failed in user-facing terms, preserve entered data where safe, and offer a specific recovery action when available. Do not expose stack traces, SQL, or infrastructure details. |

## 7. Forms and Validation

Forms MUST be easy to scan and use a consistent single-column layout unless a short, related pair of fields benefits from two columns. Group related fields with headings and spacing, not nested panels. Place persistent labels above controls. Mark required fields consistently and explain any non-obvious requirement near the relevant field or group.

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

**Whole-set editors** (the Grade scale) are used when a rule spans records, so the records are saved together: one table of editable rows with a visually hidden label per input (for example "Letter grade, band 3"), every column heading left-aligned over its input (the input sits at the start of the cell), the number inside a numeric input right-aligned, an "Add" button below the table, and a small danger-outline "Remove" button per row whose accessible name includes the row number. Removing a row only takes effect on save, and the screen says so. One primary "Save" and a secondary "Discard changes" (which reloads) sit below. The same checks the service makes run before sending; rule violations across rows (a gap or overlap) are listed in an error summary at the top, naming the affected values, and each is also shown on the row field where it starts. A refused save keeps every edit, and the summary states that nothing was changed. A "Record history" card shows the latest change to any row.

**Teacher management** follows the same list, form, and record-history patterns, with these additions. Account state is always text in a badge ("Active", "Pending setup", "Deactivated"), never color alone, and a pending or deactivated teacher also gets an info alert on the edit screen explaining it. A teacher is never deleted: the destructive action is "Deactivate teacher…" (danger outline), which asks for in-page confirmation, states that sessions end and sign-in stops while the record and history stay, and shows the service's reason (for example the last active Admin) without changing anything. Roles are a separate card with its own save: required roles are shown checked and disabled, system-managed roles (Coordinator) are described in text rather than offered as controls, and the default-role choice lists only the roles that would be assigned.

**Notification administration** (history, templates, reminder setting) follows the audit-history pattern for its history list (filters, newest first, infinite loading with a "Load more" button as the keyboard and no-observer fallback, rows kept when a later batch fails) and the maintenance-form pattern for templates and the setting. A notification's status is always written out ("Sent", "Failed", "Uncertain"...), with the badge color only reinforcing it, and "Sent" is explained as accepted by the delivery service, never as delivered or read. Content that is never stored (secure-link messages) is shown as an explicit "Content withheld" note, not as an empty area. A preview is labelled as made with sample data, states that nothing was sent or saved, shows a secure message's link only as a visibly nonfunctional placeholder, and disappears as soon as the wording is edited. A control that policy fixes (the enabled switch of a secure template) is shown checked and disabled with the reason beside it. A resend asks for explicit in-page confirmation, uses one idempotency key per confirmation, and shows the service's refusal in plain language. The three screens share a tab strip (History, Templates, Reminder setting): see **Tabs** in section 9. **Emails** are sent in one fixed IIT layout: the approved logo (alt text "IIT, University of Dhaka", loaded from a public HTTPS address) on white with a brand-primary rule, the subject as the heading in brand primary, the message in 16px primary text, a link on its own line as a brand-primary button with the address repeated as text, and a small secondary-text footer, in a table-based single column of at most 600px with inline styles for mail clients. Wording stays plain text and the layout is not editable; the logo address, brand color, portal name, footer, and button label are deployment settings (the brand color must stay dark enough for the white button text, 4.5:1).

## 9. Navigation

Navigation MUST make the user's current location and available destinations understandable. Use a consistent top header for the application identity, with a persistent sidebar on wide screens where the application structure warrants it. Mark the active item with more than color alone (such as weight, indicator, or icon treatment). Nested items MUST have clear parent/child hierarchy and expansion state.

**Account dropdown:** the compact profile trigger sits at the bottom of the sidebar, below the navigation and separated by a divider. It shows the current role and opens a small menu on activation. Keep its trigger at least 44px high and keyboard operable.
- For accounts with multiple roles, show the available roles as one labelled radio group headed "Active role"; for single-role accounts, show the role as a badge. Changing a radio switches the active role immediately.
- Include a disabled **My Profile** item until its profile destination is implemented.
- Place **Sign out** as a visually compact, full-width secondary button inside the menu, while retaining a 44px minimum hit area.
- Close the menu on Escape and outside click. Escape returns focus to the trigger. Preserve disclosure semantics; do not add a menu library solely for this pattern.

On wide screens the shell fills the viewport and only the main content scrolls, so the account section is always visible. The main region is positioned (`relative`) so visually hidden text stays inside it; otherwise it stretches the page and adds a second, empty scrollbar. On mobile it appears at the bottom of the menu drawer, one tap from any page. The header holds only the application identity (the approved IIT logo at 40px tall, original aspect ratio, beside the portal name) and, on mobile, the menu control.

**Tabs:** use a tab strip (`.tabs` and `.tab`) to move between sibling pages of one section that each have their own address, for example the notification screens. Each tab is a real link inside a labelled `nav` and `ul`, so back, bookmarks, deep links, and lazy loading keep working. The current tab is marked with `aria-current="page"`, shown by a brand-primary underline and bold brand-primary text (never color alone), and stays current on that tab's own sub-pages. Do not use `role="tablist"` or `role="tab"` for page links: that pattern is for panels within one page and needs roving focus and arrow keys. Tabs are keyboard-focusable links with the standard focus ring, at least 44px tall on touch screens, and wrap onto more lines on narrow screens. Do not use buttons for navigation between pages.

Role-specific navigation may expose different destinations, but must use the same component, spacing, active-state, and naming conventions. The server remains authoritative for authorization; navigation visibility is not a security boundary. Use breadcrumbs for deep hierarchies. On mobile, use an accessible menu or drawer with a clear open/close control, sensible focus behavior, and enough touch space. Avoid duplicating the same destination in multiple navigation regions without a clear reason.

## 10. Authentication UI

Login, forgot-password, and password-reset screens MUST be focused, uncluttered, and visibly part of the IIT Academic Portal. Use the approved logo asset at its correct aspect ratio and the approved brand token; do not add unapproved decorative brand colors. Place the form in a clear content region with a prominent heading, persistent labels, readable help/error text, and one primary action.

**Brand header:** every focused authentication screen (sign-in, forgot password, reset password, and page not found) opens with the same brand header:
- The approved IIT logo (the "IIT" lettermark above a "University of Dhaka" bar, in brand primary on a white background), 80px tall at its original 600×327 aspect ratio. Never stretch or recolor it.
- The text alternative "IIT, University of Dhaka".
- "IIT Academic Portal" set in the heading size, semibold, in brand primary.
- Both centered horizontally in the card, with 24px (`--spacing-6`) below the header.

The page heading, form, and actions stay left-aligned for scanning. Implement the header once as a shared component rather than repeating it on each page.

**Password fields:** every password input (sign-in, reset password, account setup, and their confirmations) has a show/hide button inside its right edge: an eye icon that becomes an eye with a slash while the password is shown. It is a real 44px button that keeps the shared focus ring, is named for what it will do ("Show password" / "Hide password", or "Show new password" and so on), reports its state with `aria-pressed`, and never submits the form. The password starts hidden every time the screen opens. In the frontend, wrap the input and `<app-password-toggle>` in `.password-field`.

**Favicon and app icons:** derive them from the approved logo, without recoloring it or changing its proportions.
- **Browser tab** (`favicon.ico`, 16, 32, and 48px): the "IIT" lettermark only, cropped above the "University of Dhaka" bar because the bar text cannot be read at these sizes. Center it at 92% of the width on a white rounded square (corner radius 18%), which keeps it legible on light and dark browser chrome.
- **Home screen** (`apple-touch-icon.png`, 180px): the full logo at 84% of the width on a square white tile. The platform applies its own corner mask.

Regenerate both files whenever the logo changes.

Show loading and validation states without moving controls unexpectedly. Authentication errors MUST be actionable and MUST NOT expose secrets or sensitive account-existence details. Password visibility controls, if provided, require accessible labels and state announcements. If a user can choose or switch among roles, present only the roles authorized for that user and clearly identify the active role; do not imply that a UI selection grants permission.

## 11. Dashboard Patterns

Dashboards MUST begin with a clear page heading and prioritize the most relevant summary information. Use summary cards only for concise, comparable values; avoid decorative charts or dense card grids without a clear reading purpose. Recent activity, notifications, and quick actions should each have clear headings and predictable placement. Use the same card, typography, and spacing rules as other pages.

For a template-informed dashboard, use Tailwind's generated `grid grid-cols-12 gap-4` utilities with responsive `col-span-*` utilities for widget placement. Use `.full-container` only to fill its parent; it does not remove a max-width constraint. Build summary widgets from `.card` and its direct `.card-body`, with `.card-title`, `.card-subtitle`, and `.badge-md` or `.badge-pill` as needed. Follow the reusable list and badge recipes in Section 6. Replace template palette classes such as `text-primary`, `bg-white`, or `text-dark` with the IIT color tokens in Section 3. Do not assume the template's ApexCharts or Tabler icon classes are available; adding their packages requires separate approval.

Provide loading, empty, and error states for each dashboard region. Do not define role-specific dashboard content in this document. Avoid duplicate summaries that compete with the primary content and avoid placing cards inside cards.

## 12. Feedback and System States

| State | Presentation rule |
|---|---|
| Success | Confirm what completed in plain language. Use the success token and an icon or label; use a toast only for non-critical confirmation. |
| Error | State what failed and the next useful action. Keep important errors visible; never expose internal diagnostics. |
| Warning | Explain the risk or condition that needs attention. Do not present routine information as a warning. |
| Information | Provide neutral context without implying success or failure. |
| Loading | Identify the region or action in progress; keep layout stable and prevent repeated submission when applicable. |
| Empty | State why there is no content, distinguishing no data from no matches; show a relevant next step only if one exists. |
| Unauthorized | Use a clear access-denied message without disclosing restricted data; provide a safe navigation or support path where appropriate. |
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

**Frontend implementation:** `iit-academic-portal/src/styles/tokens.css` declares these tokens, with the same names, as the Tailwind CSS v4 theme. Each token is therefore also a utility (`bg-brand-primary`, `p-4`, `rounded-md`, `md:`). Tailwind's default color palette is removed, so only the colors in this document compile. Change a value here and in that file together.

```css
/* Confirmed brand values: primary #2E3192; secondary #4B4FB0. */
:root {
  /* IIT brand colors supplied by the project. */
  --color-brand-primary: #2E3192;
  --color-brand-secondary: #4B4FB0;
  --color-brand-accent: var(--color-brand-primary);

  /* Semantic color values require contrast review. */
  --color-background: #F8F9FC;
  --color-surface: #FFFFFF;
  --color-surface-raised: #FFFFFF;
  --color-text-primary: #111827;
  --color-text-secondary: #4B5563;
  --color-text-disabled: #9CA3AF;
  --color-border: #D1D5DB;
  --color-control-border: #6B7280;
  --color-divider: #E5E7EB;
  --color-success: #15803D;
  --color-warning: #B45309;
  --color-error: #B91C1C;
  --color-info: #0369A1;
  --color-focus: #2E3192;

  /* Interaction variants are centrally derived or approved. */
  --color-brand-primary-hover: #252873;
  --color-brand-primary-active: #1A1C52;
  --color-surface-hover: #F2F3FA;
  --color-surface-selected: #EEEFF8;

  /* Typography */
  --font-family-primary: "DM Sans", "Noto Sans Bengali", sans-serif;
  --font-size-page-title: 1.5rem;
  --font-size-heading: 1.125rem;
  --font-size-body: 0.9375rem;
  --font-size-compact: 0.875rem;
  --font-size-helper: 0.8125rem;
  --font-weight-regular: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --line-height-heading: 1.25;
  --line-height-body: 1.5;

  /* Spacing */
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-3: 0.75rem;
  --spacing-4: 1rem;
  --spacing-6: 1.25rem;
  --spacing-8: 1.5rem;
  --spacing-12: 2rem;
  --spacing-16: 2.5rem;

  /* Shape, borders, elevation, and control sizing */
  --radius-sm: 0.25rem;
  --radius-md: 0.4375rem;
  --radius-lg: 0.5rem;
  --border-width: 1px;
  --shadow-overlay: 0 4px 12px rgba(17, 24, 39, 0.08), 0 1px 3px rgba(17, 24, 39, 0.06);
  --control-height-sm: 2rem;
  --control-height-md: 2.5rem;
  --control-height-lg: 2.75rem;

  /* Layout */
  --content-max-width: 80rem;
  --sidebar-width: 15rem;
  --header-height: 4rem;
  --breakpoint-sm: 40rem;
  --breakpoint-md: 48rem;
  --breakpoint-lg: 64rem;
  --breakpoint-xl: 80rem;
}
```

Use the template-informed compact shape scale: 0.25rem (4px) for small controls and 0.4375rem (7px) for standard cards/containers; do not exceed 0.5rem (8px) for routine controls or cards. Shadows MUST be subtle and limited to overlays that require elevation. Keep token scales stable; do not add near-duplicate values for one-off component styling. Document and review any token changes because they affect all modules.

## 16. UI Consistency Rules

- Do not introduce arbitrary or unapproved brand colors; use the centralized semantic tokens.
- Reuse established components and patterns before creating a new visual pattern.
- Use DM Sans and the template-informed compact sizes/radii through centralized tokens; do not copy template CSS or add its vendor packages.
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

1. Verify the supplied neutral, interaction, status, and control-border colors against actual component states and adjacent surfaces during implementation.
2. Confirm that the suggested layout dimensions and breakpoints fit the supported devices and existing frontend conventions.
3. Review existing authentication, shell, and audit screens against the template-informed typography, sidebar, header, list-group, and surface guidance at supported breakpoints; retain the IIT color tokens.
