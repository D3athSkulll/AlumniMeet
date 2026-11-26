# Alumni Directory and Missing-Alumni Search

This document explains how the search features in the AlumniMeet application work, what information they send to the MongoDB-backed API, how filters are combined, and how results are ordered and paginated.

## 1. Overall Search Flow

Both search pages are React interfaces that collect user selections and send them to the backend as URL query parameters. They do not search a local copy of the database in the browser.

The basic flow is:

1. The user enters or selects one or more filters.
2. The frontend adds the active filters to a `URLSearchParams` object.
3. The frontend requests `/api/search` or `/api/missing_alumni` from the configured backend.
4. MongoDB returns matching records, the total number of matches, and a `hasMore` flag.
5. The React page displays the records in a table and creates pagination controls.

The configured backend URL comes from `VITE_DATABASE_URL`. The frontend normalizes it so a trailing slash is always present, then appends the API path. For example:

```text
http://localhost:3001/ + api/search?name=rahul&limit=20&page=1
```

Becomes:

```text
http://localhost:3001/api/search?name=rahul&limit=20&page=1
```

If the backend returns a slow response, the frontend waits for up to 30 seconds. If it is not a JSON response, returns an error, or times out, the interface shows an error instead of displaying stale results.

## 2. Alumni Directory Search

The alumni directory is available at `/directory`. It contains both a basic search area and an advanced filter panel.

The directory is intended to work with 4,500 or more alumni records. The frontend therefore never loads the complete database at once. It requests only 20 records for the requested page.

### 2.1 Basic Filters

The basic search card contains:

- Name
- Year of Entry
- Program Name
- Company

These fields are often sufficient for a quick lookup. A user can search with only one field or combine several fields.

#### Name

The name field accepts free text. Its value is sent as the `name` query parameter.

The expected MongoDB behavior is case-insensitive partial matching. A user who types `an` should be able to match names containing “an,” while entering a complete name should narrow the result set. The backend should perform this by wrapping the supplied text in a MongoDB regular expression with case-insensitive matching.

Conceptually:

```javascript
{ name: { $regex: escapedUserText, $options: "i" } }
```

The regex is not the literal or exact JavaScript expression entered by the user. It is a MongoDB-style regular-expression pattern generated from the text. The same escaped-text and `$options: "i"` approach can make the search forgiving of uppercase and lowercase differences.

