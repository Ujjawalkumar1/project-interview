# My Role and Responsibilities: Backend Development (Detailed Code Walkthrough)

For this major project, my primary role was **Backend Development**. I designed and implemented the **FastAPI backend** that connects the frontend interface to the database, our custom AI microservices (Whisper transcription and SRS generation), and handles document exports.

Below is a detailed breakdown of the exact code I wrote, including the libraries used and step-by-step explanations of the workflow. You can use this to explain your work confidently to the teachers.

---

## 🛠️ Key Libraries Used in the Backend
Before diving into the code, here are the main Python libraries I used and why:

1. **`FastAPI`**: The core web framework. It’s an incredibly fast, modern framework for building APIs in Python. It automatically validates incoming data.
2. **`APIRouter`**: A feature of FastAPI that I used to organize different parts of the backend (transcription, srs, export) into separate files, keeping the code clean.
3. **`httpx`**: An asynchronous HTTP client (like `requests`, but modern and async). I used this to make our backend talk to the Whisper and SRS microservices without freezing the server while waiting for the AI.
4. **`pydantic` (`BaseModel`)**: A data validation library that works automatically with FastAPI. It ensures that the data sent by the frontend matches exactly what the backend expects.
5. **`motor` & `bson.objectid.ObjectId`**: Motor is the asynchronous driver for MongoDB. `ObjectId` is the unique 12-byte ID format used by MongoDB to store records.
6. **`markdown` & `xhtml2pdf`**: Used in the export feature. `markdown` converts the AI's markdown text into HTML, and `xhtml2pdf` converts that HTML layout into a properly formatted `.pdf` document.

---

## 1️⃣ The File Upload & Transcription Endpoint
This API endpoint receives an audio file from the frontend, sends it to the Whisper AI service, and saves the text result to the database.

**File Location:** `app/routers/transcribe.py`

```python
import httpx
from fastapi import APIRouter, Depends, File, HTTPException, UploadFile, status
from pydantic import BaseModel
from bson.objectid import ObjectId

# Setting up the router prefix path to /transcribe
router = APIRouter(prefix="/transcribe", tags=["transcribe"])

# The URL of the separate Whisper AI Microservice
WHISPER_SERVICE_URL = "http://127.0.0.1:7860/transcribe/"

@router.post("")
async def transcribe_audio(
    file: UploadFile = File(...),              # 1. Accepts the audio file
    current_user: dict = Depends(get_current_user), # 2. Verifies the user is logged in
    db = Depends(get_db)                       # 3. Gets a connection to MongoDB
):
    # Step A: Talk to the Whisper Microservice
    # We use 'async with httpx.AsyncClient' so our server doesn't block while waiting.
    async with httpx.AsyncClient(timeout=60.0) as client:
        # Read the audio file from memory
        file_content = await file.read()
        # Format the file exactly how the Whisper API expects to receive it
        files = {"file": (file.filename, file_content, file.content_type)}
        
        # Send a POST request to the Whisper Microservice
        response = await client.post(WHISPER_SERVICE_URL, files=files)
        
        # Parse the JSON response from Whisper
        whisper_data = response.json()
        transcription_text = whisper_data.get("transcription")
    
    # Step B: Store the Transcription in MongoDB
    # We create a dictionary formatted for MongoDB
    transcription_doc = {
        "user_id": ObjectId(current_user["_id"]), # Link this file to the specific user
        "filename": file.filename,
        "text": transcription_text,               # Store the translated text
        "created_at": datetime.utcnow()           # Timestamp
    }
    
    # Insert the document into the 'transcriptions' collection
    result = await db.transcriptions.insert_one(transcription_doc)
    
    # Step C: Return the result back to the Frontend React app
    return {
        "message": "Transcription successful",
        "transcription_id": str(result.inserted_id),
        "text": transcription_text
    }
```

**How to explain it:** "Teacher, this code acts as a bridge. It receives `UploadFile` from the frontend user and uses asynchronous `httpx` to ping our Whisper AI model. Once the AI finishes transcribing, it creates a NoSQL document using `ObjectId` references to attach the text securely to the `current_user` in our database."

---

## 2️⃣ The SRS Document Generation Endpoint
This API takes the transcription text (or manual text) and asks the second AI model to generate a formal Software Requirements Specification.

