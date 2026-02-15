---
name: htmxology-expert
description: Use when working with htmxology — the type-safe HTMX routing framework for Rust/Axum. Covers Route enums, Controller trait, RoutingController composition, template route helpers, as_htmx_attribute(), HtmxRequest detection (classic vs HTMX), Response envelope, out-of-band updates, DisplayDelegate, and ControllerRouter integration. Use when adding routes, creating controllers, building templates with route helpers, or handling HTMX vs classic requests.
---

# htmxology Expert

htmxology is a type-safe HTMX routing framework for Rust/Axum. It replaces string-based routing with enum-based routes that are verified at compile time, and provides automatic HTMX request detection, response building, and out-of-band swap support.

## Core Concepts

- **Routes are enums** — no string URLs to mistype
- **Controllers handle routes** — match arms dispatch to handlers
- **RoutingController composes controllers** — hierarchical nesting with path prefixes
- **Templates use route helpers** — type-safe URL generation in Jinja2
- **HtmxRequest detection** — automatic classic vs HTMX differentiation
- **OOB updates** — typed out-of-band swaps via Identity + Fragment traits

## Route Definition

Define routes as enum variants with `#[derive(htmxology::Route)]`:

```rust
#[derive(Debug, Clone, htmxology::Route)]
pub enum CoffeesRoute {
    /// GET /coffees/ (with query params)
    #[route("")]
    ListCoffees(#[query] SortParams),

    /// GET /coffees/new
    #[route("new")]
    NewCoffee,

    /// POST /coffees/
    #[route("", method = "POST")]
    CreateCoffee(#[body] CreateCoffee),

    /// GET /coffees/{id}
    #[route("{id}")]
    ShowCoffee(Uuid),

    /// GET /coffees/{id}/edit
    #[route("{id}/edit")]
    EditCoffee(Uuid),

    /// PUT /coffees/{id}
    #[route("{id}", method = "PUT")]
    UpdateCoffee(Uuid, #[body] CreateCoffee),

    /// POST /coffees/{id} (form compatibility)
    #[route("{id}", method = "POST")]
    UpdateCoffeePost(Uuid, #[body] CreateCoffee),

    /// DELETE /coffees/{id}
    #[route("{id}", method = "DELETE")]
    DeleteCoffee(Uuid),
}
```

### Route Attributes

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `#[route("path")]` | URL path pattern | `#[route("{id}/edit")]` |
| `method = "..."` | HTTP method (default: GET) | `method = "POST"`, `method = "DELETE"` |
| `#[query]` | Extract query parameters | `ListCoffees(#[query] SortParams)` |
| `#[body]` | Extract request body (form/JSON) | `CreateCoffee(#[body] CreateCoffee)` |
| Path params | Tuple fields become path segments | `ShowCoffee(Uuid)` matches `{id}` |

### Query Parameter Serialization

Use serde attributes for clean URLs — defaults are skipped:

```rust
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct SortParams {
    #[serde(default = "default_sort", skip_serializing_if = "is_default_sort")]
    pub sort: String,
    #[serde(default = "default_page", skip_serializing_if = "is_default_page")]
    pub page: i64,
    #[serde(default, skip_serializing_if = "String::is_empty")]
    pub q: String,
}
```

## Controller Trait

Each feature implements `Controller` to handle its routes:

```rust
use htmxology::{Controller, ServerInfo, htmx::Request as HtmxRequest};

impl Controller for CoffeesController {
    type Route = CoffeesRoute;
    type Args = AppArgs;
    type Response = Result<crate::response::Response, axum::response::Response>;

    async fn handle_request(
        &self,
        route: Self::Route,
        _htmx: HtmxRequest,
        mut parts: Parts,
        _server_info: &ServerInfo,
        _args: Self::Args,
    ) -> Self::Response {
        let state = State(self.pool.clone());
        let auth_ctx = AuthContext::from_request_parts(&mut parts, &()).await?;

        match route {
            CoffeesRoute::ListCoffees(params) => {
                handlers::list_coffees(state, auth_ctx, params).await
            }
            CoffeesRoute::NewCoffee => handlers::show_new_coffee_form(state).await,
            CoffeesRoute::CreateCoffee(body) => {
                handlers::create_coffee(state, auth_ctx, Form(body)).await
            }
            CoffeesRoute::ShowCoffee(id) => {
                handlers::show_coffee_detail(state, auth_ctx, Path(id)).await
            }
            CoffeesRoute::EditCoffee(id) => {
                handlers::show_edit_coffee_form(state, auth_ctx, Path(id)).await
            }
            CoffeesRoute::UpdateCoffee(id, body) | CoffeesRoute::UpdateCoffeePost(id, body) => {
                handlers::update_coffee(state, auth_ctx, Path(id), Form(body)).await
            }
            CoffeesRoute::DeleteCoffee(id) => {
                handlers::delete_coffee(state, auth_ctx, Path(id)).await
            }
        }
    }
}
```

