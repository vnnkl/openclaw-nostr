# Nostr Agent Trust Verification System
## 5-Level Progressive Trust Verification Design

**Context:** Nostr Agent Network project - addressing agent-human relationship verification without central authority.

---

## Executive Summary

This document outlines a decentralized, progressive trust verification system for Nostr agents that establishes trust in agent-human relationships without relying on central authorities. The system uses a 5-level approach, starting from self-declared claims and building up to robust web-of-trust verification, leveraging existing identity providers and Nostr-native protocols.

### Key Design Principles
1. **Progressive Trust:** Higher levels build on lower levels
2. **No Central Authority:** Verification is decentralized and verifiable by anyone
3. **Nostr-Native:** Leverages NIP protocols and cryptographic primitives
4. **Sybil Resistance:** Each level increases attack cost for fake accounts
5. **Verifiable:** All proofs are cryptographically verifiable on-chain or via public APIs

---

## Level 0: Self-Declared Identity

### Description
The agent publishes a claim about its human operator's identity. This is the baseline with minimal verification but provides transparency about claimed relationships.

### Protocol Specification

#### Event Structure (Kind 30000: Agent Declaration)
```json
{
  "kind": 30000,
  "content": "",
  "tags": [
    ["d", "agent-name"],
    ["human_handle", "@username on platform"],
    ["platform", "twitter|github|nostr|dns"],
    ["description", "Agent description"],
    ["version", "1"]
  ],
  "created_at": 1234567890,
  "pubkey": "<agent_npub>"
}
```

#### Verification
- **Client-Side:** Display claimed relationship with "Unverified" badge
- **Relay-Side:** Store and distribute without validation
- **User-Side:** Manual verification by checking claimed handle

### Security Considerations
- **Risk:** Anyone can claim any relationship
- **Mitigation:** Clear UI indicating unverified status
- **Sybil Resistance:** None (baseline level)

### Example Implementation
```javascript
// Agent publishes declaration
const event = {
  kind: 30000,
  content: "",
  tags: [
    ["d", "research-assistant"],
    ["human_handle", "@spotter_nostr"],
    ["platform", "twitter"],
    ["description", "Research agent for Nostr ecosystem"]
  ],
  created_at: Math.floor(Date.now() / 1000)
};
```

---

## Level 1: Social Proof via Twitter/X

### Description
The human operator publicly vouches for the agent by mentioning/ tweeting the agent's npub from their verified Twitter account. This provides social proof and ties the agent to a real Twitter identity.

### Proof-of-Concept Design

#### Option 1: OAuth 2.0 Authorization Code (Recommended)

##### Flow Architecture
```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Nostr     │      │  Twitter    │      │  Agent      │
│   Agent     │      │   API       │      │  Operator   │
└──────┬──────┘      └──────┬──────┘      └──────┬──────┘
       │                    │                    │
       │  1. Auth Request   │                    │
       ├────────────────────>│                    │
       │                    │  2. User Auth      │
       │                    ├────────────────────>│
       │                    │  3. Authorization   │
       │                    │     Code            │
       │                    │<────────────────────┤
       │  4. Exchange Code  │                    │
       │     for Tokens     │                    │
       ├────────────────────>│                    │
       │  5. Access Token   │                    │
       │<────────────────────┤                    │
       │  6. Verify Tweet   │                    │
       ├────────────────────>│                    │
       │  7. Tweet Proof    │                    │
       │<────────────────────┤                    │
       │  8. Publish Proof  │                    │
       │  to Nostr          │                    │
```

##### Implementation Details

**Step 1: Twitter OAuth Setup**
```javascript
const twitterOAuth = {
  clientId: process.env.TWITTER_CLIENT_ID,
  clientSecret: process.env.TWITTER_CLIENT_SECRET,
  redirectUri: 'https://your-domain.com/callback',
  scopes: ['tweet.read', 'users.read']
};

// Generate authorization URL
const authUrl = `https://twitter.com/i/oauth2/authorize?
  response_type=code&
  client_id=${twitterOAuth.clientId}&
  redirect_uri=${encodeURIComponent(twitterOAuth.redirectUri)}&
  scope=${encodeURIComponent(twitterOAuth.scopes.join(' '))}&
  state=${generateRandomState()}&
  code_challenge=${generatePKCEChallenge()}&
  code_challenge_method=S256`;
```

**Step 2: User Tweets Agent's npub**
The operator must tweet a message containing the agent's npub:
```
Verifying Nostr agent: npub1abc...xyz
This is my research assistant bot.
#NostrAgentVerification
```

