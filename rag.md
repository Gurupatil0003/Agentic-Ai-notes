```python
from flask import Flask, render_template, request, jsonify
import pandas as pd
import google.generativeai as genai
from dotenv import load_dotenv
import os

# ----------------------------
# Load Environment Variables
# ----------------------------
load_dotenv()

# ----------------------------
# Flask App
# ----------------------------
app = Flask(__name__)

# ----------------------------
# Configure Gemini
# ----------------------------
genai.configure(api_key=os.getenv("GOOGLE_API_KEY"))

model = genai.GenerativeModel("gemini-2.5-flash")

# ----------------------------
# Load CSV
# ----------------------------
df = pd.read_csv("qa_data (1) (1).csv")

# ----------------------------
# Convert CSV to Context
# ----------------------------
context_text = ""

for _, row in df.iterrows():

    context_text += f"""
Q: {row['question']}
A: {row['answer']}

"""

# ----------------------------
# Ask Gemini Function
# ----------------------------
def ask_gemini(query):

    prompt = f"""
You are a Q&A assistant.

Answer ONLY using the context below.

If the answer is not present, say:
No relevant Q&A found.

Context:
{context_text}

Question:
{query}
"""

    response = model.generate_content(prompt)

    return response.text.strip()

# ----------------------------
# Routes
# ----------------------------
@app.route("/")
def home():

    return render_template("index.html")

@app.route("/ask", methods=["POST"])
def ask():

    data = request.get_json()

    user_query = data["query"]

    answer = ask_gemini(user_query)

    return jsonify({
        "answer": answer
    })

# ----------------------------
# Run Flask App
# ----------------------------
if __name__ == "__main__":

    app.run(debug=True)


```

