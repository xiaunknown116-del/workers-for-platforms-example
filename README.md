import os

# Create directories
os.makedirs('generated/docs/control-plane', exist_ok=True)
os.makedirs('generated/admin-control-app', exist_ok=True)
os.makedirs('generated/config', exist_ok=True)

# File 1: webauthn-attestation-formats.md
webauthn_formats_content = """# WebAuthn Attestation Format Enforcement
# Path: docs/control-plane/webauthn-attestation-formats.md
# Reference Policy: SOP-SEC-2026-04_hardware-key-aaguid-policy.md

This document establishes the official parsing guidelines for WebAuthn metadata inside the Apex Capital Hardened AdminApp environment. All credentials targeting the Tier-0 Admin Control Plane must adhere strictly to these profiles.

## Supported Formal Attestation Formats (IANA Registry)

| Format ID | Cryptographic Mechanics | Verification Requirements | Target Environment |
| :--- | :--- | :--- | :--- |
| **`packed`** | X.509 Certificate Chain or Self-Signed Signature (ECDSA / EdDSA) | Verify cert chain path against FIDO MDS metadata or structural self-signature. | **Mandatory** — Primary YubiKey, SoloKeys, and hardware security tokens. |
| **`tpm`** | AIK (Attestation Identity Key) certificate signed via trusted TPM 2.0 | Validate parsing of `certInfo` and `pubArea`. Ensure hash integrity against SHA-256 signatures. | **Corporate Endpoints** — Windows Hello for Business enterprise workstations. |
| **`android-key`** | X.509 Certificate Extension containing key description block | Verify `teeEnforced` parameters. Assert that key usage flags match device-bound configurations. | **Mobile Hardware** — TEE/StrongBox backed Android Enterprise devices. |
| **`fido-u2f`** | Legacy FIDO U2F X.509 attestation certificate format | Extract public keys from signature payloads; handle 65-byte uncompressed EC points. | **Legacy Tokens** — Backup/secondary physical tokens. |
| **`apple`** | X.509 leaf cert containing value matching SHA-256 hash of `authData` + `clientDataHash` | Verify certificate chain roots against known Apple WebAuthn Roots. | **macOS Workstations** — Enterprise Secure Enclave (Touch ID / Face ID). |

## Forbidden and Non-Attested Types

*   **`android-safetynet`**: **Explicitly Blocked**. Deprecated by Google. Any legacy fallback attempting this mechanism must be rejected with an immediate `SecurityAlert` payload.
*   **`none`**: Allowed **only** on consumer-facing dashboards or lower tier interfaces. Tier-0 Admin Control Plane enrollment attempts containing `fmt: "none"` must throw an uncatchable initialization error.

## Execution Constraints
1. **Direct Conveyance Requirement**: Relying Party configurations must request `attestation: "direct"`.
2. **AAGUID Extraction**: The 16-byte AAGUID must be unpacked directly from the `authData` payload prior to parsing.
3. **Allowlist Cross-Reference**: Match the parsed AAGUID explicitly against `config/aaguid_whitelist.json`.
"""

with open('generated/docs/control-plane/webauthn-attestation-formats.md', 'w') as f:
    f.write(webauthn_formats_content)

# File 2: SOP-SEC-2026-04_hardware-key-aaguid-policy.md (Stub/Inferred)
sop_content = """# SOP-SEC-2026-04: Hardware Key AAGUID Enforcement Policy
# Classification: TIER-0 INTERNAL
# Effective Date: 2026-04-01

## 1. Objective
To restrict Tier-0 Admin Control Plane access to approved hardware authenticators via strict AAGUID (Authenticator Attestation GUID) allowing.

## 2. Policy Requirements
*   **All Admin Access**: Must use FIDO2/WebAuthn hardware tokens. Software authenticators are prohibited for Tier-0 operations unless using approved managed device enclaves (e.g., Corporate Managed Windows Hello TPM).
*   **AAGUID Enforcement**: The system must validate the AAGUID present in the attestation statement against the authoritative allowlist located at `config/aaguid_whitelist.json`.
*   **Prohibited Devices**: Generic U2F tokens or devices with unknown AAGUIDs (`00000000-0000-0000-0000-000000000000` except in Sandbox) are denied by default.

## 3. Approved Hardware List (Reference)
Refer to `config/aaguid_whitelist.json` for the active cryptographic allowlist.

## 4. Audit & Compliance
*   All registration attempts (Success/Failure) must be logged to the immutable SHA-256 audit chain.
*   Attestation format validation must occur *before* AAGUID checks.
"""

with open('generated/docs/control-plane/SOP-SEC-2026-04_hardware-key-aaguid-policy.md', 'w') as f:
    f.write(sop_content)