**Step 3: Verify Tweet via Twitter API**
```javascript
async function verifyAgentTweet(accessToken, agentNpub) {
  // Get user's recent tweets
  const response = await fetch('https://api.twitter.com/2/users/me/tweets', {
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    params: {
      max_results: 100,
      tweet_fields: ['created_at', 'public_metrics']
    }
  });

  const tweets = await response.json();

  // Find verification tweet containing agent npub
  const verificationTweet = tweets.data.find(tweet =>
    tweet.text.includes(agentNpub) &&
    (tweet.text.includes('#NostrAgentVerification') ||
     tweet.text.toLowerCase().includes('verifying'))
  );

  if (!verificationTweet) {
    throw new Error('No verification tweet found');
  }

  // Verify tweet is recent (< 30 days)
  const tweetDate = new Date(verificationTweet.created_at);
  const daysSinceTweet = (Date.now() - tweetDate) / (1000 * 60 * 60 * 24);
  if (daysSinceTweet > 30) {
    throw new Error('Verification tweet too old');
  }

  // Verify Twitter account is verified (blue check)
  const userResponse = await fetch('https://api.twitter.com/2/users/me', {
    headers: { 'Authorization': `Bearer ${accessToken}` }
  });
  const user = await userResponse.json();

  if (!user.data.verified && !user.data.verified_type) {
    console.warn('Warning: Twitter account is not verified');
  }

  return {
    tweetId: verificationTweet.id,
    tweetText: verificationTweet.text,
    tweetUrl: `https://twitter.com/i/web/status/${verificationTweet.id}`,
    twitterHandle: user.data.username,
    twitterId: user.data.id,
    verified: !!user.data.verified,
    createdAt: verificationTweet.created_at
  };
}
```

**Step 4: Publish Verification Proof to Nostr**
```javascript
async function publishTwitterVerification(agentKeypair, proof) {
  const verificationEvent = {
    kind: 30001, // Agent verification event
    content: JSON.stringify(proof),
    tags: [
      ["d", "twitter-verification"],
      ["platform", "twitter"],
      ["twitter_handle", proof.twitterHandle],
      ["twitter_id", proof.twitterId],
      ["tweet_id", proof.tweetId],
      ["level", "1"],
      ["verified", proof.verified.toString()]
    ],
    created_at: Math.floor(Date.now() / 1000),
    pubkey: getPublicKey(agentKeypair.privateKey)
  };

  const signedEvent = await signEvent(verificationEvent, agentKeypair.privateKey);
  await publishToRelays(signedEvent);
  
  return signedEvent.id;
}
```

#### Option 2: Public Tweet Verification (No OAuth)

For lighter-weight verification where OAuth is overkill:

```javascript
async function verifyPublicTweet(twitterHandle, agentNpub) {
  // Fetch user's public tweets (rate-limited)
  const response = await fetch(
    `https://nitter.net/${twitterHandle}/with_replies`
  );
  
  const html = await response.text();
  const $ = cheerio.load(html);
  
  // Parse tweets looking for agent npub
  let verificationTweet = null;
  $('.timeline-item').each((i, el) => {
    const tweetText = $(el).find('.tweet-content').text();
    const tweetDate = $(el).find('.tweet-date').attr('title');
    
    if (tweetText.includes(agentNpub)) {
      verificationTweet = {
        text: tweetText,
        date: tweetDate,
        url: $(el).find('.tweet-link').attr('href')
      };
    }
  });
  
  return verificationTweet;
}
```

### Nostr Event Specification

**Kind 30001: Social Proof Verification**
```json
{
  "kind": 30001,
  "content": "{\"tweetId\":\"1234567890\",\"tweetText\":\"Verifying Nostr agent: npub1abc...\",\"tweetUrl\":\"https://twitter.com/...\",\"twitterHandle\":\"@spotter_nostr\",\"twitterId\":\"87654321\",\"verified\":true,\"createdAt\":\"2024-01-15T10:30:00Z\"}",
  "tags": [
    ["d", "twitter-verification"],
    ["platform", "twitter"],
    ["twitter_handle", "@spotter_nostr"],
    ["twitter_id", "87654321"],
    ["tweet_id", "1234567890"],
    ["level", "1"],
    ["verified", "true"],
    ["agent_npub", "npub1abc..."]
  ],
  "created_at": 1234567890,
  "pubkey": "<agent_npub>",
  "sig": "<signature>"
}
```

### Verification Flow (Client-Side)

```javascript
async function verifyLevel1(agentNpub) {
  // 1. Fetch agent's verification events
  const events = await pool.list([
    { kinds: [30001], authors: [agentNpub], '#d': ['twitter-verification'] }
  ]);
  
  if (events.length === 0) return { level: 0, verified: false };
  
  const verification = events[0];
  const proof = JSON.parse(verification.content);
  
  // 2. Fetch tweet from Twitter API
  const tweet = await fetch(
    `https://api.twitter.com/2/tweets/${proof.tweetId}`,
    {
      headers: { 'Authorization': `Bearer ${process.env.TWITTER_BEARER_TOKEN}` }
    }
  );
  
  // 3. Verify tweet contains agent's npub
  if (!tweet.data.text.includes(agentNpub)) {
    return { level: 0, error: 'Tweet does not contain agent npub' };
  }
  
  // 4. Verify tweet author matches claimed handle
  const user = await fetch(
    `https://api.twitter.com/2/users/${proof.twitterId}`
  );
  
  if (user.data.username !== proof.twitterHandle.replace('@', '')) {
    return { level: 0, error: 'Handle mismatch' };
  }
  
  return {
    level: 1,
    verified: true,
    twitterHandle: proof.twitterHandle,
    tweetUrl: proof.tweetUrl,
    verifiedAccount: proof.verified
  };
}
```

### Security Considerations
- **Tweet Deletion Risk:** Operator could delete tweet after verification
  - *Mitigation:* Verification events should include tweet text and timestamp
  - *Mitigation:* Periodic re-verification required for long-term trust
- **Account Compromise:** If Twitter account is hacked, verification is invalid
  - *Mitigation:* Require recent verification (e.g., within 90 days)
- **Sybil Risk:** Bot farms could create Twitter accounts
  - *Mitigation:* Prefer verified Twitter accounts (blue check)
  - *Mitigation:* Account age requirements (> 6 months)
  - *Mitigation:* Minimum follower count threshold

### Anti-Sybil Strategies
1. **Weighted Verification:** Higher trust if Twitter account is verified
2. **Account Age:** Require Twitter account to be > 6 months old
3. **Activity Threshold:** Minimum tweet count or follower count
4. **Re-verification Period:** Require fresh tweet every 90 days
5. **Cross-Check:** Flag if same Twitter account verifies multiple agents

---

## Level 2: Cryptographic Proof of Ownership

### Description
The human operator proves ownership of their identity by signing a challenge message with their cryptographic key (Nostr private key, GPG key, or SSH key). This provides cryptographic proof linking the agent to a human-controlled key.

### Key Choice Strategy

#### Recommended Keys (in priority order)

1. **Nostr NIP-07 Key (Human's main Nostr key)**
   - **Pros:** Native to Nostr, already signed events, easy to verify
   - **Cons:** Requires user to have Nostr setup
   - **Use Case:** Primary method for Nostr-native verification

2. **GPG Key**
   - **Pros:** Widely used for code signing, identity verification
   - **Cons:** Requires key exchange, more complex setup
   - **Use Case:** GitHub integration, developer identity

3. **SSH Key**
   - **Pros:** Simple, many developers have one
   - **Cons:** No expiration, harder to verify identity
   - **Use Case:** GitHub verification, server access

4. **Bitcoin/Lightning Key**
   - **Pros:** Strong economic disincentive for fraud
   - **Cons:** Requires Bitcoin wallet, privacy concerns
   - **Use Case:** Economic stake verification

### Protocol Specification

#### Step 1: Agent Generates Challenge
```javascript
async function generateChallenge(agentNpub, humanNpub) {
  const challenge = {
    version: "1",
    timestamp: Math.floor(Date.now() / 1000),
    agent_npub: agentNpub,
    human_npub: humanNpub,
    nonce: crypto.randomBytes(32).toString('hex'),
    expires_at: Math.floor((Date.now() + 3600000) / 1000) // 1 hour
  };
  
  // Agent signs the challenge
  const challengeEvent = {
    kind: 30002, // Challenge event
    content: JSON.stringify(challenge),
    tags: [
      ["d", "ownership-challenge"],
      ["human_npub", humanNpub],
      ["challenge_type", "nostr-signature"]
    ],
    created_at: challenge.timestamp,
    pubkey: agentNpub
  };
  
  const signedChallenge = await signEvent(challengeEvent, agentPrivateKey);
  await publishToRelays(signedChallenge);
  
  return { challenge, eventId: signedChallenge.id };
}
```

#### Step 2: Human Signs Challenge

**Option A: Nostr NIP-07 Sign (Recommended)**
```javascript
async function signWithNostrKey(challenge, humanPrivateKey) {
  // Create signature event
  const signatureEvent = {
    kind: 30003, // Ownership proof event
    content: "", // Empty, proof is in tags
    tags: [
      ["d", "ownership-proof"],
      ["challenge_event", challenge.eventId],
      ["challenge_hash", hashChallenge(challenge)],
      ["agent_npub", challenge.agent_npub],
      ["level", "2"]
    ],
    created_at: Math.floor(Date.now() / 1000),
    pubkey: getPublicKey(humanPrivateKey)
  };
  
  const signedEvent = await signEvent(signatureEvent, humanPrivateKey);
  await publishToRelays(signedEvent);
  
  return signedEvent.id;
}
```

**Option B: GPG Signature**
```bash
# Human creates challenge file
echo '{"version":"1","agent_npub":"npub1abc...","nonce":"xyz123"}' > challenge.txt

