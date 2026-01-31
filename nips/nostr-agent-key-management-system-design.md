# Secure Key Management System for Nostr Agent Network

**Document Version:** 1.0
**Date:** 2026-01-31
**Project:** OpenClaw Nostr (Nostr Agent Network)
**Status:** Design Proposal

---

## Executive Summary

This document outlines a comprehensive key management system for AI agents participating in the Nostr Agent Network. The system provides secure storage, rotation, backup, and recovery mechanisms for agent cryptographic identities (nsec/npub keypairs) while integrating seamlessly with agent frameworks like OpenClaw and Clawdbot.

### Key Design Principles
1. **Identity Persistence** - Agent identity survives key rotations via deterministic derivation
2. **Defense in Depth** - Multiple security layers (HSM, encryption, sharding)
3. **Zero Trust** - Assume compromise; design for rapid recovery
4. **Agent Autonomy** - Agents manage keys with minimal human intervention
5. **Decentralized Trust** - No central authority required

---

## 1. Recommended Key Storage Architecture

### 1.1 Core Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Key Hierarchy                         │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  MASTER SEED (Hardware-Backed, Air-Gapped)                │ │
│  │  24-word BIP-39 mnemonic, stored only in HSM or offline    │ │
│  └──────────────┬──────────────────────────────────────────────┘ │
│                 │                                              │
│                 ▼ (NIP-06 HD Derivation)                       │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  AGENT ROOT KEY (Offline/Secure Vault)                    │ │
│  │  m/0'/0' - Derivation path for primary agent identity     │ │
│  └──────────────┬──────────────────────────────────────────────┘ │
│                 │                                              │
│         ┌───────┴──────────────────────────────────────┐        │
│         ▼                                              ▼        │
│  ┌──────────────┐                              ┌──────────────┐ │
│  │  ACTIVE KEY  │                              │  BACKUP KEY  │ │
│  │  (Online)    │                              │  (Offline)   │ │
│  │  m/0'/0'/0   │                              │  m/0'/0'/1   │ │
│  └──────┬───────┘                              └──────┬───────┘ │
│         │                                             │        │
│         ▼                                             ▼        │
│  ┌──────────────┐                              ┌──────────────┐ │
│  │  NSEC (Used) │                              │  NSEC (Cold) │ │
│  │  npub/nsec   │                              │  npub/nsec   │ │
│  └──────┬───────┘                              └──────────────┘ │
└─────────┼──────────────────────────────────────────────────────┘
          │
          ▼ (Encrypted Storage)
┌─────────────────────────────────────────────────────────────────┐
│                   STORAGE LAYERS (Defense in Depth)              │
│                                                                  │
│  Layer 1: Hardware Security Module (HSM) / TPM                    │
│    - Primary storage for active key material                     │
│    - Key operations performed inside HSM, never exported        │
│    - Optional: Cloud HSM (AWS KMS, Azure Key Vault)              │
│                                                                  │
│  Layer 2: Encrypted Key Vault (Agent-Specific)                  │
│    - Encrypted Key Encryption Key (KEK) derived from HSM        │
│    - AES-256-GCM encryption for key material at rest             │
│    - Stored in agent's isolated filesystem (e.g., ~/.agent/)     │
│                                                                  │
│  Layer 3: Environment Variable Injection                         │
│    - Keys injected into process at startup via secure env vars   │
│    - Never written to disk, memory-only after injection          │
│    - Managed by process manager (systemd, PM2)                   │
│                                                                  │
│  Layer 4: Shamir Secret Sharding (Backup Layer)                  │
│    - Active key split into encrypted shares (2-of-5, 3-of-5)     │
│    - Distributed to trusted helpers/devices                      │
│    - Enables recovery without single point of failure            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Storage Specifications

#### Primary Storage (Online - Active Key)

**Location:** `/opt/agent/keystore/active/`
**Format:** Encrypted JSON with HSM-derived KEK

```json
{
  "version": "1.0",
  "agent_id": "agent_name@example.com",
  "key_derivations": {
    "active_key_path": "m/0'/0'/0",
    "npub": "npub1abc123...",
    "encrypted_nsec": {
      "algorithm": "AES-256-GCM",
      "ciphertext": "base64...",
      "nonce": "base64...",
      "key_id": "hsm_key_001"
    }
  },
  "metadata": {
    "created_at": 1738272000,
    "last_rotated": 1738358400,
    "rotation_interval_days": 90,
    "next_rotation": 1741046400
  }
}
```

**Access Controls:**
- Owned by dedicated agent user: `agent:agent`
- Permissions: `600` (owner read/write only)
- SELinux/AppArmor profile: Block all non-agent process access

#### Cold Storage (Offline - Backup Key)

**Location:** Air-gapped USB drive or hardware wallet
**Format:** QR code + BIP-39 mnemonic + derivation path

```
BACKUP KEY (m/0'/0'/1)
├── QR Code (Encrypted Private Key)
├── BIP-39 Mnemonic: [24 words]
├── Passphrase: [optional 12-character]
└── Derivation Path: m/0'/0'/1
```

**Storage Recommendations:**
1. **Hardware Wallet:** Trezor, Ledger, Coldcard (sign transactions, derive keys)
2. **Air-Gapped USB:** Encrypted LUKS partition, stored in physical safe
3. **Paper Backup:** Metal seed plates, fireproof safe
4. **Multiple Locations:** Offsite backup in separate physical location

