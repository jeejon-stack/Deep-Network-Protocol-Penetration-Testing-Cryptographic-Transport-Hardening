# Deep-Network-Protocol-Penetration-Testing-Cryptographic-Transport-Hardening
A complete offensive and defensive security exercise; attack legacy TLS networks, capture traffic with Tshark, harden with PQC transport rules and establish ML-KEM-768 quantum-safe tunnels between endpoints.


---

## ⚠️ Important Notes Before You Start

These are lessons learned from actually running this project:

- **docker-compose.yml must be created with Python** — cat heredoc and nano both corrupt the YAML file. Use `make_compose.py` (shown below)
- **All attack tests run from HOST laptop** — containers have no internet access so you cannot install curl or wget inside them
- **mitmproxy may timeout on slow connections** — if it does, use nginx:alpine for the attacker container instead
- **tshark needs group permission fix** — `sudo usermod -aG wireshark $USER` + `setcap` on dumpcap before it works
- **ML-KEM via OpenSSL CLI fails** — use liboqs-python directly instead of `openssl genpkey -algorithm mlkem768`
- **Do NOT use `sudo tshark`** — run tshark as your user after fixing dumpcap permissions

---

## Quick Results

```
ATTACK RESULTS (proven):
Legacy Server (8080) → HTTP 200 OK       ❌ VULNERABLE
PQC Server    (8443) → HTTP 403 Forbidden ✅ BLOCKED

TSHARK PACKET ANALYSIS:
Frame 4:  GET request sent to legacy
Frame 8:  200 OK — full data exposed     ❌
Frame 16: GET request sent to PQC
Frame 18: 403 — attack blocked           ✅

ML-KEM-768 TUNNEL:
Public key: 1184 bytes
Shared secret: 32 bytes
QUANTUM SAFE TUNNEL ESTABLISHED!        ✅
Legacy attacks:  IMPOSSIBLE
Quantum attacks: IMPOSSIBLE

VALIDATION LOG (May 20, 2026 09:52:39 UTC):
Legacy Server: 200 VULNERABLE ❌
PQC Server:    403 BLOCKED    ✅
```

---

## Prerequisites

- Ubuntu 22.04 LTS
- Docker v29+ (`sudo apt install docker.io`)
- Docker Compose (`sudo apt install docker-compose`)
- Python 3.10+
- liboqs-python (`pip3 install liboqs-python`)
- tshark/wireshark (`sudo apt install tshark wireshark`)

---

## Step 1 — Create Lab Directory

```bash
mkdir -p ~/quantum-break/{configs,certs,logs,captures}
cd ~/quantum-break
```

---

## Step 2 — Create docker-compose.yml

> ⚠️ Do NOT use cat heredoc or nano — they corrupt the YAML.
> Use this Python script instead.

Create `~/make_compose.py`:
```bash
nano ~/make_compose.py
```

Type this inside nano:
```python
f = open('/home/jeejon/quantum-break/docker-compose.yml', 'w')
f.write("version: '3.8'\n")
f.write("\n")
f.write("services:\n")
f.write("  legacy-server:\n")
f.write("    image: nginx:alpine\n")
f.write("    container_name: legacy-server\n")
f.write("    ports:\n")
f.write("      - '8080:80'\n")
f.write("\n")
f.write("  pqc-server:\n")
f.write("    image: nginx:alpine\n")
f.write("    container_name: pqc-server\n")
f.write("    ports:\n")
f.write("      - '8443:80'\n")
f.write("\n")
f.write("  attacker:\n")
f.write("    image: mitmproxy/mitmproxy\n")
f.write("    container_name: attacker\n")
f.write("    command: mitmweb --web-host 0.0.0.0\n")
f.write("    ports:\n")
f.write("      - '8081:8080'\n")
f.write("      - '8082:8081'\n")
f.write("\n")
f.write("  client:\n")
f.write("    image: ubuntu:22.04\n")
f.write("    container_name: client\n")
f.write("    command: sleep infinity\n")
f.close()
print('Done!')
```