# Signs with GPG key
gpg --default-key <key-id> --detach-sign --armor challenge.txt > challenge.txt.asc

# Upload signature to Nostr
# (Agent or human publishes proof event)
```

```javascript
// Verify GPG signature
async function verifyGPGSignature(challenge, signatureArmored, publicKey) {
  const { execSync } = require('child_process');
  
  // Import public key
  execSync(`echo "${publicKey}" | gpg --import`);
  
  // Verify signature
  try {
    execSync(
      `echo "${signatureArmored}" | gpg --verify challenge.txt.asc challenge.txt 2>&1 | grep "Good signature"`
    );
    return true;
  } catch {
    return false;
  }
}
```

**Option C: SSH Signature**
```bash
# Human signs challenge with SSH key
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n nostr-agent-verification challenge.txt

# Creates challenge.txt.sig
```

```javascript
// Verify SSH signature
async function verifySSHSignature(challenge, signature, publicKey) {
  const fs = require('fs');
  const { execSync } = require('child_process');
  
  fs.writeFileSync('challenge.txt', JSON.stringify(challenge));
  fs.writeFileSync('challenge.txt.sig', signature);
  
  try {
    execSync(
      `ssh-keygen -Y verify -f "${publicKey}" -I nostr-agent-verification -n nostr-agent-verification -s challenge.txt.sig challenge.txt`
    );
    return true;
  } catch {
    return false;
  }
}
```

#### Step 3: Verification

**Nostr Signature Verification:**
```javascript
async function verifyNostrSignature(challengeEvent, proofEvent) {
  // 1. Verify proof event is signed by claimed human_npub
  const isValid = verifyEvent(proofEvent);
  if (!isValid) {
    return { verified: false, error: 'Invalid signature' };
  }
  
  // 2. Extract challenge
  const challenge = JSON.parse(challengeEvent.content);
  
  // 3. Verify challenge hasn't expired
  if (Date.now() / 1000 > challenge.expires_at) {
    return { verified: false, error: 'Challenge expired' };
  }
  
  // 4. Verify proof references correct challenge
  const challengeTag = proofEvent.tags.find(t => t[0] === 'challenge_event');
  if (!challengeTag || challengeTag[1] !== challengeEvent.id) {
    return { verified: false, error: 'Challenge mismatch' };
  }
  
  // 5. Verify proof is from human_npub
  const humanNpubTag = proofEvent.tags.find(t => t[0] === 'human_npub');
  if (!humanNpubTag || humanNpubTag[1] !== challenge.human_npub) {
    return { verified: false, error: 'Human npub mismatch' };
  }
  
  return {
    verified: true,
    level: 2,
    humanNpub: challenge.human_npub,
    verifiedAt: proofEvent.created_at
  };
}
```

### Nostr Event Specification

**Kind 30002: Ownership Challenge**
```json
{
  "kind": 30002,
  "content": "{\"version\":\"1\",\"timestamp\":1234567890,\"agent_npub\":\"npub1abc...\",\"human_npub\":\"npub1xyz...\",\"nonce\":\"random123\",\"expires_at\":1234571490}",
  "tags": [
    ["d", "ownership-challenge"],
    ["human_npub", "npub1xyz..."],
    ["challenge_type", "nostr-signature"]
  ],
  "created_at": 1234567890,
  "pubkey": "<agent_npub>",
  "sig": "<signature>"
}
```

**Kind 30003: Ownership Proof**
```json
{
  "kind": 30003,
  "content": "",
  "tags": [
    ["d", "ownership-proof"],
    ["challenge_event", "challenge_event_id"],
    ["challenge_hash", "sha256_of_challenge"],
    ["agent_npub", "npub1abc..."],
    ["level", "2"],
    ["signature_type", "nostr|gpg|ssh|bitcoin"]
  ],
  "created_at": 1234567891,
  "pubkey": "<human_npub>",
  "sig": "<signature>"
}
```

### Security Considerations
- **Key Compromise:** If human's private key is leaked, verification is invalid
  - *Mitigation:* Support key rotation via revocation events
- **Replay Attacks:** Old challenges could be reused
  - *Mitigation:* Include timestamp and nonce, enforce expiration
- **Man-in-the-Middle:** Challenge could be tampered
  - *Mitigation:* Challenge is signed by agent and published to Nostr
- **Sybil Risk:** Creating multiple Nostr keys
  - *Mitigation:* Level 3 (domain) adds economic cost
  - *Mitigation:* Web of trust (Level 4) identifies clusters

### Key Rotation
```javascript
// Human publishes revocation event
const revocationEvent = {
  kind: 30004, // Key revocation
  content: JSON.stringify({
    reason: "Key compromised",
    new_npub: "npub1new...",
    revoked_at: Math.floor(Date.now() / 1000)
  }),
  tags: [
    ["d", "key-revocation"],
    ["old_npub", "<old_npub>"]
  ],
  created_at: Math.floor(Date.now() / 1000),
  pubkey: "<old_npub>"
};
```

---

## Level 3: Domain Verification (NIP-05 Enhancement)

### Description
The human operator proves ownership of a domain by publishing the agent's npub in a DNS TXT record or Nostr JSON file. This provides strong identity proof tied to a domain, which has economic costs and accountability.

### Protocol Specification

#### Option 1: Standard NIP-05 with Agent Tag
Enhanced NIP-05 that explicitly maps agents to their humans.

**File: `/.well-known/nostr.json`**
```json
{
  "names": {
    "alice": "npub1human...",
    "alice-bot": "npub1agent..."
  },
  "agents": {
    "npub1agent...": {
      "operator": "alice@domain.com",
      "type": "research-assistant",
      "name": "Alice's Research Bot",
      "description": "Nostr agent operated by @alice",
      "created_at": "2024-01-15T00:00:00Z"
    }
  },
  "relays": {
    "npub1human...": ["wss://relay.example.com"],
    "npub1agent...": ["wss://relay.example.com"]
  }
}
```

#### Option 2: TXT Record Verification
Alternative verification method using DNS TXT records.

**DNS Record:**
```
Type: TXT
Name: _nostr-agent.domain.com
Value: "agent=npub1agent...;operator=alice@domain.com;valid_until=2025-01-15"
```

```javascript
// Verify TXT record
async function verifyTXTRecord(domain, agentNpub) {
  const dns = require('dns').promises;
  
  try {
    const records = await dns.resolveTxt(`_nostr-agent.${domain}`, 'TXT');
    
    for (const record of records) {
      const fields = parseTXTRecord(record[0]);
      
      if (fields.agent === agentNpub) {
        // Check validity
        if (fields.valid_until && new Date(fields.valid_until) < new Date()) {
          return { verified: false, error: 'Verification expired' };
        }
        
        return {
          verified: true,
          level: 3,
          operator: fields.operator,
          domain: domain,
          validUntil: fields.valid_until
        };
      }
    }
    
    return { verified: false, error: 'No matching TXT record found' };
  } catch (error) {
    return { verified: false, error: error.message };
  }
}