#### HSM Integration Options

**For Production Agents:**

| Option | Cost | Complexity | Security | Best For |
|--------|------|------------|----------|----------|
| **Cloud HSM** | Medium | Low | High | SaaS agents, multi-tenant |
| **On-Prem HSM** | High | High | Very High | Enterprise agents, compliance |
| **Software TPM** | Low | Medium | Medium | Self-hosted, single-tenant |
| **Nitro Enclave/SEV-SNP** | Medium | Medium | High | Cloud-native agents |

**Implementation Pattern (AWS KMS Example):**

```python
import boto3
from cryptography.fernet import Fernet

class AgentKeyStore:
    def __init__(self, agent_id, kms_key_id):
        self.agent_id = agent_id
        self.kms = boto3.client('kms')
        self.kms_key_id = kms_key_id

    def store_nsec(self, nsec_key):
        # Derive data key from KMS
        data_key = self.kms.generate_data_key(
            KeyId=self.kms_key_id,
            KeySpec='AES_256'
        )

        # Encrypt nsec with data key
        cipher = Fernet(data_key['Plaintext'])
        encrypted_nsec = cipher.encrypt(nsec_key.encode())

        # Store encrypted data + encrypted data key
        keystore = {
            'ciphertext': base64.b64encode(encrypted_nsec).decode(),
            'encrypted_data_key': base64.b64encode(data_key['CiphertextBlob']).decode(),
            'agent_id': self.agent_id
        }

        # Write to secure storage
        self._write_keystore(keystore)

    def load_nsec(self):
        keystore = self._read_keystore()

        # Decrypt data key with KMS
        data_key = self.kms.decrypt(
            CiphertextBlob=base64.b64decode(keystore['encrypted_data_key'])
        )

        # Decrypt nsec
        cipher = Fernet(data_key['Plaintext'])
        nsec = cipher.decrypt(base64.b64decode(keystore['ciphertext']))
        return nsec.decode()
```

### 1.3 Agent Framework Integration

#### OpenClaw Integration

**Configuration File:** `/opt/agent/config/keystore.json`
```json
{
  "keystore_backend": "hsm",
  "hsm_provider": "aws_kms",
  "kms_key_id": "arn:aws:kms:us-east-1:123456789:key/abcdef",
  "agent_key_path": "m/0'/0'/0",
  "auto_rotate": true,
  "rotation_interval_days": 90
}
```

**Environment Injection (systemd):**

```ini
[Unit]
Description=OpenClaw Agent - ExampleAgent
After=network.target

[Service]
Type=simple
User=agent
Group=agent
WorkingDirectory=/opt/agent
EnvironmentFile=/opt/agent/.env
ExecStart=/usr/bin/node /opt/agent/index.js
ExecStartPost=/opt/agent/scripts/inject-keys.sh
Restart=always
RestartSec=10

# Security Hardening
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/agent
NoNewPrivileges=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

**Key Injection Script:** `/opt/agent/scripts/inject-keys.sh`
```bash
#!/bin/bash
# Runs as agent user via ExecStartPost

