import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from risk_engine import run_risk_analysis

# ── Page config ──────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="BOM Risk Analyzer",
    page_icon="⚡",
    layout="wide",
    initial_sidebar_state="expanded",
)

# ── Custom CSS ────────────────────────────────────────────────────────────────
st.markdown("""
<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;600&family=IBM+Plex+Sans:wght@300;400;600&display=swap');

html, body, [class*="css"] {
    font-family: 'IBM Plex Sans', sans-serif;
    background-color: #0d0f14;
    color: #e2e8f0;
}

.stApp { background-color: #0d0f14; }

h1, h2, h3 {
    font-family: 'IBM Plex Mono', monospace;
    letter-spacing: -0.5px;
}

.metric-card {
    background: #161b27;
    border: 1px solid #2a3044;
    border-radius: 8px;
    padding: 20px 24px;
    text-align: center;
}

.metric-value {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 2.4rem;
    font-weight: 600;
    line-height: 1;
    margin-bottom: 6px;
}

.metric-label {
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 1.5px;
    color: #64748b;
}

.critical { color: #f87171; }
.elevated { color: #fbbf24; }
.low-risk { color: #34d399; }

.section-header {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 0.7rem;
    text-transform: uppercase;
    letter-spacing: 3px;
    color: #475569;
    margin-bottom: 12px;
    padding-bottom: 8px;
    border-bottom: 1px solid #1e2535;
}

div[data-testid="stSidebar"] {
    background-color: #0a0c10;
    border-right: 1px solid #1e2535;
}

.stSlider > div > div { background: #2a3044; }
</style>
""", unsafe_allow_html=True)

# ── Load Data ─────────────────────────────────────────────────────────────────
@st.cache_data
def load_data():
    return pd.read_csv("data/smart_speaker_bom.csv")

raw_df = load_data()

# ── Sidebar Controls ──────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("### ⚡ BOM Risk Analyzer")
    st.markdown("<div class='section-header'>Risk Weight Configuration</div>", unsafe_allow_html=True)
    st.caption("Adjust scoring weights to reflect your sourcing priorities")

    w_single = st.slider("Single-Source Exposure", 0, 100, 35, 5) / 100
    w_geo    = st.slider("Geographic Concentration", 0, 100, 25, 5) / 100
    w_lt     = st.slider("Lead Time Volatility", 0, 100, 25, 5) / 100
    w_sub    = st.slider("Substitutability", 0, 100, 15, 5) / 100

    total_w = w_single + w_geo + w_lt + w_sub
    if abs(total_w - 1.0) > 0.01:
        st.warning(f"Weights sum to {total_w:.2f}. Normalize to 1.0 for best results.")

    st.markdown("---")
    st.markdown("<div class='section-header'>Filters</div>", unsafe_allow_html=True)

    categories = ["All"] + sorted(raw_df["category"].unique().tolist())
    selected_cat = st.selectbox("Component Category", categories)

    tiers = ["All", "🔴 Critical", "🟡 Elevated", "🟢 Low"]
    selected_tier = st.selectbox("Risk Tier", tiers)

    countries = ["All"] + sorted(raw_df["supplier_country"].unique().tolist())
    selected_country = st.selectbox("Supplier Country", countries)

# ── Run Analysis ──────────────────────────────────────────────────────────────
weights = {
    "single_source": w_single,
    "geo_concentration": w_geo,
    "lead_time_volatility": w_lt,
    "substitutability": w_sub,
}
df = run_risk_analysis(raw_df, weights)

# Apply filters
filtered = df.copy()
if selected_cat != "All":
    filtered = filtered[filtered["category"] == selected_cat]
if selected_tier != "All":
    filtered = filtered[filtered["risk_tier"] == selected_tier]
if selected_country != "All":
    filtered = filtered[filtered["supplier_country"] == selected_country]

# ── Header ────────────────────────────────────────────────────────────────────
st.markdown("# BOM Risk Analyzer")
st.markdown("**Smart Speaker · Supply Chain Risk Intelligence**")
st.markdown("---")

