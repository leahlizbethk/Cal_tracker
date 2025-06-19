# Cal_tracker
### This Calorie Tracker is an interactive web application built using Streamlit that allows users to log, monitor, and visualize their daily calorie intake. It offers a user-friendly interface where meals can be entered along with their calorie values, and logs are automatically stored for future reference and trend analysis.###

import streamlit as st
from datetime import datetime, timedelta
import pandas as pd
import os

# Page config
st.set_page_config(page_title="Calorie Tracker", layout="centered")

# Dark mode toggle
dark_mode = st.sidebar.checkbox("🌙 Enable Dark Mode", value=False)

# CSS styling
if dark_mode:
    bg_color = "#1e1e1e"
    text_color = "#ffffff"
    box_color = "#2e2e2e"
else:
    bg_color = "linear-gradient(to bottom right, #fefcea, #f1daff)"
    text_color = "#000000"
    box_color = "#ffffff"

st.markdown(f"""
    <style>
        .stApp {{
            background: {bg_color};
            color: {text_color};
            font-family: 'Segoe UI', sans-serif;
        }}
        h1, h2, h3, h4, h5 {{
            color: {text_color};
            text-shadow: 1px 1px 2px rgba(0,0,0,0.1);
        }}
        .stButton>button {{
            background-color: #6a0dad;
            color: white;
            border-radius: 10px;
        }}
        .stTextInput>div>input, .stNumberInput input {{
            background-color: {box_color};
            color: {text_color};
        }}
    </style>
""", unsafe_allow_html=True)

st.title("🥗 Calorie Tracker")

meals = ["Breakfast", "Lunch", "Dinner", "Snacks"]
log_file = "calorie_log.csv"

# Initialize session state
if "meal_data" not in st.session_state:
    st.session_state.meal_data = {meal: {"items": "", "calories": 0} for meal in meals}

# 🍽 Meal input
st.header("🍽 Enter Meals & Calories")
for meal in meals:
    with st.expander(meal):
        st.session_state.meal_data[meal]["items"] = st.text_area(f"{meal} - Food Items", key=f"{meal}_items")
        st.session_state.meal_data[meal]["calories"] = st.number_input(f"{meal} - Calories", min_value=0,
                                                                       key=f"{meal}_calories")

# 📊 Daily summary
st.header("📊 Daily Summary")
total_calories = sum(m["calories"] for m in st.session_state.meal_data.values())
st.success(f"Total Calories Today: {total_calories} kcal")

with st.expander("📌 View What You Ate Today"):
    for meal in meals:
        st.subheader(meal)
        items = st.session_state.meal_data[meal]["items"].strip()
        calories = st.session_state.meal_data[meal]["calories"]
        if items:
            st.markdown(f"**Items:** {items}")
        else:
            st.markdown("_No items logged._")
        st.markdown(f"**Calories:** {calories} kcal")
        st.markdown("---")

# 💾 Export history
if st.button("💾 Save Log to History"):
    today = datetime.now().strftime("%Y-%m-%d")
    row = {
        "Date": today,
        **{f"{meal}_Items": st.session_state.meal_data[meal]["items"] for meal in meals},
        **{f"{meal}_Calories": st.session_state.meal_data[meal]["calories"] for meal in meals},
        "Total_Calories": total_calories
    }
    df = pd.DataFrame([row])
    if os.path.exists(log_file):
        df.to_csv(log_file, mode='a', header=False, index=False)
    else:
        df.to_csv(log_file, index=False)
    st.success("✅ Today's log saved!")

# 📈 Calorie trend chart
if os.path.exists(log_file):
    st.header("📈 Calorie Trend Over Time")
    df_log = pd.read_csv(log_file)
    if "Date" in df_log and "Total_Calories" in df_log:
        df_log["Date"] = pd.to_datetime(df_log["Date"])
        df_log = df_log.sort_values("Date")
        st.line_chart(df_log.set_index("Date")["Total_Calories"])

# 📅 View previous logs
if os.path.exists(log_file):
    st.header("📂 View Past Logs by Date")
    df_log = pd.read_csv(log_file)
    df_log["Date"] = pd.to_datetime(df_log["Date"])
    unique_dates = sorted(df_log["Date"].dt.date.unique(), reverse=True)

    selected_date = st.selectbox("Select a date to view meals:", unique_dates)
    selected_day = df_log[df_log["Date"].dt.date == selected_date]

    if not selected_day.empty:
        for meal in meals:
            st.subheader(f"{meal} on {selected_date}")
            items = selected_day[f"{meal}_Items"].values[0]
            calories = selected_day[f"{meal}_Calories"].values[0]
            st.markdown(f"**Items:** {items if items else '_No items logged._'}")
            st.markdown(f"**Calories:** {calories} kcal")
            st.markdown("---")

# ♻️ Reset data
if st.button("♻️ Reset Today's Data"):
    # Clear session state
    st.session_state.meal_data = {meal: {"items": "", "calories": 0} for meal in meals}

    # Also remove today's entry from the log file if present
    today = datetime.now().strftime("%Y-%m-%d")
    if os.path.exists(log_file):
        df = pd.read_csv(log_file)
        df["Date"] = pd.to_datetime(df["Date"]).dt.strftime("%Y-%m-%d")
        df = df[df["Date"] != today]  # Keep only rows not from today
        df.to_csv(log_file, index=False)

    st.success("✅ Today's data has been cleared from both memory and history.")
