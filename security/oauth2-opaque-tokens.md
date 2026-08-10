# Technical Guide: Sensitive Parameter Encryption & OAuth 2.0 Opaque Token Implementation

* **Target Software:** VMware Tanzu RabbitMQ (v4.2+ / v4.3)
* **Deployment Models:** Bare-Metal, RPM, OVA (Systemd Services)
* **Document Classification:** Production Operational Guide

---

## Executive Summary

Storing sensitive credentials in plain-text configuration files violates enterprise compliance standards. Furthermore, modern Identity Providers (IdPs) frequently issue Opaque Tokens instead of structured JSON Web Tokens (JWTs).

This document outlines the architecture and deployment steps required to:
1. Encrypt sensitive parameters in `/etc/rabbitmq/rabbitmq.conf` using RabbitMQ's native `config_entry_decoder`.
2. Configure active RFC 7662 OAuth 2.0 Token Introspection to support Opaque Tokens on virtual machine and bare-metal environments.

---

## Section 1: Configuration Parameter Encryption (config_entry_decoder)

VMware Tanzu RabbitMQ features a built-in configuration entry decoder that decrypts ciphertext values in memory during process startup. This ensures plain-text secrets never reside on disk inside `/etc/rabbitmq/rabbitmq.conf`.

### 1. Encryption Workflow

[ Plain-Text Secret ] 
        │
        ▼  (rabbitmqctl encode)
[ Encrypted Ciphertext ] ──► Written to /etc/rabbitmq/rabbitmq.conf
        │
        ▼  (Passphrase injected via Systemd / rabbitmq-env.conf)
[ In-Memory Decryption ] ──► Loaded directly into Erlang VM

### 2. Step-by-Step Implementation

#### Step 1: Encrypt the Plain-Text Values
Use `rabbitmqctl encode` to generate the encrypted payload. You must provide the plain-text secret and the master decryption passphrase you intend to use. Run this for every sensitive credential:

    # Encrypt Introspection Client Secret
    rabbitmqctl encode 'MySuperSecretClientPassword123!' 'YourMasterDecryptionPassphrase'
    
    # Encrypt Management UI Symmetric Key
    rabbitmqctl encode 'SecureHS256SymmetricSigningKey2026#' 'YourMasterDecryptionPassphrase'

**Example Output:**
    
    {encrypted, <<"cPAymwqmMnbPXXRVqVzpxJdrS8mHEKuo2V+3vt1u/fymexD9oztQ2G/oJ4PAaSb2c5N">>}

When moving this to `rabbitmq.conf`, drop the Erlang `{encrypted, <<"...">>}` syntax and prepend `encrypted:` to the raw base64 string:

    encrypted:cPAymwqmMnbPXXRVqVzpxJdrS8mHEKuo2V+3vt1u/fymexD9oztQ2G/oJ4PAaSb2c5N

#### Step 2: Inject the Decryption Passphrase
The Erlang VM requires the matching passphrase at boot to decrypt the entries. Store this passphrase securely.

**Option A: Systemd Override (Recommended)**
Create an environment override for the RabbitMQ service:

    systemctl edit rabbitmq-server
    # OR for Bitnami OVA installations:
    systemctl edit bitnami.vmware-rabbitmq

Add the decryption passphrase as an environment variable:

    [Service]
    Environment="RABBITMQ_CONFIG_ENTRY_DECODER_PASSPHRASE=YourMasterDecryptionPassphrase"

**Option B: rabbitmq-env.conf**
Alternatively, append the variable to `/etc/rabbitmq/rabbitmq-env.conf`:

    RABBITMQ_CONFIG_ENTRY_DECODER_PASSPHRASE="YourMasterDecryptionPassphrase"

Ensure proper permissions are set:

    chmod 600 /etc/rabbitmq/rabbitmq-env.conf
    chown rabbitmq:rabbitmq /etc/rabbitmq/rabbitmq-env.conf

---

## Section 2: OAuth 2.0 Opaque Token Architecture (RFC 7662)

Standard Open Source RabbitMQ (OSS) supports JWTs only, which are validated offline via public key signatures. Commercial VMware Tanzu RabbitMQ adds active token introspection, allowing the broker to authenticate Opaque Tokens.

### 1. Technical Comparison: JWT vs. Opaque Tokens

| Feature | JWT (JSON Web Token) | Opaque Token (RFC 7662) |
| :--- | :--- | :--- |
| **Token Format** | Base64-encoded structured JSON | Random, unstructured string / UUID |
| **Validation Method** | Offline (Public Key / JWKS signature) | Active (HTTPS call to IdP /introspect) |
| **Network Traffic** | Zero outbound calls to IdP | 1 outbound HTTPS call per connection handshake |
| **Revocation** | Dependent on short TTLs / blacklists | Instant (IdP returns active: false) |
| **Licensing** | OSS & Tanzu RabbitMQ | VMware Tanzu RabbitMQ Exclusive |

### 2. How Active Introspection Works