```python
<!DOCTYPE html>
<html lang="en">
<head>

    <meta charset="UTF-8">

    <meta name="viewport"
          content="width=device-width, initial-scale=1.0">

    <title>AI Voice Assistant</title>

    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap"
          rel="stylesheet">

    <style>

        *{
            margin:0;
            padding:0;
            box-sizing:border-box;
        }

        body{

            font-family:'Poppins', sans-serif;

            background:
            linear-gradient(
                135deg,
                #020617,
                #0f172a,
                #111827
            );

            min-height:100vh;

            display:flex;

            justify-content:center;

            align-items:center;

            overflow:hidden;

            color:white;
        }

        .bg-blur1{

            position:absolute;

            width:350px;

            height:350px;

            background:#2563eb;

            filter:blur(120px);

            opacity:0.3;

            top:-100px;

            left:-100px;
        }

        .bg-blur2{

            position:absolute;

            width:300px;

            height:300px;

            background:#06b6d4;

            filter:blur(120px);

            opacity:0.25;

            bottom:-80px;

            right:-80px;
        }

        .container{

            position:relative;

            width:90%;

            max-width:800px;

            background:rgba(255,255,255,0.07);

            backdrop-filter:blur(25px);

            border:1px solid rgba(255,255,255,0.08);

            border-radius:35px;

            padding:45px;

            box-shadow:
            0 10px 50px rgba(0,0,0,0.5);

            z-index:10;

            animation:fadeIn 1s ease;
        }

        @keyframes fadeIn{

            from{
                opacity:0;
                transform:translateY(20px);
            }

            to{
                opacity:1;
                transform:translateY(0);
            }
        }

        .top-bar{

            display:flex;

            justify-content:space-between;

            align-items:center;

            margin-bottom:40px;
        }

        .logo{

            font-size:28px;

            font-weight:700;

            background:
            linear-gradient(
                90deg,
                #38bdf8,
                #818cf8
            );

            -webkit-background-clip:text;

            -webkit-text-fill-color:transparent;
        }

        .badge{

            padding:8px 16px;

            border-radius:30px;

            background:rgba(255,255,255,0.08);

            font-size:13px;

            color:#cbd5e1;
        }

        .hero{

            text-align:center;

            margin-bottom:40px;
        }

        .hero h1{

            font-size:3rem;

            font-weight:700;

            margin-bottom:15px;
        }

        .hero p{

            color:#cbd5e1;

            font-size:16px;

            line-height:1.8;
        }

        .mic-wrapper{

            display:flex;

            justify-content:center;

            margin-bottom:35px;
        }

        .mic-btn{

            width:130px;

            height:130px;

            border-radius:50%;

            border:none;

            cursor:pointer;

            background:
            linear-gradient(
                135deg,
                #06b6d4,
                #2563eb
            );

            color:white;

            font-size:50px;

            transition:0.4s;

            box-shadow:
            0 0 40px rgba(37,99,235,0.6);

            position:relative;
        }

        .mic-btn:hover{

            transform:scale(1.08);
        }

        .mic-btn.listening{

            animation:pulse 1.2s infinite;
        }

        @keyframes pulse{

            0%{
                transform:scale(1);
                box-shadow:
                0 0 20px rgba(37,99,235,0.5);
            }

            50%{
                transform:scale(1.12);
                box-shadow:
                0 0 60px rgba(37,99,235,1);
            }

            100%{
                transform:scale(1);
                box-shadow:
                0 0 20px rgba(37,99,235,0.5);
            }
        }

        .status{

            text-align:center;

            margin-bottom:25px;

            color:#94a3b8;

            font-size:15px;
        }

        .chat-box{

            display:flex;

            flex-direction:column;

            gap:25px;
        }

        .box{

            background:rgba(255,255,255,0.05);

            border:1px solid rgba(255,255,255,0.08);

            border-radius:22px;

            padding:22px;

            transition:0.3s;
        }

        .box:hover{

            background:rgba(255,255,255,0.08);
        }

        .box-title{

            margin-bottom:15px;

            font-weight:600;

            color:#38bdf8;

            font-size:18px;
        }

        textarea{

            width:100%;

            min-height:120px;

            background:transparent;

            border:none;

            outline:none;

            color:white;

            resize:none;

            font-size:16px;

            line-height:1.8;
        }

        textarea::placeholder{

            color:#64748b;
        }

        #answer{

            line-height:1.9;

            color:#f1f5f9;
        }

        .footer{

            margin-top:35px;

            text-align:center;

            color:#64748b;

            font-size:13px;
        }

        @media(max-width:768px){

            .container{
                padding:30px;
            }

            .hero h1{
                font-size:2rem;
            }

            .mic-btn{
                width:110px;
                height:110px;
                font-size:40px;
            }
        }

    </style>

</head>

<body>

<div class="bg-blur1"></div>
<div class="bg-blur2"></div>

<div class="container">

    <div class="top-bar">

        <div class="logo">
            AI Voice Assistant
        </div>

        <div class="badge">
            Gemini + RAG
        </div>

    </div>

    <div class="hero">

        <h1>Talk With AI</h1>

        <p>
            Click the microphone and start speaking naturally.
            Your voice will be captured automatically.
        </p>

    </div>

    <div class="mic-wrapper">

        <button class="mic-btn"
                id="micBtn">

            🎤

        </button>

    </div>

    <div class="status"
         id="status">

        Waiting for your voice...

    </div>

    <div class="chat-box">

        <div class="box">

            <div class="box-title">
                 Your Speech
            </div>

            <textarea id="query"
                      placeholder="Your speech will appear here..."></textarea>

        </div>

        <div class="box">

            <div class="box-title">
                 AI Response
            </div>

            <div id="answer">

                AI response will appear here...

            </div>

        </div>

    </div>

    <div class="footer">

        Built with Flask + Gemini + Voice AI

    </div>

</div>

<script>

const micBtn =
    document.getElementById("micBtn");

const statusText =
    document.getElementById("status");

const queryBox =
    document.getElementById("query");

const answerBox =
    document.getElementById("answer");


// ------------------------------------
// Speech Recognition
// ------------------------------------
const SpeechRecognition =
    window.SpeechRecognition ||
    window.webkitSpeechRecognition;

const recognition =
    new SpeechRecognition();

recognition.lang = "en-US";

recognition.continuous = false;

recognition.interimResults = true;


// ------------------------------------
// Start Listening
// ------------------------------------
micBtn.onclick = () => {

    recognition.start();

};


// ------------------------------------
// Listening Started
// ------------------------------------
recognition.onstart = () => {

    micBtn.classList.add("listening");

    statusText.innerHTML =
        "🎤 Listening...";

};


// ------------------------------------
// Live Speech Capture
// ------------------------------------
recognition.onresult = (event) => {

    let transcript = "";

    for(let i = 0;
        i < event.results.length;
        i++){

        transcript +=
            event.results[i][0].transcript;
    }

    queryBox.value = transcript;
};


// ------------------------------------
// Speech Ended
// ------------------------------------
recognition.onend = async () => {

    micBtn.classList.remove("listening");

    statusText.innerHTML =
        " Processing...";

    const query =
        queryBox.value;

    if(query.trim() === ""){

        statusText.innerHTML =
            " No speech detected";

        return;
    }

    const response =
        await fetch("/ask", {

        method:"POST",

        headers:{
            "Content-Type":"application/json"
        },

        body:JSON.stringify({
            query:query
        })

    });

    const data =
        await response.json();

    answerBox.innerHTML =
        data.answer;

    statusText.innerHTML =
        " Response Ready";


    // ------------------------------------
    // Voice Reply
    // ------------------------------------
    const speech =
        new SpeechSynthesisUtterance(
            data.answer
        );

    speech.lang = "en-US";

    speech.rate = 1;

    window.speechSynthesis.speak(speech);

};


// ------------------------------------
// Errors
// ------------------------------------
recognition.onerror = (event) => {

    micBtn.classList.remove("listening");

    statusText.innerHTML =
        " Error: " + event.error;

};

</script>

</body>
</html>



```

```python
# Install dependencies
!pip install groq gradio PyPDF2 --quiet

import os
import gradio as gr
from PyPDF2 import PdfReader
from groq import Groq

# Set Groq API Key
os.environ["GROQ_API_KEY"] = ""
client = Groq(api_key=os.environ["GROQ_API_KEY"])

# Load manual text
def load_manual(path):
    if path.endswith(".pdf"):
        return "\n".join(
            page.extract_text() or "" for page in PdfReader(path).pages
        )
    else:
        return open(path, "r", encoding="utf-8").read()

manual_text = load_manual("/content/Complete_Business_Manual_NovaTech_Solutions (1).pdf")

# Chat function
def chat(question):
    response = client.chat.completions.create(
        model="openai/gpt-oss-120b",
        messages=[
            {"role": "system", "content": "Answer only from the given manual."},
            {"role": "user", "content": f"{manual_text}\n\nQ: {question}"}
        ],
        max_tokens=512
    )
    return response.choices[0].message.content

# Gradio UI
gr.Interface(
    fn=chat,
    inputs=gr.Textbox(
        lines=2,
        placeholder="Ask your question here...",
        label="Ask Question"
    ),
    outputs=gr.Textbox(
        lines=15,   # BIG OUTPUT BOX
        label="Answer"
    ),
    title=" Manual AI Chatbot",
    description="Ask anything from your uploaded manual",
    theme=gr.themes.Soft()
).launch()

```