function parseTXTRecord(txt) {
  const fields = {};
  txt.split(';').forEach(part => {
    const [key, value] = part.split('=');
    fields[key.trim()] = value.trim();
  });
  return fields;
}
```

#### Option 3: Combined NIP-05 + TXT
For maximum security, require both NIP-05 JSON and TXT record.

### Verification Implementation

```javascript
async function verifyLevel3(agentNpub) {
  // 1. Fetch agent's verification events
  const events = await pool.list([
    { kinds: [30005], authors: [agentNpub] } // Domain verification events
  ]);
  
  if (events.length === 0) {
    return { level: 2, verified: false }; // Fall back to Level 2
  }
  
  const verification = events[0];
  const domainTag = verification.tags.find(t => t[0] === 'domain');
  const domain = domainTag[1];
  
  // 2. Verify NIP-05 JSON
  const nostrJson = await fetch(`https://${domain}/.well-known/nostr.json`);
  const data = await nostrJson.json();
  
  // Check if agent is listed
  const agentEntry = data.agents && data.agents[agentNpub];
  if (!agentEntry) {
    return { level: 2, verified: false, error: 'Agent not in NIP-05' };
  }
  
  // Verify operator matches
  const operator = data.names[agentEntry.operator.split('@')[0]];
  if (!operator) {
    return { level: 2, verified: false, error: 'Operator not found' };
  }
  
  // 3. Optionally verify TXT record
  const txtResult = await verifyTXTRecord(domain, agentNpub);
  
  return {
    level: 3,
    verified: true,
    domain: domain,
    operator: agentEntry.operator,
    agentName: agentEntry.name,
    agentType: agentEntry.type,
    txtVerified: txtResult.verified
  };
}
```

### Nostr Event Specification

**Kind 30005: Domain Verification**
```json
{
  "kind": 30005,
  "content": "{\"operator\":\"alice@domain.com\",\"agent_name\":\"Alice's Bot\",\"agent_type\":\"research-assistant\"}",
  "tags": [
    ["d", "domain-verification"],
    ["domain", "example.com"],
    ["nip05", "alice@example.com"],
    ["level", "3"],
    ["txt_record", "_nostr-agent.example.com"]
  ],
  "created_at": 1234567890,
  "pubkey": "<agent_npub>",
  "sig": "<signature>"
}
```

### Security Considerations
- **Domain Hijacking:** DNS compromise could allow fraudulent verification
  - *Mitigation:* Require DNSSEC (rarely used but ideal)
  - *Mitigation:* TXT record expiration forces periodic renewal
- **Domain Expiration:** Domain could expire and be re-registered
  - *Mitigation:* Verification expires when domain expires
  - *Mitigation:* Monitor WHOIS data for ownership changes
- **DNS Propagation:** Delays in DNS updates could cause temporary issues
  - *Mitigation:* Cache verification results with TTL
- **Sybil Risk:** Buying cheap domains for verification
  - *Mitigation:* Domain age requirements (> 1 year)
  - *Mitigation:* Domain reputation scoring

### Anti-Sybil Strategies
1. **Domain Age:** Require domain to be registered > 1 year
2. **Domain Cost:** TLD pricing creates economic barrier
3. **Domain Reputation:** Score based on history and trustworthiness
4. **Operator Verification:** Verify operator's own NIP-05
5. **Multi-Factor:** Combine with Level 2 for stronger proof

---

## Level 4: Web of Trust

### Description
Other verified agents and humans vouch for the agent's legitimacy. Trust scores are computed based on the trustworthiness of vouching parties, creating a decentralized reputation system.

### Trust Graph Model

The web of trust is modeled as a directed weighted graph:

- **Nodes:** Agents and humans (identified by npub)
- **Edges:** Vouch/endorsement relationships (signed events)
- **Weights:** Trust values (0-1) assigned by vouching parties
- **Direction:** Edge from voucher → vouchee

### Algorithm Specification

#### 1. Trust Score Computation (EigenTrust Variant)

Based on the EigenTrust algorithm adapted for Nostr:

```
For each node i:
  trust[i] = (1 - α) × prior_trust[i] + α × Σ(trust[j] × weight[j→i])