For safe use, special regex characters supplied by a user—such as `.`, `*`, `+`, `?`, `[`, `]`, `(`, `)`, `{`, `}`, `^`, `$`, `|`, and `\`—should be escaped unless the interface explicitly promises advanced regex syntax. In normal alumni search, those characters should be treated as ordinary text rather than regex operators.

#### Year of Entry

Year of Entry is a dropdown containing every year from 1998 to 2026. The frontend sends the selected value as `yearOfEntry`.

This is a precise categorical/numeric filter, not a free-text regex filter. MongoDB should compare the stored year with the supplied year:

```javascript
{ yearOfEntry: 2020 }
```

The frontend sends the value as a string, so the backend may need to convert it to a number or compare it using the database's actual stored type.

There is no partial matching for a year such as `20`; the user chooses a full year from the dropdown.

#### Program Name

Program Name is selected from:

- BCS
- BIT
- IMG
- IMT
- MBA
- MTECH
- IPG
- PGDIT
- PGDMIT
- PHD
- DSC

The selected value is sent as `programName`.

This should normally be an exact match, rather than a regex match. Program codes are controlled values, so MongoDB should compare the complete code:

```javascript
{ programName: "BCS" }
```

Because the value comes from the controlled dropdown, escaping it and using regex is unnecessary. An exact match is also less likely to return a record with an unintended program code.

#### Company

Company searches the alumni field `lastOrganization`. The user can enter a full or partial organization name.

The frontend sends the text as `lastOrganization`. MongoDB should perform case-insensitive partial matching. For example, entering `micro` could match organizations containing “Microsoft,” “Micron,” or another matching company name.

Conceptually:

```javascript
{
  lastOrganization: {
    $regex: escapedCompanyText,
    $options: "i"
  }
}
```

The regex permits partial organization names and ignores letter case. User-entered regex operators should be escaped in a normal search UI.

### 2.2 Advanced Filters

The Advanced Filters panel contains:

- Roll Number
- Country
- City
- Nature of Job
- Specialization
- Last Position
- College Clubs

All advanced fields are combined with the basic fields. A result must satisfy every active condition.

#### Roll Number

The roll number field sends `rollNumber`.

It is a partial text search rather than an exact-only lookup. A user could enter the complete roll number or part of it. A typical MongoDB pattern is:

```javascript
{
  rollNumber: {
    $regex: escapedRollNumber,
    $options: "i"
  }
}
```

The search is case-insensitive because stored or entered roll numbers may vary in letter case.

An example input such as `2023BCS` may match roll numbers that begin with or contain that text. User input containing spaces or punctuation should be handled consistently by the backend.

#### Country

Country sends `country`.

Before sending the request, the frontend removes surrounding whitespace and converts the text to uppercase. It does not search a dropdown, so the backend should still be tolerant of data that differs in formatting or case.

The backend may use:

```javascript
{
  country: {
    $regex: escapedCountryText,
    $options: "i"
  }
}
```

For example, `india` and `INDIA` should match. A database stored as `India` can match both because of the `i` regex option.

Partial country matching is useful, but it can also be broad. Searching `united` might match several United countries or location text containing that word.

#### City

City sends `city`.

City search should be case-insensitive and partial:

```javascript
{
  city: {
    $regex: escapedCityText,
    $options: "i"
  }
}
```

Some records may store a city as part of `currentLocationIndia` or `currentOverseasLocation` rather than in a dedicated `city` field. If so, the backend search contract may need to search the relevant location fields or store and expose a normalized city field.

#### Nature of Job

Nature of Job is a controlled dropdown. The frontend sends the selected value as `natureOfJob`.

Available values include:

- ACADEMIA
- BANKING
- CAREERBREAK
- CORPORATE
- DECEASED
- ENTREPRENEUR
- FREELANCE
- GLOBAL
- GOVERNMENT

Because these values are controlled and may be stored in uppercase, MongoDB should use an exact match:

```javascript
{ natureOfJob: "CORPORATE" }
```

If older database records have inconsistent letter case, a case-insensitive regex could provide backward compatibility:

```javascript
{
  natureOfJob: {
    $regex: `^${escapedValue}$`,
    $options: "i"
  }
}
```

The anchors `^` and `$` are important here. They require the complete stored value to equal the selected category instead of allowing partial matches. For a controlled category, exact equality is preferable.

#### Specialization

Specialization is sent as `specialization`.

The options depend on the selected program. The frontend restricts combinations by mapping program codes to valid specialization codes:

| Program | Valid specializations |
| --- | --- |
| IPG | MBA, MTECH |
| MBA | BA, IFS, ISM, ITES, NFS, PSM |
| MTECH | AN, BI, CN, DC, ICS, IS, IT, SE, VLSI, WNC |
| PHD | PIT, PITF, PMG, PMGF, PAM, PAMF, PAP |

The specialization dropdown is disabled until a program is selected. If the program changes, the frontend clears the old specialization if it is no longer valid for the new program.

The specialization search should be exact:

```javascript
{ specialization: "ISM" }
```

No partial regex is needed because the UI is selecting a code. Combining an exact specialization with an exact program produces predictable results.

#### Last Position

Last Position sends `lastPosition`.

This is most useful as a case-insensitive partial search. For example, `manager` can match:

- Engineering Manager
- Product Manager
- Manager
- General Manager

The MongoDB pattern can be:

```javascript
{
  lastPosition: {
    $regex: escapedPositionText,
    $options: "i"
  }
}
```

Regular-expression matching helps when position names use different words or lengthy title variants. It does not rank records by how closely they match the text; it only decides whether the text occurs in the field.

#### College Clubs

College Clubs sends `collegeClubs`.

Club names can contain multiple words or several club names in a single field. A partial case-insensitive regex is appropriate:

```javascript
{
  collegeClubs: {
    $regex: escapedClubText,
    $options: "i"
  }
}
```

For example, a search for `coding` could match records containing “Coding Club,” “Competitive Coding,” or a longer value that includes the word. If a record stores a list or array of club names, the backend may need to use array-aware MongoDB matching rather than a regex against a single text field.

## 3. How Multiple Directory Filters Work

The frontend adds each active filter to the query string. The backend should convert those parameters into an AND-based MongoDB query.

For example:

```text
name=rahul
&yearOfEntry=2020
&programName=BCS
&lastOrganization=microsoft
&country=india
&page=1
&limit=20
```

The conceptual MongoDB query is:

```javascript
{
  name: { $regex: "rahul", $options: "i" },
  yearOfEntry: 2020,
  programName: "BCS",
  lastOrganization: { $regex: "microsoft", $options: "i" },
  country: { $regex: "india", $options: "i" }
}
```

The word “AND” means that all active conditions must match. It does not mean that the user must fill every field. No filter is required in the directory; an empty directory search requests records for all alumni.

This differs from the missing-alumni page, where at least one of the three filters must be selected before the search is allowed.

## 4. Missing-Alumni Search

The Missing-Alumni page is available at `/missing_alumni`. It is separate from the general directory because it narrows its purpose to follow-up records.

The page describes its data as alumni with unknown or deceased job status. The backend is responsible for identifying those records. The UI does not receive a separate missing-record flag from the user.

### 4.1 Required Filter Rule

Before searching, the page checks that at least one of these filters is active:

- Year of Entry
- Program Name
- Specialization

If all three are empty, it does not call the backend. It displays:

> Please select at least one filter before searching.

This prevents a broad request that could return all missing records.

### 4.2 Year of Entry

Year of Entry uses the same 1998 to 2026 dropdown used in the directory. The value is sent as `yearOfEntry`.

It should be converted to a number or matched in a way that is compatible with the database field type.

The frontend does not use regex for this filter.

### 4.3 Program Name

Program Name uses the same controlled program list. The value is sent as `programName`.

The query should use exact matching:

```javascript
{ programName: "MBA" }
```

Regex is not needed for this exact category.

### 4.4 Specialization

Specialization is also exact and controlled. It depends on the selected program.

The frontend sends `specialization` only when a specialization is selected. The backend can then match it exactly:

```javascript
{ specialization: "ISM" }
```

When the program changes, an old specialization is cleared if it does not belong to the new program.

### 4.5 Identifying Missing Records

The missing-alumni endpoint is `/api/missing_alumni`.

Conceptually, it should first identify records with unknown or deceased job status, then apply the selected alumni filters. The UI text states that records have unknown or deceased job status.

MongoDB may represent this as:

```javascript
{
  $or: [
    { natureOfJob: { $regex: "^unknown$", $options: "i" } },
    { natureOfJob: { $regex: "^deceased$", $options: "i" } },
    { natureOfJob: "" },
    { natureOfJob: { $exists: false } }
  ]
}
```

The exact field names and missing-value representation must be confirmed in the backend, but the frontend contract expects the `/api/missing_alumni` endpoint to handle this condition.

The search should also include year, program, and specialization as AND conditions. The conceptual query is:

```javascript
{
  missingCondition,
  yearOfEntry: 2018,
  programName: "MBA",
  specialization: "ISM"
}
```

If the missing condition is implemented using a nature-of-job regex, its anchors are useful:

- `^unknown$` matches only a complete value of unknown.
- `^deceased$` matches only a complete value of deceased.
- `$options: "i"` makes both checks case-insensitive.

This is safer than an unanchored regex such as `/unknown/`, which could match unrelated text containing that word.

## 5. Regex Types Used in the Search Design

The following are the regex patterns that fit the application's search behavior.

### 5.1 Partial, case-insensitive text regex

This is used for ordinary free-text fields:

- Name
- Company
- Roll Number
- Country
- City
- Last Position
- College Clubs

Pattern:

```javascript
{ $regex: escapedText, $options: "i" }
```

`i` ignores letter case. The unanchored regex matches text that appears anywhere in the stored field. For example, `mumbai` matches `Mumbai`, `Navi Mumbai`, and `Greater Mumbai`.

### 5.2 Anchored exact-category regex

This can be used for category-like text values that need exact matching:

```javascript
{ $regex: `^${escapedText}$`, $options: "i" }
```

It is useful only when the MongoDB field is inconsistent in case but should still match one complete category.

Examples:

- `^DECEASED$`
- `^UNKNOWN$`
- `^CORPORATE$`

This should not replace exact equality for a clean controlled field.

### 5.3 Multiple-name alternatives

Missing or status data may need alternatives. MongoDB supports a `$or` array:

```javascript
{
  $or: [
    { natureOfJob: { $regex: "^unknown$", $options: "i" } },
    { natureOfJob: { $regex: "^deceased$", $options: "i" } },
    { natureOfJob: "" },
    { natureOfJob: { $exists: false } }
  ]
}
```

This search means:

> Match one of these missing-data conditions.

It is a broader query than a partial text search because the field may be missing or stored in more than one representation.

### 5.4 Regex input safety

The backend should escape text that came from a free-text input. If a user searches for `C++`, for example, `+` has a special meaning in regex and should not be interpreted as a regex quantifier.

The safe approach is to convert the input into a literal text pattern:

```javascript
function escapeRegex(text) {
  return text.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

The resulting pattern is then combined with `$regex` and `$options: "i"`.

This prevents user text from unexpectedly changing the query, while still supporting normal partial and case-insensitive search behavior.

## 6. Pagination and Ordering

Both search pages request 20 records per page. The server receives:

```text
page=1
limit=20
```

Pages are calculated from the `totalCount` returned by MongoDB:

```text
number of pages = ceiling(totalCount / 20)
```

For example:

- 45 matches produce 3 pages.
- 44 matches produce 3 pages.
- 40 matches produce 2 pages.

### 6.1 Previous and Next Navigation

The Previous button is disabled on page 1.

The Next button is enabled only when `hasMore` is true. The backend reports `hasMore` because it knows whether more records exist after the current page.

The page-number area displays:

- The first page
- The last page
- Pages within two positions of the current page
- Ellipses between separated page ranges

For example, on page 10 of 20, it may show:

```text
1 … 8 9 10 11 12 … 20
```

### 6.2 Directory Ordering

The directory performs a JavaScript sort after it receives the page:

```javascript
(a.yearOfEntry ?? Number.MAX_SAFE_INTEGER) -
(b.yearOfEntry ?? Number.MAX_SAFE_INTEGER)
```

This sorts records by ascending `yearOfEntry`.

The behavior is:

- Earliest known entry year appears first.
- 1998 appears before 1999.
- 2025 appears before 2026.
- Records without a valid year are placed at the end.

Within the same year, the frontend does not define a second sort key. Names, roll numbers, or other fields are not guaranteed to be alphabetically ordered unless the backend already applied a stable secondary sort before returning the page.

There is an important implementation distinction: sorting happens after a 20-record page arrives from the backend. If MongoDB applies pagination before the frontend sort, each individual page is ordered by entry year, but the complete result set is not necessarily globally sorted across every page. A globally exact order should ideally be applied by MongoDB using `sort({ yearOfEntry: 1, name: 1 })` before pagination with `skip` and `limit`.

Records missing `yearOfEntry` use `Number.MAX_SAFE_INTEGER`, ensuring that they are sorted after records with known years in the returned page.

### 6.3 Missing-Alumni Ordering

The missing-alumni page does not perform an additional frontend sort.

Therefore, the ordering shown on that page is the ordering returned by `/api/missing_alumni`. It is best practice for the backend to provide a stable explicit order, such as:

```javascript
sort({ yearOfEntry: 1, name: 1 })
```

This would display missing alumni in ascending entry-year order and alphabetically within each year.

Until the backend guarantees and documents a sort, missing-alumni results should be understood as “backend/API order,” not a user-controlled name or year sort.

### 6.4 UI Tables

The directory table displays:

- Name
- Roll Number
- Department
- Year of Entry
- Company
- Location

The missing-alumni table displays:

- Name
- Roll Number
- Year of Entry
- Program
- Specialization
- Company
- Location

Missing values are shown as `NA`. Location selection prefers `currentLocationIndia`, then `currentOverseasLocation`, and then `NA`.

The directory also shows an Update Details button while hovering over a result row. This button opens the profile update form for that specific roll number.

## 7. Profile Update Request Behavior

The directory allows profile updates through an edit dialog.

The request includes:

- `rollNumber`
- `oldData`
- `newData`

The roll number identifies the record. It is not editable in the form.

The submitted request is stored or processed by the backend and is later visible in `/volunteer-portal-secret`. The volunteer portal retrieves pending, approved, or rejected requests, and an authorized volunteer can approve or reject the update.

No update should be treated as live until the backend:

- Authenticates the volunteer
- Validates the submitted fields
- Applies the approved changes to the correct roll number
- Records the reviewed state and optional notes

## 8. Search States

The frontend handles three main search outcomes.

### Loading

The directory and missing-alumni pages show a spinning indicator and a loading message while the fetch is running.

Search buttons are disabled while loading.

### No Matches

If the API returns an empty array with no error, the UI shows:

- “No alumni found matching your criteria” in the directory
- “No missing alumni found for selected filters” in the missing-alumni page

### Errors

An error may occur if:

- The backend is unavailable
- The server is starting slowly
- A request times out
- The backend does not return JSON
- The backend returns a non-success response

The directory and missing-alumni pages clear previous results and display the error message rather than leaving old data on the screen.

## 9. Summary

The directory is a multi-filter MongoDB search interface that combines partial text regex searches with exact academic and category filters.

Its main design rules are:

- Partial case-insensitive regex: name, company, roll number, country, city, position, and clubs
- Exact equality: year, program, specialization, and controlled category values
- AND combination: every active filter must match
- 20 records per page
- Directory records sorted by year of entry in ascending order after receiving a page
- Missing-alumni results ordered according to the backend unless it also explicitly sorts
- Regex input escaped to prevent user text from becoming executable query syntax
- Missing-alumni search requiring at least one of year, program, or specialization

These rules keep the interface broad enough to find similar names and organizations, while controlled filters make program, specialization, year, and job categories precise and predictable.
