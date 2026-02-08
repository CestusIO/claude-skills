# htmx Server-Side Patterns

## Core Principle

htmx servers return **HTML fragments**, not JSON. The server renders HTML that gets swapped directly into the DOM.

## Detecting htmx Requests

htmx sends an `HX-Request: true` header with every request. Check it to decide between full page and partial responses.

### Rust (Axum)

```rust
use axum::{
    http::HeaderMap,
    response::{Html, IntoResponse},
};

async fn list_items(headers: HeaderMap) -> impl IntoResponse {
    let items = get_items().await;
    let is_htmx = headers.get("HX-Request").is_some();

    if is_htmx {
        // htmx request — return partial HTML
        let html = ItemsListPartial { items }.render().unwrap();
        Html(html)
    } else {
        // Regular request — return full page
        let html = ItemsPage { items }.render().unwrap();
        Html(html)
    }
}
```

### Go (net/http)

```go
func itemsHandler(w http.ResponseWriter, r *http.Request) {
    items := getItems()

    if r.Header.Get("HX-Request") == "true" {
        tmpl.ExecuteTemplate(w, "_items_list.html", items)
    } else {
        tmpl.ExecuteTemplate(w, "items.html", items)
    }
}
```

## Response Headers

### Triggering Client Events

```rust
use axum::{
    http::HeaderValue,
    response::{Html, IntoResponse, Response},
};

async fn create_item() -> Response {
    let html = ItemTemplate { /* ... */ }.render().unwrap();
    let mut response = Html(html).into_response();
    response.headers_mut().insert(
        "HX-Trigger",
        HeaderValue::from_static("itemCreated"),
    );
    response
}
```

With data:
```rust
use axum::http::HeaderValue;

response.headers_mut().insert(
    "HX-Trigger",
    HeaderValue::from_str(
        r#"{"itemCreated": {"id": "123", "name": "New Item"}}"#
    ).unwrap(),
);
```

### Client-Side Redirect

```rust
use axum::{
    http::{HeaderValue, StatusCode},
    response::{Html, IntoResponse, Response},
};

async fn login() -> Response {
    if authenticate() {
        let mut response = StatusCode::OK.into_response();
        response.headers_mut().insert(
            "HX-Redirect",
            HeaderValue::from_static("/dashboard"),
        );
        response
    } else {
        let html = LoginErrorTemplate {}.render().unwrap();
        (StatusCode::UNAUTHORIZED, Html(html)).into_response()
    }
}
```

### Override Swap Behavior

```rust
use axum::{http::HeaderValue, response::{Html, IntoResponse, Response}};

async fn submit() -> Response {
    let html = SuccessTemplate {}.render().unwrap();
    let mut response = Html(html).into_response();
    response.headers_mut().insert(
        "HX-Retarget",
        HeaderValue::from_static("#notifications"),
    );
    response.headers_mut().insert(
        "HX-Reswap",
        HeaderValue::from_static("beforeend"),
    );
    response
}
```

### Push URL to History

```rust
use axum::{
    extract::Path,
    http::HeaderValue,
    response::{Html, IntoResponse, Response},
};
use uuid::Uuid;

async fn show_item(Path(id): Path<Uuid>) -> Response {
    let html = ItemDetailTemplate { /* ... */ }.render().unwrap();
    let mut response = Html(html).into_response();
    response.headers_mut().insert(
        "HX-Push-Url",
        HeaderValue::from_str(&format!("/items/{id}")).unwrap(),
    );
    response
}
```

## Out-of-Band Updates

Update multiple elements with a single response. The main content goes to `hx-target`, and elements with `hx-swap-oob="true"` replace their matching element by ID:

```html
<!-- Main content goes to hx-target -->
<tr id="item-{{ item.id }}">
    <td>{{ item.name }}</td>
    <td>{{ item.price }}</td>
</tr>

<!-- OOB updates go to elements matching their id -->
<span id="item-count" hx-swap-oob="true">{{ total_items }}</span>
<div id="notification" hx-swap-oob="true">
    Item "{{ item.name }}" created!
</div>
```

### OOB in Templates (Jinja2/Askama)