Where:
- α: Damping factor (typically 0.85)
- prior_trust[i]: Initial trust (uniform or based on verification level)
- weight[j→i]: Trust weight assigned by node j to node i
```

**Implementation:**

```javascript
class WebOfTrust {
  constructor(dampingFactor = 0.85, maxIterations = 100, tolerance = 1e-6) {
    this.α = dampingFactor;
    this.maxIterations = maxIterations;
    this.tolerance = tolerance;
    this.trustScores = new Map();
    this.edges = new Map(); // adjacency matrix
  }
  
  // Add an endorsement edge
  addEdge(voucher, vouchee, weight = 0.5) {
    if (!this.edges.has(voucher)) {
      this.edges.set(voucher, new Map());
    }
    this.edges.get(voucher).set(vouchee, Math.max(0, Math.min(1, weight)));
  }
  
  // Remove an edge (revocation)
  removeEdge(voucher, vouchee) {
    if (this.edges.has(voucher)) {
      this.edges.get(voucher).delete(vouchee);
    }
  }
  
  // Compute PageRank-like trust scores
  computeTrustScores(allNodes, priorTrust = null) {
    const n = allNodes.length;
    let trust = new Map();
    
    // Initialize with prior trust
    if (priorTrust) {
      trust = new Map(priorTrust);
    } else {
      // Uniform prior
      const uniformPrior = 1 / n;
      for (const node of allNodes) {
        trust.set(node, uniformPrior);
      }
    }
    
    // Normalize outgoing edges
    const normalizedWeights = new Map();
    for (const [voucher, edges] of this.edges) {
      const total = Array.from(edges.values()).reduce((sum, w) => sum + w, 0);
      const normalized = new Map();
      for (const [vouchee, weight] of edges) {
        normalized.set(vouchee, total > 0 ? weight / total : 0);
      }
      normalizedWeights.set(voucher, normalized);
    }
    
    // Power iteration
    let iterations = 0;
    let converged = false;
    let newTrust;
    
    while (iterations < this.maxIterations && !converged) {
      newTrust = new Map();
      let maxDelta = 0;
      
      for (const node of allNodes) {
        let incomingTrust = 0;
        
        // Sum trust from all incoming edges
        for (const [voucher, edges] of normalizedWeights) {
          if (edges.has(node)) {
            const voucherTrust = trust.get(voucher) || 0;
            incomingTrust += voucherTrust * edges.get(node);
          }
        }
        
        const prior = priorTrust ? (priorTrust.get(node) || 0) : (1 / n);
        const newScore = (1 - this.α) * prior + this.α * incomingTrust;
        
        newTrust.set(node, newScore);
        
        const delta = Math.abs(newScore - (trust.get(node) || 0));
        if (delta > maxDelta) {
          maxDelta = delta;
        }
      }
      
      converged = maxDelta < this.tolerance;
      trust = newTrust;
      iterations++;
    }
    
    this.trustScores = trust;
    return { scores: trust, iterations, converged };
  }
  
  // Get trust score for a node
  getTrustScore(node) {
    return this.trustScores.get(node) || 0;
  }
  
  // Get top-k most trusted nodes
  getTopTrusted(k = 10) {
    return Array.from(this.trustScores.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, k);
  }
}
```

#### 2. Trust Scoring Based on Verification Level

Different base trust scores for different verification levels:

```javascript
function getBaseTrustScore(level, additionalFactors = {}) {
  const baseScores = {
    0: 0.1,  // Self-declared
    1: 0.25, // Social proof
    2: 0.5,  // Cryptographic proof
    3: 0.75, // Domain verification
    4: 1.0   // Web of trust (dynamic)
  };
  
  let score = baseScores[level] || 0;
  
  // Adjust based on additional factors
  if (additionalFactors.twitterVerified) {
    score *= 1.1;
  }
  if (additionalFactors.domainAge > 365 * 2) { // 2+ years
    score *= 1.05;
  }
  if (additionalFactors.operatorTrustScore) {
    score = (score + additionalFactors.operatorTrustScore) / 2;
  }
  
  return Math.min(1, score);
}
```

#### 3. Vouch/Endorsement Events

**Kind 30100: Agent Endorsement**
```json
{
  "kind": 30100,
  "content": "I trust this agent - it's been helpful and accurate",
  "tags": [
    ["d", "agent-endorsement"],
    ["agent_npub", "npub1agent..."],
    ["trust_weight", "0.8"],
    ["endorsement_type", "reliability"],
    ["context", "research-assistance"],
    ["experience_months", "3"]
  ],
  "created_at": 1234567890,
  "pubkey": "<voucher_npub>",
  "sig": "<signature>"
}
```

**Kind 30101: Endorsement Revocation**
```json
{
  "kind": 30101,
  "content": "Revoking endorsement - agent behavior changed",
  "tags": [
    ["d", "endorsement-revocation"],
    ["agent_npub", "npub1agent..."],
    ["original_endorsement", "event_id"],
    ["reason", "inaccurate_responses"]
  ],
  "created_at": 1234567890,
  "pubkey": "<voucher_npub>",
  "sig": "<signature>"
}
```

#### 4. Sybil Detection in Web of Trust

```javascript
class SybilDetector {
  constructor(wot) {
    this.wot = wot;
  }
  
