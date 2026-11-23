# Zoho Books custom fields: how they work, where they show, what users complain about, and what it means for Comsky Books

> **Status:** research and design input only. Nothing here is built. Written 2026-10-21 for the owner's request (item 3:
> custom fields per object, mandatory or not, a field type, and "where it is visible", including whether a field added on a
> voucher type should appear in reports; items 1 and 2, the report grid and quick-saved views, depend on the answer).
> **Method:** Zoho's own public help pages, knowledge-base (KB) pages, API reference pages, monthly product-update posts and
> community-forum threads, read through a web-fetch tool that **summarises each page with a small model**. That means detail can
> be dropped: where I write "not stated on the page I read", that is all it means, not that Zoho lacks the feature. Search-engine
> summaries are tagged separately. Repo facts were read from the working tree, which was not modified. Everything is paraphrased;
> no Zoho text or screenshot is reproduced (one short quote, section 2.6).
> **Evidence tags:** **[W]** a Zoho page I fetched and read (via the summarising tool). **[Ws]** a search-result summary only: I
> did not read the page, or the fetch showed only the opening post of a forum thread. **[C]** read in this repo's code (file
> named). **[J]** my judgement. **[unverified]** I looked and could not confirm it. A forum post with no date shown is marked
> "undated"; most are.
> **Sibling doc:** `docs/research/zoho-reporting-tags.md` was untracked in the tree when I looked (another lane). I have not read
> it. Section 3 here keeps only the reporting-tag facts that bear on custom fields, so the two may overlap; where they differ,
> check the page URL in section 5.

---

## 0. Read this first

### 0.1 What I could not check

1. **No live Zoho Books organisation.** Nothing below was seen on a screen. The same is true of Tally and Pipedrive: this is a
   map of documents, to be confirmed in a trial organisation before a spec is locked.
2. **Zoho's help changes monthly and varies by region.** Two regional plan-comparison pages (Australia, India) disagree on
   per-type field counts (section 2.10). Older KB pages describe form options that the current help page does not list
   (section 2.6). The monthly product-update posts for May to October 2026 add custom-field features nearly every month.
3. **Forum threads mostly came back as an opening post only**; replies, staff answers and dates are known to me only through
   search summaries **[Ws]**, and I say so each time. Two "limit reached" threads do not name the Zoho product (it may be
   Zoho Inventory or another), so they prove a pattern, not a Books number. One line-item thread is about Zoho Invoice, the
   sibling product.
4. **No documented list of which modules take custom fields** on any page I read (section 2.1).
5. **Zoho "profitability tag": not found as a name.** What the owner describes (a tag created once, attached to many
   transaction types, reports filtered and compared by it) is **Reporting Tags**. Zoho's own help example for them is a travel
   agency comparing profitability by destination **[W]**. A Zoho Projects-side profitability feature is a different thing and
   was out of scope **[unverified]**.
6. **What an inactive or deleted dropdown option does to old records:** not stated on the page I read **[unverified]**.
7. **Whether a hidden-by-role field also disappears from reports and exports:** not stated **[unverified]**.
8. **The "View Associated Components" feature**, which my first search placed in June 2026, is in the **July 2026** post **[W]**;
   its exact screen is not described.

### 0.2 The ten findings that change what we build

1. **Zoho runs two tools, not one.** A *custom field* is a typed attribute of one module (a customer, an item, an invoice, a line
   of an invoice). A *reporting tag* is a company-level dimension: a named list of options created once, attachable to many
   record types and to lines, built to be filtered and compared in reports **[W]**. The owner's clothes-colour and zone cases
   are custom fields; his "profitability tag" is a reporting tag.
2. **A Zoho custom field has no "show in reports" setting.** The create form (label, type, help text, data privacy, unique,
   input format, default or inherited value, mandatory, "show in all PDFs") has no report switch **[W]**. Zoho decides
   report by report which fields a report can use, and that coverage has grown piecemeal: a custom field as a column of two
   reports in 2016, Chart of Accounts fields as Trial Balance columns in October 2026, sales-order fields as filters in one
   inventory report in September 2026 **[W]**. Users have asked for "reports on custom fields" since 2011 **[W]**. Zoho's answer
   to the owner's open question is therefore "neither: not automatic, not a per-field choice, report by report".
3. **"Where is it visible" is four separate controls in Zoho, none of them "report".** (a) the module's entry form, (b) role
   access (read and write, read only, hidden), (c) one yes/no "show in all PDFs", (d) portal display (custom modules, and
   attachments on invoices and quotes). Item fields add "show when creating transactions" and "show in transaction types"
   **[W]**.
4. **An item field becomes a column of the transaction's item table**, and the value is carried per line (the API has
   `item_custom_fields` per line). That is Zoho's answer to "Colour at item level". It is rolled out transaction type by
   transaction type (Delivery Challans and Inventory Adjustments only in October 2026) **[W]**.
5. **Mandatory means "cannot save or update without a value"** **[W]**. The page does not say what happens to a record that
   predates the field. Hidden consequence for us: see trap T1, section 4.3.
6. **Deleting is made deliberately hard:** a field must be inactive first, and if it holds data a Force Delete removes the field
   and all its data for good **[W]**. A "View Associated Components" view (July 2026) shows where a field is used **[W]**.
7. **Limits are per data type, per module and per plan, and users hit them** (for example five dropdown fields on a low plan)
   **[W][Ws]**. The two regional tables disagree, so any number is perishable.
8. **The API treats a custom field as `{ customfield_id, value }` plus an API name starting `cf_`**; an invoice can be updated
   or upserted through a unique custom field; definitions cannot be created through a documented API. The Custom Views API is
   the best model for a saved grid view I found: criteria that name custom fields, columns with width and a pinned flag,
   sharing, favourite and default **[W]**.
9. **Reporting tags (advanced mode)** sit at header or line level, can be mandatory, have a default option, nest five levels,
   show options conditionally, and can restrict which records a role sees (on one tag only); old mode had 10 tags with 500
   options and no mandatory flag (an undated Zoho post confirms the gap). Tags on lines cannot customise the Invoice Details
   report (use Receivable Details) **[W]**.
10. **Comsky's existing custom fields (UDF) have the right skeleton and four holes**: two objects only (ledger, stock item), a
    `required` flag nothing enforces, options that cannot be edited, and a delete that orphans stored values and then silently
    drops them on the next save of the master. Nothing outside the master form reads a custom value **[C]** (section 4.1). The
    cheapest high-value moves are in section 4.2.

---

## 1. How Zoho splits the work (one table)