When an AMQP client or web browser presents an Opaque Token:
1. **Handshake Received:** RabbitMQ intercepts the opaque access token.
2. **Introspection Query:** RabbitMQ issues an authenticated HTTPS POST request to the IdP's `introspection_endpoint`.
3. **Payload Processing:** The IdP evaluates the token and returns a JSON response containing `active: true/false`, `scope`, `sub`, and `exp`.
4. **Authorization:** RabbitMQ maps the returned scope claims to local virtual host permissions.

### 3. Management Web UI Requirements

The RabbitMQ Management Web UI requires a JWT structure to maintain stateless browser sessions. When opaque tokens are used, RabbitMQ generates an in-memory session token for the user browser.

To support this, you must configure `auth_oauth2.opaque_token_signing_key.key`. Without this symmetric key, opaque tokens will authenticate AMQP/MQTT protocol traffic, but logging into the Management Web UI will fail.

---

## Section 3: Unified Production Configuration

Below is a complete, hardened `/etc/rabbitmq/rabbitmq.conf` combining active RFC 7662 Opaque Token Introspection with encrypted credentials.

    # ==============================================================================
    # 1. AUTHENTICATION BACKEND SELECTION
    # ==============================================================================
    # Enables OAuth 2.0 as primary authentication with local Mnesia fallback
    auth_backends.1 = oauth2
    auth_backends.2 = internal
    
    # The Resource Server ID expected in token audience/scope claims
    auth_oauth2.resource_server_id = rabbitmq
    
    # ==============================================================================
    # 2. OPAQUE TOKEN INTROSPECTION CONFIGURATION (RFC 7662)
    # ==============================================================================
    # Root Identity Provider Issuer URL (for OpenID Discovery)
    auth_oauth2.issuer = https://idp.example.com/oauth2
    
    # Explicit RFC 7662 Introspection Endpoint
    auth_oauth2.introspection_endpoint = https://idp.example.com/oauth2/v1/introspect
    
    # Authentication method used by RabbitMQ against the IdP endpoint
    # Options: basic (HTTP Basic Auth) | request_param (POST body params)
    auth_oauth2.introspection_client_auth_method = basic
    auth_oauth2.introspection_client_id = rabbitmq-server-introspection-client
    
    # ENCRYPTED: Introspection Client Secret
    auth_oauth2.introspection_client_secret = encrypted:cPAymwqmMnbPXXRVqVzpxJdrS8mHEKuo2V+3vt1u/fymexD9oztQ2G/oJ4PAaSb2c5N
    
    # ==============================================================================
    # 3. MANAGEMENT WEB UI SESSION SIGNING (REQUIRED FOR OPAQUE TOKENS)
    # ==============================================================================
    auth_oauth2.opaque_token_signing_key.id = rabbit_kid
    
    # ENCRYPTED: HS256 Symmetric Key used to mint local browser session tokens
    auth_oauth2.opaque_token_signing_key.key = encrypted:z8vN0qP3xX9mW1rL4kS7jF2vC5bN8mQ1vT4rP7kX9mW1rL4kS7jF2vC5bN8mQ1vT
    
    # ==============================================================================
    # 4. HTTPS & CUSTOM CA TRUST STORE (IF IDP USES INTERNAL CA)
    # ==============================================================================
    # Uncomment if your Identity Provider uses an enterprise internal CA
    # auth_oauth2.https.cacertfile = /etc/rabbitmq/certs/enterprise-ca.crt

---

## Section 4: Operational Deployment Sequence

Execute this sequence on each cluster node to deploy the configuration cleanly:

### 1. Enable Required Plugins

    rabbitmq-plugins enable rabbitmq_auth_backend_oauth2

### 2. Set File System Permissions
Restrict access to the configuration files containing ciphertext and environment variables:

    chmod 640 /etc/rabbitmq/rabbitmq.conf
    chown root:rabbitmq /etc/rabbitmq/rabbitmq.conf
    
    chmod 600 /etc/rabbitmq/rabbitmq-env.conf
    chown rabbitmq:rabbitmq /etc/rabbitmq/rabbitmq-env.conf

### 3. Restart the Broker

    systemctl restart rabbitmq-server
    # OR for Bitnami OVA installations:
    systemctl restart bitnami.vmware-rabbitmq

---

## Section 5: Verification & Troubleshooting

### 1. Verify Configuration Decryption at Startup
Check the startup log to ensure the `config_entry_decoder` successfully decrypted the config entries without throwing bad passphrase errors:

    grep -i -E "decoder|config|oauth" /var/log/rabbitmq/rabbit@$(hostname).log

*Success Indicator:* The log indicates `config_entry_decoder` initialized successfully and `rabbitmq_auth_backend_oauth2` loaded without syntax or decryption errors.

### 2. Verify Introspection Network Reachability
Ensure the RabbitMQ host can communicate with the Identity Provider's introspection port (HTTPS 443):

    curl -i -X POST -u "rabbitmq-server-introspection-client:MyClientPassword" \
      https://idp.example.com/oauth2/v1/introspect

### 3. Test Opaque Token Authentication via AMQP
Pass an active opaque token in the password field using standard AMQP client tools or `perf-test`:

    /usr/local/bin/rabbitmq-perf-test \
      --uri "amqp://rabbitmq:YOUR_OPAQUE_TOKEN@localhost:5672/%2f" \
      --consumers 1 --producers 1 --time 10