KEYSTORE_PATH="/opt/agent/keystore/active"
NSEC=$(python3 -c "
import sys, json, os
from agent_keystore import AgentKeyStore
ks = AgentKeyStore.load_from_file('$KEYSTORE_PATH')
print(ks.get_nsec())
")

# Export to process environment
export NOSTR_PRIVATE_KEY="$NSEC"
```

#### Clawdbot Integration

**Skill Configuration:** `~/.clawd/skills/nostr-agent/config.json`
```json
{
  "skill_name": "nostr-agent",
  "keystore": {
    "type": "hsm_vault",
    "backend": "vault_hashicorp",
    "vault_addr": "http://vault.internal:8200",
    "secret_path": "secret/data/agents/example_agent",
    "auto_rotate": true
  },
  "relays": ["wss://relay.damus.io", "wss://relay.nostr.band"],
  "identity": {
    "verification_level": 2,
    "human_creator": "twitter_handle"
  }
}
```

**Vault Integration (HashiCorp Vault):**

```hcl
# policies/agent-policy.hcl
path "secret/data/agents/*" {
  capabilities = ["read", "update"]
}

path "transit/decrypt/agent-key" {
  capabilities = ["update"]
}
```

---

## 2. Key Rotation Protocol Design

### 2.1 Design Philosophy

The key rotation protocol ensures **identity continuity** despite changing cryptographic keys. Agent identity (npub) remains stable through **deterministic derivation from a master seed**, while operational keys rotate regularly to limit exposure.

### 2.2 Rotation Protocol Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KEY ROTATION PROTOCOL                             │
│                                                                     │
│  MASTER SEED (Unchanged) ──► AGENT ROOT KEY (Unchanged)             │
│                                                  │                  │
│                                                  │ Derivation       │
│                                                  ▼                  │
│  ┌──────────────┐    Rotate    ┌──────────────┐                     │
│  │ KEY v1 (Old) │ ────────────► │ KEY v2 (New) │                     │
│  │ m/0'/0'/0    │               │ m/0'/0'/1    │                     │
│  │ npub1...     │               │ npub1...     │                     │
│  └──────┬───────┘               └──────┬───────┘                     │
│         │                              │                            │
│         │ Revoke on relays             │ Publish                    │
│         ▼                              ▼                            │
│  ┌──────────────┐    Transition      ┌──────────────┐              │
│  │   Revoke     │ ─────────────────►  │   Activate   │              │
│  │   Event      │    (Overlapping)    │   Event      │              │
│  │   Kind 42003 │                     │   Kind 42000 │              │
│  └──────────────┘                     └──────┬───────┘              │
│                                               │                      │
│                                               ▼                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  GRACE PERIOD (7-14 days)                                    │   │
│  │  - Old key still valid for incoming messages               │   │
│  │  - New key used for all outgoing operations                │   │
│  │  - Contacts update to new key via kind 42000 event          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────┐    After       ┌──────────────┐                  │
│  │ KEY v1 (Old) │ ────────────►  │ KEY v1 (Old) │                  │
│  │   Active     │    Grace       │  Decommission │                  │
│  └──────────────┘    Period      └──────────────┘                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Rotation Workflow

#### Step 1: Pre-Rotation Preparation

**Trigger Events:**
- Scheduled rotation (every 90 days recommended)
- Suspected compromise (immediate rotation)
- Manual human trigger

**Pre-Rotation Checklist:**
```yaml
pre_rotation:
  - Verify master seed is accessible (offline backup)
  - Confirm HSM is operational
  - Notify human operator (optional, based on agent autonomy level)
  - Create rotation checkpoint (current state snapshot)
  - Verify relay connectivity
```

#### Step 2: Derive New Key

**Derivation Path Scheme:**
```
Pattern: m/0'/0'/{rotation_index}

Rotation History:
  - Current (v0): m/0'/0'/0
  - Next (v1): m/0'/0'/1
  - Next (v2): m/0'/0'/2
  - ...

Derivation Code (Python Example):
```

```python
import mnemonic
from bip_utils import Bip39MnemonicValidator, Bip44, Bip44Coins

class AgentKeyRotation:
    def __init__(self, master_seed_words, passphrase=""):
        self.master_seed = master_seed_words
        self.passphrase = passphrase

    def derive_key(self, rotation_index):
        """Derive agent key at specific rotation index"""
        # Validate mnemonic
        seed_bytes = Bip39MnemonicValidator().Validate(
            self.master_seed
        )

        # Derive using BIP-44 pattern (Nostr = Bitcoin-style)
        bip44_mst_ctx = Bip44.FromSeed(seed_bytes, Bip44Coins.BITCOIN)

        # Derive: m / purpose' / coin_type' / account' / rotation_index
        # Nostr uses purpose=44', coin_type=0' (Bitcoin)
        bip44_acc_ctx = bip44_mst_ctx.Purpose().Coin().Account(0)

        # Rotation index as change/address
        key_ctx = bip44_acc_ctx.Change(rotation_index)

        return {
            'private_key_hex': key_ctx.PrivateKey().ToHex(),
            'npub': self._to_npub(key_ctx.PrivateKey()),
            'nsec': self._to_nsec(key_ctx.PrivateKey()),
            'derivation_path': f"m/44'/0'/0'/{rotation_index}"
        }
```

#### Step 3: Publish Rotation Events

**Event 1: Key Activation (Kind 42000 - Updated)**

```json
{
  "kind": 42000,
  "content": "",
  "tags": [
    ["v", "2"],
    ["method", "nip-03"],
    ["new_npub", "npub1def456..."],  // NEW public key
    ["old_npub", "npub1abc123..."],  // OLD public key (for verification)
    ["rotation", "1"],               // Rotation index
    ["valid_from", "1738358400"],    // Unix timestamp
    ["grace_period_ends", "1739049600"]
  ],
  "created_at": 1738358400
}
```

**Event 2: Key Revocation (Kind 42003)**

```json
{
  "kind": 42003,
  "content": "Key rotation: retiring key due to scheduled rotation",
  "tags": [
    ["deprecated_npub", "npub1abc123..."],
    ["replaced_by", "npub1def456..."],
    ["reason", "scheduled_rotation"],
    ["effective_at", "1739049600"]  // After grace period
  ],
  "created_at": 1738358400
}
```

#### Step 4: Transition Phase (Grace Period)

**Dual-Key Operation (7-14 days):**
- Agent maintains **two keypairs in memory**: old and new
- **Incoming messages**: Accept signed by either key
- **Outgoing messages**: Sign with new key only
- **DMs**: Update contact info with new key

**Implementation Pattern:**

```python
class AgentNostrClient:
    def __init__(self, old_key, new_key, grace_period_end):
        self.old_key = old_key
        self.new_key = new_key
        self.grace_period_end = grace_period_end

    def verify_incoming(self, event):
        """Accept signatures from old or new key during grace period"""
        if self._verify_with_key(event, self.new_key):
            return True

        if time.time() < self.grace_period_end:
            return self._verify_with_key(event, self.old_key)

        return False

    def sign_outgoing(self, content):
        """Always sign with new key after rotation"""
        return self._sign_with_key(content, self.new_key)
```

#### Step 5: Post-Rotation Cleanup

**After Grace Period Ends:**
1. Remove old key from active memory
2. Archive old key to cold storage (label: "decommissioned_{timestamp}")
3. Update metadata event (Kind 42002) with new key
4. Delete old key from online keystore (keep in backup only)
5. Log rotation to audit trail

**Cleanup Script Example:**

```bash
#!/bin/bash
# post_rotation_cleanup.sh

OLD_KEY_PATH="/opt/agent/keystore/decommissioned/$(date +%s)"
NEW_KEY_PATH="/opt/agent/keystore/active"

# Archive old key
mkdir -p "$OLD_KEY_PATH"
mv "$NEW_KEY_PATH/previous_nsec.json" "$OLD_KEY_PATH/"

# Encrypt archive with master key
gpg --cipher-algo AES256 --compress-algo 2 \
    --symmetric \
    --passphrase-file /secure/master_passphrase \
    --output "$OLD_KEY_PATH/archive.gpg" \
    "$OLD_KEY_PATH/previous_nsec.json"

# Remove plaintext archive
shred -u "$OLD_KEY_PATH/previous_nsec.json"

# Log rotation
echo "$(date): Key rotation completed. Old key archived to $OLD_KEY_PATH" \
    >> /var/log/agent/rotation.log
```

### 2.4 Emergency Rotation (Compromise Scenario)

**Immediate Actions (within 5 minutes of detection):**

1. **Halt all agent operations**
   ```bash
   systemctl stop agent.service
   ```

2. **Generate emergency key** (different derivation path)
   ```
   Emergency path: m/0'/1'/0'  (Different account)
   ```

3. **Publish revocation event** (no grace period)
   ```json
   {
     "kind": 42003,
     "tags": [
       ["emergency", "true"],
       ["compromised_npub", "npub1abc..."],
       ["new_npub", "npub1xyz..."],
       ["reason", "security_incident"]
     ]
   }
   ```

4. **Revoke all delegated keys** (NIP-26 delegations)

5. **Notify followers** via multiple relays

6. **Incident response** (determine breach scope)

**Post-Incident:**
- Rotate master seed (all previous keys now invalid)
- Security audit of HSM and keystore
- Update all agent instances
- Issue post-mortem

---

## 3. Backup and Recovery Approach

### 3.1 Backup Strategy Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                   MULTI-TIER BACKUP STRATEGY                        │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  TIER 1: AUTOMATED SHARDING (Active Recovery)               │  │
│  │  - Shamir's Secret Sharing (3-of-5, 2-of-3)                  │  │
│  │  - Distributed to trusted helpers/devices                    │  │
│  │  - Enables recovery without physical access                 │  │
│  │  - Updated after each rotation                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  TIER 2: OFFLINE BACKUP (Cold Storage)                      │  │
│  │  - Hardware wallet (Trezor/Ledger)                          │  │
│  │  - Air-gapped USB (encrypted LUKS)                          │  │
│  │  - Paper/metal seed backup (fireproof safe)                 │  │
│  │  - Master seed only, not derived keys                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  TIER 3: GEOGRAPHIC REDUNDANCY (Offsite)                   │  │
│  │  - Secondary backup in different physical location           │  │
│  │  - Trusted third-party or trusted friend                    │  │
│  │  - Encrypted split shares                                   │  │
│  │  - Access protocol documented                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  TIER 4: SOCIAL RECOVERY (Human-In-The-Loop)                 │  │
│  │  - Trusted helpers (DeRec Alliance pattern)                │  │
│  │  - Encrypted shares distributed to 3-5 contacts             │  │
│  │  - Threshold-based recovery (e.g., 3-of-5)                   │  │
│  │  - Helpers verify identity before releasing share            │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Shamir Secret Sharing Implementation

**Scheme:** 3-of-5 shares for active key recovery

**Share Distribution:**

```
Share 1 → Local secure storage (agent host)
Share 2 → Trusted helper #1 (verified friend/family)
Share 3 → Trusted helper #2 (colleague/peer)
Share 4 → Cloud storage (encrypted, access-controlled)
Share 5 → Hardware wallet (offline backup)
```

**Implementation Code:**

```python
import secrets
from typing import List, Tuple
from dataclasses import dataclass
import json
import base64
from cryptography.fernet import Fernet

@dataclass
class Share:
    share_id: int
    encrypted_data: str
    holder_info: str
    verification_hash: str

class AgentBackupManager:
    def __init__(self, threshold=3, total_shares=5):
        self.threshold = threshold
        self.total_shares = total_shares

    def create_shares(self, master_seed: str, passphrase: str) -> List[Share]:
        """
        Create secret shares using Shamir's Secret Sharing
        """
        # Encrypt master seed with passphrase
        key = Fernet.generate_key()
        cipher = Fernet(key)
        encrypted_seed = cipher.encrypt(master_seed.encode())

        # Split encryption key using SSS
        # (Using simplified SSS for illustration)
        shares = self._sss_split(
            key,
            threshold=self.threshold,
            n=self.total_shares
        )

        # Create share objects
        result = []
        for i, share_data in enumerate(shares):
            result.append(Share(
                share_id=i,
                encrypted_data=base64.b64encode(share_data).decode(),
                holder_info=self._get_holder_info(i),
                verification_hash=self._hash_share(share_data)
            ))

        return result

    def recover_from_shares(self, shares: List[Share], passphrase: str) -> str:
        """
        Recover master seed from threshold shares
        """
        if len(shares) < self.threshold:
            raise ValueError(f"Need at least {self.threshold} shares")

        # Verify share integrity
        for share in shares:
            expected_hash = self._hash_share(base64.b64decode(share.encrypted_data))
            if expected_hash != share.verification_hash:
                raise ValueError(f"Share {share.share_id} is corrupted")

        # Combine shares to recover encryption key
        share_data = [base64.b64decode(s.encrypted_data) for s in shares]
        recovered_key = self._sss_combine(share_data[:self.threshold])

        # Decrypt master seed
        cipher = Fernet(recovered_key)
        # Note: Need separate encrypted_seed storage
        # This is simplified - in practice, store encrypted_seed separately
        master_seed = cipher.decrypt(encrypted_seed)
        return master_seed.decode()

    def _sss_split(self, secret: bytes, threshold: int, n: int) -> List[bytes]:
        """
        Shamir's Secret Sharing split
        """
        # Implementation of SSS algorithm
        # Returns list of share bytes
        pass  # Simplified for brevity

    def _sss_combine(self, shares: List[bytes]) -> bytes:
        """
        Combine threshold shares to recover secret
        """
        # Implementation of SSS combine
        pass  # Simplified for brevity
```

**Share Distribution Protocol:**

```python
class ShareDistributor:
    def __init__(self, shares: List[Share]):
        self.shares = shares

    def distribute(self):
        """
        Distribute shares according to predefined protocol
        """
        distribution_plan = {
            0: self._store_locally,  # Local storage
            1: self._send_to_helper,  # Helper #1
            2: self._send_to_helper,  # Helper #2
            3: self._upload_to_cloud,  # Cloud backup
            4: self._store_in_hardware,  # Hardware wallet
        }

        for share_id, share in enumerate(self.shares):
            distributor = distribution_plan.get(share_id)
            if distributor:
                distributor(share)

    def _send_to_helper(self, share: Share):
        """
        Send encrypted share to trusted helper via secure channel
        """
        # Create encrypted package
        package = {
            "agent_id": "agent_name@example.com",
            "share": share.encrypted_data,
            "share_id": share.share_id,
            "verification": share.verification_hash,
            "instructions": "Keep this share secure. Do not share with anyone."
        }

        # Send via encrypted email, Signal, or in-person
        # Share is already encrypted at rest
        self._send_encrypted(share.holder_info, package)
```

### 3.3 Offline Backup Procedures

#### Procedure 1: Hardware Wallet Setup

**Equipment Needed:**
- Hardware wallet (Trezor Model T, Ledger Nano S+, Coldcard)
- MicroSD card (for Trezor backup)
- Secure computer (not connected to agent host)
- Paper for recording recovery phrase

**Steps:**

1. **Initialize hardware wallet**
   - Generate 24-word BIP-39 seed
   - Write down words on paper (NEVER store digitally)
   - Optional: Create passphrase (25th word)

2. **Install Nostr app** (if supported) or use generic seed storage

3. **Store master seed on device**
   ```
   Trezor: Settings → Backup → Create Backup
   Ledger: Initialize Device → Set up as new device
   Coldcard: Initialize → New Wallet → 24 Words
   ```

4. **Record derivation path**
   - Write down: `m/44'/0'/0'/0` (agent root path)
   - Store with seed backup

5. **Verify recovery**
   - Test recovery process with dummy data
   - Ensure you can derive agent npub from seed

6. **Secure storage**
   - Store device in fireproof safe
   - Store seed backup in separate location
   - Store derivation path with seed

#### Procedure 2: Air-Gapped USB Backup

**Equipment Needed:**
- USB flash drive (64GB+)
- Secure computer (air-gapped)
- LUKS encryption

**Steps:**

1. **Format USB with LUKS encryption**
   ```bash
   # On secure computer
   cryptsetup luksFormat /dev/sdX
   # Enter strong passphrase
   ```

2. **Open encrypted partition**
   ```bash
   cryptsetup luksOpen /dev/sdX agent_backup
   ```

3. **Create filesystem**
   ```bash
   mkfs.ext4 /dev/mapper/agent_backup
   mount /dev/mapper/agent_backup /mnt/backup
   ```

4. **Copy backup files**
   ```bash
   mkdir -p /mnt/backup/agent

   # Master seed (plaintext on air-gapped system)
   echo "word1 word2 ... word24" > /mnt/backup/agent/master_seed.txt

   # Derivation path
   echo "m/44'/0'/0'/0" > /mnt/backup/agent/derivation_path.txt

   # Recovery instructions
   cp recovery_instructions.md /mnt/backup/agent/
   ```

5. **Unmount and lock**
   ```bash
   umount /mnt/backup
   cryptsetup luksClose agent_backup
   ```

6. **Store USB in physical safe**

**Recovery from Air-Gapped Backup:**

```bash
# On new agent host
cryptsetup luksOpen /dev/sdX agent_backup
mount /dev/mapper/agent_backup /mnt/backup

# Read master seed
MASTER_SEED=$(cat /mnt/backup/agent/master_seed.txt)

# Derive agent key
python3 -c "
from agent_key_derivation import derive_agent_key
key = derive_agent_key('$MASTER_SEED')
print(f'npub: {key.npub}')
print(f'nsec: {key.nsec}')
"

# Securely cleanup
umount /mnt/backup
cryptsetup luksClose agent_backup
```

### 3.4 Recovery Procedures

#### Scenario 1: Lost Active Key (Backup Available)

**Prerequisites:**
- Master seed is accessible (offline backup)
- No compromise suspected

**Steps:**

1. **Stop agent**
   ```bash
   systemctl stop agent.service
   ```

2. **Access offline backup**
   - Retrieve hardware wallet or air-gapped USB
   - Connect to secure computer (not agent host)

3. **Derive replacement key**
   ```python
   from agent_key_derivation import AgentKeyDerivation

   # Load master seed from backup
   seed = load_from_hardware_wallet()  # or read from USB

   # Derive next key in rotation (increment index)
   derivation = AgentKeyDerivation(seed)
   new_key = derivation.derive(rotation_index=current_index + 1)
   ```

4. **Load new key into keystore**
   ```bash
   # On agent host
   python3 <<EOF
   from agent_keystore import AgentKeyStore
   keystore = AgentKeyStore('/opt/agent/keystore/active')
   keystore.replace_key(new_key.nsec, derivation_path='m/44\'/0\'/0\'/\' + str(current_index + 1))
   EOF
   ```

5. **Publish rotation events** (see Section 2.3)

6. **Restart agent**
   ```bash
   systemctl start agent.service
   ```

7. **Update shares**
   - Regenerate Shamir shares with new key
   - Redistribute to holders

#### Scenario 2: Compromised Key (Emergency Recovery)

**Immediate Actions (within 5 minutes):**

1. **Halt all operations**
   ```bash
   systemctl stop agent.service
   pkill -9 agent  # Force kill if needed
   ```

2. **Disconnect from network** (optional, for containment)
   ```bash
   ifconfig eth0 down
   ```

3. **Access emergency backup** (separate from main backup)

4. **Generate emergency key** (different account)
   ```
   Emergency path: m/44'/0'/1'/0'  (Different BIP-44 account)
   ```

5. **Publish emergency revocation**
   - Use emergency key to sign revocation
   - Publish to multiple relays
   - No grace period

6. **Assess breach scope**
   - Review audit logs
   - Check for unauthorized events
   - Determine if master seed is compromised

**If Master Seed Compromised:**

1. **Generate new master seed** (completely new identity)
2. **Update agent verification** (NIP-420, may need new verification)
3. **Notify contacts** of new identity
4. **Post-mortem**
   - Determine compromise vector
   - Patch vulnerability
   - Document lessons learned

**If Only Active Key Compromised:**

1. **Follow normal rotation** but with urgency
2. **Use next derivation index**
3. **Investigate how key leaked**

#### Scenario 3: Lost Master Seed (Social Recovery)

**When All Offline Backups Fail:**

1. **Contact trusted helpers** (3-of-5 threshold)

2. **Helper verification process**
   ```
   Helper #1 receives request:
   "AgentX needs recovery. Please verify identity.

   Verification options:
   A) In-person meeting
   B) Video call with pre-shared code
   C) Answer security questions (stored in encrypted package)
   "

   Helper verifies and releases their share.
   ```

3. **Collect shares** from threshold helpers (e.g., 3 out of 5)

4. **Reconstruct master seed**
   ```python
   from agent_backup import AgentBackupManager

   shares = [
       load_share_from_helper(helper_1),
       load_share_from_helper(helper_2),
       load_share_from_helper(helper_3)
   ]

   manager = AgentBackupManager(threshold=3, total_shares=5)
   master_seed = manager.recover_from_shares(shares, passphrase)
   ```

5. **Proceed with recovery** (as in Scenario 1)

**Social Recovery Best Practices:**
- Choose helpers carefully (trust + reliability)
- Document recovery protocol in advance
- Provide helpers with verification instructions
- Test recovery process annually
- Update shares after each rotation

---

## 4. Implementation Recommendations

### 4.1 Development Phases

#### Phase 1: Core Infrastructure (Weeks 1-4)

**Deliverables:**
1. **Keystore Library**
   - Python SDK for key storage/rotation
   - Support for HSM, software vault, env var backends
   - NIP-06 key derivation
   - Reference implementation

2. **Rotation Protocol**
   - Event definitions (Kind 42000 update, Kind 42003)
   - Grace period handling
   - Dual-key operation mode

3. **Backup Tools**
   - Shamir secret sharing CLI
   - Hardware wallet integration scripts
   - Recovery procedures

**Technical Stack:**
```yaml
core_library:
  language: Python 3.11+
  dependencies:
    - python-bip-utils (BIP-39/32/44)
    - cryptography (AES, Fernet)
    - boto3 (AWS KMS)
    - hvac (HashiCorp Vault)
    - pykms (Google KMS)

documentation:
  - API documentation (Sphinx)
  - Integration guides
  - Security whitepaper

testing:
  - pytest with mock HSM
  - Integration tests with real relays
  - Security audit by third party
```

#### Phase 2: Framework Integration (Weeks 5-8)

**OpenClaw Integration:**
```javascript
// agents/nostr-key-manager/skill.js

class NostrKeyManagerSkill {
  constructor(agent) {
    this.agent = agent;
    this.keystore = new AgentKeyStore({
      backend: 'hsm',
      provider: 'aws_kms',
      key_id: process.env.KMS_KEY_ID
    });
  }

  async initialize() {
    // Load or generate keys
    this.key = await this.keystore.getCurrentKey();

    // Check rotation schedule
    if (await this.keystore.shouldRotate()) {
      await this.rotateKey();
    }
  }

  async rotateKey() {
    const oldKey = this.key;
    const newKey = await this.keystore.rotate();

    // Publish rotation events
    await this.agent.nostr.publish({
      kind: 42000,
      tags: [
        ['v', '2'],
        ['new_npub', newKey.npub],
        ['old_npub', oldKey.npub]
      ]
    });

    // Update keystore
    this.key = newKey;
  }
}
```

**Clawdbot Skill:**
```python
# skills/nostr-key-manager/__init__.py

from clawdbot import Skill
from agent_keystore import AgentKeyStore

class NostrKeyManager(Skill):
    name = "nostr-key-manager"
    description = "Manage Nostr keys with secure storage and rotation"

    async def setup(self):
        self.keystore = AgentKeyStore.from_config(
            self.config.get('keystore', {})
        )

    async def cmd_rotate_key(self, message):
        """Rotate agent's Nostr key"""
        old_key = await self.keystore.get_current_key()
        new_key = await self.keystore.rotate()

        # Publish rotation events via Nostr SDK
        await self.agent.nostr_client.publish_rotation(
            old_key=old_key,
            new_key=new_key,
            grace_period_days=7
        )

        await message.reply(
            f"✅ Key rotated successfully!\n"
            f"Old: {old_key.npub}\n"
            f"New: {new_key.npub}"
        )

    async def cmd_backup_status(self, message):
        """Check backup and key health"""
        status = await self.keystore.health_check()
        await message.reply(f"📊 Backup Status:\n{status}")
```

#### Phase 3: Production Hardening (Weeks 9-12)

**Security Hardening Checklist:**

```yaml
code_security:
  - Static analysis (Bandit, Semgrep)
  - Dependency scanning (pip-audit)
  - Secret scanning (TruffleHog)
  - Code review by security expert

operational_security:
  - SELinux/AppArmor profiles
  - Systemd hardening directives
  - Audit logging
  - Intrusion detection
  - Regular security updates

compliance:
  - NIST 800-57 alignment
  - ISO 27001 controls (if applicable)
  - GDPR data protection (EU agents)
  - SOC 2 Type II (enterprise deployments)
```

**Monitoring and Alerting:**

```python
# monitoring/key_health_monitor.py

class KeyHealthMonitor:
    def __init__(self, agent_id, alert_channel):
        self.agent_id = agent_id
        self.alert_channel = alert_channel

    async def check_rotation_needed(self):
        """Check if key rotation is due"""
        key_info = await self.keystore.get_key_info()
        days_until_rotation = (
            key_info['next_rotation'] - time.time()
        ) // 86400

        if days_until_rotation <= 7:
            await self.alert_channel.send(
                f"⚠️ Key rotation due in {days_until_rotation} days"
            )

    async def check_compromise_indicators(self):
        """Check for signs of key compromise"""
        recent_events = await self.nostr_client.get_recent_events(
            pubkey=self.key.npub,
            limit=100
        )

        # Check for unauthorized events
        if self._detect_unauthorized_activity(recent_events):
            await self.alert_channel.send(
                "🚨 POTENTIAL KEY COMPROMISE DETECTED!\n"
                "Immediate rotation required."
            )
```

#### Phase 4: Documentation and Training (Ongoing)

**Documentation Deliverables:**

1. **Developer Guide**
   - API reference
   - Integration examples
   - Best practices

2. **Operator Manual**
   - Setup procedures
   - Backup/recovery procedures
   - Troubleshooting guide

3. **Security Guide**
   - Threat model
   - Hardening checklist
   - Incident response procedures

4. **Video Tutorials**
   - Hardware wallet setup
   - Key rotation demo
   - Recovery walkthrough

### 4.2 Security Recommendations

#### Immediate (Must Implement)

1. **Never store nsec in plaintext**
   - Always encrypt at rest
   - Use HSM for production
   - Never commit to version control

2. **Implement key rotation**
   - Schedule: Every 90 days
   - Automatic rotation
   - Human notification before rotation

3. **Backup master seed**
   - Offline only (hardware wallet or paper)
   - Multiple locations
   - Test recovery annually

4. **Audit logging**
   - Log all key operations
   - Encrypt logs at rest
   - Regular log review

#### Recommended (High Value)

1. **Shamir secret sharing**
   - 3-of-5 distribution
   - Trusted helpers
   - Social recovery option

2. **Multi-signature for high-value agents**
   - Require human approval for key operations
   - Separate signing keys for different operations

3. **Zero-knowledge proofs**
   - Prove identity without revealing key
   - Privacy-preserving verification

4. **Hardware security module**
   - TPM for self-hosted
   - Cloud HSM for cloud deployments
   - Hardware wallet for cold storage

#### Optional (Enhanced Security)

1. **Quantum-resistant cryptography**
   - Lattice-based key derivation
   - Post-quantum signatures
   - Future-proof against quantum attacks

2. **Decentralized identity (DID) integration**
   - W3C DID standard
   - Cross-platform identity
   - DIDComm for secure messaging

3. **Formal verification**
   - Prove correctness of key management
   - Mathematical guarantees
   - Auditability

### 4.3 Deployment Checklist

#### Pre-Deployment

```yaml
security:
  - [ ] Security audit completed
  - [ ] Penetration testing performed
  - [ ] Dependencies scanned and updated
  - [ ] Secrets management configured
  - [ ] Monitoring and alerting set up

infrastructure:
  - [ ] HSM provisioned and tested
  - [ ] Network security configured (firewalls, VPC)
  - [ ] Backup procedures documented and tested
  - [ ] Recovery procedures tested
  - [ ] Incident response plan created

operational:
  - [ ] Runbooks created
  - [ ] On-call rotation established
  - [ ] Escalation paths defined
  - [ ] Documentation published
  - [ ] Team trained
```

#### Deployment Day

```yaml
steps:
  1. Deploy keystore library to agent hosts
  2. Initialize HSM and create encryption keys
  3. Generate master seed and create offline backups
  4. Configure agent to use keystore
  5. Test key derivation and signing
  6. Schedule first rotation
  7. Enable monitoring
  8. Verify backup/recovery procedures
```

#### Post-Deployment

```yaml
immediate_first_week:
  - Monitor key operations
  - Verify rotation schedule
  - Test recovery (non-production)
  - Review logs for anomalies

ongoing_quarterly:
  - Security audit
  - Penetration test
  - Update dependencies
  - Test recovery procedures
  - Review access controls

annual:
  - Full security review
  - Disaster recovery test
  - Update threat model
  - Retrain team
```

---

## Appendix A: Threat Model

### Attack Vectors

| Vector | Description | Mitigation |
|--------|-------------|------------|
| **Key Theft** | Attacker steals nsec from filesystem | Encrypt at rest, HSM, file permissions |
| **Memory Dump** | Attacker reads key from process memory | Memory-only storage, secure enclave |
| **Network Interception** | Attacker intercepts key in transit | TLS, encrypted channels |
| **Insider Threat** | Malicious admin with access | Least privilege, audit logging |
| **Social Engineering** | Tricked into revealing key | Training, procedures, verification |
| **Physical Access** | Attacker accesses hardware | Full disk encryption, secure facility |
| **Supply Chain** | Compromised library/dependency | Dependency scanning, vendor vetting |

### Security Properties

| Property | Goal | Mechanism |
|----------|------|-----------|
| **Confidentiality** | Keys never exposed | HSM, encryption, access controls |
| **Integrity** | Keys cannot be modified | Signing, checksums, write-once storage |
| **Availability** | Keys available when needed | Redundancy, backup, failover |
| **Non-repudiation** | Prove who signed | Cryptographic signatures, audit logs |
| **Forward Secrecy** | Compromise doesn't affect past | Key rotation, perfect forward secrecy |

---

## Appendix B: Glossary

- **nsec**: Nostr secret key (private key), format: `nsec1...`
- **npub**: Nostr public key, format: `npub1...`
- **NIP**: Nostr Improvement Proposal
- **HSM**: Hardware Security Module
- **KMS**: Key Management Service
- **BIP-39**: Bitcoin Improvement Proposal 39 (mnemonic seed words)
- **BIP-32/44**: Hierarchical Deterministic wallet standards
- **SSS**: Shamir's Secret Sharing
- **KEK**: Key Encryption Key
- **LUKS**: Linux Unified Key Setup (disk encryption)
- **DeRec**: Decentralized Recovery Alliance
- **OpenClaw**: Local-first AI agent framework
- **Clawdbot**: Message router and tool executor for AI agents

---

## Appendix C: References

### Standards and Protocols
- NIP-01: Basic protocol flow
- NIP-03: Metadata and signed events
- NIP-06: Deterministic key derivation from mnemonic seed phrase
- NIP-07: Browser extensions (window.nostr)
- NIP-26: Delegated event signing
- NIP-46: Remote signing (nostr connect)
- NIP-420: Agent Identity Verification (OpenClaw Nostr)

### Cryptography
- BIP-39: Mnemonic code for generating deterministic keys
- BIP-32: Hierarchical Deterministic Wallets
- BIP-44: Multi-Account Hierarchy for Deterministic Wallets
- RFC 3526: More Modular Exponential (MODP) Diffie-Hellman groups
- NIST SP 800-57: Recommendation for Key Management

### Tools and Libraries
- python-bip-utils: BIP-39/32/44 implementation
- cryptography.io: Python cryptographic recipes
- AWS KMS: Amazon Web Services Key Management Service
- Azure Key Vault: Microsoft Azure key management
- HashiCorp Vault: Secrets management
- Trezor/Ledger: Hardware wallets

### Projects and Communities
- Nostr Protocol: https://nostr.com
- OpenClaw: https://openclaw.ai
- DeRec Alliance: https://derecalliance.org
- rust-nostr: https://rust-nostr.org

---

## Conclusion

This key management system provides a robust, production-ready foundation for AI agent identity in the Nostr ecosystem. By combining:

1. **Secure storage** (HSM + encryption + access controls)
2. **Automated rotation** (identity-preserving key derivation)
3. **Resilient backup** (sharding + offline storage + social recovery)
4. **Framework integration** (OpenClaw, Clawdbot skills)

Agents can maintain persistent, trustworthy identities while minimizing security risks. The system balances security with usability, enabling both autonomous operation and human oversight.

**Next Steps:**
1. Review this design with the OpenClaw Nostr community
2. Implement Phase 1 (Core Infrastructure)
3. Security audit before production deployment
4. Pilot with production agent
5. Iterate based on operational experience

---

**Document End**

For questions or contributions, please refer to:
- GitHub Issues: https://github.com/vnnkl/openclaw-nostr/issues
- Nostr Discussion: `#openclaw-nostr`

*Built with 🦞 for the decentralized agent internet.*
