# Verdant Sales Dashboard — Technical Specification

## 1. Overview
A single-page, dark-themed sales analytics dashboard that visualizes revenue, profit, and regional performance from a static CSV dataset. The design language is “black & emerald” — deep charcoal backgrounds with vivid green accents and nature-inspired chart colors.

**Target rebuild:** Any developer should be able to recreate the exact UI and behavior using this document, the provided CSV schema, and standard React + Tailwind + Recharts tooling.

---

## 2. Tech Stack
| Layer | Technology |
|-------|------------|
| Framework | TanStack Start (React 19 + Vite) |
| Styling | Tailwind CSS v4 (`@theme` inline tokens) |
| Charts | Recharts (`recharts`) |
| Icons | `lucide-react` |
| Fonts | Google Fonts — Space Grotesk, Inter, JetBrains Mono |

---

## 3. Data Source
### 3.1 File
`src/data/sales.csv` — imported as a raw string (`?raw`) then parsed at runtime.

### 3.2 CSV Columns
| Column | Type | Description |
|--------|------|-------------|
| `Customer Name` | string | Account name |
| `Product Category` | string | Category: Software, Hardware, Electronics, Accessories, Services |
| `Item Description` | string | SKU / line-item description |
| `Quantity` | integer | Units sold |
| `Unit Price` | number | Price per unit |
| `Total Sale` | number | `Quantity × Unit Price` |
| `COGS` | number | Cost of goods sold |
| `Profit` | number | `Total Sale − COGS` |
| `Profit Margin %` | string | Percentage margin (e.g. `72%`) |
| `Date` | string (`YYYY-MM-DD`) | Transaction date |
| `Region` | string | Geographic region: Northeast, Southwest, Southeast, Midwest, West |

### 3.3 Parsed Data Interface (`SaleRow`)
```ts
interface SaleRow {
  customer: string;
  category: string;
  item: string;
  quantity: number;
  unitPrice: number;
  totalSale: number;
  cogs: number;
  profit: number;
  margin: number;      // parsed as float from "%" string
  date: string;        // ISO date
  region: string;
}
```

### 3.4 Computed Aggregates
All derived in `src/lib/sales-data.ts` and exported as constants:

| Export | Derivation |
|--------|------------|
| `sales` | `SaleRow[]` — full parsed array |
| `totalRevenue` | `Σ totalSale` |
| `totalProfit` | `Σ profit` |
| `totalCogs` | `Σ cogs` |
| `orderCount` | `sales.length` |
| `avgMargin` | `(totalProfit / totalRevenue) × 100` |
| `profitByCustomer` | Top 10 customers by `Σ profit`, descending |
| `salesByRegion` | All regions by `Σ totalSale`, descending |
| `monthlyRevenue` | Group by `YYYY-MM`, sum `revenue` and `profit`, sorted chronologically; month label formatted as `"Jan"`, `"Feb"`, etc. |

---

## 4. Design Tokens
All colors are defined in `src/styles.css` using `oklch()`.

### 4.1 Semantic Palette (`:root`)
| Token | Value | Usage |
|-------|-------|-------|
| `--background` | `oklch(0.16 0.012 160)` | Page background |
| `--foreground` | `oklch(0.96 0.01 150)` | Primary text |
| `--card` | `oklch(0.20 0.014 160)` | Card surfaces |
| `--card-foreground` | `oklch(0.96 0.01 150)` | Card text |
| `--popover` | `oklch(0.20 0.014 160)` | Tooltip background base |
| `--primary` | `oklch(0.78 0.19 145)` | Emerald accent (icons, badges, lines) |
| `--primary-foreground` | `oklch(0.14 0.02 160)` | Text on primary |
| `--muted` | `oklch(0.24 0.015 160)` | Subtle backgrounds |
| `--muted-foreground` | `oklch(0.70 0.02 150)` | Secondary text, axis labels |
| `--border` | `oklch(0.30 0.02 160 / 60%)` | Card borders, grid lines |
| `--success` | `oklch(0.78 0.19 145)` | Positive indicators |

### 4.2 Chart Palette (Pie / Categorical)
Stored in both `REGION_COLORS` (JS array) and matching CSS `--chart-1`…`--chart-5`:

