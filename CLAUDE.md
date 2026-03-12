# Claude Guidelines: Fishbowl iReport Development

## Environment & Toolchain

- **Report designer**: iReport 5.6.0
- **Rendering engine**: JasperReports (compatible with iReport 5.6.0)
- **Database**: MySQL 5.x (Fishbowl's embedded MySQL)
- **Source system**: Fishbowl Inventory Management
- **Barcode generation**: bwipjs API — `https://bwipjs-api.metafloor.com/?bcid=...`

---

## SQL Rules

- Always target **MySQL 5.x** syntax — no CTEs, no window functions
- Never start SQL output with comments — begin directly with `SELECT`
- Never change the capitalisation of existing parameters, field names, or aliases
- Use `$P!{}` (bang syntax) for parameters substituted directly into SQL strings (e.g. date ranges, IDs passed from Fishbowl modules)
- Use `$P{}` for JDBC-parameterised values
- **Exclude `$P!{}` parameters from `GROUP BY`** — they are pre-resolved constants, not columns
- Use `GROUP_CONCAT(DISTINCT NULLIF(field, '') SEPARATOR ', ')` pattern to collapse multi-row data (e.g. tracking numbers) while suppressing blank entries
- Custom field access pattern: use `CustomFieldByName('FieldName', recordId)` or the appropriate Fishbowl custom field view/function

### Common Fishbowl Tables

| Table | Purpose |
|-------|---------|
| `so`, `soItem` | Sales orders and line items |
| `product`, `part` | Products and parts master |
| `ship`, `shipItem`, `shipCarton` | Shipments, shipped items, cartons |
| `pick`, `pickItem` | Pick orders and picked items |
| `postso` | Posted SO records |
| `customer` | Customer master |
| `accountgroup` | Customer account groupings |
| `sysuser` | Fishbowl users (e.g. salesperson) |
| `carrier` | Shipping carriers |
| `WO` and related | Work orders / manufacturing |
| `location`, `locationgroup` | Warehouse locations |

### Standard Parameter Names (Fishbowl conventions)

| Parameter | Type | Usage |
|-----------|------|-------|
| `dateRange1` | `java.util.Date` | Start date |
| `dateRange2` | `java.util.Date` | End date |
| `shipID` | `java.lang.Integer` | Ship record ID |
| `pickNum` | `java.lang.String` | Pick order number |
| `soNum` | `java.lang.String` | Sales order number |
| `locationID` | `java.lang.Integer` | Location ID |

---

## JRXML Rules

### UUIDs
- **Always generate valid RFC 4122 UUIDs** for all `uuid` attributes — never use placeholder/fake values like `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
- Each band, field, element, and group must have a unique UUID

### Parameters
- Declare all parameters at the top of the JRXML with correct Java types
- Use `<defaultValueExpression>` where a sensible fallback exists
- `$P!{}` parameters: type as `java.lang.String` unless numeric computation is needed — but be aware Fishbowl may pass numeric IDs, so match the type to what the calling module actually sends

### Expressions
- Null-safe pattern: `$F{fieldName} != null ? $F{fieldName} : ""`
- Conditional formatting (e.g. red for negative): use `printWhenExpression` or a `<conditionalStyle>` with `<conditionExpression>`
- For boolean show/hide: `new Boolean($F{fieldName} != null && !$F{fieldName}.isEmpty())`

### Layout & Measurements
- Units are **points** (1 inch = 72 pts)
- Avery L7060 (21-up location labels): each label ~151.9 × 75.7 pts; verify top/left margins from client measurements — this is the #1 source of misalignment
- Avery L7148 (10-up part labels): each label ~198.4 × 113.4 pts; same margin caveat
- Always include a border rectangle in detail bands during development for alignment verification — remove or comment before final delivery
- Page margins: match physical sheet margins precisely; don't trust defaults

### HTML / Rich Text in Fields
- Use `markup="html"` on the `<textElement>` tag
- Line breaks must be `<br/>` — **not** `</br>` or `\n`
- Wrap expression content in `"<html>" + value + "</html>"` if needed

### Barcodes & QR Codes
- Use image elements with URL expression pointing to bwipjs API
- Example QR: `"https://bwipjs-api.metafloor.com/?bcid=qrcode&text=" + java.net.URLEncoder.encode($F{someField}, "UTF-8") + "&scale=3"`
- Example Code128: `"https://bwipjs-api.metafloor.com/?bcid=code128&text=" + $F{barcode} + "&includetext&scale=2"`
- Set `onErrorType="Blank"` on image elements to fail silently when field is null

### Subreports
- Pass parent parameters explicitly via `<subreportParameter>`
- Connection: use `$P{REPORT_CONNECTION}` to share the parent DB connection
- Prefer `GROUP_CONCAT` in the main query over subreports for simple aggregations (e.g. tracking numbers) — fewer moving parts

### Groups
- Group header/footer bands: set `isSplitAllowed="false"` on detail bands for label reports to prevent mid-label page breaks
- `isStartNewPage="true"` on group headers only when a page-per-group layout is required

---

## iReport 5.6.0 Quirks & Known Issues

- **Row repetition via cross-join number generators** (e.g. `mysql.help_topic`) may silently produce "document has no pages" — the iReport DB user may lack SELECT on system tables. Test the query directly in MySQL first to confirm it works, then check if iReport's configured connection user has the same privileges.
- **`$P!{}` in GROUP BY**: will cause a MySQL error — always exclude these from GROUP BY clauses
- **Parameter type mismatch**: if Fishbowl passes a numeric ship/pick ID but the parameter is declared as `java.lang.String`, the report may error or return no data. Match declared type to what the module actually sends.
- **iReport preview vs. production**: preview uses iReport's own DB connection; production uses Fishbowl's connection user which may have different permissions
- **Large JRXML edits**: use targeted/surgical patches rather than full-file regeneration to avoid UUID and structure drift

---

## Workflow & Delivery Preferences

- **Don't regenerate complete files** — provide targeted diffs/patches and prompt before doing a full rewrite
- **Test SQL in MySQL directly** before embedding in JRXML to isolate query vs. iReport issues
- **Don't add README files** unless explicitly requested — brief bullet-point summaries are preferred
- When creating new JRXML from scratch, provide the complete file with all valid UUIDs generated
- When patching existing JRXML, provide only the changed XML blocks with clear location context (e.g. "replace the `<field>` block for `trackingNumber`")
- Label reports: always confirm physical measurements with client before finalising margins

---

## Common Patterns (Reference Snippets)

### Null-safe field with fallback
```xml
<textFieldExpression><![CDATA[$F{customerName} != null ? $F{customerName} : "N/A"]]></textFieldExpression>
```

### Conditional red style for negative value
```xml
<conditionalStyle>
  <conditionExpression><![CDATA[$F{variance} != null && $F{variance}.doubleValue() < 0]]></conditionExpression>
  <style forecolor="#FF0000" isBold="true"/>
</conditionalStyle>
```

### GROUP_CONCAT tracking numbers suppressing blanks
```sql
GROUP_CONCAT(DISTINCT NULLIF(sc.trackingNum, '') ORDER BY sc.trackingNum SEPARATOR ', ') AS trackingNumbers
```

### bwipjs QR code image expression
```xml
<imageExpression><![CDATA["https://bwipjs-api.metafloor.com/?bcid=qrcode&text="
  + java.net.URLEncoder.encode($F{bomURL} != null ? $F{bomURL} : "", "UTF-8")
  + "&scale=3"]]></imageExpression>
```

### printWhenExpression — show only if field has value
```xml
<printWhenExpression><![CDATA[$F{bomURL} != null && !$F{bomURL}.trim().isEmpty()]]></printWhenExpression>
```
