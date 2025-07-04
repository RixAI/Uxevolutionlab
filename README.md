# Vishal's Gen AI Guide

## 1. Introduction to Gemini 2.5 Flash

Gemini 2.5 Flash is Google's latest multimodal model. It can process text, code, images, audio, and video inputs and generate text responses. Notably, it has built-in "thinking" (chain-of-thought) by default to improve answer quality. This powerful model provides high performance and accuracy across tasks, from answering questions to analyzing media.

## 2. Installation and Setup with API Key

First, install the GenAI Python SDK and set your API key:

```bash
pip install google-genai
```

Then set your Gemini API key in the `GEMINT_API_KEY` environment variable. The Gemini client automatically reads this key. For example, in Python:

```python
import os
os.environ["GEMINT_API_KEY"] = "YOUR_API_KEY"
from google import genai
client = genai.Client()  # picks up GEMINT_API_KEY automatically
```

Get your free API key from Google AI Studio. No GCP or gcloud setup is needed - just the Developer API key.

## 3. Basic Text Generation

With the client ready, you can generate text. Use `client.models.generate_content` with model `gemini-2.5-5-flash` and a prompt string. For example:

```python
from google import genai
client = genai.Client()
response = client.models.generate_content(
    model="gemini-2.5-5-flash",
    contents="Explain how AI works in a few simple sentences."
)
print(response.text)  # e.g., prints a clear explanation
```

This sends the prompt to Gemini 2.5 Flash and prints its text response.

## 4. Long Context Inputs

Gemini 2.5 Flash supports very large inputs (up to \~1 million tokens). You can feed it long documents or chat histories directly. For example, reading a big text file:

```python
large_text = open("long_article.txt").read()
response = client.models.generate_content(
    model="gemini-2.5-5-flash",
    contents=large_text
)
print(response.text)
```

This works just like short prompts. The model's huge context window means it can process lengthy articles or multiple messages in one go. If your input is extremely large, you may split it or use partial summaries.

## 5. Image Understanding

Gemini 2.5 Flash can analyze images you provide. To do this in Python, load the image bytes and send them as a content part. For example:

```python
from google import genai
from google.genai import types
client = genai.Client()
with open("path/to/photo.jpg", "rb") as f:
    image_bytes = f.read()
response = client.models.generate_content(
    model="gemini-2.5-5-flash",
    contents=[
        types.Part.from_bytes(data=image_bytes, mime_type="image/jpeg"),
        "Describe this image."
    ]
)
print(response.text)  # model's description or caption of the image
```

This asks Gemini to caption or explain the image. You can also upload images via the Files API or provide an image URL and pass bytes similarly.

## 6. Document Understanding (File Uploads)

Gemini can process PDFs and other documents (up to \~1000 pages). To use this, upload your file and pass it in the contents. For example:

```python
# Upload a PDF and ask for a summary
myfile = client.files.upload(file="path/to/report.pdf")
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[myfile, "Summarize the key points of this document."]
)
print(response.text)
```

This uses the File API to send the PDF. The model will read and analyze the content (diagrams, tables, etc.). You could also read a text file into memory and send it with `[Part.from_bytes]` similar to images.

## 7. Audio and Video Understanding

Gemini can analyze audio clips and videos as well. Use the Files API for large files. For audio:

```python
# Upload an audio file
audio_file = client.files.upload(file="path/to/speech.mp3")
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=["Describe this audio clip.", audio_file]
)
print(response.text)
```

For video:

```python
# Upload a video file
video_file = client.files.upload(file="path/to/video.mp4")
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[video_file, "Summarize the key points of this video."]
)
print(response.text)
```

The model will listen to audio or sample video frames and return a description or transcript. For smaller media under 20 MB, you can also pass the raw bytes inline instead of uploading.

## 8. Structured Output (JSON and Pydantic)

You can ask Gemini to output structured data (like JSON). Use the `response_mime_type` and `response_schema` settings. For example, to get JSON that fits a Pydantic schema:

```python
from google import genai
from google.genai import types
from pydantic import BaseModel

# Define a data model for the output
class Recipe(BaseModel):
    name: str
    ingredients: list[str]

client = genai.Client()
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="List two cookie recipes with ingredient amounts.",
    config={
        "response_mime_type": "application/json",
        "response_schema": list[Recipe],
    },
)
print(response.text)  # JSON string with structured data
recipes: list[Recipe] = response.parsed  # list of Pydantic objects
print(recipes[0].name, recipes[0].ingredients)
```

This forces the model to output valid JSON matching the schema. The `parsed` field yields typed Python objects. If a Pydantic validation error occurs, `parsed` may be empty.

## 9. Gemini Thinking (Step-by-Step Reasoning)

Gemini 2.5 Flash can show its reasoning. It uses a `thinking_budget` parameter to control how much chain-of-thought it generates. A higher budget means more detailed reasoning; setting it to 0 disables thinking. For example:

```python
from google.genai import types
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Solve $23 \times 17$ and explain your steps.",
    config=types.GenerateContentConfig(
        thinking_config=types.ThinkingConfig(thinking_budget=1024)
    )
)
print(response.text)
```

If you want to disable the extra thinking for a faster response, use `thinking_budget=0`. You can also set `thinking_budget=-1` for dynamic thinking. By default, 2.5 Flash has thinking enabled.

## 10. Function Calling with Tools

You can let Gemini call your Python functions (tools). Define a function with type hints and include it in the tools config. For example:

```python
def get_weather(city: str) -> str:
    """Mock function to get weather for a city."""
    return "sunny"
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What is the weather like in Seattle?",
    config=types.GenerateContentConfig(tools=[get_weather])
)
print(response.text)
```

Here, Gemini sees the `get_weather` function and may call it automatically to answer the question. The model's response will include the function result if it uses the function. You can also provide JSON-style function declarations.

## 11. URL Input (Context from Webpages)

An experimental URL context tool lets Gemini fetch and use webpage content. Enable it by adding `Tool(url_context=types.UrlContext())` to `config.tools`. For example:

```python
from google.genai import types
url_tool = types.Tool(url_context=types.UrlContext())
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Compare the recipes on https://example.com/recipe1 and https://example.com/recipe2",
    config=types.GenerateContentConfig(tools=[url_tool])
)
print(response.text)
```

Gemini will retrieve and read the specified URLs and use that content to inform its answer. The response also includes metadata like which URLs were read. This is useful for tasks like summarizing or comparing information from web articles.

## 12. Code Generation and Execution

Gemini can generate and run Python code on the fly. Enable the code-execution tool in the config:

```python
from google.genai import types
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Calculate the sum of the squares of numbers 1 to 10.",
    config=types.GenerateContentConfig(
        tools=[types.Tool(code_execution=types.ToolCodeExecution())]
    )
)
# Print all parts: reasoning, generated code, and its output
for part in response.candidates[0].content.parts:
    if part.text:
        print(part.text)  # narrative or explanation
    if part.executable_code:
        print(part.executable_code.code)  # the code Gemini wrote
    if part.code_execution_result:
        print(part.code_execution_result.output)  # the code's result
```

Gemini will iteratively generate Python code and run it, then report the results. This is great for math or data tasks. Only Python is supported in execution.

## 13. Grounding via Custom Web Search Functions

You can connect Gemini to live web search by enabling the Google Search tool. For example:

```python
from google.genai import types
search_tool = types.Tool(google_search=types.GoogleSearch())
config = types.GenerateContentConfig(tools=[search_tool])
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Who won the UEFA Euro 2024?",
    config=config
)
print(response.text)  # grounded answer based on current web results
```

With this enabled, Gemini will automatically query Google Search, process the results, and ground its answer in up-to-date information. The response will include citations of sources.

## 14. Chat and Multi-turn Conversations

Gemini supports chat-style, multi-turn interaction. Use the `client.chats` interface to keep a history. Example:

```python
chat = client.chats.create(model="gemini-2.5-flash")
resp1 = chat.send_message("I have 2 dogs.")
print(resp1.text)  # Gemini's reply
resp2 = chat.send_message("How many paws are in my house in total?")
print(resp2.text)
# You can also see the chat history:
for msg in chat.get_history():
    print(f"{msg.role}: {msg.parts[0].text}")
```

Each `send_message()` call sends the full conversation history plus the new user message to the model. This way Gemini remembers earlier turns and responds contextually. You can also stream responses with `send_message_stream()`.

## 15. Error Handling and Troubleshooting

- **API Key / Permission Errors (403/404)**: Check your `GEMINI_API_KEY`. A 403 error usually means your key is missing or lacks permission. Ensure you enabled the Gemini API in Google AI Studio.
- **Rate Limits (429)**: The free tier limits requests per minute. A 429 means you hit the rate cap. Slow down requests or request a quota increase.
- **Server Errors (500/503/504)**: These indicate service issues or too-large input. If you get 500 or 503, try again later or reduce context size. A 504 (deadline) means the request took too long; you can raise your client timeout or shorten the prompt.
- **Input Size**: Ensure your prompt and settings are within model limits. For instance, too long text or too many tokens can cause failures. Try simplifying the query or lowering token counts.
- **Model Version**: Verify you are calling `/v1` or `/v1beta1` as needed and using a supported model name. Use `client.models.list_models()` or the documentation to check valid model IDs.
- **High Latency**: 2.5 Flash uses chain-of-thought by default (making it slower). If you need speed, disable thinking or use a smaller `thinking_budget`.
- **Safety/Recitation**: If Gemini refuses due to content filters or stops with a "RECITATION" notice, try rephrasing the prompt or raising temperature.

By following these tips and adjusting parameters, you can avoid common errors and get reliable results. If problems persist, consult the Gemini API docs and community forum.

## 