| Index | Color | Description |
|-------|-------|-------------|
| 1 | `oklch(0.80 0.19 145)` | Emerald green |
| 2 | `oklch(0.75 0.16 195)` | Teal / cyan |
| 3 | `oklch(0.82 0.15 85)` | Amber / gold |
| 4 | `oklch(0.60 0.14 160)` | Deep forest green |
| 5 | `oklch(0.72 0.13 35)` | Copper / terracotta |

### 4.3 Gradients
- **Revenue line gradient** (`id="rev"`): `oklch(0.82 0.20 145)` → `oklch(0.55 0.15 165)`
- **Profit bar gradient** (`id="barFill"`): `oklch(0.55 0.15 165)` → `oklch(0.82 0.20 145)`
- **Page background**: Two fixed radial gradients (see §6.2)

### 4.4 Typography
| Role | Font | Weight |
|------|------|--------|
| Display / Headings | Space Grotesk | 500–700 |
| Body / UI | Inter | 400–600 |
| Mono / Labels | JetBrains Mono | 400–500 |

Letter-spacing on headings: `-0.02em`. Labels use `uppercase tracking-widest`.

---

## 5. Layout Architecture
### 5.1 Responsive Grid
| Section | Mobile | `sm` (640px+) | `lg` (1024px+) |
|---------|--------|---------------|----------------|
| Stat cards | 1 column | 2 columns | 4 columns |
| Charts | 1 column | 1 column | 3 columns |

### 5.2 Page Structure
```
<body>
  <div class="max-w-7xl mx-auto px-6 py-10 lg:px-10">
    <header>          <!-- Brand + live indicator -->
    <section>         <!-- 4 StatCards -->
    <section>         <!-- 3 ChartCards (2-row grid) -->
      <ChartCard class="lg:col-span-2">   <!-- LineChart -->
      <ChartCard>                           <!-- PieChart -->
      <ChartCard class="lg:col-span-3">   <!-- BarChart -->
    <footer>
  </div>
</body>
```

### 5.3 Card Component Specs

#### `StatCard`
- **Wrapper**: `rounded-xl border border-border bg-card p-6`
- **Hover**: `hover:border-primary/60`, optional `ring-1 ring-primary/30` when `accent=true`
- **Glow**: Absolute-positioned blurred circle (`bg-primary/10 blur-2xl`) in top-right corner
- **Label**: Mono, uppercase, `text-muted-foreground`
- **Value**: Display font, `text-3xl font-semibold`
- **Delta** (optional): `text-primary text-xs`
- **Icon**: Rounded container `bg-primary/15 text-primary p-2.5`

#### `ChartCard`
- **Wrapper**: Same rounded card style as `StatCard`
- **Header**: Title (`font-display text-lg font-semibold`) + optional subtitle (`text-xs text-muted-foreground`)
- **Status dot**: Small `h-2 w-2` circle with `bg-primary` and `shadow-[0_0_12px] shadow-primary`
- **Body**: Responsive container (`width="100%"`), chart-specific height

---

## 6. Global Background & Surface
The body uses a fixed dual-radial gradient overlay on top of `--background`:

```css
background-image:
  radial-gradient(ellipse 80% 50% at 50% -10%, oklch(0.30 0.10 150 / 0.4), transparent),
  radial-gradient(ellipse 60% 40% at 100% 100%, oklch(0.25 0.08 145 / 0.3), transparent);
background-attachment: fixed;
```

---

## 7. Charts (Detailed Specs)

### 7.1 Monthly Revenue Trend — LineChart
**Container**: `ChartCard` with `lg:col-span-2`, chart height `300px`

| Property | Value |
|----------|-------|
| Chart type | `LineChart` |
| Data | `monthlyRevenue` |
| X-axis | Month name (`month`), stroke `oklch(0.70 0.02 150)`, fontSize `12` |
| Y-axis | Currency formatter `fmtMoney` (`$1.2k` shorthand), same stroke |
| Grid | `strokeDasharray="3 3"`, stroke `oklch(0.30 0.02 160 / 0.3)` |
| Tooltip | Custom `TooltipBox` with `fmtFull` (`$12,345`) |
| Legend | `wrapperStyle={{ fontSize: 12, paddingTop: 8 }}` |