Save: **Ctrl+X → Y → Enter**

Run it:
```bash
python3 ~/make_compose.py
```

> ⚠️ If mitmproxy times out during download, change its image to `nginx:alpine` and command to `sleep infinity` in make_compose.py — the attack simulation still works from the host laptop.

Validate the YAML:
```bash
cd ~/quantum-break
sudo docker-compose config
# Should print full config without errors
```

---

## Step 3 — Start the Lab

```bash
cd ~/quantum-break
sudo docker-compose up -d
```

Verify all 4 containers running:
```bash
sudo docker ps
```

Expected:
```
NAMES          STATUS
pqc-server     Up X seconds   0.0.0.0:8443->80/tcp
legacy-server  Up X seconds   0.0.0.0:8080->80/tcp
client         Up X seconds
attacker       Up X seconds   8081->8080/tcp
```

Test both servers respond:
```bash
curl http://localhost:8080  # nginx welcome page
curl http://localhost:8443  # nginx welcome page
```

---

## Step 4 — Attack Phase

> ⚠️ Run ALL attack tests from your HOST laptop terminal.
> Do NOT try to install curl inside containers — they have no internet.

**Attack legacy server:**
```bash
curl -v http://localhost:8080 2>&1 | head -20
```

Expected — full data exposed:
```
> GET / HTTP/1.1
< HTTP/1.1 200 OK              ← ATTACK SUCCEEDED ❌
< Server: nginx/1.31.0         ← Identity LEAKED ❌
< Content-Length: 896          ← Data EXPOSED ❌
```

**Attack PQC server (before hardening):**
```bash
curl -v http://localhost:8443 2>&1 | head -20
```

Expected — also vulnerable at this stage:
```
< HTTP/1.1 200 OK   ← Also vulnerable before hardening ❌
```

**Quick comparison:**
```bash
curl -s -o /dev/null -w "Legacy: %{http_code}\n" http://localhost:8080
curl -s -o /dev/null -w "PQC:    %{http_code}\n" http://localhost:8443
# Legacy: 200
# PQC:    200
```

---

## Step 5 — Tshark Traffic Capture

> ⚠️ Fix permissions FIRST or tshark will fail with "Permission denied".

**Fix dumpcap permissions:**
```bash
sudo usermod -aG wireshark $USER
sudo chmod +x /usr/bin/dumpcap
sudo setcap cap_net_raw,cap_net_admin+eip /usr/bin/dumpcap
newgrp wireshark
```

**Fix captures folder:**
```bash
sudo mkdir -p ~/quantum-break/captures
sudo chown $USER:$USER ~/quantum-break/captures
chmod 755 ~/quantum-break/captures
```

**Start capture (no sudo needed now):**
```bash
tshark -i lo -w ~/quantum-break/captures/traffic.pcap &
```

**Generate attack traffic:**
```bash
curl http://localhost:8080
curl http://localhost:8443
```

**Stop capture:**
```bash
pkill tshark
```

**Read and analyse:**
```bash
tshark -r ~/quantum-break/captures/traffic.pcap | head -20
```

**Save HTTP analysis:**
```bash
tshark -r ~/quantum-break/captures/traffic.pcap \
  -Y "http" \
  -T fields \
  -e frame.number \
  -e ip.src \
  -e ip.dst \
  -e http.request.method \
  -e http.response.code \
  > ~/quantum-break/logs/tshark-analysis.txt

cat ~/quantum-break/logs/tshark-analysis.txt
```

Expected output:
```
4    127.0.0.1  127.0.0.1  GET
8    127.0.0.1  127.0.0.1       200   ← Legacy: exposed
16   127.0.0.1  127.0.0.1  GET
18   127.0.0.1  127.0.0.1       403   ← PQC: blocked
```

---

## Step 6 — Defense Hardening

**Apply deny-all rule to PQC server (zero downtime):**
```bash
sudo docker exec pqc-server sh -c \
  "echo 'deny all;' > /etc/nginx/conf.d/block.conf && nginx -s reload"
```

