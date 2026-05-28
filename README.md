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

```python

import streamlit as st
import requests
from groq import Groq

st.set_page_config("Weather Agent", "🌤️")

API_KEY = ""

# 🔑 Groq API Key (set in environment for safety)
# export GROQ_API_KEY="your_key"
client = Groq(api_key="")  # <-- replace here or use env


def weather(city):
    r = requests.get(
        "http://api.openweathermap.org/data/2.5/weather",
        params={"q": city, "appid": API_KEY, "units": "metric"}
    ).json()

    if r.get("cod") != 200:
        return None

    return {
        "temp": r["main"]["temp"],
        "hum": r["main"]["humidity"],
        "cond": r["weather"][0]["description"]
    }


def rating(t, c):
    c = c.lower()
    return "⛈️ Bad" if "rain" in c else "🔥 Hot" if t > 35 else "❄️ Cold" if t < 10 else "🌤️ Perfect" if 20 <= t <= 30 else "🌥️ Ok"


def ai(city, d):
    prompt = f"""City:{city}
Temp:{d['temp']}°C
Humidity:{d['hum']}%
Condition:{d['cond']}

Give:
- Insight
- Outfit
- Activity (short)"""

    res = client.chat.completions.create(
        model="llama-3.1-8b-instant",
        messages=[{"role": "user", "content": prompt}]
    )
    return res.choices[0].message.content


# ---------- UI ----------
st.title("🌍 Weather Agent (Groq Powered)")

city = st.text_input("Enter city")

if st.button("Check Weather") and city:
    data = weather(city)

    if not data:
        st.error("City not found")
    else:
        st.subheader(f"📍 {city.title()}")

        c1, c2, c3 = st.columns(3)
        c1.metric("🌡️ Temp", f"{data['temp']}°C")
        c2.metric("💧 Humidity", f"{data['hum']}%")
        c3.metric("☁️ Condition", data["cond"].title())

        st.success("Weather: " + rating(data["temp"], data["cond"]))

        with st.expander("🧠 AI Weather Agent"):
            st.write(ai(city, data))

```

```python
from flask import Flask, render_template, request
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

app = Flask(__name__)

# Fast Model
model = SentenceTransformer('paraphrase-MiniLM-L3-v2')

def calculate_scores(jd, resume):

    # Semantic Search Score
    docs = [jd, resume]

    embeddings = model.encode(
        docs,
        convert_to_numpy=True
    )

    index = faiss.IndexFlatL2(
        embeddings.shape[1]
    )

    index.add(embeddings)

    query = model.encode(
        [jd],
        convert_to_numpy=True
    )

    D, I = index.search(query, k=2)

    semantic_score = max(0, 100 - D[0][1])

    # Keyword Match Score
    jd_words = set(jd.lower().split())

    resume_words = set(resume.lower().split())

    matched = jd_words.intersection(
        resume_words
    )

    keyword_score = (
        len(matched) / len(jd_words)
    ) * 100

    # Final ATS Score
    ats_score = (
        semantic_score * 0.7 +
        keyword_score * 0.3
    )

    return (
        round(semantic_score, 2),
        round(keyword_score, 2),
        round(ats_score, 2),
        matched
    )


@app.route('/', methods=['GET', 'POST'])
def home():

    semantic_score = None
    keyword_score = None
    ats_score = None
    matched_keywords = []

    if request.method == 'POST':

        jd = request.form['job_desc']

        resume = request.form['resume_text']

        (
            semantic_score,
            keyword_score,
            ats_score,
            matched_keywords

        ) = calculate_scores(jd, resume)

    return render_template(
        'index.html',
        semantic_score=semantic_score,
        keyword_score=keyword_score,
        ats_score=ats_score,
        matched_keywords=matched_keywords
    )


if __name__ == '__main__':
    app.run(debug=True)

```