**Lines:**
1. **Revenue** — `type="monotone"`, stroke `url(#rev)` (gradient), `strokeWidth={3}`, dots `r=4` filled `oklch(0.82 0.20 145)`, `activeDot r=6`
2. **Profit** — `type="monotone"`, stroke `oklch(0.55 0.15 165)`, `strokeWidth={2}`, `strokeDasharray="4 4"`, dots `r=3`

### 7.2 Sales by Region — PieChart
**Container**: `ChartCard`, chart height `300px`

| Property | Value |
|----------|-------|
| Chart type | `PieChart` |
| Data | `salesByRegion` |
| `innerRadius` | `55` |
| `outerRadius` | `95` |
| `paddingAngle` | `3` |
| Stroke | `oklch(0.20 0.014 160)`, width `2` |
| Cells | Mapped to `REGION_COLORS` array (index order) |
| Tooltip | Custom `TooltipBox` with `fmtFull` |
| Legend | `iconType="circle"`, `fontSize: 11`, `paddingTop: 8` |

### 7.3 Top Customers by Profit — BarChart
**Container**: `ChartCard` with `lg:col-span-3`, chart height `340px`

| Property | Value |
|----------|-------|
| Chart type | `BarChart` with `layout="vertical"` |
| Data | `profitByCustomer` (top 10) |
| X-axis | `type="number"`, currency shorthand formatter |
| Y-axis | `type="category"`, `dataKey="name"`, width `140`, fontSize `12` |
| Grid | Vertical only (`horizontal={false}`), same dash/stroke as line chart |
| Tooltip | Custom `TooltipBox` with `fmtFull` |
| Cursor | `fill: oklch(0.30 0.02 160 / 0.3)` |
| Bars | `dataKey="profit"`, fill `url(#barFill)`, `radius={[0, 6, 6, 0]}` |

### 7.4 Tooltip Formatting Helpers
```ts
const fmtMoney = (n: number) => n >= 1000 ? `$${(n / 1000).toFixed(1)}k` : `$${n.toFixed(0)}`;
const fmtFull = (n: number) => `$${n.toLocaleString("en-US", { maximumFractionDigits: 0 })}`;
```

Tooltip container style:
- Background: `bg-popover/95` with `backdrop-blur`
- Border: `border border-border`
- Shadow: `shadow-xl`
- Text: `text-xs`

---

## 8. Interactivity & Filtering

### 8.1 Implemented Interactivity
- **Hover states**: Stat cards border brightens (`hover:border-primary/60`)
- **Chart tooltips**: Custom styled box showing exact values on hover for all three charts
- **Active dots**: Revenue line dots enlarge on hover (`activeDot r=6`)
- **Bar chart cursor**: Semi-transparent vertical highlight on hover

### 8.2 Filtering (Explicitly NOT Implemented)
The dashboard is **static / read-only**. There is no:
- Date range picker
- Region or customer filter
- Category drill-down
- Live data refresh or polling
- Search

If adding filters, they would operate on the `sales` array before the aggregate exports are computed.

---

## 9. Rebuild Checklist

### 9.1 Files to Create
```
src/data/sales.csv                 # populate with full dataset
src/lib/sales-data.ts              # CSV parser + aggregate exports
src/components/dashboard/StatCard.tsx
src/components/dashboard/ChartCard.tsx
src/routes/index.tsx               # Dashboard page route
src/styles.css                    # oklch theme tokens + fonts
```

### 9.2 NPM Dependencies
```bash
bun add recharts lucide-react
```

### 9.3 Fonts (Google Fonts link in `head()`)
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&family=Space+Grotesk:wght@500;600;700&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
```

### 9.4 SEO / Meta
- Title: `Verdant — Sales Analytics`
- Description: `Real-time sales, profit and regional performance dashboard.`

---

## 10. Data Snapshot (Quick Reference)
| Metric | Value |
|--------|-------|
| Total Revenue | ~$730,000 |
| Total Profit | ~$450,000 |
| Orders | 90 |
| Avg Margin | ~62% |
| Date Range | Jan 2026 – May 2026 |
| Regions | 5 (Northeast, Southwest, Southeast, Midwest, West) |
| Top Customer | Stark Industries (software license) |

---

*Document version: 1.0 — Generated for the Verdant Sales Dashboard rebuild.*