### Controller Associated Types

| Type | Purpose |
|------|---------|
| `Route` | The route enum this controller handles |
| `Args` | Shared context passed to handlers (e.g., DB pool + user prefs) |
| `Response` | Return type — typically `Result<Response, axum::response::Response>` |

### Controller Constructor Pattern

Controllers provide a `from_app` factory for the RoutingController to instantiate them:

```rust
impl CoffeesController {
    pub fn new(pool: sqlx::PgPool) -> Self {
        Self { pool }
    }

    pub fn from_app(app: &AppController) -> Self {
        Self::new(app.pool().clone())
    }
}
```

## RoutingController (Composite)

Use `#[derive(RoutingController)]` to compose multiple controllers into a hierarchy:

```rust
use htmxology::RoutingController;

#[derive(Clone, RoutingController)]
#[controller(AppControllerRoute, response = Result<crate::response::Response, axum::response::Response>, args = AppArgs)]
#[subcontroller(HomeController, route = Home)]
#[subcontroller(AuthController, route = Auth, path = "auth/", convert_response = "AppController::convert_auth_response")]
#[subcontroller(CoffeesController, route = Coffees, path = "coffees/", convert_with = "CoffeesController::from_app")]
#[subcontroller(RoastersController, route = Roasters, path = "roasters/", convert_with = "RoastersController::from_app")]
#[subcontroller(UserController, route = User, path = "user/", convert_with = "UserController::from_app")]
#[subcontroller(PreferencesController, route = Preferences, path = "preferences/", convert_response = "AppController::convert_preferences_response")]
pub struct AppController {
    pool: PgPool,
}
```

### RoutingController Attributes

| Attribute | Purpose |
|-----------|---------|
| `#[controller(RouteName, response = ..., args = ...)]` | Defines the auto-generated route enum name and types |
| `route = Variant` | Variant name in the generated `AppControllerRoute` enum |
| `path = "prefix/"` | URL path prefix for this subcontroller |
| `convert_with = "fn"` | Factory function to create subcontroller from parent |
| `convert_response = "fn"` | Custom function to convert subcontroller response to parent response |

### Auto-Generated Route Enum

The macro generates a composite route enum:

```rust
// Auto-generated by RoutingController derive
pub enum AppControllerRoute {
    Home(HomeRoute),
    Auth(AuthRoute),
    Coffees(CoffeesRoute),
    Roasters(RoastersRoute),
    User(UserRoute),
    Preferences(PreferencesRoute),
}
```

This enum implements `Display` (for URL generation) and `Route` (for parsing).

### Response Conversion

When a subcontroller has a different response type, provide a converter:

```rust
impl AppController {
    fn convert_auth_response(
        &self,
        _htmx: &htmxology::htmx::Request,
        _parts: &http::request::Parts,
        _server_info: &htmxology::ServerInfo,
        _args: &AppArgs,
        response: <AuthController as htmxology::Controller>::Response,
    ) -> <Self as htmxology::Controller>::Response {
        Err(response.unwrap_or_else(|e| e))
    }
}
```

## Template Route Helpers

Template structs expose methods that return `AppControllerRoute` values for type-safe URL generation:

```rust
use askama::Template;

#[derive(Template)]
#[template(path = "coffees/templates/list.html")]
pub struct CoffeesListTemplate {
    pub coffees: Vec<CoffeeListItem>,
    pub sort: String,
    pub q: String,
}

impl CoffeesListTemplate {
    pub fn new_route(&self) -> AppControllerRoute {
        AppControllerRoute::Coffees(CoffeesRoute::NewCoffee)
    }

    pub fn list_route(&self) -> AppControllerRoute {
        AppControllerRoute::Coffees(CoffeesRoute::ListCoffees(SortParams::default()))
    }

    pub fn detail_route(&self, id: &Uuid) -> AppControllerRoute {
        AppControllerRoute::Coffees(CoffeesRoute::ShowCoffee(*id))
    }

    pub fn edit_route(&self, id: &Uuid) -> AppControllerRoute {
        AppControllerRoute::Coffees(CoffeesRoute::EditCoffee(*id))
    }

    pub fn delete_route(&self, id: &Uuid) -> AppControllerRoute {
        AppControllerRoute::Coffees(CoffeesRoute::DeleteCoffee(*id))
    }
}
```

### Using Routes in Jinja2 Templates

Routes auto-convert to URL strings via their `Display` implementation:

