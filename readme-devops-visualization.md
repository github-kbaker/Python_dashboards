import pandas as pd
import panel as pn
import hvplot.pandas  # noqa: F401

pn.extension("tabulator")

# -----------------------------
# 1) Load data
# -----------------------------
DATA_PATH = "my_apps/data.csv"

df = pd.read_csv(DATA_PATH)

if df.empty:
    raise ValueError(f"Loaded dataframe is empty. Check if '{DATA_PATH}' exists and has rows.")

# Normalize column names (strip spaces)
df.columns = [c.strip() for c in df.columns]

# Ensure date exists + is datetime (if present)
if "date" in df.columns:
    df["date"] = pd.to_datetime(df["date"], errors="coerce")

# -----------------------------
# 2) Pick columns safely
# -----------------------------
# Category column
if "category" in df.columns:
    category_col = "category"
else:
    text_cols = df.select_dtypes(include=["object", "string"]).columns.tolist()
    category_col = text_cols[0] if text_cols else df.columns[0]

# Numeric/value column
if "value" in df.columns:
    df["value"] = pd.to_numeric(df["value"], errors="coerce")

numeric_cols = df.select_dtypes(include=["number"]).columns.tolist()
if "value" in numeric_cols:
    value_col = "value"
elif numeric_cols:
    value_col = numeric_cols[0]
else:
    raise ValueError("No numeric columns found. Add a numeric column like 'value'.")

# Drop rows where value is missing
df = df.dropna(subset=[value_col])

# -----------------------------
# 3) Widgets (filters)
# -----------------------------
categories = sorted(df[category_col].dropna().unique().tolist())
category_select = pn.widgets.MultiChoice(
    name="Category",
    options=categories,
    value=categories[:],
)

vmin = float(df[value_col].min())
vmax = float(df[value_col].max())
value_range = pn.widgets.RangeSlider(
    name=f"{value_col} range",
    start=vmin,
    end=vmax,
    value=(vmin, vmax),
)

date_range = None
if "date" in df.columns and df["date"].notna().any():
    dmin = df["date"].min()
    dmax = df["date"].max()
    date_range = pn.widgets.DateRangeSlider(
        name="Date range",
        start=dmin,
        end=dmax,
        value=(dmin, dmax),
    )

# -----------------------------
# 4) Filtering + views (bind-based, reliable)
# -----------------------------
def filtered_df(selected_categories, vrange, drange=None):
    dff = df.copy()

    if selected_categories:
        dff = dff[dff[category_col].isin(selected_categories)]

    lo, hi = vrange
    dff = dff[(dff[value_col] >= lo) & (dff[value_col] <= hi)]

    if drange is not None and "date" in dff.columns and dff["date"].notna().any():
        start, end = drange
        dff = dff[(dff["date"] >= start) & (dff["date"] <= end)]

    return dff


def kpis(dff):
    total = float(dff[value_col].sum()) if not dff.empty else 0.0
    avg = float(dff[value_col].mean()) if not dff.empty else 0.0
    rows = int(len(dff))

    return pn.Row(
        pn.Card(f"### Total\n# {total:,.0f}", width=220),
        pn.Card(f"### Average\n# {avg:,.1f}", width=220),
        pn.Card(f"### Rows\n# {rows:,}", width=220),
    )


def chart(dff):
    if dff.empty:
        return pn.pane.Markdown("No data after filtering.")

    summary = dff.groupby(category_col, as_index=False)[value_col].sum()

    return summary.hvplot.bar(
        x=category_col,
        y=value_col,
        height=320,
        responsive=True,
        title="Value by Category",
        rot=0,
    )


def table(dff):
    return pn.widgets.Tabulator(
        dff,
        pagination="remote",
        page_size=12,
        sizing_mode="stretch_width",
        height=360,
    )


# Build a reactive dataframe from widgets
dff = pn.bind(
    filtered_df,
    category_select,
    value_range,
    date_range if date_range is not None else None,
)

# -----------------------------
# 5) Layout
# -----------------------------
controls_items = [category_select, value_range]
if date_range is not None:
    controls_items.append(date_range)

controls = pn.Card(*controls_items, title="Filters")

dashboard = pn.Row(
    pn.Column("# DevOps Visualization Dashboard", controls, width=360),
    pn.Column(
        pn.bind(kpis, dff),
        pn.bind(chart, dff),
        pn.bind(table, dff),
        sizing_mode="stretch_width",
    ),
    sizing_mode="stretch_width",
)

dashboard.servable()