# File 3: webauthn_verifier.py (The backend stub)
verifier_content = """\"\"\"
Module: admin_app/control_plane/webauthn_verifier.py
Description: Hardened WebAuthn attestation parser for Tier-0 Admin Control Plane.
Compliance: SOP-SEC-2026-04 Hardware-Key Policy.
Environment: Controlled Sandbox (source: "sandbox-mock")
\"\"\"

import json
import hashlib
import hmac
import logging
from typing import Dict, Any, Tuple

# Setup isolation logging
logger = logging.getLogger("ApexAdminControlPlane")

# Simulated path targets from infrastructure configuration
AAGUID_ALLOWLIST_PATH = "config/aaguid_whitelist.json"

class AttestationValidationError(Exception):
    \"\"\"Custom exception wrapper for high-impact authentication validation issues.\"\"\"
    pass

class HardenedWebAuthnVerifier:
    def __init__(self, allowlist_source: str = AAGUID_ALLOWLIST_PATH):
        self.allowlist_source = allowlist_source
        # Valid cryptographic attestation formats allowed in Tier-0 Control Plane
        self.permitted_formats = {"packed", "tpm", "android-key", "fido-u2f", "apple"}
        # Strictly explicitly prohibited/deprecated formats
        self.forbidden_formats = {"android-safetynet", "none"}

    def _load_aaguid_allowlist(self) -> Dict[str, Any]:
        \"\"\"Loads and parses the allowed hardware authenticators from the master policy list.\"\"\"
        try:
            # In a live system, this reads directly from secure storage
            # Mocking data layer isolation pattern for sandbox validation
            mock_allowlist = {
                "00000000-0000-0000-0000-000000000000": "System Mock Token",
                "f81d4fae-7dec-11d0-a765-00a0c91e6bf6": "YubiKey 5 Series NFC",
                "7c526a0c-43f1-48fb-9c88-e25dfddc3b28": "Apple Secure Enclave Platform Token"
            }
            return mock_allowlist
        except Exception as e:
            raise AttestationValidationError(f"Failed to populate AAGUID security dictionary: {str(e)}")

    def _write_to_sha256_audit_chain(self, event_type: str, status: str, details: Dict[str, Any]) -> str:
        \"\"\"
        Emits records to the append-only cryptographic log layer.
        Ensures strict compliance with sandbox-mock source constraints.
        \"\"\"
        payload = {
            "source": "sandbox-mock",
            "layer": "Tier-0 Admin Control Plane",
            "event": event_type,
            "status": status,
            "meta": details
        }
        serialized = json.dumps(payload, sort_keys=True)
        record_hash = hashlib.sha256(serialized.encode('utf-8')).hexdigest()
        
        # Log payload emission targeting append-only block simulation
        logger.info(f"[AUDIT-CHAIN-EMIT] Hash: {record_hash} | Payload: {serialized}")
        return record_hash

    def parse_and_verify_attestation(self, fmt: str, auth_data: bytes, att_stmt: Dict[str, Any]) -> Tuple[bool, str]:
        \"\"\"
        Parses incoming credentials, isolates structural format identifiers, 
        unpacks the underlying AAGUID, and assesses security posture.
        \"\"\"
        audit_meta = {"format": fmt}
        
        try:
            # 1. Evaluate format validity against formal registries
            if fmt in self.forbidden_formats:
                raise AttestationValidationError(f"Registration rejected. Format '{fmt}' is explicitly prohibited.")
            
            if fmt not in self.permitted_formats:
                raise AttestationValidationError(f"Registration rejected. Format '{fmt}' unrecognized by Policy.")
            
            # 2. Extract structural binary elements from Authenticator Data (authData)
            # authData structural breakdown: 
            # RpIdHash (32 bytes) -> Flags (1 byte) -> SignCount (4 bytes) -> AttestedCredData (Variable)
            if len(auth_data) < 37:
                raise AttestationValidationError("Malformed authData block: payload under minimum required byte limit.")
            
            # Attested credential data starts at byte 37 if flags specify its presence
            flags = auth_data
            # Bit 6 (0x40) specifies presence of Attested Credential Data
            if not (flags & 0x40):
                raise AttestationValidationError("Invalid context status: Credential Data flags absent in Tier-0 operation.")
            
            # Extract 16-byte AAGUID from offset 37 to 53
            aaguid_bytes = auth_data[37:53]
            if len(aaguid_bytes) != 16:
                raise AttestationValidationError("Failed to unpack complete 16-byte block for AAGUID isolation.")
            
            # Convert binary array to canonical string representation format
            aaguid_str = "-".join([
                aaguid_bytes[0:4].hex(),
                aaguid_bytes[4:6].hex(),
                aaguid_bytes[6:8].hex(),
                aaguid_bytes[8:10].hex(),
                aaguid_bytes[10:16].hex()
            ])
            
            audit_meta["extracted_aaguid"] = aaguid_str
            
            # 3. Assess extracted token signature scope against verified hardware list
            allowlist = self._load_aaguid_allowlist()
            if aaguid_str not in allowlist:
                raise AttestationValidationError(f"AAGUID '{aaguid_str}' missing from explicit hardware whitelist.")
            
            # Record explicit description target for policy compliance checking
            audit_meta["device_profile"] = allowlist[aaguid_str]
            
            # 4. Finalize state change validation loop
            chain_hash = self._write_to_sha256_audit_chain(
                event_type="HARDWARE_KEY_REGISTRATION",
                status="SUCCESS",
                details=audit_meta
            )
            return True, chain_hash

        except AttestationValidationError as exc:
            audit_meta["error_message"] = str(exc)
            self._write_to_sha256_audit_chain(
                event_type="HARDWARE_KEY_REGISTRATION_DENIED",
                status="FAILURE",
                details=audit_meta
            )
            # Escalate verification exception details back up to caller context
            raise exc
"""

