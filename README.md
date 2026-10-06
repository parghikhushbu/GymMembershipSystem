# Gym & Fitness Center — Membership Subscription System

ASP.NET Core MVC (net10.0) + EF Core + SQLite. No client-side framework and no
JavaScript: every screen works with scripting switched off.

## Running it

```bash
cd GymMembershipSystem
dotnet run
```

The database is created and seeded on first run (`db.Database.EnsureCreated()`).
**If you already have a `gym.db` from an earlier version, delete it** — the
schema gained a column (`Subscription.IsRenewal`) and `EnsureCreated` will not
migrate an existing file.

Seed data covers six months of members, subscriptions and payments, so every
report has something to show the moment you open it. Some seeded memberships are
deliberately set to expire in the next few days to exercise the expiry report.

No login system: the "signed-in" member is held in Session and defaults to the
first seeded member. Swap `MemberAreaBaseController.GetCurrentMemberIdAsync()`
for an Identity lookup when you add real authentication.

---

## Requirement coverage

### Core transactions

| Requirement | Where it lives |
| --- | --- |
| Plan selection (Monthly / Annual) | `Areas/Member/Controllers/PlansController` → `Views/Plans/Index`. `PlanType` drives the badge and card colour. Only `IsActive` plans are offered. |
| Personal trainer add-on cart | `Areas/Member/Controllers/CartController` — add, remove, clear, with a duplicate guard and a unique index on `(MemberId, TrainerId)`. Inactive trainers are rejected. |
| Payment gateway integration | `Areas/Member/Controllers/PaymentController` → `Checkout` / `Confirm` / `Success`. Simulated gateway: card, UPI or net banking, validated against an allow-list, producing a `Payment` row with a transaction reference. |
| Subscription renewal payment | `Renew(subscriptionId)` on `PaymentController`, surfaced as a button on the member profile. `SubscriptionService.CheckoutAsync` stacks the new term on top of any days left on the current one, closes off the superseded subscription, and flags the new row `IsRenewal`. |

### Dashboard reports

| Report | Where it lives |
| --- | --- |
| Memberships expiring in 7 days | `Admin/DashboardController` — date-window query, rendered with the member's email so the front desk can call them. |
| Payment collection breakdown | `GroupBy` over plan name with `Count()` and `Sum()`, drawn as a CSS meter chart. |
| Active vs inactive member statistics | Counts plus a share bar. Statuses are recomputed on load rather than trusted (see below). |
| Extra: six-month revenue, renewal rate, plan popularity, payment-method split, member growth, recent transactions | `Admin/ReportsController` |

### Syllabus stack

- **Areas (Admin / Member)** — both registered, area route mapped before the
  default route in `Program.cs`. Each area has its own `_ViewImports`,
  `_ViewStart` and layout.
- **Custom master layout** — three of them: `Views/Shared/_Layout` (public),
  `Areas/Member/Views/Shared/_MemberLayout` (member, with a live cart badge),
  `Areas/Admin/Views/Shared/_AdminLayout` (staff, with a sidebar that marks the
  current section).
- **LINQ aggregation** — `GroupBy` / `Sum` / `Count` / `Average` across the
  dashboard, reports, plan list (live members per plan) and member profile
  (lifetime spend). Nothing is stored as a running total.
- **TempData** — every write action redirects with `TempData["SuccessMessage"]`
  or `TempData["ErrorMessage"]`, rendered by all three layouts. The payment
  receipt uses `TempData.Peek` so a refresh doesn't blank the page, and
  `Payment/Success` redirects away if there's no message to show.
- **Photo upload** — `Services/PhotoService` validates extension and size, writes
  to `wwwroot/uploads/{members,trainers}` under a GUID filename, and deletes the
  old file when one is replaced.

### Things that now actually work

`SubscriptionService.RefreshStatusesAsync()` runs before any status-dependent
screen: it expires lapsed subscriptions and syncs each member's Active/Inactive
flag to whether they hold a live subscription. Without it the "active vs
inactive" report was reporting a flag nobody ever updated.

Also added: member payment history, admin member search and status filter, manual
member suspend/reinstate, plan on/off sale toggle, trainer pause and delete (which
clears the trainer out of any member carts first), and photo removal.

---

## Front end

One stylesheet: `wwwroot/css/site.css`. No framework.

**Colour carries meaning.** Six hues, each with a fixed job, so the same colour
always says the same thing:

| Hue | Means |
| --- | --- |
| violet | brand, primary actions, membership plans |
| cyan | trainers and add-ons |
| lime | active, paid, healthy |
| amber | expiring soon, needs attention |
| rose | expired, inactive, destructive |
| indigo | money and totals |

Tokens live in `:root` — change them there and the whole site follows. Display
type is Outfit, body text is Inter, both from Google Fonts with a system
fallback stack if the machine is offline.

**Responsive behaviour**

- Nav collapses to a hamburger below 860px using a hidden checkbox and a sibling
  selector — no JavaScript, still keyboard reachable.
- Data tables reflow below 780px: the header row is visually hidden and each row
  becomes its own labelled card. This depends on every `<td>` carrying a
  `data-label` matching its column heading — **add one to any new cell.**
- The admin sidebar becomes a horizontally scrolling tab row below 920px.
- Card grids use `auto-fill` + `minmax`, so no breakpoints are needed.
- Type scales with `clamp()`.

**Accessibility floor.** Skip link on every layout, one visible focus ring,
44px+ touch targets, 16px form inputs (stops iOS zooming on focus),
`prefers-reduced-motion` respected, alerts announced with `role="status"` /
`role="alert"`, charts carry text alternatives.

**Charts** are pure CSS — `.bar-chart` for monthly series, `.meter` for ranked
comparisons, `.split-bar` for a two-way share. No charting library.

**Component classes.** `.panel`, `.card` (+ `.capped` and a `cap-*` hue),
`.card-grid`, `.stat-box` (+ `tile-*`), `.badge` (+ `badge-*`), `.table-wrap`
with `.data-table`, `.form-block`, `.radio-cards`, `.filter-bar`, `.empty`,
`.page-head`, `.btn` (+ `-heat` / `-cool` / `-fresh` / `-secondary` / `-ghost` /
`-danger` / `-sm` / `-block`).
