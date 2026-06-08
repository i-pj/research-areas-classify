# Networking & Security

The Research Taxonomy Service is distributed across two primary environments: a Dokploy VPS (running FastAPI and FastEmbed) and a local Apple M2 Mac (running Ollama and Qdrant). Secure, low-latency communication between these environments is critical.

---

## 1. Tailscale Setup

We use **Tailscale** to create a secure, zero-trust virtual private network (tailnet) connecting the Dokploy server and the M2 Mac.

### Installation
1. Install Tailscale on the Dokploy server and authenticate it to your tailnet.
2. Install Tailscale on the Apple M2 Mac and authenticate it to the same tailnet.

### MagicDNS Configuration
Instead of using ephemeral tailnet IPs (e.g., `100.x.x.x`), configure services using **MagicDNS** hostnames. This ensures connectivity survives IP changes.
- **Format:** `http://<machine-hostname>.<tailnet-name>.ts.net:<port>`
- **Example:** `http://m2-mac.tail12345.ts.net:11434`

Set your `.env` variables (`OLLAMA_BASE_URL` and `QDRANT_URL`) using these MagicDNS addresses.

---

## 2. Security Model

### Ollama Authentication
Ollama **does not have built-in authentication**. Therefore, Tailscale *is* the authentication layer.
- **Action:** Bind Ollama to listen on all interfaces or specifically on the Tailscale interface by setting `OLLAMA_HOST="0.0.0.0:11434"` (or `OLLAMA_HOST="100.x.x.x:11434"`).
- **Critical:** Ensure your Dokploy/Mac firewalls do not expose port `11434` to the public internet. It must only be accessible via the `tailscale0` interface.

### Qdrant Defense in Depth
While Tailscale provides network-level security, we implement defense-in-depth for the vector database:
- **Action:** Enable Qdrant API key authentication.
- Set `QDRANT__SERVICE__API_KEY` when starting the Qdrant container on the Mac.
- Set `QDRANT_API_KEY` in the FastAPI `.env` file on Dokploy.

---

## 3. Latency Considerations

### WireGuard Overhead
Tailscale uses WireGuard under the hood. Expect a baseline network latency overhead of **~1-3ms** per request between Dokploy and the Mac.

### FastEmbed Locality
Why do we run FastEmbed (BM25 and ColBERT generation) on Dokploy instead of the Mac?
- Generating a ColBERT multivector for a 100-token document yields an array of `[100, 96]` floats.
- Sending raw text to Dokploy and generating embeddings locally avoids serializing and transmitting large arrays of floats across the tailnet.
- FastEmbed runs highly optimized ONNX runtimes on the CPU, making it perfectly suited for the Dokploy environment.

---

## 4. Troubleshooting

If the FastAPI service cannot reach Ollama or Qdrant:
1. Run `tailscale status` on Dokploy to verify the Mac is listed and online.
2. Ping the MagicDNS hostname from Dokploy: `ping m2-mac.tail12345.ts.net`.
3. Check Mac firewall settings to ensure Tailscale traffic is allowed on ports 11434 and 6333.