```python
<!DOCTYPE html>
<html lang="en">

<head>

    <meta charset="UTF-8">

    <meta name="viewport"
          content="width=device-width, initial-scale=1.0">

    <title>AI Resume Matcher</title>

    <link rel="stylesheet"
          href="{{ url_for('static', filename='style.css') }}">

    <link rel="preconnect"
          href="https://fonts.googleapis.com">

    <link rel="preconnect"
          href="https://fonts.gstatic.com"
          crossorigin>

    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap"
          rel="stylesheet">

</head>

<body>

<div class="bg-blur"></div>

<div class="main-container">

    <div class="hero-section">

        <h1>🚀 AI Resume Matcher</h1>

        <p>
            Semantic Search + Keyword Matching + ATS Score
        </p>

    </div>

    <form method="POST">

        <div class="input-grid">

            <div class="input-card">

                <h2>📄 Job Description</h2>

                <textarea
                    name="job_desc"
                    placeholder="Paste Job Description Here..."
                    required></textarea>

            </div>

            <div class="input-card">

                <h2>👨‍💻 Resume</h2>

                <textarea
                    name="resume_text"
                    placeholder="Paste Resume Here..."
                    required></textarea>

            </div>

        </div>

        <button type="submit">
            Analyze Resume
        </button>

    </form>

    {% if ats_score %}

    <div class="result-grid">

        <div class="score-card">

            <h2>{{ semantic_score }}%</h2>

            <p>Semantic Search</p>

        </div>

        <div class="score-card">

            <h2>{{ keyword_score }}%</h2>

            <p>Keyword Match</p>

        </div>

        <div class="score-card final-card">

            <h2>{{ ats_score }}%</h2>

            <p>ATS Score</p>

        </div>

    </div>

    <div class="keywords-box">

        <h2>🔥 Matched Keywords</h2>

        <div class="keyword-list">

            {% for word in matched_keywords %}

                <span>{{ word }}</span>

            {% endfor %}

        </div>

    </div>

    {% endif %}

</div>

</body>
</html>

```

```python
*{
    margin:0;
    padding:0;
    box-sizing:border-box;
    font-family:'Poppins', sans-serif;
}

body{
    background:#020617;
    color:white;
    min-height:100vh;
    overflow-x:hidden;
}

.bg-blur{
    position:fixed;
    width:500px;
    height:500px;
    background:#2563eb;
    filter:blur(180px);
    top:-150px;
    right:-100px;
    opacity:0.4;
    z-index:-1;
}

.main-container{
    width:90%;
    max-width:1400px;
    margin:auto;
    padding:50px 0;
}

.hero-section{
    text-align:center;
    margin-bottom:40px;
}

.hero-section h1{
    font-size:60px;
    font-weight:700;
    background:linear-gradient(to right,#38bdf8,#818cf8);
    -webkit-background-clip:text;
    -webkit-text-fill-color:transparent;
    margin-bottom:15px;
}

.hero-section p{
    color:#cbd5e1;
    font-size:18px;
}

.input-grid{
    display:grid;
    grid-template-columns:1fr 1fr;
    gap:30px;
}

.input-card{
    background:rgba(15,23,42,0.7);
    border:1px solid rgba(255,255,255,0.08);
    border-radius:25px;
    padding:25px;
    backdrop-filter:blur(20px);
    box-shadow:0px 10px 30px rgba(0,0,0,0.3);
}

.input-card h2{
    margin-bottom:20px;
    color:#38bdf8;
}

textarea{
    width:100%;
    height:350px;
    background:#0f172a;
    border:none;
    outline:none;
    border-radius:18px;
    padding:20px;
    color:white;
    font-size:15px;
    resize:none;
}

textarea::placeholder{
    color:#94a3b8;
}

button{
    width:100%;
    margin-top:30px;
    padding:18px;
    border:none;
    border-radius:18px;
    background:linear-gradient(to right,#38bdf8,#818cf8);
    color:white;
    font-size:20px;
    font-weight:600;
    cursor:pointer;
    transition:0.3s;
}

button:hover{
    transform:translateY(-3px);
    box-shadow:0px 10px 20px rgba(56,189,248,0.4);
}

.result-grid{
    display:grid;
    grid-template-columns:1fr 1fr 1fr;
    gap:25px;
    margin-top:50px;
}

.score-card{
    background:rgba(15,23,42,0.8);
    border:1px solid rgba(255,255,255,0.08);
    border-radius:25px;
    padding:35px;
    text-align:center;
    backdrop-filter:blur(20px);
    transition:0.3s;
}

.score-card:hover{
    transform:translateY(-5px);
}

.score-card h2{
    font-size:55px;
    color:#22c55e;
    margin-bottom:10px;
}

.score-card p{
    color:#cbd5e1;
    font-size:18px;
}

.final-card h2{
    color:#f59e0b;
}

.keywords-box{
    margin-top:40px;
    background:rgba(15,23,42,0.8);
    border-radius:25px;
    padding:30px;
    border:1px solid rgba(255,255,255,0.08);
}

.keywords-box h2{
    margin-bottom:20px;
    color:#38bdf8;
}

.keyword-list{
    display:flex;
    flex-wrap:wrap;
    gap:15px;
}

.keyword-list span{
    background:linear-gradient(to right,#38bdf8,#818cf8);
    color:white;
    padding:12px 18px;
    border-radius:50px;
    font-size:14px;
    font-weight:500;
}

@media(max-width:900px){

    .input-grid{
        grid-template-columns:1fr;
    }

    .result-grid{
        grid-template-columns:1fr;
    }

    .hero-section h1{
        font-size:40px;
    }

}


```