**File Location:** `app/routers/srs.py`

```python
import httpx
from fastapi import APIRouter, Depends, HTTPException, status
from bson.objectid import ObjectId

router = APIRouter(prefix="/generate_srs", tags=["srs"])
SRS_SERVICE_URL = "http://127.0.0.1:9000/api/v1/generate/quick" # URL of our SRS Microservice

@router.post("")
async def generate_srs(
    request: SRSRequest,          # Pydantic model for input data validation
    current_user: dict = Depends(get_current_user), 
    db = Depends(get_db)
):
    # Step A: Retrieve input text
    input_text = ""
    # If the user passed an ID of a previous transcription:
    if request.transcription_id:
        trans_oid = ObjectId(request.transcription_id)
        # Search the database for the transcription text
        transcription = await db.transcriptions.find_one({ "_id": trans_oid })
        input_text = transcription.get("text", "")
    
    # Step B: Call the SRS Generation Microservice
    # We formulate a payload containing the user's idea and preferred template (e.g., Agile)
    payload = {
        "user_requirement": input_text,
        "template_name": request.template_name or "agile"
    }

    # Asynchronously await the AI response (large 300s timeout due to LLM generation speeds)
    async with httpx.AsyncClient(timeout=300.0) as client:
        response = await client.post(SRS_SERVICE_URL, json=payload)
        srs_data = response.json()
        generated_srs = srs_data.get("srs_document")
        template_used = srs_data.get("template_used")

    # Step C: Save generated Software Requirements Spec into the database
    srs_doc = {
        "user_id": ObjectId(current_user["_id"]),
        "input_text": input_text,
        "srs_document": generated_srs,
        "template_used": template_used,
        "created_at": datetime.utcnow()
    }
    result = await db.srs_documents.insert_one(srs_doc)

    return {
        "message": "SRS generated successfully",
        "srs_id": str(result.inserted_id),
        "srs_document": generated_srs
    }
```

**How to explain it:** "For generating the SRS, my backend checks if the request contains a valid `transcription_id`. It searches our MongoDB database to pull the raw text. Then it creates a JSON payload to send to our Generative AI pipeline. LLMs (Large Language Models) take a long time to think, so I set an async timeout of 300 seconds to prevent connection drops, and wait for the Markdown formatted SRS to return."

---

## 3️⃣ The PDF Export Endpoint
This endpoint converts the generated markdown (which is how LLMs act naturally) into an actual graphical PDF that the user can download.

**File Location:** `app/routers/export.py`

```python
import tempfile
import os
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from fastapi.responses import FileResponse
from bson.objectid import ObjectId

router = APIRouter(prefix="/export", tags=["export"])

@router.get("/{srs_id}")
async def export_srs(
    srs_id: str,
    background_tasks: BackgroundTasks, # FastAPI feature to run functions after the response is sent
    current_user: dict = Depends(get_current_user),
    db = Depends(get_db)
):
    # Step A: Get exactly which document the user wants
    srs_doc = await db.srs_documents.find_one({
        "_id": ObjectId(srs_id),
        "user_id": ObjectId(current_user["_id"]) # Security check: ensures user owns this document
    })
    srs_text = srs_doc.get("srs_document", "")

    # Step B: Data Conversion Pipeline (Markdown -> HTML -> PDF)
    # mkstemp creates a temporary empty file safely on the operating system
    fd, path = tempfile.mkstemp(suffix=".pdf")
    os.close(fd) 
    
    from markdown import markdown
    # Convert pure Markdown text into raw HTML (with support for markdown tables)
    srs_html = markdown(srs_text, extensions=['tables'])

    # CSS string omitted for brevity (I wrote CSS logic to style tables and fonts)
    style_css = "... CSS properties for a nice document ..."

    html_content = f"<html><head><style>{{style_css}}</style></head><body>{{srs_html}}</body></html>"

    from xhtml2pdf import pisa
    # Write the beautifully styled HTML into a binary PDF format
    with open(path, "w+b") as result_file:
        pisa_status = pisa.CreatePDF(html_content, dest=result_file)

    # Step C: Deliver the file & Cleanup
    # Uses FastAPI BackgroundTasks to delete the temporary PDF file immediately after the user downloads it
    background_tasks.add_task(os.unlink, path)
    
    # Return FileResponse triggers a file download in the user's browser
    return FileResponse(
        path=path, 
        filename=f"srs_{srs_id}.pdf", 
        media_type="application/pdf"
    )
```