```html
{# Regular link — route renders as URL string #}
<a href="{{ self.new_route() }}" class="btn btn-primary">+ Add Coffee</a>

{# Link with path parameter #}
<a href="{{ self.detail_route(coffee.id) }}">{{ coffee.name }}</a>

{# HTMX GET with type-safe route #}
<input hx-get="{{ self.list_route() }}"
       hx-trigger="input changed delay:500ms, search"
       hx-target="#coffee-list-container"
       hx-select="#coffee-list-container"
       hx-swap="innerHTML"
       hx-push-url="true" />
```

### `as_htmx_attribute()` for Method-Aware Routes

For routes with non-GET methods (POST, PUT, DELETE), use `as_htmx_attribute()` to automatically generate the correct `hx-*` attribute:

```html
{# Generates: hx-delete="/coffees/{id}" #}
<button {{ self.delete_route(coffee.id).as_htmx_attribute() | safe }}
        hx-confirm="Delete this coffee?"
        hx-target="closest tr"
        hx-swap="outerHTML swap:1s">
  Delete
</button>
```

**Critical**: Always use `| safe` filter with `as_htmx_attribute()` — it outputs raw HTML attributes.

### Cross-Controller Routes

Template route helpers can reference routes from other controllers:

```rust
impl NewCoffeeTemplate {
    pub fn roaster_autocomplete_route(&self) -> AppControllerRoute {
        AppControllerRoute::Roasters(RoastersRoute::Autocomplete(AutocompleteParams::default()))
    }
}
```

## HTMX Request Detection

htmxology automatically detects HTMX requests via the `HX-Request` header and provides a typed enum:

```rust
use htmxology::htmx::Request as HtmxRequest;

match htmx {
    HtmxRequest::Classic => {
        // Full browser navigation — wrap content in Root template
        let root = Root {
            breadcrumbs: breadcrumbs.unwrap_or_default(),
            main_container: MainContainer(page),
            color_theme,
        };
        root.render_into_response()
    }
    HtmxRequest::Htmx { .. } => {
        // HTMX request — return partial HTML (MainContainer only)
        let main_container = MainContainer(page);
        let mut response = main_container.into_htmx_response();

        // Attach out-of-band updates
        if let Some(breadcrumbs) = breadcrumbs {
            response = response.with_oob(breadcrumbs);
        }

        response.into_response()
    }
}
```

The `HtmxRequest::Htmx` variant also carries metadata:
- `boosted: bool` — whether `hx-boost` triggered the request
- `current_url: String` — the page URL when the request was made
- `target: Option<...>` — the `hx-target` element ID
- `trigger: Option<...>` — the triggering element ID

## Response Envelope

Use a response enum to standardize what handlers return:

```rust
pub enum Response {
    /// Empty response — e.g., after a DELETE
    Empty,

    /// Page with optional breadcrumbs — App wrapper handles classic vs HTMX
    Page {
        page: Page,
        breadcrumbs: Option<Breadcrumbs>,
    },

    /// Direct HTMX response — bypasses App wrapper entirely
    Htmx(htmxology::htmx::Response<String>),
}

pub type HtmxResponse = htmxology::htmx::Response<String>;
```

### Handler Return Patterns

```rust
// Page response (most common) — App wraps in Root or MainContainer
Ok(Response::Page {
    page: Page::Coffees(Box::new(Pages::List(template))),
    breadcrumbs: None,
})

// Empty response — returns 200 OK with no body
Ok(Response::Empty)

// Redirect via error path — Axum's Redirect is not a Response variant
Err(Redirect::to("/coffees/").into_response())

// Error response
Err((StatusCode::NOT_FOUND, "Coffee not found").into_response())
```

## Out-of-Band Updates

OOB swaps let a single response update multiple page sections. htmxology provides typed support.

### Define an OOB Fragment

Implement `Identity` (stable HTML id) and `Fragment` (swap strategy) traits:

```rust
use askama::Template;
use htmxology::htmx::{Fragment, HtmlId, Identity, InsertStrategy};

#[derive(Template, Default)]
#[template(path = "breadcrumbs.html")]
pub struct Breadcrumbs {
    items: Vec<BreadcrumbItem>,
}

impl Identity for Breadcrumbs {
    fn id(&self) -> HtmlId {
        HtmlId::from_static("breadcrumbs").expect("valid id")
    }
}

impl Fragment for Breadcrumbs {
    fn insert_strategy(&self) -> InsertStrategy {
        InsertStrategy::OuterHtml
    }
}
```

### Attach OOB Fragments to Responses

```rust
let main_container = MainContainer(page);
let mut response = main_container.into_htmx_response();

if let Some(breadcrumbs) = breadcrumbs {
    response = response.with_oob(breadcrumbs);
}

response.into_response()
```

### Template-Side OOB

In templates, use `hx-swap-oob="true"` on elements with stable IDs:

```html
<!-- OOB: updates the hidden sort input -->
<input type="hidden" id="current-sort" name="sort" value="{{ sort }}" hx-swap-oob="true" />

<!-- OOB: updates the sort buttons -->
<div id="sort-buttons" class="d-flex gap-2" hx-swap-oob="true">
  {% include "coffees/templates/sort_buttons.html" %}
</div>

<!-- Primary content (goes to hx-target) -->
{% include "coffees/templates/list_table.html" %}
```

## DisplayDelegate

Use `#[derive(DisplayDelegate)]` to auto-implement `Display` for page enums that delegate to their inner types:

```rust
use htmxology::DisplayDelegate;

#[derive(DisplayDelegate)]
pub enum Page {
    Coffees(Box<coffees::pages::Pages>),
    Roasters(roasters::pages::Pages),
    User(user::pages::Pages),
}

#[derive(DisplayDelegate)]
pub enum Pages {
    List(CoffeesListTemplate),
    Detail(CoffeeDetailTemplate),
    New(NewCoffeeTemplate),
    Edit(EditCoffeeTemplate),
}
```

This allows `{{ page|safe }}` in templates to render whichever variant is active.

## ControllerRouter Integration

Wire the controller hierarchy into Axum:

```rust
use htmxology::ControllerRouter;

// Create router from controller + args factory
let controller_router = ControllerRouter::new(controller, |c| {
    let pool = c.pool().clone();
    async move { pool }
});

// Convert to Axum Router
let router: axum::Router = controller_router.into();
```

### Server Setup

```rust
use htmxology::{Server, ServerOptions};

let base_url: http::Uri = "http://localhost:3000".parse()?;
let options = ServerOptions {
    base_url: Some(base_url),
};

let server = Server::builder(listener)
    .with_options(options)
    .build();

server.serve_with_router(controller_router).await?;
```

## Feature Module Pattern

Each feature is self-contained — Rust code **and** HTML templates live together in the same module directory under `src/`. This keeps features cohesive and easy to move or refactor.

Standard directory layout for each feature:

```
src/{feature}/
├── mod.rs              # Module exports
├── controller.rs       # Route enum + Controller impl
├── handlers.rs         # Async handler functions
├── pages.rs            # Pages enum with DisplayDelegate
├── templates.rs        # Askama template structs + route helpers
├── data.rs             # Data access layer (optional)
├── queries.rs          # SQL queries (optional)
└── templates/
    ├── list.html       # List page
    ├── list_table.html # Table partial (HTMX swappable)
    ├── list_results.html # Results with OOB updates
    ├── sort_buttons.html # Sort controls
    ├── new.html        # Create form
    ├── edit.html       # Edit form
    └── detail.html     # Detail view
```

### Adding a New Feature

1. Create `src/{feature}/` directory with the files above
2. Define `Route` enum with `#[derive(htmxology::Route)]`
3. Implement `Controller` trait with route matching
4. Create template structs with route helper methods
5. Wire into `AppController` with `#[subcontroller(...)]`

## Common Patterns

### hx-boost for SPA-like Navigation

Set on the body to boost all links and forms:

```html
<body hx-boost="true"
      hx-target="#main-container"
      hx-swap="outerHTML"
      hx-indicator=".htmx-indicator">
```

All regular links become HTMX requests. The App controller detects `HtmxRequest::Htmx` and returns partials.

### Delete with Row Removal

```html
<button {{ self.delete_route(coffee.id).as_htmx_attribute() | safe }}
        hx-confirm="Delete this coffee?"
        hx-target="closest tr"
        hx-swap="outerHTML swap:1s">
  <i class="bi bi-trash"></i>
</button>
```

Handler returns `Ok(Response::Empty)` — the row is removed from the DOM.

### Search + Sort + Filter

Combine `hx-include` and `hx-vals` to compose query parameters from multiple inputs:

```html
<!-- Search input includes sort param -->
<input type="search" name="q"
       hx-get="{{ self.list_route() }}"
       hx-trigger="input changed delay:500ms, search"
       hx-target="#coffee-list-container"
       hx-select="#coffee-list-container"
       hx-swap="innerHTML"
       hx-include="#current-sort"
       hx-push-url="true" />

<!-- Sort buttons include search param -->
<button hx-get="{{ self.list_route() }}"
        hx-target="#coffee-list-container"
        hx-select="#coffee-list-container"
        hx-swap="innerHTML"
        hx-vals='{"sort": "name"}'
        hx-include="#coffee-filter"
        hx-push-url="true">
  Name
</button>
```

### RenderIntoResponse for Full Pages

```rust
use htmxology::RenderIntoResponse;

// Renders Askama template directly into an axum Response
root.render_into_response()
```

### into_htmx_response() for Partials

```rust
// Renders template into an HTMX-aware response (supports .with_oob())
let response = main_container.into_htmx_response();
```
