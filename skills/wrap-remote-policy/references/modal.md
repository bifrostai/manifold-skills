# Modal: route hosting

Modal's `@modal.web_endpoint` decorator gives each function its own
hostname. Two functions decorated this way end up on two separate
URLs:

```python
@app.cls(image=image, gpu="A100")
class Policy:
    @modal.web_endpoint(method="GET")
    def config(self): ...

    @modal.web_endpoint(method="POST")
    def infer(self, obs: dict): ...
```

The Manifold driver reads one base URL from the environment and
appends path segments to it. If `/config` and `/infer` are on
different hosts, the driver cannot reach both.

The fix is to serve all routes through one ASGI app:

```python
@app.cls(image=image, gpu="A100")
class Policy:
    @modal.enter()
    def load(self):
        self.model = ...

    @modal.asgi_app()
    def web(self):
        from fastapi import FastAPI
        api = FastAPI()

        @api.get("/config")
        def config():
            return {"checkpoint": "...", "cameras": [...]}

        @api.post("/infer")
        def infer(obs: dict):
            return {"actions": self.model(obs)}

        @api.get("/health")
        def health():
            return {"status": "ok"}

        return api
```

This gives one hostname for all three routes (`/config`, `/infer`,
`/health`).
