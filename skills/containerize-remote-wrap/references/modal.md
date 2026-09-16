# Modal: auth tokens

Modal endpoints can be secured with custom authentication headers.
A common pattern uses two env vars for a key pair:

```
MODAL_TOKEN_ID=ak-...
MODAL_TOKEN_SECRET=as-...
```

The driver reads these from the environment and attaches them as
headers on each request. At registration, pass them with `--env`:

```sh
manifold policy init <slug> \
    --image <image>:<tag> \
    --minimum-gpu-memory-gb 0 \
    --env MY_SERVER_URL=<endpoint-url> \
    --env MODAL_TOKEN_ID=ak-... \
    --env MODAL_TOKEN_SECRET=as-...
```

The platform stores `config.env` in plaintext. If the user would
rather not store tokens this way, they can restrict the endpoint by
network (IP allowlist to the runner's egress, or a private network)
and drop the token vars.
