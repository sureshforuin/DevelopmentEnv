# Implementation Plan: AG Grid Custom Pagination with Angular

## 1. Objective

Implement custom pagination for an Angular application using AG Grid,
with server-side data retrieval and a reusable pagination footer.

The application should efficiently handle 50,000+ records without
downloading the entire dataset. The initial page size should default to
**1,000 records**, and users should be able to increase or decrease the
page size.

### Expected user experience

The footer should display:

-   The total number of matching records (for example, **52,350**).
-   The current page and total number of pages.
-   The currently displayed record range (for example, **Showing
    1--1,000 of 52,350 records**).
-   First, previous, next, and last page controls.
-   A direct page-number input so users can jump to a page.
-   A page-size selector, defaulting to 1,000.

## 2. Proposed architecture

``` text
Angular page
  ├── AG Grid
  │     └── Displays the records for the current page
  │
  └── Custom pagination footer
        ├── Current page
        ├── Total pages
        ├── Total record count
        ├── Page navigation
        └── Page-size selector
              |
              v
        Angular data service
              |
              v
        Backend API
        ├── Filtering
        ├── Sorting
        ├── Pagination
        └── Total matching count
              |
              v
           Database
```

The Angular component coordinates the grid and footer. The Angular
service sends pagination, search, and sorting parameters to the API. The
backend applies those operations to the complete dataset and returns
only the requested page.

## 3. API contract

The backend API must support pagination and return both the requested
records and the total number of matching records.

### Example request

``` http
GET /api/records?page=1&pageSize=1000
```

### Example response

``` json
{
  "data": [
    { "id": 1, "name": "Record 1" },
    { "id": 2, "name": "Record 2" }
  ],
  "totalRecords": 52350,
  "page": 1,
  "pageSize": 1000
}
```

The `data` array above is abbreviated for illustration. In the actual
response, it should contain up to 1,000 records.

**Important:** `totalRecords` must represent the total number of records
matching the current filters, not merely the number of records in the
returned page. The backend should calculate this count independently of
the pagination limit.

### Suggested request parameters

  Parameter     Purpose                          Example
  ------------- -------------------------------- -----------
  `page`        1-based page number              `1`
  `pageSize`    Number of records per page       `1000`
  `search`      Optional global search keyword   `invoice`
  `sortBy`      Optional sortable field          `name`
  `sortOrder`   Sort direction                   `asc`

## 4. Pagination calculations

Assume the API reports 52,350 matching records and the selected page
size is 1,000.

-   Total pages: `ceil(52350 / 1000) = 53`
-   Page 1: records 1--1,000
-   Page 2: records 1,001--2,000
-   Page 53: records 52,001--52,350

The backend can calculate the offset as:

``` typescript
const offset = (page - 1) * pageSize;
```

For example, page 10 with a page size of 1,000:

``` text
offset = (10 - 1) * 1000
offset = 9000
limit  = 1000
```

The API should return the requested slice and the total matching count.

**Page-jump consideration:** With 52,350 records and a page size of
1,000, there are 53 pages, so page 200 is not valid. Users can jump to
page 200 only when the selected page size results in at least 200 pages.
For 52,350 records, a page size of 261 or less allows that.

When the page size changes, reset the current page to page 1 to avoid
requesting an invalid page.

## 5. Angular implementation structure

Suggested files:

  -----------------------------------------------------------------------
  File                                Responsibility
  ----------------------------------- -----------------------------------
  `records.component.ts`              Main AG Grid configuration, state,
                                      and data loading

  `records.component.html`            Grid layout and footer placement

  `grid-pagination.component.ts`      Pagination state, inputs, and
                                      output events

  `grid-pagination.component.html`    Footer controls and record-count
                                      display

  `records.service.ts`                Backend API integration

  `records.model.ts`                  API request and response interfaces
  -----------------------------------------------------------------------

### A. Main grid component

Responsibilities:

-   Initialize AG Grid.
-   Fetch the first 1,000 records on initial load.
-   Bind the returned records to AG Grid.
-   Maintain the current page, page size, and total record count.
-   Listen for page and page-size changes from the custom footer.
-   Request the selected page from the API.
-   Update the grid and footer after each successful response.
-   Handle loading and API error states.

### B. Custom pagination component

Responsibilities:

-   Display the total matching record count.
-   Display the current page and total pages.
-   Provide first, previous, next, and last page controls.
-   Allow direct page-number entry.
-   Provide a page-size selector.
-   Emit page-change and page-size-change events to the parent
    component.
-   Disable navigation controls when the user is already on the first or
    last page.

Suggested page-size options:

-   100
-   500
-   1,000 (default)
-   2,000

Reset to page 1 when the page size changes.

### C. Angular data service

The service should accept the page, page size, optional search keyword,
and optional sorting information.

Example:

``` typescript
getRecords(params: {
  page: number;
  pageSize: number;
  search?: string;
  sortBy?: string;
  sortOrder?: string;
}) {
  return this.http.get<ApiResponse>(
    '/api/records',
    { params: { ...params } }
  );
}
```

