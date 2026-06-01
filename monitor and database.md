```python
import os
from datetime import datetime, UTC

from dotenv import load_dotenv
from flask import Flask, render_template, request, jsonify

from pymongo import MongoClient
from langchain_google_genai import ChatGoogleGenerativeAI
from langsmith import traceable

load_dotenv()

app = Flask(__name__)

# MongoDB
mongo_client = MongoClient(
    os.getenv("MONGODB_URI")
)

db = mongo_client["ai_agent"]
chat_collection = db["chat_history"]

# Gemini
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    temperature=0.7
)


@traceable(name="SaveChat")
def save_chat(question, answer):

    chat_collection.insert_one(
        {
            "question": question,
            "answer": answer,
            "created_at": datetime.now(UTC)
        }
    )


@traceable(name="GeminiMongoAgent")
def ask_agent(question):

    response = llm.invoke(question)

    answer = response.content

    save_chat(question, answer)

    return answer


@app.route("/")
def home():

    return render_template("index.html")


@app.route("/chat", methods=["POST"])
def chat():

    user_message = request.json["message"]

    answer = ask_agent(user_message)

    return jsonify(
        {
            "response": answer
        }
    )


if __name__ == "__main__":
    app.run(debug=True)



```

```python

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>AI Agent Dashboard</title>

<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">

<style>

:root{
    --bg:#0b1120;
    --card:#111827;
    --border:#293548;
    --accent:#4f46e5;
    --accent2:#06b6d4;
    --text:#f8fafc;
    --muted:#94a3b8;
}

*{
    margin:0;
    padding:0;
    box-sizing:border-box;
    font-family:'Inter',sans-serif;
}

body{
    height:100vh;
    overflow:hidden;
    background:
    radial-gradient(circle at top left,#312e81,transparent 35%),
    radial-gradient(circle at bottom right,#0891b2,transparent 35%),
    var(--bg);
    color:white;
}

.wrapper{
    display:flex;
    height:100vh;
}

/* SIDEBAR */

.sidebar{
    width:260px;
    background:rgba(17,24,39,.75);
    backdrop-filter:blur(20px);
    border-right:1px solid rgba(255,255,255,.08);
    padding:25px;
}

.logo{
    font-size:26px;
    font-weight:700;
    margin-bottom:40px;
}

.logo span{
    background:linear-gradient(90deg,#60a5fa,#22d3ee);
    -webkit-background-clip:text;
    -webkit-text-fill-color:transparent;
}

.card{
    background:rgba(255,255,255,.05);
    border:1px solid rgba(255,255,255,.08);
    border-radius:18px;
    padding:18px;
    margin-bottom:15px;
}

.card h3{
    margin-bottom:8px;
}

.card p{
    color:var(--muted);
    font-size:14px;
}

/* MAIN */

.main{
    flex:1;
    display:flex;
    flex-direction:column;
}

.header{
    padding:25px;
    border-bottom:1px solid rgba(255,255,255,.08);
    backdrop-filter:blur(20px);
}

.header h1{
    font-size:30px;
}

.header p{
    color:var(--muted);
    margin-top:5px;
}

.chat-container{
    flex:1;
    overflow-y:auto;
    padding:30px;
}

.message-row{
    display:flex;
    margin-bottom:25px;
    animation:fade .3s ease;
}

.user-row{
    justify-content:flex-end;
}

.bot-row{
    justify-content:flex-start;
}

.message{
    max-width:70%;
    padding:18px;
    border-radius:20px;
    line-height:1.7;
    white-space:pre-wrap;
}

.user{
    background:linear-gradient(
        135deg,
        #4f46e5,
        #06b6d4
    );
}

.bot{
    background:rgba(255,255,255,.06);
    border:1px solid rgba(255,255,255,.08);
}

.input-section{
    padding:25px;
    border-top:1px solid rgba(255,255,255,.08);
}

.input-box{
    display:flex;
    gap:10px;
}

input{
    flex:1;
    padding:18px;
    border:none;
    border-radius:18px;
    background:rgba(255,255,255,.08);
    color:white;
    outline:none;
    font-size:16px;
}

button{
    border:none;
    border-radius:18px;
    padding:0 30px;
    cursor:pointer;
    color:white;
    font-weight:600;
    background:linear-gradient(
        135deg,
        #4f46e5,
        #06b6d4
    );
}

.typing{
    display:none;
    color:#60a5fa;
    padding:10px;
}

@keyframes fade{
    from{
        opacity:0;
        transform:translateY(10px);
    }
    to{
        opacity:1;
        transform:translateY(0);
    }
}

</style>
</head>

<body>

<div class="wrapper">

    <div class="sidebar">

        <div class="logo">
            <span>AI Agent</span>
        </div>

        <div class="card">
            <h3>Model</h3>
            <p>Gemini 2.5 Flash</p>
        </div>

        <div class="card">
            <h3>Database</h3>
            <p>MongoDB Atlas</p>
        </div>

        <div class="card">
            <h3>Monitoring</h3>
            <p>LangSmith Tracing</p>
        </div>

    </div>

    <div class="main">

        <div class="header">
            <h1>🤖 AI Agent Dashboard</h1>
            <p>Gemini + MongoDB + LangSmith</p>
        </div>

        <div
            class="chat-container"
            id="chatBox">

            <div class="message-row bot-row">
                <div class="message bot">
                    Hello 👋
                    <br><br>
                    I'm your AI Agent.
                    Ask me anything.
                </div>
            </div>

        </div>

        <div
            class="typing"
            id="typing">
            Gemini is thinking...
        </div>

        <div class="input-section">

            <div class="input-box">

                <input
                    id="message"
                    placeholder="Ask anything..."
                >

                <button onclick="sendMessage()">
                    Send
                </button>

            </div>

        </div>

    </div>

</div>

<script>

async function sendMessage(){

    let input =
    document.getElementById("message");

    let text =
    input.value.trim();

    if(!text) return;

    let chat =
    document.getElementById("chatBox");

    chat.innerHTML += `
    <div class="message-row user-row">
        <div class="message user">
            ${text}
        </div>
    </div>
    `;

    input.value="";

    document.getElementById("typing")
    .style.display="block";

    const response =
    await fetch("/chat",{
        method:"POST",
        headers:{
            "Content-Type":"application/json"
        },
        body:JSON.stringify({
            message:text
        })
    });

    const data =
    await response.json();

    document.getElementById("typing")
    .style.display="none";

    chat.innerHTML += `
    <div class="message-row bot-row">
        <div class="message bot">
            ${data.response}
        </div>
    </div>
    `;

    chat.scrollTop =
    chat.scrollHeight;
}

document
.getElementById("message")
.addEventListener("keypress",
function(e){

    if(e.key==="Enter"){
        sendMessage();
    }

});

</script>

</body>
</html>


```