A common pattern — return the main list content plus OOB updates for related UI:

```html
<!-- OOB: updates a hidden input elsewhere on the page -->
<input type="hidden" id="current-sort" name="sort" value="{{ sort }}" hx-swap-oob="true" />

<!-- OOB: updates the sort buttons -->
<div id="sort-buttons" class="d-flex gap-2" hx-swap-oob="true">
  {% include "sort_buttons.html" %}
</div>

<!-- Primary content (goes to hx-target) -->
{% include "list_table.html" %}
```

### OOB Swap Strategies

```html
<!-- Replace content (default — outerHTML) -->
<div id="target" hx-swap-oob="true">New content</div>

<!-- Append to element -->
<div id="target" hx-swap-oob="beforeend">Appended content</div>

<!-- Delete element -->
<div id="target" hx-swap-oob="delete"></div>
```

## Error Handling

### Return Appropriate Status Codes

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::{Html, IntoResponse, Response},
};
use sqlx::PgPool;
use uuid::Uuid;

async fn delete_item(
    State(pool): State<PgPool>,
    Path(id): Path<Uuid>,
) -> Response {
    match sqlx::query("DELETE FROM items WHERE id = $1")
        .bind(id)
        .execute(&pool)
        .await
    {
        Ok(_) => StatusCode::OK.into_response(),
        Err(e) => {
            tracing::error!("Database error: {}", e);
            let html = ErrorTemplate { message: "Failed to delete" }.render().unwrap();
            (StatusCode::INTERNAL_SERVER_ERROR, Html(html)).into_response()
        }
    }
}
```

### Client-Side Error Handling

Configure htmx to swap on error responses:

```javascript
htmx.config.responseHandling = [
    {code:"204", swap: false},
    {code:"[23]..", swap: true},
    {code:"422", swap: true, error: false, target: "#errors"},
    {code:"[45]..", swap: true, error: true}
];
```

## Validation Patterns

### Inline Validation

```rust
use axum::{extract::Form, response::{Html, IntoResponse}};
use serde::Deserialize;

#[derive(Deserialize)]
struct EmailInput {
    email: String,
}

