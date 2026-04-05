# Streamlit Presentation Layer

The UI is a multi-page Streamlit application defined in `main.py`. It renders resource management forms, job dispatch controls, and evaluation dashboards. All data operations go through the gRPC client -- the Streamlit layer has no direct database access.

## Entry Point

`main.py` sets the page configuration and selects a navigation mode based on the `IS_COMPOSABLE` environment variable:

```python
st.set_page_config(
    page_title="Fine Tuning Studio",
    page_icon=IconPaths.FineTuningStudio.FINE_TUNING_STUDIO,
    layout="wide"
)
```

The layout is always `"wide"`. The page icon is loaded from the `resources/images/` directory via `ft.consts.IconPaths`.

## Navigation Modes

```d2
direction: right

env: IS_COMPOSABLE env var

composable: Composable Mode {
  label: "Horizontal Navbar\n(dropdown menus)"
  nav: st_navbar + HTML dropdowns
  pages: st.navigation(position='hidden')
}

standard: Standard Mode {
  label: "Sidebar Navigation\n(section headers + icons)"
  nav: st.sidebar page_links
  pages: st.navigation(position='hidden')
}

env -> composable: IS_COMPOSABLE is set
env -> standard: IS_COMPOSABLE is unset
```

### Composable Mode

Activated when `IS_COMPOSABLE` is set to any non-empty value. Uses `streamlit_navigation_bar` (`st_navbar`) combined with custom HTML/CSS for dropdown menus. Navigation groups:

| Group | Pages |
|---|---|
| Home | Home |
| Database Import Export | Database Import and Export |
| Resources | Import Datasets, View Datasets, Import Base Models, View Base Models, Create Prompts, View Prompts |
| Experiment | Train a New Adapter, Monitor Training Jobs, Local Adapter Comparison, Run MLFlow Evaluation, View MLflow Runs |
| AI Workbench | Export And Deploy Model |
| Examples | Ticketing Agent App |
| Feedback | Provide Feedback |

The navbar is rendered as a fixed-position HTML `<nav>` element with CSS dropdown menus. Links use `target="_self"` to navigate within the Streamlit app. All pages are registered with `st.navigation(position="hidden")` so that Streamlit handles routing internally while the custom navbar provides the visible UI.

### Standard Mode (Default)

When `IS_COMPOSABLE` is not set, the sidebar renders section headers and page links with Material Design icons:

```python
with st.sidebar:
    st.image("./resources/images/ft-logo.png")
    st.markdown("Navigation")
    st.page_link("pgs/home.py", label="Home", icon=":material/home:")
    st.page_link("pgs/database.py", label="Database Import and Export", icon=":material/database:")

    st.markdown("Resources")
    st.page_link("pgs/datasets.py", label="Import Datasets", icon=":material/publish:")
    st.page_link("pgs/view_datasets.py", label="View Datasets", icon=":material/data_object:")
    # ... remaining pages
```

Sidebar sections: Navigation, Resources, Experiments, AI Workbench, Examples, Feedback. The sidebar footer displays the current project owner and a link to the CML domain.

## Page Inventory

All page modules live in the `pgs/` directory:

| File | Title | Section |
|---|---|---|
| `pgs/home.py` | Home | Navigation |
| `pgs/database.py` | Database Import and Export | Database |
| `pgs/datasets.py` | Import Datasets | Resources |
| `pgs/view_datasets.py` | View Datasets | Resources |
| `pgs/models.py` | Import Base Models | Resources |
| `pgs/view_models.py` | View Base Models | Resources |
| `pgs/prompts.py` | Create Prompts | Resources |
| `pgs/view_prompts.py` | View Prompts | Resources |
| `pgs/train_adapter.py` | Train a New Adapter | Experiments |
| `pgs/jobs.py` | Training Job Tracking | Experiments |
| `pgs/evaluate.py` | Local Adapter Comparison | Experiments |
| `pgs/mlflow.py` | Run MLFlow Evaluation | Experiments |
| `pgs/mlflow_jobs.py` | View MLflow Runs | Experiments |
| `pgs/export.py` | Export And Deploy Model | AI Workbench |
| `pgs/sample_ticketing_agent_app_embed.py` | Sample Ticketing Agent App | Examples |
| `pgs/feedback.py` | Feedback | Feedback |

