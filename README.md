```python


import os
from dotenv import load_dotenv

from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    MessageHandler,
    ContextTypes,
    filters
)

from autogen import AssistantAgent

# LOAD ENV VARIABLES
load_dotenv()

TOKEN = os.getenv("TELEGRAM_TOKEN")
GROQ_API_KEY = os.getenv("GROQ_API_KEY")

# GROQ LLM CONFIG
llm_config = {
    "config_list": [
        {
            "model": "llama-3.1-8b-instant",
            "api_key": GROQ_API_KEY,
            "base_url": "https://api.groq.com/openai/v1",
            "price": [0, 0]
        }
    ],
    "temperature": 0.7
}

# THINKER AGENT
thinker = AssistantAgent(
    name="Thinker",
    llm_config=llm_config,
    system_message="Understand the user message briefly."
)

# WRITER AGENT
writer = AssistantAgent(
    name="Writer",
    llm_config=llm_config,
    system_message="Reply shortly in max 10 lines."
)

# CHATBOT FUNCTION
async def chatbot(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    # user message
    user_message = update.message.text

    print("👤 USER MESSAGE:")
    print(user_message)

    # thinker agent
    analysis = thinker.generate_reply(
        messages=[
            {
                "role": "user",
                "content": user_message
            }
        ]
    )

    print("\n🧠 THINKER AGENT:")
    print(analysis)

    # writer agent
    reply = writer.generate_reply(
        messages=[
            {
                "role": "user",
                "content": f"""
                User: {user_message}

                Analysis: {analysis}
                """
            }
        ]
    )

    print("\n✍️ WRITER AGENT:")
    print(reply)


    # telegram reply
    await update.message.reply_text(
        str(reply)
    )

# TELEGRAM APP
app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(
    MessageHandler(
        filters.TEXT & ~filters.COMMAND,
        chatbot
    )
)

print("🚀 Bot Running...")

app.run_polling()


```



```python
import streamlit as st
import requests
from groq import Groq

st.set_page_config(page_title="Weather Agent", page_icon="🌤️")

API_KEY = "cf489bc6aa06db0227f38e282c919ce7"

# 🔑 Groq API Key (set in environment for safety)
# export GROQ_API_KEY="your_key"
client = Groq(api_key="")  # <-- replace here or use env


def weather(city):
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={API_KEY}&units=metric"
    r = requests.get(url).json()

    if r.get("cod") != 200:
        return None

    return {
        "temp": r["main"]["temp"],
        "hum": r["main"]["humidity"],
        "cond": r["weather"][0]["description"]
    }


def rating(t, c):
    c = c.lower()
    return (
        "⛈️ Bad" if "rain" in c else
        "🔥 Hot" if t > 35 else
        "❄️ Cold" if t < 10 else
        "🌤️ Perfect" if 20 <= t <= 30 else
        "🌥️ Ok"
    )


# 🧠 GROQ AGENT FUNCTION
def ai_agent(city, data):
    prompt = f"""
You are a smart AI Weather Agent.

City: {city}
Temperature: {data['temp']}°C
Humidity: {data['hum']}%
Condition: {data['cond']}

Do:
1. Give short weather insight
2. Suggest outfit
3. Suggest activity

Keep response short, friendly, and practical.
"""

    try:
        response = client.chat.completions.create(
            model="llama-3.1-8b-instant",
            messages=[
                {"role": "user", "content": prompt}
            ]
        )
        return response.choices[0].message.content

    except Exception as e:
        return f"Groq Error: {str(e)}"


# ---------- UI ----------
st.title("🌍 Weather Agent (Groq Powered)")

city = st.text_input("Enter city")

if st.button("Check Weather") and city:
    data = weather(city)

    if data:
        st.subheader(f"📍 {city.title()}")

        col1, col2, col3 = st.columns(3)
        col1.metric("🌡️ Temp", f"{data['temp']}°C")
        col2.metric("💧 Humidity", f"{data['hum']}%")
        col3.metric("☁️ Condition", data["cond"].title())

        st.success("Weather: " + rating(data["temp"], data["cond"]))

        # 🧠 AI AGENT OUTPUT
        with st.expander("🧠 AI Weather Agent (Groq)"):
            st.write(ai_agent(city, data))

    else:
        st.error("City not found")



```