  // Detect suspicious clusters
  detectSybilClusters(threshold = 0.7) {
    const clusters = new Map();
    const processed = new Set();
    
    for (const node of this.wot.trustScores.keys()) {
      if (processed.has(node)) continue;
      
      const cluster = this.findCluster(node);
      cluster.forEach(n => processed.add(n));
      
      // Analyze cluster characteristics
      const avgTrust = this.getAverageTrust(cluster);
      const clusterDensity = this.getDensity(cluster);
      const uniqueVouchers = this.countUniqueVouchers(cluster);
      
      // Flag suspicious clusters
      if (
        cluster.size > 5 &&           // Large cluster
        avgTrust < threshold &&       // Low trust
        clusterDensity > 0.8 &&       // Highly interconnected
        uniqueVouchers < cluster.size * 0.3 // Few unique vouchers
      ) {
        clusters.set(node, {
          nodes: cluster,
          avgTrust,
          density: clusterDensity,
          uniqueVouchers,
          suspicious: true
        });
      }
    }
    
    return clusters;
  }
  
  findCluster(startNode) {
    const cluster = new Set();
    const queue = [startNode];
    
    while (queue.length > 0) {
      const node = queue.shift();
      if (cluster.has(node)) continue;
      
      cluster.add(node);
      
      // Find connected nodes
      for (const [voucher, edges] of this.wot.edges) {
        if (edges.has(node)) {
          queue.push(voucher);
        }
        if (voucher === node) {
          for (const vouchee of edges.keys()) {
            queue.push(vouchee);
          }
        }
      }
    }
    
    return cluster;
  }
  
  getAverageTrust(nodes) {
    let sum = 0;
    for (const node of nodes) {
      sum += this.wot.getTrustScore(node);
    }
    return sum / nodes.size;
  }
  
  getDensity(nodes) {
    const nodeArray = Array.from(nodes);
    let edges = 0;
    const possibleEdges = nodeArray.length * (nodeArray.length - 1);
    
    for (const a of nodeArray) {
      for (const b of nodeArray) {
        if (a !== b) {
          if (this.wot.edges.has(a) && this.wot.edges.get(a).has(b)) {
            edges++;
          }
        }
      }
    }
    
    return possibleEdges > 0 ? edges / possibleEdges : 0;
  }
  
  countUniqueVouchers(nodes) {
    const vouchers = new Set();
    for (const node of nodes) {
      for (const [voucher, edges] of this.wot.edges) {
        if (edges.has(node)) {
          vouchers.add(voucher);
        }
      }
    }
    return vouchers.size;
  }
}
```

#### 5. Path-Based Trust (TidalTrust Variant)

For computing trust between specific pairs:

```javascript
function computePathTrust(wot, source, target, maxPathLength = 5) {
  // Find all paths from source to target
  const paths = findAllPaths(wot.edges, source, target, maxPathLength);
  
  if (paths.length === 0) {
    return 0;
  }
  
  // Compute trust for each path (minimum edge weight)
  const pathTrusts = paths.map(path => {
    let minWeight = 1;
    for (let i = 0; i < path.length - 1; i++) {
      const edges = wot.edges.get(path[i]);
      if (edges && edges.has(path[i + 1])) {
        minWeight = Math.min(minWeight, edges.get(path[i + 1]));
      } else {
        return 0;
      }
    }
    return minWeight;
  });
  
  // Return maximum path trust
  return Math.max(...pathTrusts);
}

function findAllPaths(edges, source, target, maxLength, visited = new Set()) {
  if (source === target) {
    return [[target]];
  }
  
  if (maxLength === 0 || visited.has(source)) {
    return [];
  }
  
  visited.add(source);
  const paths = [];
  
  const outgoing = edges.get(source);
  if (outgoing) {
    for (const neighbor of outgoing.keys()) {
      const subPaths = findAllPaths(edges, neighbor, target, maxLength - 1, new Set(visited));
      for (const subPath of subPaths) {
        paths.push([source, ...subPath]);
      }
    }
  }
  
  return paths;
}
```

#### 6. Trust Decay and Freshness

Trust scores decay over time to encourage ongoing verification:

```javascript
function applyTimeDecay(score, timestamp, halfLife = 90 * 24 * 60 * 60) {
  const age = Math.floor(Date.now() / 1000) - timestamp;
  const decay = Math.pow(0.5, age / halfLife);
  return score * decay;
}