```python
GOOGLE_API_KEY=


MONGODB_URI=mongodb+srv://gurupatil327_db_user:UhT312XizrKJnZEu@cluster0.phfkokq.mongodb.net/?appName=Cluster0

LANGSMITH_API_KEY=
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=Gemini-Agent




```

```python
flask
gunicorn
pymongo
python-dotenv
langchain
langchain-google-genai
langsmith

```

```python
web: gunicorn app:app

```


```python

import os
from datetime import datetime, UTC

from dotenv import load_dotenv
load_dotenv()

# -----------------------------
# LangSmith Debug
# -----------------------------
print("LANGSMITH_API_KEY:", bool(os.getenv("LANGSMITH_API_KEY")))
print("LANGSMITH_TRACING:", os.getenv("LANGSMITH_TRACING"))
print("LANGCHAIN_TRACING_V2:", os.getenv("LANGCHAIN_TRACING_V2"))
print("LANGSMITH_PROJECT:", os.getenv("LANGSMITH_PROJECT"))

# -----------------------------
# Imports
# -----------------------------
from pymongo import MongoClient
from langsmith import traceable
from langchain_google_genai import ChatGoogleGenerativeAI

# -----------------------------
# MongoDB
# -----------------------------
mongo_client = MongoClient(
    os.getenv("MONGODB_URI")
)

mongo_client.admin.command("ping")

db = mongo_client["ai_agent"]
chat_collection = db["chat_history"]

print("✅ MongoDB Connected")

# -----------------------------
# Gemini
# -----------------------------
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    temperature=0.7
)

print("✅ Gemini Connected")


# -----------------------------
# Mongo Save Trace
# -----------------------------
@traceable(name="MongoDB_Save")
def save_chat(question, answer):

    chat_collection.insert_one(
        {
            "question": question,
            "answer": answer,
            "created_at": datetime.now(UTC)
        }
    )


# -----------------------------
# Agent Trace
# -----------------------------
@traceable(
    run_type="chain",
    name="GeminiMongoAgent"
)
def ask_agent(question):

    print("Tracing Question:", question)

    response = llm.invoke(question)

    answer = response.content

    save_chat(
        question,
        answer
    )

    return answer


# -----------------------------
# Chat Loop
# -----------------------------
print("\n🤖 Gemini Agent Started")
print("Type 'exit' to quit\n")

while True:

    query = input("You: ")

    if query.lower() == "exit":
        break

    try:

        answer = ask_agent(query)

        print("\nAgent:")
        print(answer)

    except Exception as e:

        print("\nError:", e)

```
