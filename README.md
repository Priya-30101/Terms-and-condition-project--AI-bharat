# Terms-and-condition-project--AI-bharat
This project builds an AI and NLP powered system that makes long and complicated Terms &amp; Conditions easy to understand. It automatically finds important clauses, highlights possible risks, and converts legal language into simple summaries, and also provide voice explanations.
# AI Model & Cloud Services

# AI Model
- TinyLlama – Used to analyze Terms & Conditions and generate summaries and risk scores.
- Ollama – Used to run the TinyLlama model locally for fast inference.

# AWS Cloud Services
- Amazon EC2 – Hosting the backend server and running the application.
- Amazon DynamoDB – Storing user data and analysis results.
- Amazon S3 – Storing uploaded Terms & Conditions documents.
- AWS Lambda – Processing uploaded documents.
Backend code :

from fastapi import FastAPI, UploadFile, File
import pdfplumber
import requests

app = FastAPI()

OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL_NAME = "tinyllama"


def analyze_contract(text):

    prompt = f"""
You are a legal contract risk analyzer.

Analyze the Terms and Conditions below.

Return output STRICTLY in this format:

Dangerous Statements:
- list only risky clauses

Risk Score:
- score from 1 to 10

Conclusion:
- 1 short line final verdict

TEXT:
{text}
"""

    payload = {
        "model": MODEL_NAME,
        "prompt": prompt,
        "stream": False
    }

    response = requests.post(OLLAMA_URL, json=payload)

    data = response.json()

    # Safe return to avoid KeyError
    return data.get("response", "No response from model")


@app.post("/analyze-pdf")
async def analyze_pdf(file: UploadFile = File(...)):

    # Save PDF
    with open(file.filename, "wb") as f:
        f.write(await file.read())

    # Extract text
    text = ""
    with pdfplumber.open(file.filename) as pdf:
        for page in pdf.pages:
            page_text = page.extract_text()
            if page_text:
                text += page_text + "\n"

    result = analyze_contract(text)

     return {
        "analysis": result
    }
        "analysis": result
    }