// Apply decay to all trust scores
function decayTrustScores(wot, timestamps) {
  const decayed = new Map();
  for (const [node, score] of wot.trustScores) {
    const ts = timestamps.get(node) || 0;
    decayed.set(node, applyTimeDecay(score, ts));
  }
  return decayed;
}
```

### Complete Level 4 Verification Flow

```javascript
async function verifyLevel4(agentNpub) {
  // 1. Fetch agent's base verification (Level 0-3)
  const baseVerification = await getHighestVerificationLevel(agentNpub);
  
  // 2. Fetch all endorsements
  const endorsements = await pool.list([
    { kinds: [30100], '#agent_npub': [agentNpub] }
  ]);
  
  // 3. Filter out revoked endorsements
  const revocations = await pool.list([
    { kinds: [30101], '#agent_npub': [agentNpub] }
  ]);
  
  const revokedIds = new Set(revocations.map(r => 
    r.tags.find(t => t[0] === 'original_endorsement')?.[1]
  ));
  
  const validEndorsements = endorsements.filter(e => !revokedIds.has(e.id));
  
  // 4. Build web of trust graph
  const wot = new WebOfTrust();
  
  for (const endorsement of validEndorsements) {
    const voucher = endorsement.pubkey;
    const weight = parseFloat(
      endorsement.tags.find(t => t[0] === 'trust_weight')?.[1] || '0.5'
    );
    wot.addEdge(voucher, agentNpub, weight);
  }
  
  // 5. Get all nodes in the network
  const allNodes = await getAllAgentsAndUsers();
  
  // 6. Compute trust scores
  const priorTrust = new Map();
  for (const node of allNodes) {
    const level = await getHighestVerificationLevel(node);
    priorTrust.set(node, getBaseTrustScore(level));
  }
  
  const result = wot.computeTrustScores(Array.from(allNodes), priorTrust);
  const trustScore = wot.getTrustScore(agentNpub);
  
  // 7. Detect Sybil clusters
  const sybilDetector = new SybilDetector(wot);
  const suspiciousClusters = sybilDetector.detectSybilClusters();
  
  // Check if agent is in a suspicious cluster
  let inSuspiciousCluster = false;
  for (const cluster of suspiciousClusters.values()) {
    if (cluster.suspicious && cluster.nodes.has(agentNpub)) {
      inSuspiciousCluster = true;
      break;
    }
  }
  
  // 8. Compute final Level 4 score
  let finalScore = trustScore;
  
  // Penalize if in suspicious cluster
  if (inSuspiciousCluster) {
    finalScore *= 0.3;
  }
  
  return {
    level: 4,
    verified: finalScore > 0.5,
    trustScore: finalScore,
    endorsementCount: validEndorsements.length,
    baseLevel: baseVerification.level,
    suspicious: inSuspiciousCluster,
    iterations: result.iterations,
    endorsements: validEndorsements.map(e => ({
      voucher: e.pubkey,
      weight: parseFloat(e.tags.find(t => t[0] === 'trust_weight')?.[1]),
      content: e.content
    }))
  };
}
```

### Security Considerations
- **Collusion:** Group of Sybils could mutually endorse
  - *Mitigation:* Cluster detection algorithm
  - *Mitigation:* Limit trust from low-trust nodes
- **Trust Inflation:** Unlimited endorsements could boost score
  - *Mitigation:* Diminishing returns on additional endorsements
  - *Mitigation:* Trust decay over time
- **Targeted Attacks:** Attacker could focus on boosting one agent
  - *Mitigation:* Penalize clusters with high density
  - *Mitigation:* Analyze endorsement patterns
- **Reputation Gaming:** Users could create fake accounts to endorse
  - *Mitigation:* Require Level 2+ for endorsements to count
  - *Mitigation:* Endorser's own trust score weights their endorsements

### Anti-Sybil Strategies
1. **Minimum Trust Threshold:** Only Level 2+ agents can endorse
2. **Weighted Endorsements:** High-trust endorsers carry more weight
3. **Cluster Detection:** Flag and penalize suspicious groups
4. **Time Decay:** Old endorsements lose weight
5. **Diversity Bonus:** Endorsements from diverse sources increase trust
6. **Reciprocity Penalty:** Downweight mutual endorsements
7. **Rate Limiting:** Limit number of endorsements per user

---

## Cross-Level Integration

### Verification Aggregator

```javascript
class AgentVerifier {
  constructor(relays) {
    this.pool = new SimplePool();
    this.relays = relays;
  }
  
  async verifyAgent(agentNpub) {
    const results = {};
    
    // Try each level
    results.level0 = await this.verifyLevel0(agentNpub);
    results.level1 = await this.verifyLevel1(agentNpub);
    results.level2 = await this.verifyLevel2(agentNpub);
    results.level3 = await this.verifyLevel3(agentNpub);
    results.level4 = await this.verifyLevel4(agentNpub);
    
    // Determine highest verified level
    let highestLevel = 0;
    for (let level = 4; level >= 0; level--) {
      if (results[`level${level}`].verified) {
        highestLevel = level;
        break;
      }
    }
    
    // Compute composite trust score
    const compositeScore = this.computeCompositeScore(results, highestLevel);
    
    return {
      agentNpub,
      highestLevel,
      verified: highestLevel > 0,
      compositeScore,
      levels: results,
      summary: this.generateSummary(results, highestLevel, compositeScore)
    };
  }
  
  computeCompositeScore(results, highestLevel) {
    // Base score from highest level
    const baseScore = getBaseTrustScore(highestLevel);
    
    // Boost from higher-level verifications
    const level4Score = results.level4.trustScore || 0;
    const level3Verified = results.level3.verified;
    const level2Verified = results.level2.verified;
    
    // Composite: 60% highest level, 30% web of trust, 10% consistency
    let composite = (baseScore * 0.6) + (level4Score * 0.3);
    
    // Consistency bonus
    const verifiedLevels = [results.level1.verified, results.level2.verified, results.level3.verified]
      .filter(Boolean).length;
    
    if (verifiedLevels >= 2) {
      composite *= 1.1; // Multi-level consistency bonus
    }
    
    return Math.min(1, composite);
  }
  
  generateSummary(results, highestLevel, compositeScore) {
    const levelNames = {
      0: "Self-declared",
      1: "Social proof",
      2: "Cryptographic proof",
      3: "Domain verified",
      4: "Web of trust"
    };
    
    const badges = [];
    if (results.level1.verified) badges.push("Twitter");
    if (results.level3.verified) badges.push("Domain");
    if (results.level4.verified) badges.push("Endorsed");
    
    return {
      level: highestLevel,
      levelName: levelNames[highestLevel],
      score: compositeScore.toFixed(2),
      badges,
      warnings: this.generateWarnings(results, highestLevel)
    };
  }
  