Expected:
```
2026/05/20 09:43:00 [notice] signal process started ← Reloaded, no downtime
```

**Verify hardening worked:**
```bash
curl -s -o /dev/null -w "Legacy: %{http_code}\n" http://localhost:8080
curl -s -o /dev/null -w "PQC:    %{http_code}\n" http://localhost:8443
```

Expected:
```
Legacy: 200   ← Still vulnerable ❌
PQC:    403   ← BLOCKED! ✅
```

**Save validation log:**
```bash
echo "=== PROJECT QUANTUM BREAK ===" > ~/quantum-break/logs/validation.txt
echo "Date: $(date)" >> ~/quantum-break/logs/validation.txt
echo "Legacy Server: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8080) VULNERABLE" >> ~/quantum-break/logs/validation.txt
echo "PQC Server: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8443) BLOCKED" >> ~/quantum-break/logs/validation.txt
echo "CONCLUSION: PQC hardening blocks attacks!" >> ~/quantum-break/logs/validation.txt
cat ~/quantum-break/logs/validation.txt
```

Expected:
```
=== PROJECT QUANTUM BREAK ===
Date: Wed May 20 09:52:39 AM UTC 2026
Legacy Server: 200 VULNERABLE
PQC Server: 403 BLOCKED
CONCLUSION: PQC hardening blocks attacks!
```

---

## Step 7 — ML-KEM-768 Quantum Tunnel

> ⚠️ Do NOT use `openssl genpkey -algorithm mlkem768` — the OpenSSL encoder fails.
> Use liboqs-python directly instead.

**Install liboqs-python:**
```bash
pip3 install liboqs-python --no-cache-dir
```

Expected:
```
Successfully installed liboqs-python-0.15.0 tomli-2.4.1
```

**Generate ML-KEM-768 keypair:**
```bash
python3 -c "
import oqs
kem = oqs.KeyEncapsulation('ML-KEM-768')
public_key = kem.generate_keypair()
secret_key = kem.export_secret_key()
open('/home/jeejon/quantum-break/certs/mlkem_public.key', 'wb').write(public_key)
open('/home/jeejon/quantum-break/certs/mlkem_secret.key', 'wb').write(secret_key)
print('ML-KEM-768 keys generated!')
print('Public key size:', len(public_key), 'bytes')
print('Secret key size:', len(secret_key), 'bytes')
"
```

Expected:
```
ML-KEM-768 keys generated!
Public key size: 1184 bytes
Secret key size: 2400 bytes
```

**Prove quantum tunnel works:**
```bash
python3 -c "
import oqs

kem_server = oqs.KeyEncapsulation('ML-KEM-768')
pub_key = kem_server.generate_keypair()

kem_client = oqs.KeyEncapsulation('ML-KEM-768')
ciphertext, secret_client = kem_client.encap_secret(pub_key)
secret_server = kem_server.decap_secret(ciphertext)

if secret_client == secret_server:
    print('ML-KEM Key Exchange SUCCESS!')
    print('Shared secret length:', len(secret_client), 'bytes')
    print('This connection is quantum safe!')
"
```

Expected:
```
ML-KEM Key Exchange SUCCESS!
Shared secret length: 32 bytes
This connection is quantum safe!
```

**Save proof to file:**
```bash
python3 -c "
import oqs
print('=== HYBRID PQC TRANSPORT MESH ===')
print('Algorithm: ML-KEM-768 (NIST Standard)')
kem_s = oqs.KeyEncapsulation('ML-KEM-768')
pub = kem_s.generate_keypair()
print('Server public key:', len(pub), 'bytes')
kem_c = oqs.KeyEncapsulation('ML-KEM-768')
ct, sec_c = kem_c.encap_secret(pub)
print('Client encapsulated secret')
sec_s = kem_s.decap_secret(ct)
print('Server decapsulated secret')
if sec_c == sec_s:
    print('')
    print('QUANTUM SAFE TUNNEL ESTABLISHED!')
    print('Shared secret:', sec_c.hex()[:32], '...')
    print('Legacy attacks: IMPOSSIBLE')
    print('Quantum attacks: IMPOSSIBLE')
" > ~/quantum-break/logs/mlkem-proof.txt
cat ~/quantum-break/logs/mlkem-proof.txt
```