# ── KPI Row ───────────────────────────────────────────────────────────────────
col1, col2, col3, col4, col5 = st.columns(5)

critical_count = len(df[df["risk_tier"] == "🔴 Critical"])
elevated_count = len(df[df["risk_tier"] == "🟡 Elevated"])
low_count       = len(df[df["risk_tier"] == "🟢 Low"])
avg_score       = df["composite_risk_score"].mean()
total_spend     = df["annual_spend_usd"].sum()

with col1:
    st.markdown(f"""
    <div class='metric-card'>
        <div class='metric-value critical'>{critical_count}</div>
        <div class='metric-label'>Critical Risk</div>
    </div>""", unsafe_allow_html=True)

with col2:
    st.markdown(f"""
    <div class='metric-card'>
        <div class='metric-value elevated'>{elevated_count}</div>
        <div class='metric-label'>Elevated Risk</div>
    </div>""", unsafe_allow_html=True)

with col3:
    st.markdown(f"""
    <div class='metric-card'>
        <div class='metric-value low-risk'>{low_count}</div>
        <div class='metric-label'>Low Risk</div>
    </div>""", unsafe_allow_html=True)

with col4:
    st.markdown(f"""
    <div class='metric-card'>
        <div class='metric-value' style='color:#7c93c3'>{avg_score:.0f}</div>
        <div class='metric-label'>Avg Risk Score</div>
    </div>""", unsafe_allow_html=True)

with col5:
    st.markdown(f"""
    <div class='metric-card'>
        <div class='metric-value' style='color:#a78bfa'>${total_spend/1e6:.2f}M</div>
        <div class='metric-label'>Annual Spend</div>
    </div>""", unsafe_allow_html=True)

st.markdown("<br>", unsafe_allow_html=True)

# ── Charts Row ────────────────────────────────────────────────────────────────
left, right = st.columns([3, 2])

with left:
    st.markdown("<div class='section-header'>Component Risk Heatmap</div>", unsafe_allow_html=True)

    color_map = {"🔴 Critical": "#f87171", "🟡 Elevated": "#fbbf24", "🟢 Low": "#34d399"}

    fig_bar = px.bar(
        filtered.sort_values("composite_risk_score", ascending=True),
        x="composite_risk_score",
        y="component_name",
        orientation="h",
        color="risk_tier",
        color_discrete_map=color_map,
        text="composite_risk_score",
        hover_data=["supplier", "supplier_country", "lead_time_weeks", "single_sourced"],
    )
    fig_bar.update_traces(texttemplate='%{text:.0f}', textposition='outside')
    fig_bar.update_layout(
        plot_bgcolor="#161b27",
        paper_bgcolor="#0d0f14",
        font=dict(color="#e2e8f0", family="IBM Plex Mono"),
        xaxis=dict(title="Composite Risk Score (0–100)", gridcolor="#1e2535", range=[0, 115]),
        yaxis=dict(title="", gridcolor="#1e2535"),
        legend_title="Risk Tier",
        height=480,
        margin=dict(l=10, r=30, t=10, b=10),
        showlegend=False,
    )
    st.plotly_chart(fig_bar, use_container_width=True)