| The need | Zoho's tool | Notes |
|---|---|---|
| An extra typed attribute on a record (Industry on a customer, Stock# on an invoice) | Custom field of that module | Per module, per data type, per plan limits. Can be printed on PDFs. |
| An extra column on the item table of a transaction | Custom field on the **Items** module with "show when creating transactions" on, optionally limited to some transaction types | The column then exists on those transactions; value per line **[W KB]** |
| An internal reporting dimension (region, department, branch) applied across many record types | Reporting tag | Filter and compare in reports; not for printing **[W]** |
| A saved set of columns and filters on a list page | Custom view | Shared with me, selected people, everyone; favourite **[W]** |
| A saved report layout | Custom report | Plan-capped count (10, 25, 50, unlimited) **[W]** |
| Restrict a value to a pattern | Input format (standard or regex) on a text field | Single and multi-line text only **[W]** |
| Enforce a business rule across fields | Validation rule | Premium and above; whether custom fields can be used in the criteria is not stated **[W]** |
| Show, hide, require or auto-fill a field depending on another | Layout rule | **Custom modules only** on the page I read **[W]** |

---

## 2. Custom fields in detail

### 2.1 Which objects take them

No page I read lists the supported modules **[unverified as a list]**. What the pages and posts together show **[W]**:

- **The help page** (examples only): Customers, Invoices, and custom modules. It routes every creation through the module's
  own Fields tab: Settings, the module, Fields, New Field.
- **Items** (Settings, Items, Field Customisation) and **Customers and Vendors** (Settings, Preferences, Customers and
  Vendors, Field Customisation) per KB pages. Customers and vendors have **three scopes**: fields for customers and vendors,
  fields for customers' contact persons, and fields for vendors' contact persons.
- **History:** custom fields began on invoices and estimates in May 2011, three per module and text only **[W blog]**. An
  undated community announcement later added Expenses and Payments Received with a short type list (text, number, decimal,
  amount, auto number, check box) **[W]**.
- **Product posts, 2026 [W]:** credit-note date fields for the German edition (May); contact persons and lookup fields that
  can point at Journal or Account (September); Delivery Challans (customer fields carried over; item-level fields), Inventory
  Adjustments (item-level fields), bank deposits, locations and the Chart of Accounts (October).
- **Lookup page** names Items, Projects, Customers, Users, Invoices, Sales Orders, Purchase Orders and custom modules as places
  a lookup's related list appears **[W]**.
- **API:** the `custom_fields` array is documented on invoices and items **[W]**; line items carry their own (`item_custom_fields`
  is named in forum code **[Ws]**).

Reading: custom fields exist on essentially every master and transaction module, grown module by module; a user cannot assume a
new transaction type has them **[J]**.

### 2.2 The create form, option by option **[W]**

| Option | What it does |
|---|---|
| Label name | What users see on the record |
| Data type | See 2.3 |
| Help text | A hint beside the field |
| Data privacy | **PII** (sensitive: encrypted, visible only to users with PII access; not sensitive: unencrypted but searchable by users with protected-data access) for text, email, URL, phone, number, date. **ePHI** (always encrypted, not searchable) for text, email, URL, phone, date |
| Prevent duplicate values | Each value must be unique. Not available with data privacy on. Turning it on for a module with more than 10,000 records takes time and sends an in-app notification when done |
| Input format | Restricts the characters and length of a single-line or multi-line text field: standard formats (digits; alphanumeric with or without spaces; alphanumeric with hyphen and underscore; letters with or without spaces) or a regular expression (the example is a national insurance number) |
| Default value | Pre-fills when nothing is entered. Or **inherit** the default from a field of a related module (a customer field into an invoice field). Not with data privacy. With "prevent duplicates" the user must still type a unique value |
| Is mandatory | Yes: the record cannot be saved or updated without a value. No: optional |
| Show in all PDFs | Yes: the field prints on transaction PDFs. One switch for all PDFs, not per template or per transaction type |

After saving, the field has an **API field name** (found by editing it) used in API calls.

Not on this page **[W]**: any "show in" placement option, per-transaction-type scope (that is in the item KB, 2.6), report
visibility, import or export, templates beyond PDFs.

### 2.3 Field types **[W]**

21 standard types and 3 AI types.

| Type | Notes |
|---|---|
| Single-line text | Up to 255 characters |
| Multi-line text | Up to 36,000 characters, formatting excluded |
| Email, URL (optional hyperlink label), Phone | Plain validation |
| Number, Decimal, Percent | No ranges stated |
| Amount | Uses the organisation's currency code |
| Date | Fixed or relative (Today, Tomorrow, first of month) |
| Date and time | Shown YYYY-MM-DD HH:MM |
| Check box | Can default to ticked |
| Auto-generate number | Starting number, optional prefix and suffix, increments by 1. "Apply to existing invoices" is a one-time choice that **cannot be changed later** |
| Dropdown | One option from a list (2.5) |
| Multi-select | Up to **30** options; same set-up as dropdown |
| Lookup | Links one record of another module; shows as a related list on the other side |
| Multi-select lookup | Many-to-many; chosen through a pop-up or a dropdown |
| External lookup | Pulls one field from another Zoho app |
| Attachment | One file under 7 MB; on invoices and quotes you can choose whether customers see it in the portal; only the file name shows on the details page |
| Image | Up to 5 MB (JPG, JPEG, PNG, BMP); shows on the details page; printable through a placeholder |
| Formula | Output type chosen; built from functions, fields and operators; the page says the result shows on the **details page, not on create or edit** |
| AI: keyword extraction, sentiment analysis, image to text | Values created when a record is created or edited; need Settings, AI Preferences; Premium and above |

### 2.4 Mandatory, default, validation, unique

- **Mandatory** is a single yes/no per field. There is no "mandatory only when" on standard modules; conditional mandatory exists
  only as layout rules on **custom modules** **[W layout-rules]**. The reporting-tag workaround post (2.13) shows users
  routing around it with a dropdown plus a script.
- **Default** can be static or inherited from a related record's field, a lightweight form of "Zone follows the customer".
- **Validation:** only the input-format pattern on text fields lives on the field. Cross-field rules are **validation rules**
  (Premium and above; discount cap, required terms, date conditions as examples; an alert blocks saving). The page does not say
  whether custom fields are usable in the criteria **[W]**. July 2026 extended validation rules to credit notes and vendor
  credits, October 2026 to bank deposits, Delivery Challans (India) and two inventory documents **[W]**.
- **Unique** fields are a premium feature (4 allowed on Premium and up on the pricing page; November 2023 note **[Ws]**) and
  they unlock "update by this field" in the API (2.12).

### 2.5 How options are defined and edited (dropdown and multi-select) **[W]**

Add one at a time (hover and `+`), paste several (one per line), or insert a predefined set (days of the week, months,
countries). Each option can be deleted or marked active or inactive from its menu. Optional colour per option, shown beside the
option or wrapped around it. The page does not mention a default option for a standard dropdown, and does not say what an
inactive option does to records that already hold it **[unverified]**. A consultancy article about reporting tags (3.8) says
deleting a tag option removes it from future entry but leaves it on past records **[W: third-party article, see 3.4]**; I found no Zoho
statement for dropdown options.

### 2.6 Where a field is visible, and who may see it

| Axis | Control | Evidence |
|---|---|---|
| Entry form | The module the field belongs to. No layout builder for standard modules was described; **layout rules** and a form builder exist for custom modules only | [W] |
| Which transaction types (item fields only) | KB: Settings, Items, Field Customisation, New Custom Field: "Show when creating transactions" (Yes/No) and "Show in transaction types" (pick which) | [W KB]. The **current help page's create-form list does not include these two options**, so the screen may have changed **[unverified]** |
| PDFs | "Show in all PDFs": one yes/no. An item field with it on prints its value under the item in the item table; empty prints nothing | [W] |
| Roles | Configure Access per role: read and write, read only, hide. All roles have access by default; the Admin role always has full access and cannot be restricted | [W] |
| Customer or vendor portal | "Display in portal" and "Edit in portal" on **custom module** fields; not supported for attachment, lookup, external lookup fields. Standard-module portal settings: not found. (The data-types page separately gives an attachment on invoices and quotes a show-or-hide-in-portal choice [W].) | [Ws] for the custom-module toggle names and the unsupported types |
| Reports and list columns | **Not a field setting.** Report by report, see 2.8 | [W] |
| Privacy | PII and ePHI flags change who can see and search the value | [W] |

The one short quote worth keeping, from the item KB on the column behaviour: it says to create a custom field in the Items
module "to add a new column to the item table in transactions".

### 2.7 Item-level (per-line) fields

- **How Zoho does it** **[W KB]**: create the field on the Items module with "show when creating transactions" on, pick the
  transaction types, optionally show on PDFs. It appears as a column of the item table on those transactions. A community answer
  (search summary **[Ws]**) says these columns sit before Quantity and that only they can be reordered.
- **Value is per line.** Forum code names an `item_custom_fields` array on each line, and a developer thread shows a line
  value being set on an invoice rather than on the item master **[W thread; it concerns Zoho Invoice, the sibling product]**.
  Whether a line's value **defaults from the item master's value** is not stated **[unverified]**.
- **Roll-out is per transaction type**: October 2026 added item-level fields to Delivery Challans (India) and Inventory
  Adjustments, and customer field values now flow into Delivery Challans automatically **[W]**.
- **Gaps users reported** (all undated, **[W]** threads except where marked): line custom fields are described as hard to
  reach from a script (a Deluge `invoice.get("line_items")` did not expose one) **[Ws]**; an API error reports a **100-character**
  cap on a line custom field that invoice-level fields do not have (sample date 2018); a feature request says lines of the
  same item are merged by a commerce flow and the variation in custom values is lost (the post speaks of a commerce
  product, so it may not be Books); and a user asked for per-line dates and could not find an option **[Ws]**.
- **Attachment and image fields now carry over when a record is converted or cloned** (July 2026) **[W]**, which implies they
  did not before.

### 2.8 Can a custom field be a column or a filter? Surface by surface

| Surface | Finding | Evidence |
|---|---|---|
| **Custom Reports builder** (parent module plus child modules; general filters; show/hide columns; layout; permissions) | Page says nothing about custom fields as columns, group-by or filter fields. Reporting tags appear only as an advanced filter in the "from an existing report" flow | [W] |
| **Which reports can be customised** | Two Zoho pages disagree: the manage-reports page says only Profit and Loss, Balance Sheet, Cash Flow and Invoice Details; the custom-reports page lets you start from any report; a KB page lists some forty reports under two headings ("save as custom reports" and "reporting tags") without saying which is which. Plan-gated | [W] |
| **Invoice Details, Expenses by Customer** | A November 2016 release note says a custom field (example: a customer's expense limit) can be added as a column by customising the report | [W] |
| Same report, staff reply on the forum | Custom fields shown as separate columns and usable as filters in Invoice Details | [Ws] |
| **Receivable reports** | Show/Hide Columns lets you pick "fields under Customer"; whether customer custom fields are in that list is not stated | [W] |
| **Trial Balance** | Chart of Accounts custom fields can be columns (October 2026) | [W] |
| **Order Fulfillment By Item** (Inventory add-on) | Sales-order custom fields are available as advanced filters (September 2026) | [W] |
| **Custom views (list pages)** | Help page silent. **API accepts custom fields in view criteria** (`${invoice.cf_text_field}`, `${PLACEHOLDER.EMPTY}` for blank). Forum: a user believed custom fields could not be list columns (no reply visible); a roughly five-year-old staff reply says they appear in Customize Columns, and multi-select, multi-line and attachment types cannot be columns | [W] API; [W] thread; [Ws] reply |
| **Zoho Analytics** | A user's invoice check-box field was not offered in an Analytics report (no answer shown) | [W] |
| **Sales and payments reports** | A user exports the report and the customer list and joins them with VLOOKUP to get a customer field | [W] undated |
| **Export** | Export templates pick the fields; the page does not mention custom fields; only the first 25,000 rows export | [W] |

**What this says:** Zoho never made "a field exists, so every report can use it" true. Each report got custom-field columns
when someone built them. That is the pattern Comsky should not copy (section 4.3, T2).

### 2.9 Zoho's own history on reporting by custom field **[W]**

The May 2011 launch post and its comments: users immediately asked for reports and search on custom fields, for more than
three fields, and for dropdown, date and check-box types. Zoho replied on 26 May 2011 that reporting on custom fields was not
possible and that votes decide priority. A user in October 2011 asked again, adding that field types would reduce entry errors.
Fifteen years later most of those asks are shipped, but the report side is still per report.

### 2.10 Limits per plan (they are tiered, regional and perishable)

The help page gives no numbers: limits apply per data-type group (numerical, decimal, string, boolean, long text, multi-select,
formula), vary by plan, and a **Custom Fields Usage** pop-up on each module's Fields tab shows what is left **[W]**. The two
plan-comparison pages I read give different counts **[W, tables summarised by the tool]**:

| Field group (each type counted separately where noted) | Australia page: Standard / Professional / Premium / Elite / Ultimate | India page: same order |
|---|---|---|
| Single-line text, email, URL, phone (each) | 20 / 20 / 50 / 50 / 50 | 20 / 20 / 50 / 50 / 50 |
| Number, date, date-time (each) | 12 / 12 / 30 / 30 / 30 | 12 / 12 / 30 / 30 / 30 |
| Decimal, amount, percent (each) | 4 / 4 / 10 / 10 / 10 | 10 / 20 / 20 / 20 / 20 |
| Dropdown | 5 / 5 / 20 / 20 / 20 | 10 / 25 / 25 / 25 / 25 |
| Check box | 3 / 3 / 3 / 3 / 3 | 10 / 20 / 20 / 20 / 20 |
| Attachment | 4 all | 4 / 10 / 10 / 10 / 10 |
| Multi-line text | 5 all | 5 all |
| Multi-select | 2 / 2 / 5 / 5 / 5 | 2 / 2 / 5 / 5 / 5 |
| Lookup | 5 all | 5 all |
| Formula | none / none / 2 / 5 / 5 | none / none / 2 / 5 / 5 |
| Unique fields | none / none / 4 / 4 / 4 | none / none / 4 / 4 / 4 |
| AI fields | none / none / 2 / 2 / 2 | none / none / 2 / 2 / 2 |
| Multi-select lookup | none / none / 1 / 1 / 1 | none / none / 1 / 1 / 1 |
| Auto-generate number | 1 all | 1 all |
| Custom views | every plan, including Free | every plan |
| Custom reports (count) | 10 / 25 / 50 / unlimited / unlimited | same |
| Validation rules | Premium and up | Premium and up |
| Workflow rules | none / 10 / 200 / 200 / 200 | not shown |
| Custom modules | Premium 10, Elite 10, Ultimate 25 | same |

Users hit the caps: a third decimal field returned "limit reached for this data type" and a staff member pointed to the
limits page (product not named; the staff reply is from a search summary **[Ws]**); another user with five of ten allowed fields
was told to upgrade (the URL slug suggests 2015; product not named). **[W]**

### 2.11 Lifecycle: edit, inactive, delete

- **Edit** from the Fields tab; "prevent duplicates" on a big module is applied in the background **[W]**.
- **Mark inactive** stops data entry in the field; **mark active** restores it. The page does not say whether inactive fields
  still print, still report, or still show old values **[unverified]**.
- **Delete** is offered on an **inactive** field. If records hold values a Force Delete prompt appears: delete those
  transactions first, or force, which removes the field and all its data permanently **[W]**.
- **View Associated Components** (July 2026) shows every place a field is in use **[W]**.
- **Auto-number** setup for existing invoices is one-time and not changeable **[W]**.

### 2.12 In the API

| Point | Finding | Evidence |
|---|---|---|
| Value shape | `custom_fields`: array of `{ customfield_id, value }` on create, update and get for invoices and items | [W] |
| Field identity | Numeric `customfield_id`, and an **API name** (`cf_...`). A forum answer says the API name can stand in for the id in Deluge line-item code | [W] id; [Ws] API name |
| Reporting tags on the same record | `tags`: request `{ tag_id, tag_option_id }`; response adds `tag_name`, `tag_option_name`, `is_tag_mandatory`; tags also appear inside each line item | [W] |
| Update or upsert by a unique field | `PUT /invoices` with headers naming the field's API name and the value (`X-Unique-Identifier-Key`, `X-Unique-Identifier-Value`); `X-Upsert: true` creates when none matches. Only fields set to prevent duplicates qualify | [W] |
| Search by field | The invoice list documents a `custom_field` search parameter with `_startswith` and `_contains` variants. A forum poster saw every invoice returned when filtering by a custom field value; an unanswered thread asks how the variants work | [W] docs; [Ws] conflict |
| Field endpoints | `GET /invoices/{id}/customfields` and `PUT /invoice/{id}/customfields` (note the singular) are listed; bodies not read | [W] index only |
| List versus detail | In a **Zoho Subscriptions** thread (same platform family), some `cf_` fields were present in the list response and others only in the detail response, which carried a `custom_field_hash` object. Not shown for Books | [W] thread; unverified for Books |
| Creating definitions | No documented API. A forum post reports an undocumented settings endpoint that worked when copied from the browser | [Ws] |
| Custom Views API | `POST/GET/PUT/DELETE /customviews`, reorder, `GET /advancesearch` (fields and operators). Criteria: a `rule` with `columns` (`index`, `field`, `comparator`, `value`) and a `criteria_string` such as `( 1 AND 2 )`. Columns: `select_columns` with `field`, `width`, `is_pinned`. Also `shared_with` (everyone, onlyme, selected), `is_favorite`, `is_default`, `column_orientation_type` (clip or wrap) | [W] |

### 2.13 Import, export, templates, automation

- **Import** takes CSV, TSV and XLS; a Map Fields step pairs file columns with fields and mappings can be saved. The migration
  guide tells users to create a custom field for any item attribute Zoho lacks, then map it; unmapped columns are not
  imported; the page does not discuss dropdown values that do not exist in the list **[W]**. A reporting tag imports as its own
  column whose header matches the tag's name **[W]**. Item reporting tags can be bulk-updated (June 2026) **[W]**.
- **Export** is by template of chosen fields, CSV, XLS or XLSX, 25,000 rows at most **[W]**.
- **PDF templates:** "show in all PDFs" is the standard-module control. The visual template editor page lists placeholders for
  header, footer, address formats and item name, without a list or syntax **[W]**. **HTML templates with `${...}` placeholders,
  including `TableCF.<module>.cf_x.Value` for table custom fields, are available for custom modules only** (August 2026)
  **[W]**. So a standard-module custom field prints only through the one switch **[J]**.
- **Workflow rules:** event or date triggers, up to 10 criteria with 10 conditions each; the page does not say custom fields
  are usable in criteria. A February 2025 product note says rules can now compare one field with another **[Ws]**. **Field
  updates** change a chosen field of a module when a rule fires (fixed value only; three per rule); custom fields not
  mentioned **[W]**.
- **Layout rules** (show, hide, enable, require, auto-fill by condition): custom modules only, and plan-gated **[W]**.

### 2.14 What users complain about

| Complaint | Where | Date | Evidence |
|---|---|---|---|
| "I want reports and search on my custom fields", "only three fields", "more types" | Zoho blog comments | May and Oct 2011 | [W] |
| A "Stock#" field on invoices cannot be put on the sales reports | Forum | undated | [W] opening post |
| A customer custom field is missing from Payments Received; fixed by exporting both and doing a VLOOKUP | Forum | undated | [W] |
| Custom fields cannot be columns in estimate or invoice custom views | Forum | undated | [W]; staff reply [Ws] |
| An invoice custom field is invisible to Zoho Analytics | Forum | undated | [W] |
| "Limit reached for this data type" on a third item field; "limit reached, upgrade" with five of ten used | Forum, product not named | undated (second slug suggests 2015) | [W] |
| A line-item custom field value is capped at 100 characters in the API; invoice-level fields are not | Forum | sample date 2018 | [W] |
| Lines of the same item merged, losing per-line custom values | Forum (commerce product) | undated | [W] |
| List endpoint omits some `cf_` fields that the detail endpoint returns (Subscriptions API) | Forum | undated | [W] |
| Reporting tags cannot be mandatory (old mode) | Zoho function post | undated | [W] |
| Reporting tags on invoice lines mean Invoice Details cannot be customised by tag | KB | undated | [W] |
| A "tag" cannot be assigned automatically from a rule | Blog comments | July 2016 | [W] |

---

## 3. Reporting tags, only as they bear on custom fields

(Full treatment: the sibling doc. Facts below are **[W]** unless marked.)

### 3.1 Two versions

| | Older flow | Advanced reporting tags (opt-in, one-way, cannot be turned off) |
|---|---|---|
| Count | Up to **10 tags**, up to **500 options** each | Option counts by plan: India comparison page says 2,000 per tag (Standard to Premium) and 5,000 (Elite, Ultimate); tag-count limit not restated **[unverified]** |
| Where applied | Individual **line items** (expenses: a bottom section) | **Transaction (header) or line level** for sales, purchases, journals and inventory adjustments; masters carry a default |
| Mandatory | No flag on the page (an undated Zoho function post confirmed it was missing and offered a workaround) | **Yes**: "make this reporting tag mandatory"; needs at least one active option |
| Options | Flat list | Nested to **five levels**; reorder by drag; mark one as default; new tags start "Not Ready" until they have an option |
| Conditional options | No | **Visibility conditions**: map each option to options of earlier tags (or Locations); five filters and 200 mapped options per tag; Elite and above per the pricing page |
| Access control | No | Per **role**: users see only records carrying their assigned options; up to 500 options per user; **only one tag per organisation** may have it; Elite and above |
| Reports | Filter in "Customize Report" | Advanced Filters in many reports (tag, comparator, option; "is empty" finds untagged), **Compare With** by tags (also by hierarchy), "Include Child Options" (May 2026) |

### 3.2 Records and reports

- **Records that carry a tag (advanced):** customers, vendors, items, quotes, sales orders, invoices, recurring invoices, credit
  notes, expenses, recurring expenses, purchase orders, bills, recurring bills, vendor credits, retainer invoices, customer and
  vendor payments, inventory adjustments, sales receipts, fixed assets, banking, journals.
- **An item can only be linked to a tag applied at line level**; a customer's or vendor's tag applies by default to their
  transactions; an item's tag applies by default to lines.
- **Invoice Details and Bills Details cannot be customised with tags** because tags sit on the lines; use Receivable Details or
  Payable Details, where invoices and bills can be grouped and filtered by tag.
- **Profit and Loss by tag:** Compare With, choose reporting tags; or More Filters, the tag, "is in", the options. The page does
  not describe the layout of the result **[W]**.

### 3.3 API

Tag endpoints cover create, list, update, delete, mark default option, options, visibility criteria, active and inactive for the
tag and for each option, and reorder. A tag's create body names `entities` (customers, vendors, items, fixed asset, banking) and
`multi_preference_entities` with a `preference` of `transaction` or `line_item` for sales, purchases, journals and inventory
adjustment; the list returns `is_active`, `is_mandatory`, `is_class`, `is_draft` **[W]**.

### 3.4 Practitioner view (promotional, third party, July 2026) **[W]**

A Dubai consultancy's article says: tags are for internal analysis and custom fields for client-facing details; deleting an
option keeps it on past records and marking it inactive is advised; untagged amounts fall into an "Unassociated" bucket that
can distort margins; inconsistent tagging by staff skews results; retroactive tagging is expensive, so start a structure at a
quarter boundary; limit who can create tags. Treat it as opinion from a seller of Zoho services.

---

## 4. What this means for Comsky Books **[J]**

### 4.1 Where we are today **[C]**

- **Definition:** `CustomFieldDef { id, cid, objectKind: 'ledger' | 'stockItem', key (slug), label, type: text | number | date |
  toggle | select, options?, required?, order? }` (`src/types.ts`). **Values:** `udf` on `Ledger` and `StockItem` only. `Voucher`
  and `InventoryLine` have no `udf`.
- **Entry:** `ui/udf.ts` appends the fields under "Additional Details" on the ledger and stock-item masters. `required` is
  stored and printed in the admin table, and **nothing reads it**: no form, import, bulk or API path enforces it.
- **Admin screen** (`ui/admin/CustomFields.tsx`): add and delete only. No edit, rename, reorder or inactive state. Options are one
  comma-separated box and **cannot be changed after the field is created**. Access is administrator-only (`core/access.ts`).
- **Delete is unguarded:** `checkDelete` has no `customFields` case (it falls through to `null`). Values stay in each master's
  `udf`, and `collectUdf` loops only over **current** definitions, so the next save of that ledger or item silently drops the
  orphaned values.
- **Two lossy details:** a number field set to zero is not stored (`if (n)`), and a toggle stores only true, so "No" and
  "not answered" cannot be told apart in a filter (`is No` versus `is empty`).
- **Outside the form, nothing:** no report column, no filter, no saved-view field, no print. The catalogue plan reserves a "Your
  custom fields" group with ids `udf:ledger:<key>` and `udf:stockItem:<key>` (`docs/research/pipedrive-list-ux.md` 6.1 to 6.3)
  but only for **master** fields; no voucher or line field is planned there.
- **API and export:** definitions are read-only through the REST API (`ApiDocs.tsx` `READ_ONLY_COLLECTIONS`, mirrored by
  `backend/src/restPolicy.js`). The Tally XML export lists "masters with custom fields" under **what the file does not carry**
  (`core/tallyExport.ts`). The Tally importer reads "the fields we understand and drops the rest" (header of
  `core/tallyImport.ts`), so fields a Tally customer's own customisation added would be dropped on import **[J: I did not
  verify UDF tag handling beyond that header]**.
- **Reusable precedents already in the code:** a voucher panel that is out of the Enter walk and absent unless filled
  (dispatch and e-way bill, `Alt+S`), an `Alt+H` grid for GST history, the shared master-form engine, the saved-view store with
  a personal and a company list, the row filter that runs before totals, and cost centres and categories as Tally's own
  dimension (`CostCentre`, `CostCategory`, `VoucherEntry.costs`).

### 4.2 Ideas worth borrowing from Zoho, in the order I would take them

1. **Two more object kinds before anything else: voucher and voucher line.** Zone is a voucher field; Colour per sale line is a
   line field. Without them the owner's own examples cannot be built. Add `udf?` to `Voucher` and to `InventoryLine`, absent
   unless used. (Zoho's item-field-becomes-a-line-column is the precedent.)
2. **Scope a field with three independent switches, not one "visible"**, mirroring Zoho's separation but adding the one it lacks:
   - **Where it is entered:** which voucher types or masters (Zoho's "show in transaction types" is the closest precedent);
   - **Whether reports may use it:** "Available as a column and a filter" (default yes) and "Show by default" (default no);
   - **Whether it prints:** "Show on the printed voucher" (default no; Zoho's "show in all PDFs", but ours should be per
     voucher type because the invoice customiser already decides print layout).
3. **Answer to "should a field added to Sales appear in the Sales report automatically?"** Make it **automatically available in
   the Columns picker and the row filter** of every report whose rows are that object (and the object it joins to, for a party
   or item field), **never added to a standard layout without the administrator asking.** The first half avoids Zoho's fifteen
   years of "missing from my report"; the second keeps the rule that the standard layout equals today's table.
4. **Saved views carry custom-field columns and filters.** Zoho's saved-view model (criteria that reference custom fields,
   columns with width and pinned flag, shared with me or selected or everyone, favourite, default) is what `SavedViewStored`
   (personal and `Company.reportViews`) should grow into. A saved condition naming a deleted or inactive field must **match
   nothing and be kept**, never be dropped (the widening-direction rule in the Phase 26 gotchas).
5. **Option management as a real editor:** add, paste a block, reorder, mark inactive, a default option, optional colour. Options
   need **stable ids**; renaming changes the label, never the stored value, so a saved filter "Zone = North" survives a rename.
6. **Inherit a default from the related record:** Zone defaults from the party's Zone; Colour defaults from the item's Colour.
   Zoho's "inherit value" is exactly this and costs little.
7. **Inactive before delete, and a usage count.** Zoho's order (inactive, then delete, then a force-delete warning that data goes
   too) and its "View Associated Components" are cheap to match: show "N records hold a value" before a delete is allowed.
8. **Unique and input-format validation later.** Real needs (a gate-pass or LR number must be unique; a vehicle number has a
   shape) but not the first slice.
9. **Custom fields as a landing place for import.** Zoho's migration advice is "create a field for any column that has no home
   and map it". Our CSV mapper (`MappedImport`, `core/importMap.ts`) could offer "create a custom field for this column"; the
   Tally importer could map unknown tags the same way. Open item, not first slice.
10. **A tag-like dimension** (section 4.4, case D): header or line, mandatory, default from the party or item, and a Compare
    mode in reports.

### 4.3 Traps: Zoho's gaps and our own gotchas that meet them

- **T1. Mandatory against existing records and the frozen Enter walk.** Zoho says a mandatory field blocks saving *or updating*
  without a value; it does not say how a record created before the field is handled **[W]**. Read literally, adding a mandatory
  field makes every old record un-editable until filled **[J]**. For us: a required field binds **new** vouchers; on alteration
  it binds only if the voucher already holds a value or the administrator chose "enforce on alter"; and there is a way to fill
  blanks in bulk. A new voucher field is **appended after the last existing field or sits in an Alt panel**, never inserted in the
  walk (Phase 24 gotcha: a habitual Enter walk typed into the wrong box; a field after Dr/Cr stole the Accept-on-Enter).
  Enforcement must live at the **write guard**, not only in the form, or Bulk Entry, import, API and sync skip it. Today
  `required` is not enforced anywhere, so turning it on is itself a behaviour change for existing companies.
- **T2. A field that is missing from a report.** Zoho's 15-year complaint. The design should include an assertion that walks every
  report whose rows are the object (the registers, Day Book, Ledger Vouchers, Receivables and Payables, Stock Summary and
  movement) and fails if the picker lacks a definition, the same style as the existing `dashboard-custom.assert`
  source-reading checks.
- **T3. Orphaned values and silent drops.** Delete today orphans values and the next master save erases them. Inactive first;
  keep values; a tombstone definition keeps old values readable and filterable; delete only offered at zero usage or with an
  explicit, named force that says how many records lose data.
- **T4. Option rename or removal against history.** Zoho does not document it. Store option ids; deleting an option in use is
  refused (or replaced by an inactive option); reports always show the stored value, with "(removed)" if the option is gone.
- **T5. Line values are a snapshot.** Copy the item's Colour onto the line **at posting** (the GST-snapshot gotcha), or editing
  the item re-labels last year's sales in every colour report.
- **T6. Do not merge lines that differ in a custom value.** Zoho's commerce thread lost the variation. Any place that combines
  lines of the same item (print grouping, e-invoice item list, HSN summary) must stay a function of money and quantity only and
  leave the custom value out, so a field can never move a statutory figure **[J]**.
- **T7. A field must never move a figure.** Filtered totals say "Total (filtered)" and a filtered Ledger Vouchers keeps the true
  closing balance (Phase 26 gotcha); a custom column is display and filter, not a new basis. A report that groups by a field
  must reconcile its groups plus "none" to the paisa to the unfiltered total (Phase 28 gotcha).
- **T8. Role-hidden fields leak through reports.** Zoho can hide a field per role but I could not confirm that columns, filters,
  exports and the API honour it **[unverified]**. For us a hidden field must vanish from the picker, the filter list, CSV and
  Excel, saved views, Go To and the API read, in one place (`core/access.ts`), or "hidden" is cosmetic.
- **T9. Older clients erase what they do not know.** An older app re-saving a voucher drops an unknown `udf`, exactly as it did
  to `Company.ui` and layout `cfg` (Phase 27 and 28 gotchas). A voucher alteration must carry `udf` through untouched, as
  `dispatch` and `ewb` do today, and a layout or definition reader must preserve unknown fields.
- **T10. Do not copy plan caps.** Zoho's counts (five dropdowns on a low plan) generated complaints; our plan limits are
  display-only (CLAUDE.md). Keep technical caps only (field count per object, options per list, 20 report columns), and say
  what each cap protects.
- **T11. Multi-value fields break columns and filters.** Zoho cannot show multi-select, multi-line text or attachments as list
  columns **[Ws]**. Ship single-choice list, text, number, date, Yes/No first; multi-select later with an "includes" filter and
  a joined-text column.
- **T12. Toggle and zero are values.** Store Yes/No/empty as three states and store 0; otherwise "is No" and "is empty" cannot be
  told apart and a nil answer vanishes. (Both fail today, 4.1.)
- **T13. Names are display text.** Field labels and option labels go through `cleanLabel` (Phase 27), keep an immutable key,
  and a duplicate like "Colour" and "Color" is caught by a normalised comparison.
- **T14. Clone, duplicate and recall must say what they copy.** Zoho only fixed attachments on clone and convert in July 2026.
  Alt+2 duplicate, ghost recall, recurring entries and credit-note-from-invoice each need a stated rule per field (copy,
  reset or ask); Zone should copy, a one-off gate-pass number should not.
- **T15. Printed paper and statutory outputs.** A custom field never reaches a GST return, e-invoice or e-way payload, and
  prints only when the administrator turned it on (never legal wording the owner did not choose: Phase 20-21 gotcha).
- **T16. Parity honesty.** None of this is Tally parity (Tally's own user-defined fields come from TDL, outside this product's
  scope). It is a "beyond" row, as the existing UDF one is.

### 4.4 The owner's cases, worked through

**A. Clothes dealer: Colour at item level, with a list of options, visible in reports.**
- *Two readings; support both, because they differ.* (1) **An attribute of the item** (this item is Blue): a list field on the
  Stock Item master. The current UDF can already store it; what is missing is editing the options and using it anywhere
  outside the master form. (2) **A choice made per sale line** (one shirt item sold in red, blue and black): a line field.
  Zoho's item-field-as-line-column is that second reading, and its API stores a value per line.
- *Model:* one definition, `objectKind: 'stockItem'`, with a flag "also ask on voucher lines" (the "show when creating
  transactions" analogue), applying to chosen voucher types. The line's value defaults from the item's, is editable, and is
  stored on the line as a snapshot (T5).
- *Where on the line:* after the last existing line column or in the line's Alt grid, never in the frozen walk (T1).
- *Reports:* Stock Item Vouchers, Stock Summary movement and any line-grain report get a Colour column and a filter ("Colour is
  Red"). A voucher-grain report (Sales Register) shows the one-to-many collapse already planned for Godown and Cost centre: the
  one colour, or "Multiple" (`pipedrive-list-ux.md` 6.1).
- *The honest limit:* **a field labels stock, it does not split stock.** If the dealer wants *closing stock by colour*, a custom
  field cannot give it; that needs separate items (variants) or batches. Zoho's own variant items live in its inventory add-on
  **[W July 2026]**. Say this in the product, or the first demo disappoints.

**B. Invoice-level Zone: North, South, East, West, on Sales vouchers, filterable in the Sales report.**
- *Model:* `objectKind: 'voucher'`, type list with four options, applies to Sales (and optionally Credit Note), not mandatory by
  default; default inherited from the party's own "Zone" field if the party master has one (Zoho's inherit rule).
- *Entry:* appended after the last existing voucher field or in an Alt panel; if mandatory, it must be reachable by Tab and
  named in the "cannot accept" message, and the Accept-on-Enter chain must stop on it (T1).
- *Reports:* a Zone column and filter on the Sales register (flat level), Day Book, and the other voucher-grain reports;
  Receivables and Payables need the bill-to-voucher link that `BillState` does not carry today (flagged in the planned P4 of
  `pipedrive-list-ux.md`). Totals read "Total (filtered)". "Sales by Zone" is a group-by with "no Zone" as its own row so the
  groups add to the register total (T7).
- *Saved view:* "Sales, North, this quarter" is one saved view carrying the filter and columns (4.2 item 4); a company view if the
  administrator shares it.

**C. "Visible on the voucher type; should it also be on the reports by default?"** Zoho's answer is no switch at all (2.8).
Ours, per 4.2 items 2 and 3: **entry scope** (which voucher types), **reports** (available yes, shown by default no), **print**
(no). A field removed from the entry scope (Zone no longer asked on Credit Notes) stays in the picker for the vouchers that hold a
value, because history must remain reportable (T3).

**D. The profitability or reporting tag.** Tally already has this dimension under another name: **cost category and cost centre**,
applied to ledger entries with amounts, only where a ledger is flagged for cost centres, with a Cost-Centre P&L Breakup report
(Phase 15) **[C]**. A Zoho tag is simpler (one option per header or per line, no amount split) and adds things cost centres do
not: mandatory per record type, a default from the party or item, one option list used by sales, purchase, expense and journal
alike, and a **Compare** layout. Three options, **[J]**:
1. *No new object:* make cost centres easier to use as the profitability dimension (a default from the party, mandatory per
   voucher type, a Compare view). Smallest; keeps Tally's meaning.
2. *A voucher or line list field flagged "use as a reporting dimension"*, with group-by and Compare in the registers and a
   P&L-style report. Answers the owner's example (Sales and Purchases by Zone, side by side) with machinery from 4.2.
3. *A separate "Tag" object* mirroring Zoho (hierarchy, conditional options, role access). Largest; I would defer it.
The trap in options 2 and 3: **a P&L is built from ledger postings, not vouchers.** "Profit by Zone" needs a rule that every
posting inherits a tag: the header tag covers all postings of the voucher; a line tag covers that line's item value and its tax
share; freight, round-off and TDS postings follow the header tag or fall to "Untagged". The test is the Phase 28 rule: each
option plus "Untagged" must add to the P&L's figure to the paisa. If the owner only wants **sales by Zone**, that rule is not
needed and option 2 is cheap.

### 4.5 Questions for the owner

1. Is Colour **an attribute of the item** (each shirt has a colour), **a per-sale choice**, or does he want **stock by colour**?
   The answers need three different features (4.4 A).
2. Should a required field apply to old vouchers when someone alters them (T1), or only to new ones?
3. Do they need **profit** by Zone (a P&L cut) or **sales and purchases** by Zone (a register cut)? (4.4 D)
4. May a clerk see a field but not edit it, or should some fields be hidden from some operators? That decides whether T8 is
   needed in the first slice.
5. Should a custom field print on the invoice? Zoho's one global switch is the floor.
6. Are Tally customers asking for **TDL-style fields that must come across on import**, so the importer should map unknown tags
   to custom fields (4.2 item 9)?

### 4.6 What the design must be able to prove (checks to write before building)

- With no voucher or line definitions, a company's vouchers, print, Enter walk and every report equal today's, asserted by
  identity (the Phase 27 opt-in rule); a definition created and then made inactive changes no figure.
- Delete: a definition with values cannot be removed without a named force, and old values stay readable (T3).
- A required voucher field blocks a **new** voucher through the form, Bulk Entry, import and API alike, and does not block
  altering an old voucher by default (T1).
- Every report whose rows are the object offers the field in its picker and filter; a deleted option or field in a saved view
  matches nothing and is kept (T2, 4.2 item 4).
- A list rename changes no stored value or saved filter (T4); a line value is unchanged by later edits to the item (T5).
- Grouping by a field reconciles to the unfiltered total including "none" (T7); a hidden field is absent from the picker, CSV,
  Excel, views, Go To and the API read (T8); an older client's save of a voucher leaves its `udf` intact (T9).

---

## 5. Sources read (all fetched 2026-10-21; **[W]** unless marked)

**Zoho help and KB**
- Custom fields: https://www.zoho.com/books/help/settings/customization/custom-fields.html (and the `.md` copy
  https://www.zoho.com/books/help/settings/customization/custom-fields.md)
- Field data types: https://www.zoho.com/books/help/settings/customization/custom-field-data-types.html and
  https://zstatic.zohostatic.com/books/help/settings/customization/custom-field-data-types.md
- Lookup: https://www.zoho.com/books/help/settings/customization/lookup.md
- Formula: https://www.zoho.com/books/help/settings/customization/formula.md
- Validation rules: https://www.zoho.com/books/help/settings/customization/validation-rules.md
- Custom views: https://www.zoho.com/books/help/settings/customization/custom-views.md
- Custom reports: https://www.zoho.com/books/help/reports/custom-reports.md
- Manage reports: https://www.zoho.com/books/help/reports/manage-reports.md
- Receivables reports: https://www.zoho.com/books/help/reports/receivables.html
- Workflow rules: https://www.zoho.com/books/help/settings/automation/workflow-rules.md
- Field updates: https://www.zoho.com/books/help/settings/automation/workflow-actions/field-updates.md
- PDF templates: https://www.zoho.com/books/help/settings/pdf-templates/edit-templates.md
- HTML PDF templates: https://www.zoho.com/books/help/settings/html-pdf-templates.md
- Import: https://www.zoho.com/books/help/import-export/import.md; export:
  https://www.zoho.com/books/help/import-export/export.md
- Migration guide: https://www.zoho.com/in/books/help/migration/migrating-to-zoho-books-from-other-software.html
- Custom modules index and basic functions: https://www.zoho.com/books/help/custom-modules/index.md,
  https://www.zoho.com/books/help/custom-modules/basic-functions.md
- Layout rules (custom modules): https://www.zoho.com/books/help/custom-modules/layout-rules.md
- Reporting tags (older flow): https://www.zoho.com/books/help/settings/reporting-tags.html and
  https://www.zoho.com/us/books/help/settings/reporting-tags.md
- Reporting tags (advanced): https://www.zoho.com/books/help/settings/reporting-tag.md and
  https://www.zoho.com/in/books/help/settings/advanced-reporting-tag.html
- KB, item column: https://www.zoho.com/books/kb/items/add-new-column-in-item-table.html; item info on PDFs:
  https://www.zoho.com/books/kb/templates/display-additional-item-info.html
- KB, contact-person fields: https://www.zoho.com/us/books/kb/contacts/contact-person-custom-fields.md; contact fields:
  https://www.zoho.com/in/books/kb/contacts/custom-fields-contacts.md
- KB, tags in Invoice Details: https://www.zoho.com/books/kb/reports/reporting-tags-in-invoice-details-report.html; Bills Details:
  https://www.zoho.com/books/kb/reports/reporting-tags-in-bills-details-report.html; P&L by tag:
  https://www.zoho.com/in/books/kb/reports/profit-and-loss-report-reporting-tag-wise.md; reports that can be customised:
  https://www.zoho.com/books/kb/reports/list-of-reports-that-can-be-customized.html; tag import:
  https://www.zoho.com/in/books/kb/import-export/import-transaction-with-reporting-tag.html
- Plan comparison: https://www.zoho.com/au/books/pricing/pricing-comparison.html and
  https://www.zoho.com/in/books/pricing/pricing-comparison.html; pricing: https://www.zoho.com/books/pricing/
- Documentation index (used to find pages): https://www.zoho.com/books/help/llms.txt

**Zoho API reference**
- Invoices: https://www.zoho.com/books/api/v3/invoices/ (read in three offsets; the page is about 236,000 characters, and the
  tool read most but not all); Items: https://www.zoho.com/books/api/v3/items/; Reporting tags:
  https://www.zoho.com/books/api/v3/reporting-tags/ (first 100,000 of 150,000 characters); Custom views:
  https://www.zoho.com/books/api/v3/custom-views/

**Zoho blog and product-update posts**
- Custom fields launch, May 2011: https://www.zoho.com/blog/general/just-in-custom-fields-for-invoices-and-estimates.html
- Reporting tags launch, July 2016: https://www.zoho.com/blog/books/introducing-reporting-tags-in-zoho-books-sorting-your-reports-is-now-just-a-tag-away.html
- What's New, November 2016: https://help.zoho.com/portal/en/community/topic/what-s-new-in-zoho-books-november-2016
- Product updates: May 2026 https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-may-2026;
  June 2026 https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-june-2026-7-6-2026;
  July 2026 https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-july-2026;
  August 2026 https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-august-2026;
  September 2026 https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-september-2026;
  October 2026 https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-october-2026
- Advanced filters enhancement: https://help.zoho.com/portal/en/community/topic/enhancement-to-advanced-filters-in-reports
- Expense and Payments Received custom fields:
  https://help.zoho.com/portal/en/community/topic/custom-fields-with-data-types-for-expense-module-in-zoho-books
- Mandatory reporting tag workaround: https://help.zoho.com/portal/en/community/topic/function-3-assign-a-mandatory-reporting-tag-to-invoices

**Forum threads (opening post only came back; replies, where quoted, are [Ws])**
- https://help.zoho.com/portal/en/community/topic/sales-report-with-custom-field
- https://help.zoho.com/portal/en/community/topic/how-can-i-add-a-custom-field-to-the-reports-in-sales
- https://help.zoho.com/portal/en/community/topic/custom-view-and-custom-fields-on-zoho-books (and the `en-gb` copy)
- https://help.zoho.com/portal/es/community/topic/zoho-books-invoice-custom-fields-are-not-available-in-analytics
- https://help.zoho.com/portal/en/community/topic/error-custom-field-limit-has-been-reached-for-this-data-type
- https://help.zoho.com/portal/en/community/topic/custom-fields-limit-10-2-2015
- https://help.zoho.com/portal/pt/community/topic/is-there-a-limit-of-100-characters-in-line-items-custom-fileds
- https://help.zoho.com/portal/en/community/topic/line-items-of-the-same-item-id-should-not-be-automatically-consolidated-as-it-fails-to-recognize-variation-in-custom-fields-as-well-as-distinct-items-of-the-same-id-sold-in-varying-units-of-measure
- https://help.zoho.com/portal/en/community/topic/need-help-with-deluge-syntax-for-invoice-item-custom-fields
- https://help.zoho.com/portal/en/community/topic/missing-custom-fields-into-api-call-invoices (Subscriptions API)
- https://help.zoho.com/portal/en/community/topic/books-api-search-invoices-by-custom-fields-variants-custom-field-startswith-and-custom-field-contains
- https://help.zoho.com/portal/en/community/topic/how-to-add-a-custom-column-in-the-invoice-table-in-zoho-books
- https://help.zoho.com/portal/ja/community/topic/missing-custom-fields-in-zoho-reports?page=8

**Third party**
- https://akmcme.com/how-to-use-reporting-tags-in-zoho-books-for-insights/ (consultancy, promotional)

**Search summaries only [Ws] (pages not read)**
- Staff replies in the "sales report with custom field" and custom-view threads; the community post on creating custom fields
  through an undocumented endpoint (https://help.zoho.com/portal/pt/community/topic/create-custom-fields-and-custom-buttons-through-api?page=8);
  the February 2025 workflow-rules note on field comparison; the November 2023 unique-fields note; a community answer that item
  columns sit before Quantity; a forum report that the portal setting for custom module fields is "Display in Portal" and
  "Edit in Portal"; Zoho Subscriptions help (10-tag cap).

**Repo files read (not modified):** `CLAUDE.md`; `src/types.ts`; `src/ui/udf.ts`; `src/ui/admin/CustomFields.tsx`;
`src/ui/masters.tsx` and `src/ui/inventory/masters.tsx` (UDF call sites, by search); `src/state/store.ts` (`checkDelete`);
`src/core/savedViews.ts`; `src/core/tallyExport.ts`; `src/core/tallyImport.ts` (header); `src/ui/admin/ApiDocs.tsx`;
`docs/research/pipedrive-list-ux.md`; `docs/research/wave-a2-plan.md`; `docs/research/zoho-books-reference.md` (format only).
