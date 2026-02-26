# RunPod HTTP Server (Remote Dev Access)

How to spin up a simple HTTP server on a RunPod pod so you can verify the port is reachable from outside — useful for confirming remote coding/dev access is working.

## File Location

```
/root/.openclaw/workspace/server.py
```

## The Server

```python
from flask import Flask, jsonify
import datetime

app = Flask(__name__)

@app.route('/')
def index():
    return '<h1>Echo is here ✈️</h1><p>Flask running on RunPod port 8000</p>'

@app.route('/ping')
def ping():
    return jsonify({
        'status': 'ok',
        'time': datetime.datetime.utcnow().isoformat() + 'Z',
        'message': 'Echo is alive ✈️'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=False)
```

## Starting It

```bash
# Background (non-blocking)
cd /root/.openclaw/workspace && python3 server.py &

# Verify it's up locally
curl http://localhost:8000/ping
```

## Stopping It

```bash
pkill -f "python3.*server.py"
```

## Public URL

RunPod exposes HTTP ports via their proxy. Format:

```
https://{POD_ID}-{PORT}.proxy.runpod.net
```

Get your pod ID:
```bash
echo $RUNPOD_POD_ID
```

Example:
```
https://jbw5hlylhocnkx-8000.proxy.runpod.net
https://jbw5hlylhocnkx-8000.proxy.runpod.net/ping
```

The pod ID changes each time you create a new pod.

## Notes

- Port `8000` must be added as an **HTTP Port** in the RunPod pod config (not TCP) for the proxy URL to work
- Server dies on pod/container restart — add a boot hook to auto-start if needed
- Flask is already installed in this environment (`pip install flask --ignore-installed`)
- Pod info available via env vars: `RUNPOD_POD_ID`, `RUNPOD_PUBLIC_IP`, `RUNPOD_TCP_PORT_*`