The exact interface should be aligned with the backend API contract.

### D. Search functionality

Search must work across the entire dataset, not only the records
currently loaded in the grid.

Recommended flow:

1.  The user enters a search keyword.
2.  Apply a 300--500 ms debounce.
3.  Send the search keyword to the backend.
4.  The backend filters the complete dataset.
5.  The backend returns the requested page and the filtered total count.
6.  Reset pagination to page 1.
7.  Update the grid and footer.

The backend should use appropriate database indexes for frequently
searched fields where practical.

### E. Sorting

Sorting must also be performed by the backend so it applies to the
complete dataset, rather than only the current page.

Example:

``` http
GET /api/records?page=1&pageSize=1000&sortBy=name&sortOrder=asc
```

Only allow sorting by fields supported by the backend. Validate
sort-field names and sort directions on the server rather than
interpolating arbitrary client input into SQL.

## 6. AG Grid configuration

For this design, use AG Grid's **Client-Side Row Model** to display the
records for the currently selected page. The application, rather than AG
Grid's built-in pagination, controls the API requests and page
navigation.

Example configuration:

``` typescript
gridOptions: GridOptions = {
  rowModelType: 'clientSide',
  pagination: false,
  animateRows: true,
  columnDefs: this.columnDefs,
  defaultColDef: {
    sortable: false,
    filter: false,
    resizable: true
  }
};
```

In this approach:

-   The grid receives only the current page of records.
-   The custom Angular footer controls pagination.
-   The application fetches a new page when the user navigates.
-   Server-side search and sorting are handled through API requests.
-   AG Grid Enterprise is not required for this approach.

Do not enable AG Grid's built-in pagination at the same time as the
custom footer. Also, disable or intercept grid sorting interactions if
sorting is implemented through custom server-side controls;
alternatively, handle AG Grid sort-change events and translate them into
API requests.

## 7. Implementation phases

### Phase 1 --- Backend API changes

-   Add `page` and `pageSize` request parameters.
-   Implement offset/limit or equivalent database pagination.
-   Return `totalRecords` independently of the page limit.
-   Support server-side search.
-   Support server-side sorting.
-   Validate page sizes, page numbers, and sortable fields.

### Phase 2 --- Angular grid integration

-   Configure AG Grid.
-   Fetch the initial 1,000 records.
-   Bind the API response to the grid.
-   Maintain pagination state in the parent component.
-   Add loading indicators and error handling.

### Phase 3 --- Custom pagination footer

-   Build a reusable Angular pagination component.
-   Display total records, current page, total pages, and record range.
-   Implement first, previous, next, and last navigation.
-   Implement direct page jumping.
-   Implement page-size selection.
-   Reset to page 1 when the page size changes.

### Phase 4 --- Search and sorting

-   Implement debounced global search.
-   Implement backend sorting.
-   Reset pagination when search criteria change.
-   Update the total count after filtering.
-   Ensure the displayed record range reflects the filtered result set.

### Phase 5 --- Testing and optimization

-   Test with 50,000+ records.
-   Test all supported page sizes.
-   Test direct page navigation and invalid page numbers.
-   Test search and sorting across the full dataset.
-   Verify the total count and displayed record range.
-   Test rapid page changes and prevent stale API responses from
    replacing newer results.
-   Measure API response time and grid rendering performance.
-   Check database query plans and indexes for frequently used filters
    and sort fields.

## 8. Acceptance criteria

-   [ ] Initial load fetches no more than 1,000 records.
-   [ ] Footer displays the actual total matching record count (for
    example, 52,350).
-   [ ] Footer displays the current page and total number of pages.
-   [ ] Footer displays the correct record range.
-   [ ] User can navigate to any valid page.
-   [ ] User can change page size to 100, 500, 1,000, or 2,000.
-   [ ] Default page size is 1,000.
-   [ ] Total page count updates when page size changes.
-   [ ] Pagination resets to page 1 when page size or search criteria
    change.
-   [ ] Search works across the complete dataset.
-   [ ] Sorting works across the complete dataset.
-   [ ] Footer displays the correct filtered record count.
-   [ ] Invalid page numbers are handled safely.
-   [ ] Older, slower API responses cannot overwrite newer page results.
-   [ ] The grid does not download the entire 50,000+ record dataset.

## 9. Final recommendation

  Area                 Recommended approach
  -------------------- -----------------------------------
  Grid                 AG Grid Community
  Row model            Client-Side Row Model
  Data fetching        Custom API pagination
  Default batch size   1,000 records
  Pagination UI        Reusable custom Angular component
  Total count          Backend response
  Search               Server-side
  Sorting              Server-side
  Page navigation      Custom footer
  AG Grid Enterprise   Not required

This design is **manual server-side pagination using AG Grid's
Client-Side Row Model for each fetched page**. It is different from AG
Grid's Enterprise Server-Side Row Model and its Community Infinite Row
Model.

It provides full control over the footer and API requests while keeping
the grid implementation relatively simple and avoiding an AG Grid
Enterprise dependency.