## Client Caching

Shared client instances are cached at the Streamlit server level using `@st.cache_resource`. This avoids creating a new gRPC channel or CML API client on every page render. Both helpers are defined in `pgs/streamlit_utils.py`:

```python
@st.cache_resource
def get_fine_tuning_studio_client() -> FineTuningStudioClient:
    client = FineTuningStudioClient()
    return client

@st.cache_resource
def get_cml_client() -> CMLServiceApi:
    client = default_client()
    return client
```

`@st.cache_resource` ensures a single instance per Streamlit server process. The gRPC client connects to the address specified by `FINE_TUNING_SERVICE_IP` and `FINE_TUNING_SERVICE_PORT` environment variables. The CML client uses `cmlapi.default_client()`, which reads CML connection parameters from the pod environment.

## Data Flow

```d2
direction: right

page: Streamlit Page {
  label: "pgs/datasets.py"
}

cache: "@st.cache_resource" {
  shape: diamond
}

client: FineTuningStudioClient {
  label: "gRPC Client"
}

grpc: gRPC Server {
  label: "port 50051"
}

domain: Domain Logic

dao: DAO

sqlite: SQLite {
  shape: cylinder
}

page -> cache: get_fine_tuning_studio_client()
cache -> client: cached instance
client -> grpc: AddDataset(AddDatasetRequest(...))
grpc -> domain: add_dataset(request, cml, dao)
domain -> dao: session.add(Dataset.from_message(request))
dao -> sqlite: INSERT
```

Every user interaction follows this path: Streamlit widget event triggers a page callback, the page calls the cached client, the client sends a gRPC request, the server delegates to domain logic, and the domain function uses the DAO to read or write SQLite.

## How to Add a New Page

1. **Create the page module** at `pgs/my_page.py`:

```python
import streamlit as st
from pgs.streamlit_utils import get_fine_tuning_studio_client

st.header("My New Page")

client = get_fine_tuning_studio_client()

# Use the client to interact with the gRPC service
models = client.get_models()
for model in models:
    st.write(model.name)
```

2. **Register the page in both navigation modes** in `main.py`:

In the composable mode `setup_navigation()` function, add:
```python
st.Page("pgs/my_page.py", title="My New Page"),
```

In the composable mode HTML navbar, add a link in the appropriate dropdown:
```html
<a href="/my_page" target="_self"><span class="material-icons">icon_name</span> My New Page</a>
```

In the standard mode `setup_navigation_sidebar()` function, add:
```python
st.Page("pgs/my_page.py", title="My New Page"),
```

In the standard mode `setup_sidebar()` function, add under the appropriate section:
```python
st.page_link("pgs/my_page.py", label="My New Page", icon=":material/icon_name:")
```

3. **If the page requires a new RPC**, add it to the protobuf definition, regenerate, implement the servicer method, and add the domain function. See [gRPC Service Design](./grpc-service.md).

## Custom CSS

Both navigation modes inject custom CSS to control typography and layout:

- Heading sizes (`h3` reduced to `1.1rem`)
- Tab label font sizes (`0.9rem`)
- Sidebar theming (dark background `#16262c`, white text) in standard mode
- Navbar positioning and dropdown behavior in composable mode

CSS is injected via `st.markdown(css, unsafe_allow_html=True)`.

## Cross-References

- [System Overview](./overview.md) -- startup sequence and environment variables
- [gRPC Service Design](./grpc-service.md) -- client wrapper and API surface
- [Data Tier](./data-tier.md) -- database schema backing the resources displayed in the UI