with open('generated/admin-control-app/webauthn_verifier.py', 'w') as f:
    f.write(verifier_content)

# File 4: aaguid_whitelist.json
whitelist_content = """{
    "00000000-0000-0000-0000-000000000000": "System Mock Token",
    "f81d4fae-7dec-11d0-a765-00a0c91e6bf6": "YubiKey 5 Series NFC",
    "7c526a0c-43f1-48fb-9c88-e25dfddc3b28": "Apple Secure Enclave Platform Token"
}"""

with open('generated/config/aaguid_whitelist.json', 'w') as f:
    f.write(whitelist_content)

print(f"Files generated: {os.listdir('generated/docs/control-plane')}, {os.listdir('generated/admin-control-app')}, {os.listdir('generated/config')}")
# Workers for Platforms Example Project

- [Blog post](https://blog.cloudflare.com/workers-for-platforms/)
- [Docs](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms)
- [Discord](https://discord.cloudflare.com/)

This is a **minimal Workers for Platforms** example that demonstrates the core concepts of dynamic dispatch. The platform allows users to create and upload custom Workers through a simple web interface, then access them via friendly URLs.

Workers for Platforms gives your customers the ability to build services and customizations (powered by Workers) while you retain full control over how their code is executed and billed. The **dynamic dispatch namespaces** feature makes this possible.

By creating a dispatch namespace and using the `dispatch_namespaces` binding in a regular fetch handler, you have a "dispatch Worker":

```javascript
export default {
  async fetch(request, env) {
    // "dispatcher" is a binding defined in wrangler.jsonc
    // "my-user-worker" is a script previously uploaded to the dispatch namespace
    const worker = env.dispatcher.get("my-user-worker");
    return await worker.fetch(request);
  }
}
```

This is the perfect way for a platform to create boilerplate functions, handle routing to "user Workers", and sanitize responses. You can manage thousands of Workers with a single Cloudflare Workers account!

## In this example

Users can upload Workers scripts through a simple web form. The platform uploads the script to a dispatch namespace and stores a name → Worker ID mapping in Workers KV. Users can then access their Workers via URLs like `/user-workers/my-worker`.

This minimal example focuses on the core Workers for Platforms concepts:
- Dynamic dispatch using the `dispatcher` binding
- Worker upload via the Cloudflare API
- Simple name-based routing using KV storage

## Key Features

- **Simple Worker Creation**: Web form for uploading Worker code
- **Dynamic Dispatch**: Route requests to user Workers by name
- **KV Storage**: Store friendly name mappings
- **No Dependencies**: Pure Workers runtime with minimal external dependencies

## Getting started

Your Cloudflare account needs access to Workers for Platforms.

1. Install the package and dependencies:

   ```
   npm install
   ```

2. Create an API token with Workers Scripts (Edit) permission:

   Visit [https://dash.cloudflare.com/?to=/:account/api-tokens](https://dash.cloudflare.com/?to=/:account/api-tokens) and create a new token with the "Workers Scripts (Edit)" permission.

3. Copy the `.env.test` file to `.env` and set the `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` secrets:

   ```sh
   cp .env.test .env
   ```

   Then edit the `.env` file with your actual values:

   ```sh
   CLOUDFLARE_ACCOUNT_ID = "your_actual_account_id"
   CLOUDFLARE_API_TOKEN = "your_actual_api_token"
   ```

   The `.env` file is already in `.gitignore` and will not be committed to git.

   Then run the following commands to add these secrets to your Worker in production:

   ```
   npx wrangler secret put CLOUDFLARE_API_TOKEN
   ```

   ```
   npx wrangler secret put CLOUDFLARE_ACCOUNT_ID
   ```

4. Create a KV namespace for Worker mappings:

   ```
   npx wrangler kv:namespace create "WORKER_MAPPINGS"
   ```

   Copy the namespace ID and preview ID into `wrangler.jsonc` under the `kv_namespaces` binding.

5. Create a dispatch namespace:

   ```
   npx wrangler dispatch-namespace create workers-for-platforms-example-project
   ```

6. Run the Worker in dev mode:
   ```
   npm run dev
   ```
   Or deploy to production:
   ```
   npm run deploy
   ```

Once the Worker is live, visit [localhost:8787](http://localhost:8787/) in a browser. You can create a new Worker via the "/upload" link. Access your Workers at `/user-workers/{name}`!

Then access it at: `http://localhost:8787/user-workers/my-worker`
