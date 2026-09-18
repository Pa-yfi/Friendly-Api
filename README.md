# Terminal chatbot

A tiny Python chatbot that runs in your terminal. You type a question, it sends the question to Google's Gemini API, and prints the answer back. Type `exit` to quit.

This repo is meant as a beginner-friendly example of what an API is and how you actually call one from code.

---

## What is an API?

API stands for **Application Programming Interface**. It is a way for one program to ask another program to do something.

The usual analogy is a restaurant. You (your program) don't walk into the kitchen and cook. You give your order to a waiter, the waiter takes it to the kitchen, and brings back the food. The API is the waiter: it defines what you're allowed to order, what format the order has to be in, and what you get back.

In practice, a web API works like this:

1. Your program sends a **request** over the internet to a server. The request contains the data you want to send (here, your question) and usually an **API key** that proves who you are.
2. The server does the heavy work. In this case, a huge language model runs on Google's machines, not yours.
3. The server sends back a **response** — usually structured data like JSON — which your program reads and uses.

The important idea: you don't need the model on your computer. You just need permission to ask for it. That is what an API key is — a private password that identifies your account, so the provider knows who is making requests and who to bill or rate-limit.

Some terms you'll see everywhere:

- **Endpoint** — the specific address/function you're calling (here, "generate content").
- **Request / response** — what you send, what comes back.
- **API key / token** — your credential. Treat it like a password.
- **SDK / client library** — code the provider gives you (like `google-genai`) so you don't have to build raw HTTP requests by hand.
- **Rate limit** — how many requests you're allowed in a period of time.

---

## The code

```python
from google import genai

client = genai.Client(api_key="")

while True:
    question = input("You: ")

    if question.lower() == "exit":
        break

    response = client.models.generate_content(
        model="gemini-3.6-flash",
        contents=question,
    )

    print("Gemini:", response.text)
```

Walking through it top to bottom:

`from google import genai` imports Google's GenAI SDK. This is the client library that handles the networking for you — building the HTTP request, attaching your key, sending it, and parsing the JSON that comes back. Without it you'd be writing raw requests yourself.

`client = genai.Client(api_key="")` creates the client object and hands it your API key. The client is the thing that holds your credentials and knows the address of Google's servers, so every call you make through it is already authenticated. Right now the key is an empty string, so the script won't work until you put a real one in (see the setup section below for a safer way to do it).

`while True:` starts an infinite loop. This is what makes it a chat instead of a one-shot script — it keeps asking for input forever until something breaks the loop.

`question = input("You: ")` prints the prompt `You: ` and waits for you to type a line and press Enter. Whatever you typed is stored as a string in `question`.

`if question.lower() == "exit": break` is the exit condition. `.lower()` converts the input to lowercase first, so `EXIT`, `Exit`, and `exit` all work. `break` leaves the `while` loop, which ends the program.

`response = client.models.generate_content(...)` is the actual API call — the only line that touches the internet. `model="gemini-3.6-flash"` picks which model answers, and `contents=question` is the text you're sending. The program pauses here while the request travels to Google, the model generates an answer, and the response comes back.

`print("Gemini:", response.text)` prints the reply. The response object contains more than just text (things like token usage and safety metadata), so `.text` pulls out just the generated answer. Then the loop starts over and asks for your next question.

---

## Setup

```bash
pip install google-genai
```

Get an API key from [Google AI Studio](https://aistudio.google.com/), then run:

```bash
python chatbot.py
```

---

## Two things to fix before you push this

**Don't hardcode your key.** Never commit an API key to GitHub. Bots scrape public repos for leaked keys within minutes. Read it from an environment variable instead:

```python
import os
from google import genai

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
```

Then set it in your shell before running:

```bash
# Linux / macOS
export GEMINI_API_KEY="your-key-here"
```

```powershell
# Windows PowerShell
$env:GEMINI_API_KEY="your-key-here"
```

Also add a `.gitignore` with `.env` in it if you decide to keep the key in a file.

**Check the model name.** Double-check `gemini-3.6-flash` against the current model list in Google's docs — model IDs change, and a wrong one gives you a 404 rather than a useful error.

---

## Known limitation: it has no memory

Each loop sends only your latest question. The model doesn't see anything you said earlier, so follow-ups like "explain that again more simply" won't work — there is no "that" as far as the model is concerned.

To fix it, keep a list of the conversation and send the whole thing each time:

```python
history = []

while True:
    question = input("You: ")
    if question.lower() == "exit":
        break

    history.append({"role": "user", "parts": [{"text": question}]})

    response = client.models.generate_content(
        model="gemini-3.6-flash",
        contents=history,
    )

    history.append({"role": "model", "parts": [{"text": response.text}]})
    print("Gemini:", response.text)
```

Now every request carries the full conversation, so the model has context. The trade-off is that requests get longer (and more expensive) as the chat grows.

---

## Ideas to build on this

- Wrap the API call in `try/except` so a network error doesn't crash the whole script.
- Add a system instruction to give the bot a personality.
- Stream the response so text appears word by word instead of all at once.
- Save the conversation to a text file when you exit.
