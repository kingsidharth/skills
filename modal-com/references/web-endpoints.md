# Web Endpoints Reference

## FastAPI Shortcut

```python
from pydantic import BaseModel

class Request(BaseModel):
    text: str

@app.function()
@modal.web_endpoint(method="POST")
def process(request: Request):
    return {"result": request.text.upper()}

# GET with query params
@app.function()
@modal.web_endpoint(method="GET")
def search(q: str, limit: int = 10):
    return {"query": q, "limit": limit}
```

## ASGI Apps (FastAPI, Starlette)

```python
from fastapi import FastAPI

web_app = FastAPI()

@web_app.post("/predict")
def predict(data: dict):
    return {"prediction": ...}

@app.function(allow_concurrent_inputs=100)
@modal.asgi_app()
def fastapi_app():
    return web_app
```

## WSGI Apps (Flask, Django)

```python
from flask import Flask

flask_app = Flask(__name__)

@flask_app.route("/")
def home():
    return "Hello"

@app.function()
@modal.wsgi_app()
def flask_endpoint():
    return flask_app
```

## Streaming Responses

```python
@app.function()
@modal.web_endpoint(method="POST")
def stream_tokens(prompt: str):
    from fastapi.responses import StreamingResponse
    
    def generate():
        for token in model.generate_stream(prompt):
            yield token
    
    return StreamingResponse(generate(), media_type="text/plain")
```

## WebSockets

```python
@app.function(allow_concurrent_inputs=10)
@modal.asgi_app()
def websocket_app():
    from fastapi import FastAPI, WebSocket
    
    app = FastAPI()
    
    @app.websocket("/ws")
    async def websocket_endpoint(websocket: WebSocket):
        await websocket.accept()
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    
    return app
```

## URLs

Deployed endpoints get stable URLs:
`https://yourname--appname-functionname.modal.run`