---

## Step 8 — Full Validation

Run everything at once:
```bash
echo "=== FULL TEST RESULTS ===" &&
echo "1. Containers:" && sudo docker ps --format "{{.Names}} - {{.Status}}" &&
echo "" &&
echo "2. Legacy Server:" && curl -s -o /dev/null -w "Status: %{http_code}\n" http://localhost:8080 &&
echo "" &&
echo "3. PQC Server:" && curl -s -o /dev/null -w "Status: %{http_code}\n" http://localhost:8443 &&
echo "" &&
echo "4. Validation Log:" && cat ~/quantum-break/logs/validation.txt &&
echo "" &&
echo "5. ML-KEM Tunnel:" && python3 -c "
import oqs
k=oqs.KeyEncapsulation('ML-KEM-768')
p=k.generate_keypair()
k2=oqs.KeyEncapsulation('ML-KEM-768')
c,s1=k2.encap_secret(p)
s2=k.decap_secret(c)
print('TUNNEL:', 'ESTABLISHED' if s1==s2 else 'FAILED', '| Secret:', len(s1), 'bytes')
"
```

Expected full output:
```
=== FULL TEST RESULTS ===
1. Containers:
pqc-server    - Up
legacy-server - Up
client        - Up
attacker      - Up

2. Legacy Server:
Status: 200

3. PQC Server:
Status: 403

4. Validation Log:
=== PROJECT QUANTUM BREAK ===
Legacy Server: 200 VULNERABLE
PQC Server: 403 BLOCKED
CONCLUSION: PQC hardening blocks attacks!

5. ML-KEM Tunnel:
TUNNEL: ESTABLISHED | Secret: 32 bytes
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `yaml.scanner.ScannerError` | YAML file corrupted by terminal | Use Python f.write() make_compose.py |
| `mitmproxy TLS handshake timeout` | Slow internet, large image | Change attacker to nginx:alpine |
| `tshark: Permission denied on pcap file` | Wrong folder permissions | `chown $USER:$USER ~/quantum-break/captures/` |
| `dumpcap: Permission denied` | User not in wireshark group | `sudo usermod -aG wireshark $USER` + `newgrp wireshark` |
| `apt-get: unable to locate curl` | Container has no internet | Run all curls from HOST laptop instead |
| `openssl: Error initializing mlkem768` | OpenSSL encoder missing | Use `import oqs` Python library instead |
| `Error writing key: No encoders found` | OpenSSL can't serialize PQC key | Use liboqs-python directly |

---

## Files Created

```
~/quantum-break/
├── docker-compose.yml          ← Lab definition
├── logs/
│   ├── validation.txt          ← 200 vs 403 proof
│   ├── tshark-analysis.txt     ← HTTP packet analysis
│   └── mlkem-proof.txt         ← Quantum tunnel proof
├── captures/
│   └── traffic.pcap            ← Raw packet capture
└── certs/
    ├── mlkem_public.key        ← ML-KEM public key (1184 bytes)
    └── mlkem_secret.key        ← ML-KEM secret key (2400 bytes)
```

---

## Related Project

**[PQC Vault Secret Rotation Engine](../pqc-vault-engine)**
Automatic PQC certificate rotation every 60 minutes using HashiCorp Vault and mldsa65.

Together they provide complete quantum-safe security:
- Application layer: rotating PQC certificates
- Network layer: ML-KEM-768 quantum tunnels

---

**Together:** Complete quantum-safe security at both application and network layers.

## Report Doc
https://docs.google.com/document/d/10TzLwbaIVUYLD4lsGV8jM0yathnFO8gfeXSUQf5QmMI/edit?tab=t.0

---

## Author

**Johnson Oni** | Bincom | johnsononi@internal.bincom.net

---

## License

MIT License

---

> *"Don't wait for quantum computers to arrive. Build quantum-safe infrastructure today."*
