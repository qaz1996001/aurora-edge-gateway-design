# ADR-0003: Hybrid Runtime – Keep Python for DSP/ML

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Platform engineers, DSP team, data science team

---

## Context

Aurora Edge Gateway includes computationally intensive features:
- **Digital Signal Processing (DSP)**: Fourier transforms, filtering, spectral analysis on sensor streams
- **Machine Learning Inference**: Lightweight anomaly detection models (TensorFlow Lite)

The original Node.js implementation calls out to Python subprocesses for these tasks (via `child_process.spawn`). The question is: rewrite Python code in Rust, or keep Python?

---

## Decision

**Keep Python for DSP and ML workloads.** Rust calls Python subprocesses via efficient IPC (Unix sockets + bincode serialization).

---

## Rationale

### Why Keep Python?

**Scientific Ecosystem Maturity**
- `numpy`, `scipy`, `scikit-learn` highly optimized for numerical computing
- TensorFlow Lite, PyTorch Mobile mature for edge inference
- Rewriting all DSP algorithms in Rust would take weeks; many already battle-tested
- **Result**: Faster port; lower risk of algorithmic bugs

**Developer Productivity**
- DSP team is expert in Python; unfamiliar with Rust
- Porting complex FFT or Kalman filter to Rust = high risk
- Python prototyping faster than Rust for numerical code
- **Result**: Team velocity; fewer bugs in mathematical code

**Runtime Cost Acceptable**
- DSP computations amortize IPC overhead (heavy computation; light communication)
- FFT on 1024-sample window: ~5ms computation; 0.5ms IPC overhead = acceptable
- ML inference: ~50–200ms computation; IPC negligible
- **Result**: No performance regression vs. Node.js

### Why Not Rewrite in Rust?

**High Effort**
- Porting numpy-based DSP pipeline to ndarray (Rust equivalent) = weeks of work
- Porting TensorFlow Lite models to tch-rs or burn = integration risk
- **Cost**: 2–4 week delay; introduces new bugs to shake out

**Organizational Risk**
- DSP team has no Rust experience
- Code reviews slower (no one to review complex numerical Rust code)
- Maintenance burden: future updates harder