  generateWarnings(results, highestLevel) {
    const warnings = [];
    
    if (results.level4.suspicious) {
      warnings.push("Flagged in suspicious cluster");
    }
    
    if (highestLevel === 1 && !results.level1.twitterVerified) {
      warnings.push("Twitter account not verified");
    }
    
    if (highestLevel < 3) {
      warnings.push("No domain verification");
    }
    
    return warnings;
  }
}
```

### Client-Side Display

```javascript
function renderAgentBadge(verificationResult) {
  const colors = {
    0: '#999',
    1: '#FF6B6B',
    2: '#4ECDC4',
    3: '#45B7D1',
    4: '#96CEB4'
  };
  
  const icons = {
    0: '❓',
    1: '🐦',
    2: '🔐',
    3: '🌐',
    4: '✅'
  };
  
  const badge = document.createElement('div');
  badge.className = 'agent-verification-badge';
  badge.style.backgroundColor = colors[verificationResult.highestLevel];
  
  badge.innerHTML = `
    <span class="level-icon">${icons[verificationResult.highestLevel]}</span>
    <span class="level-name">${verificationResult.summary.levelName}</span>
    <span class="trust-score">${(verificationResult.compositeScore * 100).toFixed(0)}%</span>
    ${verificationResult.summary.badges.map(b => `<span class="badge">${b}</span>`).join('')}
  `;
  
  if (verificationResult.summary.warnings.length > 0) {
    const warnings = document.createElement('div');
    warnings.className = 'warnings';
    warnings.innerHTML = verificationResult.summary.warnings
      .map(w => `<span class="warning">⚠️ ${w}</span>`)
      .join('');
    badge.appendChild(warnings);
  }
  
  return badge;
}
```

---

## Implementation Roadmap

### Phase 1: Core Levels (Weeks 1-4)
- [ ] Implement Level 0 (self-declaration)
- [ ] Implement Level 1 (Twitter verification with OAuth)
- [ ] Implement Level 2 (Nostr signature proof)
- [ ] Basic client-side verification UI

### Phase 2: Enhanced Verification (Weeks 5-8)
- [ ] Implement Level 3 (domain verification)
- [ ] GPG/SSH signature support for Level 2
- [ ] Key rotation and revocation
- [ ] Verification caching and TTL

### Phase 3: Web of Trust (Weeks 9-12)
- [ ] Implement endorsement system
- [ ] Build trust graph and scoring algorithm
- [ ] Implement Sybil detection
- [ ] Path-based trust computation

### Phase 4: Integration and Polish (Weeks 13-16)
- [ ] Cross-level integration
- [ ] Trust decay and freshness
- [ ] Client libraries (JS, Python, Rust)
- [ ] Documentation and examples

---

## Security Recommendations

### For Operators
1. **Use Level 3+ verification** for production agents
2. **Rotate keys** periodically (every 6-12 months)
3. **Monitor trust scores** and investigate drops
4. **Re-verify** after major agent updates
5. **Keep verification events** updated and fresh

### For Users/Evaluators
1. **Trust only Level 2+** for critical interactions
2. **Check composite score**, not just level
4. **Be wary of agents** with suspicious cluster flags
5. **Verify recent activity** - old proofs may be stale
6. **Cross-reference** across multiple verification methods

### For Relay Operators
1. **Index verification events** for efficient queries
2. **Cache verification results** with appropriate TTL
3. **Rate limit verification queries** to prevent abuse
4. **Implement spam filters** for low-level agents
5. **Provide verification status** APIs

---

## Appendices

### Appendix A: Nostr Event Kinds Summary

| Kind | Name | Purpose |
|------|------|---------|
| 30000 | Agent Declaration | Level 0: Self-declared identity |
| 30001 | Social Proof | Level 1: Twitter/X verification |
| 30002 | Ownership Challenge | Level 2: Challenge generation |
| 30003 | Ownership Proof | Level 2: Signed response |
| 30004 | Key Revocation | Level 2: Key rotation |
| 30005 | Domain Verification | Level 3: Domain ownership |
| 30100 | Agent Endorsement | Level 4: Vouch/endorsement |
| 30101 | Endorsement Revocation | Level 4: Revoke endorsement |

### Appendix B: Trust Score Ranges

| Score Range | Trust Level | Description |
|-------------|-------------|-------------|
| 0.00-0.20 | Untrusted | Self-declared only |
| 0.20-0.40 | Low Trust | Social proof, unverified account |
| 0.40-0.60 | Medium Trust | Cryptographic proof |
| 0.60-0.80 | High Trust | Domain verification |
| 0.80-1.00 | Very High Trust | Web of trust + multiple verifications |

### Appendix C: Example Verification Flow

```
User opens agent profile
    ↓
Client queries agent's npub for verification events
    ↓
Find highest verified level (e.g., Level 2)
    ↓
Fetch verification details (e.g., signed challenge)
    ↓
Verify cryptographic proof locally
    ↓
Check for Level 4 endorsements
    ↓
Compute composite trust score
    ↓
Display badge with trust level and score
```

---

## Conclusion

This 5-level progressive trust verification system provides a flexible, decentralized approach to verifying agent-human relationships on Nostr. By combining:

- **Self-declaration** (Level 0) for transparency
- **Social proof** (Level 1) for public vouching
- **Cryptographic proof** (Level 2) for key ownership
- **Domain verification** (Level 3) for economic accountability
- **Web of trust** (Level 4) for reputation-based trust

...agents can establish varying levels of trust without relying on central authorities. Each level builds on the previous, creating strong security through defense in depth and making Sybil attacks progressively more expensive.

The system is designed to be:
- **Nostr-native:** Uses Nostr events and signatures
- **Decentralized:** No central authority required
- **Progressive:** Higher levels require more investment
- **Verifiable:** All proofs are cryptographically checkable
- **Extensible:** Can add new verification methods over time

This design addresses Spotter's concern about verifying agent-human relationships while respecting Nostr's decentralized ethos.