**How to explain it:** "The AI returns string data formatted in Markdown. I created an export mechanism that first fetches the document securely checking for ownership. Then, using Python's `markdown` package, I convert that raw text into HTML. I inject custom CSS into the HTML, and utilize `xhtml2pdf` to render a final, cleanly formatted PDF file. Most importantly, I utilized FastAPI's `BackgroundTasks` feature to safely delete (`os.unlink()`) the PDF file off the server memory right after transmitting it to the client, preventing our server memory from filling up over time."

---

## 🌐 How the Frontend and Backend Communicate

Explaining how our React frontend talks to the FastAPI backend is critical. Here is how I set up that connection.

### 1. CORS (Cross-Origin Resource Sharing)
By default, web browsers block web pages from making requests to a different domain or port for security reasons. Because our frontend React app runs on one port (e.g., `localhost:3000`) and our backend runs on another (e.g., `localhost:8000`), the browser would normally block them from talking.

**My Solution:** In `main.py`, I implemented `CORSMiddleware`.
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], # Allows requests from any frontend origin
    allow_credentials=True,
    allow_methods=["*"], # Allows GET, POST, PUT, DELETE
    allow_headers=["*"], # Allows headers like Authorization (for our passwords/tokens)
)
```
**How to explain it:** "I configured the backend with CORS middleware. This tells the backend server to explicitly trust incoming HTTP requests from our React frontend, allowing it to send data (like user passwords and audio files) without the browser blocking it."

### 2. JSON Web Tokens (JWT) for Authentication
When a user logs in on the frontend, how does the backend remember who they are for their next request? I implemented JWT token authentication in `app/routers/auth.py`.

```python
@router.post("/login", response_model=Token)
async def login(credentials: LoginSchema, db=Depends(get_db)):
    # 1. Finds the user email in the MongoDB database
    user = await db.users.find_one({"email": credentials.email.lower()})
    
    # 2. Uses bcrypt to securely verify their hashed password matches
    if not verify_password(credentials.password, user["hashed_password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # 3. Creates a secure, encrypted JWT String containing their User ID
    token = create_access_token({"sub": str(user["_id"])})
    
    # 4. Returns this token to the frontend
    return Token(access_token=token)
```
**How to explain it:** "When users log in, the frontend sends a POST request to my `/login` endpoint. My code verifies the hashed password using `bcrypt`. If it's correct, I generate a JWT (JSON Web Token) using the `python-jose` library. The frontend saves this token and securely attaches it to the 'Headers' of every future request. This is how the backend knows exactly whose audio file it is processing without asking them to log in again."

---

## 🚀 The Brain of the App: `main.py`

This is the central entry point of the entire backend. It doesn't do the heavy lifting itself, but it routes all traffic.

**File Location:** `app/main.py`

```python
from fastapi import FastAPI
from . import mongo # Our custom database connection file
from .routers import auth as auth_router, transcribe, srs, export

# 1. Initialize the App
app = FastAPI(title="GenAI SRS Backend (Mongo Auth)")

# 2. Attach the Routers (Connecting the wires)
app.include_router(auth_router.router)
app.include_router(transcribe.router)
app.include_router(srs.router)
app.include_router(export.router)

# 3. Setup CORS (as explained above)
app.add_middleware(CORSMiddleware, ...)

# 4. Startup & Shutdown Events
@app.on_event("startup")
async def startup_event():
    # As soon as the server turns on, connect to MongoDB automatically
    await mongo.connect_to_mongo()

@app.on_event("shutdown")
async def shutdown_event():
    # When the server turns off, cleanly close the database connection
    await mongo.close_mongo_connection()

# 5. Health Check
@app.get("/health")
def health():
    return {"status": "ok"}
```

**How to explain it:** "`main.py` is the conductor of the orchestra. In this file, I instantiated the core `FastAPI` application. Instead of putting hundreds of lines of code here, I imported the specific routers (auth, transcribe, srs, export) and attached them to the main app tree. I also used unique FastAPI lifecycle decorators (`@app.on_event`) to ensure that the moment our backend turns on, it actively opens the connection to our MongoDB database, and when it shuts down, it safely closes it to prevent memory leaks."
