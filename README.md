import numpy as np
import pandas as pd
import streamlit as st
#tests
# ---------------------------
# 1. Create / load data
# ---------------------------

# 🔹 Option A: Generate sample traffic data (works out of the box)
def generate_sample_data():
    np.random.seed(42)

    # 7 days of hourly data
    time_index = pd.date_range(start="2025-01-01", periods=24*7, freq="H")

    segment_ids = [f"Segment_{i}" for i in range(1, 6)]  # 5 road segments

    data = []
    for seg in segment_ids:
        base_volume = np.random.randint(200, 800)
        base_speed = np.random.randint(30, 70)

        volume = base_volume + np.random.randint(-100, 100, size=len(time_index))
        volume = np.clip(volume, 0, None)

        # Speeds decrease when volume is high
        speed = base_speed - (volume - volume.mean()) / 80
        speed = np.clip(speed, 5, 100)

        df_seg = pd.DataFrame({
            "timestamp": time_index,
            "segment": seg,
            "volume": volume,
            "avg_speed": speed
        })
        data.append(df_seg)

    df = pd.concat(data, ignore_index=True)
    return df


# 🔹 Option B: Load your own CSV instead
# Make sure your CSV has at least: timestamp, segment, volume, avg_speed columns
def load_data():
    # Example: replace this with your real file path
    # df = pd.read_csv("traffic_data.csv", parse_dates=["timestamp"])
    # return df
    return generate_sample_data()


df = load_data()

# ---------------------------
# 2. Streamlit dashboard UI
# ---------------------------

st.set_page_config(page_title="Traffic Flow Dashboard", layout="wide")
st.title("🚦 Traffic Flow Analysis Dashboard")

st.markdown(
    "Use the controls in the sidebar to explore traffic volume and speeds "
    "across different road segments and time periods."
)

# Sidebar filters
st.sidebar.header("Filters")

# Segment filter
all_segments = sorted(df["segment"].unique())
selected_segment = st.sidebar.selectbox(
    "Select road segment", 
    ["All"] + all_segments
)

# Date range filter
min_date = df["timestamp"].min().date()
max_date = df["timestamp"].max().date()

date_range = st.sidebar.date_input(
    "Select date range",
    value=(min_date, max_date),
    min_value=min_date,
    max_value=max_date
)

# Apply filters
filtered_df = df.copy()

# Filter by segment
if selected_segment != "All":
    filtered_df = filtered_df[filtered_df["segment"] == selected_segment]

# Filter by date range
if isinstance(date_range, tuple) or isinstance(date_range, list):
    start_date, end_date = date_range
else:
    start_date = end_date = date_range

filtered_df = filtered_df[
    (filtered_df["timestamp"].dt.date >= start_date) &
    (filtered_df["timestamp"].dt.date <= end_date)
]

# ---------------------------
# 3. KPI cards
# ---------------------------

col1, col2, col3 = st.columns(3)

total_volume = int(filtered_df["volume"].sum())
avg_speed_overall = filtered_df["avg_speed"].mean()
peak_hour = filtered_df.groupby(filtered_df["timestamp"].dt.hour)["volume"].sum().idxmax()

col1.metric("Total Volume (vehicles)", f"{total_volume:,}")
col2.metric("Average Speed (km/h)", f"{avg_speed_overall:,.1f}")
col3.metric("Peak Hour (24h clock)", f"{peak_hour}:00")

# ---------------------------
# 4. Charts
# ---------------------------

st.subheader("Traffic Volume Over Time")
volume_time = (
    filtered_df
    .set_index("timestamp")
    .resample("1H")["volume"]
    .sum()
)

st.line_chart(volume_time)

st.subheader("Average Speed Over Time")
speed_time = (
    filtered_df
    .set_index("timestamp")
    .resample("1H")["avg_speed"]
    .mean()
)

st.line_chart(speed_time)

# Volume by segment (even if single segment, it shows context)
st.subheader("Average Volume by Segment")
vol_by_segment = (
    filtered_df
    .groupby("segment")["volume"]
    .mean()
    .sort_values(ascending=False)
)

st.bar_chart(vol_by_segment)

# ---------------------------
# 5. Raw data preview
# ---------------------------

with st.expander("Show raw data"):
    st.dataframe(filtered_df.sort_values("timestamp"))