with right:
    st.markdown("<div class='section-header'>Geographic Exposure</div>", unsafe_allow_html=True)

    geo_spend = df.groupby("supplier_country")["annual_spend_usd"].sum().reset_index()
    fig_pie = px.pie(
        geo_spend,
        names="supplier_country",
        values="annual_spend_usd",
        color_discrete_sequence=["#f87171","#fbbf24","#34d399","#7c93c3","#a78bfa","#38bdf8"],
        hole=0.55,
    )
    fig_pie.update_layout(
        plot_bgcolor="#161b27",
        paper_bgcolor="#0d0f14",
        font=dict(color="#e2e8f0", family="IBM Plex Mono"),
        legend=dict(bgcolor="#0d0f14"),
        height=300,
        margin=dict(l=10, r=10, t=10, b=10),
    )
    st.plotly_chart(fig_pie, use_container_width=True)

    st.markdown("<div class='section-header'>Risk Score by Category</div>", unsafe_allow_html=True)
    cat_risk = df.groupby("category")["composite_risk_score"].mean().reset_index().sort_values("composite_risk_score", ascending=False)
    fig_cat = px.bar(
        cat_risk,
        x="composite_risk_score",
        y="category",
        orientation="h",
        color="composite_risk_score",
        color_continuous_scale=["#34d399", "#fbbf24", "#f87171"],
        range_color=[0, 100],
    )
    fig_cat.update_layout(
        plot_bgcolor="#161b27",
        paper_bgcolor="#0d0f14",
        font=dict(color="#e2e8f0", family="IBM Plex Mono"),
        xaxis=dict(title="Avg Risk Score", gridcolor="#1e2535"),
        yaxis=dict(title="", gridcolor="#1e2535"),
        coloraxis_showscale=False,
        height=220,
        margin=dict(l=10, r=10, t=10, b=10),
    )
    st.plotly_chart(fig_cat, use_container_width=True)

# ── Spider / Radar Chart ──────────────────────────────────────────────────────
st.markdown("<div class='section-header'>Risk Dimension Breakdown — Top 5 Critical Components</div>", unsafe_allow_html=True)

top5 = df.nlargest(5, "composite_risk_score")
dims = ["single_source_score", "geo_concentration_score", "lead_time_volatility_score", "substitutability_score"]
dim_labels = ["Single Source", "Geo Concentration", "Lead Time Volatility", "Substitutability"]

fig_radar = go.Figure()
colors = ["#f87171", "#fbbf24", "#fb923c", "#e879f9", "#38bdf8"]

for i, (_, row) in enumerate(top5.iterrows()):
    values = [row[d] for d in dims]
    values += [values[0]]
    fig_radar.add_trace(go.Scatterpolar(
        r=values,
        theta=dim_labels + [dim_labels[0]],
        fill='toself',
        name=row["component_name"],
        line_color=colors[i],
        fillcolor=colors[i],
        opacity=0.2,
    ))

fig_radar.update_layout(
    polar=dict(
        bgcolor="#161b27",
        radialaxis=dict(visible=True, range=[0, 100], gridcolor="#2a3044", color="#64748b"),
        angularaxis=dict(gridcolor="#2a3044", color="#94a3b8"),
    ),
    plot_bgcolor="#0d0f14",
    paper_bgcolor="#0d0f14",
    font=dict(color="#e2e8f0", family="IBM Plex Mono"),
    legend=dict(bgcolor="#0d0f14", bordercolor="#2a3044"),
    height=380,
    margin=dict(l=50, r=50, t=20, b=20),
)
st.plotly_chart(fig_radar, use_container_width=True)

# ── Data Table ────────────────────────────────────────────────────────────────
st.markdown("<div class='section-header'>Full BOM Risk Table</div>", unsafe_allow_html=True)

display_cols = [
    "component_name", "category", "supplier", "supplier_country",
    "lead_time_weeks", "single_sourced", "num_alternatives",
    "composite_risk_score", "risk_tier", "annual_spend_usd"
]

display_df = filtered[display_cols].sort_values("composite_risk_score", ascending=False).copy()
display_df["annual_spend_usd"] = display_df["annual_spend_usd"].apply(lambda x: f"${x:,.0f}")
display_df.columns = [
    "Component", "Category", "Supplier", "Country",
    "Lead Time (wks)", "Single Sourced", "# Alternatives",
    "Risk Score", "Risk Tier", "Annual Spend"
]

st.dataframe(display_df, use_container_width=True, hide_index=True)

# ── Footer ────────────────────────────────────────────────────────────────────
st.markdown("---")
st.caption("BOM Risk Analyzer · Smart Speaker Supply Chain · Built with Streamlit + Plotly")