async fn validate_email(Form(input): Form<EmailInput>) -> impl IntoResponse {
    if input.email.is_empty() {
        return Html(r#"<span class="error">Email is required</span>"#);
    }
    if !input.email.contains('@') {
        return Html(r#"<span class="error">Invalid email format</span>"#);
    }

    Html(r#"<span class="success">Email available</span>"#)
}
```

HTML:
```html
<input type="email" name="email"
       hx-post="/validate/email"
       hx-trigger="blur changed"
       hx-target="next .validation">
<span class="validation"></span>
```

### Form Validation Response

```rust
use axum::{
    extract::Form,
    http::StatusCode,
    response::{Html, IntoResponse, Redirect, Response},
};
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateItem {
    name: String,
}

async fn create_item(Form(input): Form<CreateItem>) -> Response {
    if input.name.is_empty() {
        let html = FormErrorTemplate { error: "Name is required" }.render().unwrap();
        return (StatusCode::UNPROCESSABLE_ENTITY, Html(html)).into_response();
    }

    // Process valid submission...
    Redirect::to("/items/").into_response()
}
```

## Partial Templates

### Template Organization (Askama + Jinja2)

```
src/{feature}/
├── handlers.rs         # HTTP handler functions
├── templates.rs        # Askama template structs
└── templates/
    ├── list.html       # List page
    ├── list_table.html # Table partial (swappable via htmx)
    ├── list_results.html # Results with OOB updates
    ├── new.html        # Create form
    ├── edit.html       # Edit form
    └── detail.html     # Detail view
```

### Askama Template Struct

```rust
use askama::Template;

#[derive(Template)]
#[template(path = "items/templates/list.html")]
struct ItemsListTemplate {
    items: Vec<Item>,
    sort: String,
    page: i64,
    total_pages: i64,
    q: String,
}
```

### Using Templates in Handlers

```rust
use askama::Template;
use axum::{extract::State, response::{Html, IntoResponse}};

async fn list_items(State(pool): State<PgPool>) -> impl IntoResponse {
    let items = fetch_items(&pool).await.unwrap_or_default();
    let template = ItemsListTemplate {
        items,
        sort: "name".to_string(),
        page: 1,
        total_pages: 1,
        q: String::new(),
    };
    Html(template.render().unwrap())
}
```

### Jinja2 Template Examples

Item row with edit and delete:
```html
<tr id="item-{{ item.id }}">
    <td>{{ item.name }}</td>
    <td>
        <button hx-get="/items/{{ item.id }}/edit"
                hx-target="closest tr"
                hx-swap="outerHTML">
            Edit
        </button>
        <button hx-delete="/items/{{ item.id }}"
                hx-target="closest tr"
                hx-swap="outerHTML swap:1s"
                hx-confirm="Delete this item?">
            Delete
        </button>
    </td>
</tr>
```

List template including partials:
```html
<table id="items-table">
    <thead>
        <tr><th>Name</th><th>Actions</th></tr>
    </thead>
    <tbody>
        {% for item in items %}
            {% include "items/templates/_row.html" %}
        {% endfor %}
    </tbody>
</table>
```

## Pagination

### Handler with Query Parameters

```rust
use axum::{extract::Query, response::{Html, IntoResponse}};
use serde::Deserialize;

#[derive(Deserialize)]
struct PaginationParams {
    #[serde(default = "default_page")]
    page: i64,
    #[serde(default = "default_page_size")]
    page_size: i64,
    #[serde(default)]
    q: String,
}

fn default_page() -> i64 { 1 }
fn default_page_size() -> i64 { 20 }

async fn list_items(Query(params): Query<PaginationParams>) -> impl IntoResponse {
    let offset = (params.page - 1) * params.page_size;
    let items = fetch_page(offset, params.page_size).await;
    let has_next = items.len() as i64 == params.page_size;

    let template = ItemsPageTemplate {
        items,
        has_next,
        next_page: params.page + 1,
    };
    Html(template.render().unwrap())
}
```

### Load More Pattern (HTML)

```html
{% for item in items %}
<div class="item">{{ item.name }}</div>
{% endfor %}

{% if has_next %}
<div hx-get="/items?page={{ next_page }}"
     hx-trigger="revealed"
     hx-swap="outerHTML">
    Loading more...
</div>
{% endif %}
```

## Search with Debounce

### Search Input with Filter + Sort

```html
<input
  type="search"
  id="item-filter"
  name="q"
  class="form-control"
  placeholder="Search items..."
  value="{{ q }}"
  autocomplete="off"
  hx-get="/items"
  hx-trigger="input changed delay:500ms, search"
  hx-target="#item-list-container"
  hx-select="#item-list-container"
  hx-swap="innerHTML"
  hx-include="#current-sort"
  hx-push-url="true"
/>
```

### Sort Buttons with hx-vals

```html
<button
  type="button"
  hx-get="/items"
  hx-target="#item-list-container"
  hx-select="#item-list-container"
  hx-swap="innerHTML"
  hx-vals='{"sort": "name"}'
  hx-include="#item-filter"
  hx-push-url="true"
>
  Name
</button>
```

Key patterns:
- `hx-vals='{"sort": "name"}'` — Override query parameter
- `hx-include="#item-filter"` — Include filter input value in request
- `hx-push-url="true"` — Update browser URL with new parameters
- `hx-select="#item-list-container"` — Filter response to swap only this section

## WebSocket/SSE Integration

For real-time updates, combine htmx with SSE extension:

```html
<div hx-ext="sse" sse-connect="/events" sse-swap="message">
    <!-- Content updated via SSE -->
</div>
```

Server (Rust/Axum):
```rust
use axum::response::sse::{Event, Sse};
use futures::stream::Stream;
use std::convert::Infallible;
use std::time::Duration;
use tokio_stream::StreamExt;

async fn events() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = tokio_stream::wrappers::IntervalStream::new(
        tokio::time::interval(Duration::from_secs(1)),
    )
    .map(|_| {
        let html = "<div>Updated content</div>".to_string();
        Ok(Event::default().data(html))
    });

    Sse::new(stream)
}
```