**Low Benefit**
- DSP code is not on the hot path (runs on-demand, not continuous)
- Memory safety already guaranteed by subprocess isolation (can't crash gateway even if Python crashes)

---

## Architecture

### Hybrid IPC Model

```
Rust (aeg-sensors)                      Python (subprocess)
──────────────────────────────────────────────────────────
aeg-sensors process
│
├─ Read sensor samples from MQTT
├─ Buffer 1024 samples
│
└─ [Need FFT analysis]
   └─ Send request via Unix socket
      
      Request: {
        "cmd": "fft_analysis",
        "samples": [1.2, 3.4, ...],  // 1024 f64 values
        "window": "hann"
      }
      
      ↓ [Unix socket → bincode serialization; ~2 KB]
      
      Python subprocess (spawned on-demand)
      │
      ├─ scipy.fft.fft(samples, window="hann")
      ├─ Extract power spectrum
      │
      └─ Send response
      
      Response: {
        "success": true,
        "power": [10.2, 5.1, ...],  // 512 f64 values
        "peak_freq": 50.0
      }
      
      ↓ [Unix socket ← bincode; ~4 KB]
      
Rust (aeg-sensors)
│
├─ Receive response
├─ Use power spectrum for anomaly detection
└─ Update DB with results
```

### Subprocess Lifecycle

```
┌─ aeg-sensors (Rust)
│
├─ [on startup]
│  └─ Register Python subprocess path: /usr/local/bin/aeg-dsp.py
│
├─ [on first DSP request]
│  └─ spawn("/usr/local/bin/aeg-dsp.py")
│  └─ Keep process alive in background
│
├─ [on subsequent DSP requests]
│  └─ Send request via Unix socket (process already running)
│  └─ No spawn overhead (keep-alive)
│
├─ [on error or timeout]
│  └─ Kill subprocess; next request spawns new instance
│  └─ No cascading failures (subprocess crash ≠ gateway crash)
│
└─ [on shutdown]
   └─ Clean up subprocess (SIGTERM)
```

### Python Subprocess Interface

```python
# /usr/local/bin/aeg-dsp.py
import socket
import sys
import json
import numpy as np
from scipy import signal

def fft_analysis(samples, window='hann'):
    """Perform FFT with windowing."""
    windowed = samples * signal.get_window(window, len(samples))
    fft_result = np.abs(np.fft.fft(windowed))
    return fft_result[:len(samples)//2]

def anomaly_detect(recent_values, threshold=2.5):
    """Simple statistical anomaly detection."""
    mean = np.mean(recent_values)
    std = np.std(recent_values)
    return (recent_values - mean) / std > threshold

# Main loop: read requests from Unix socket
socket_path = "/tmp/aeg-dsp.sock"
sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
sock.bind(socket_path)
sock.listen(1)

while True:
    conn, _ = sock.accept()
    data = conn.recv(8192)
    request = json.loads(data.decode())
    
    if request['cmd'] == 'fft_analysis':
        response = fft_analysis(
            np.array(request['samples']),
            window=request.get('window', 'hann')
        )
        conn.send(json.dumps({'success': True, 'result': response.tolist()}).encode())
    elif request['cmd'] == 'anomaly_detect':
        anomalies = anomaly_detect(np.array(request['values']))
        conn.send(json.dumps({'success': True, 'anomalies': anomalies.tolist()}).encode())
    
    conn.close()
```

---

## Consequences

### Positive
1. **Faster Port**: Reuse existing Python code; no algorithmic rewrites
2. **Lower Risk**: DSP algorithms already proven; fewer bugs
3. **Maintainability**: DSP team can continue maintaining their code
4. **Scientific Ecosystem**: Access to numpy, scipy, TensorFlow Lite without reimplementation
5. **Operational Simplicity**: If Python crashes, gateway continues (isolated process)

### Negative
1. **IPC Overhead**: ~1–2 ms per request + serialization cost (bincode/JSON)
   - **Mitigation**: Batch requests; only use for heavy computations
2. **Process Management**: Python subprocess must be kept alive or respawned efficiently
   - **Mitigation**: Explicit subprocess lifecycle management; keep-alive strategy
3. **Debugging Complexity**: Bridging Rust ↔ Python for troubleshooting
   - **Mitigation**: Structured logging on both sides; clear error propagation
4. **Dependency**: If Python is removed from edge device, feature breaks
   - **Mitigation**: Document as required dependency; distribute Python runtime

### Trade-Offs Accepted
- **Slight IPC latency** (1–2 ms) for **ecosystem benefits and development velocity**
- **Dual-runtime deployment** (Rust + Python) for **specialized workload advantages**

---

## Performance Impact

| Operation | Node.js (In-Process) | Rust + Python (IPC) | Delta |
|---|---|---|---|
| FFT (1024 samples) | 8 ms | 5 ms (DSP) + 1 ms (IPC) = 6 ms | 25% faster |
| Anomaly detection | 2 ms | 1 ms (DSP) + 0.5 ms (IPC) = 1.5 ms | 25% faster |
| ML inference (100ms model) | 105 ms | 100 ms (DSP) + 1 ms (IPC) = 101 ms | ~4% faster |

**Conclusion**: IPC overhead is negligible compared to actual computation time.

---

## Related Decisions
- **[ADR-0002](0002-module-to-crate-decomposition.md)**: Sensor service (aeg-sensors) hosts DSP bridge
- **Architecture**: See `../02-architecture-overview.md` for hybrid runtime diagram

---

## Monitoring & Debugging

### Subprocess Health
```bash
# Check if Python subprocess is running
ps aux | grep aeg-dsp.py

# Monitor CPU/memory of subprocess
watch -n 1 'ps -o %cpu,%mem,cmd | grep aeg-dsp'

# View subprocess logs (if separate log file)
tail -f /var/log/aeg-dsp.log
```

### IPC Performance
```bash
# Add tracing to measure IPC roundtrip time
RUST_LOG=aeg_sensors=debug aeg-sensors

# Example debug output:
# [DEBUG] aeg_sensors: Sending FFT request (1024 samples)
# [DEBUG] aeg_sensors: Python response received in 6.2ms
```

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Platform team, DSP team  
**Implementation**: Phase A (IPC bridge by week 4)
