# Keychain Services from Terminal

The macOS Keychain is a secure, encrypted database that stores passwords, certificates, encryption keys, and secure notes. While most users interact with Keychain through the Keychain Access GUI application, the `security` command-line tool provides full access to Keychain services for automation, scripting, and administrative tasks.

## Understanding Keychains

macOS maintains several keychains by default:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Keychain Hierarchy                            │
├─────────────────────────────────────────────────────────────────┤
│  System Keychain                                                 │
│  /Library/Keychains/System.keychain                             │
│  - System-wide certificates and passwords                        │
│  - Requires admin authentication                                 │
├─────────────────────────────────────────────────────────────────┤
│  System Roots                                                    │
│  /System/Library/Keychains/SystemRootCertificates.keychain      │
│  - Apple's trusted root certificates                             │
│  - Protected by SIP                                              │
├─────────────────────────────────────────────────────────────────┤
│  Login Keychain                                                  │
│  ~/Library/Keychains/login.keychain-db                          │
│  - User's personal passwords and certificates                    │
│  - Unlocked at login by default                                  │
├─────────────────────────────────────────────────────────────────┤
│  Local Items (iCloud Keychain)                                   │
│  ~/Library/Keychains/[UUID]/                                    │
│  - Synced across devices via iCloud                              │
│  - Protected by Secure Enclave                                   │
└─────────────────────────────────────────────────────────────────┘
```

## The security Command

### Keychain Management

```bash
# List all keychains in search list
$ security list-keychains
    "/Users/david/Library/Keychains/login.keychain-db"
    "/Library/Keychains/System.keychain"

# Show default keychain
$ security default-keychain
    "/Users/david/Library/Keychains/login.keychain-db"

# Set default keychain
$ security default-keychain -s ~/Library/Keychains/login.keychain-db

# Show login keychain
$ security login-keychain
    "/Users/david/Library/Keychains/login.keychain-db"

# Get keychain info
$ security show-keychain-info ~/Library/Keychains/login.keychain-db
Keychain "/Users/david/Library/Keychains/login.keychain-db"
    lock-on-sleep
    timeout=21600s
```

### Creating and Deleting Keychains

```bash
# Create a new keychain
$ security create-keychain -p "password" ~/Library/Keychains/custom.keychain

# Create with password prompt (more secure)
$ security create-keychain ~/Library/Keychains/custom.keychain
password for new keychain:

# Add to search list
$ security list-keychains -s ~/Library/Keychains/login.keychain-db \
    ~/Library/Keychains/custom.keychain

# Delete a keychain
$ security delete-keychain ~/Library/Keychains/custom.keychain
```

### Locking and Unlocking

```bash
# Lock a keychain
$ security lock-keychain ~/Library/Keychains/login.keychain-db

# Unlock a keychain (prompts for password)
$ security unlock-keychain ~/Library/Keychains/login.keychain-db

# Unlock with password (for scripts - be careful with password exposure)
$ security unlock-keychain -p "password" ~/Library/Keychains/login.keychain-db

# Lock all keychains
$ security lock-keychain -a

# Set keychain to lock after timeout
$ security set-keychain-settings -t 3600 ~/Library/Keychains/login.keychain-db
# Locks after 1 hour of inactivity

# Set to lock on sleep
$ security set-keychain-settings -l ~/Library/Keychains/login.keychain-db
```

## Managing Passwords

### Finding Passwords

```bash
# Find a generic password
$ security find-generic-password -a "username" -s "service" \
    ~/Library/Keychains/login.keychain-db

# Output includes metadata:
# keychain: "/Users/david/Library/Keychains/login.keychain-db"
# class: "genp"
# attributes:
#     "acct"<blob>="username"
#     "svce"<blob>="service"
#     ...

# Get just the password (to stdout)
$ security find-generic-password -a "username" -s "service" -w
mypassword

# Find internet password (website credentials)
$ security find-internet-password -a "username" -s "example.com" -w

# Find by label
$ security find-generic-password -l "My Label" -w

# Find with domain and protocol
$ security find-internet-password -s "github.com" -r "htps" -w
```

### Adding Passwords

```bash
# Add a generic password
$ security add-generic-password \
    -a "username" \
    -s "service-name" \
    -w "password" \
    -l "Human Readable Label" \
    ~/Library/Keychains/login.keychain-db

# Add without specifying password (prompts)
$ security add-generic-password \
    -a "username" \
    -s "service-name" \
    -l "My Service" \
    -T "" \
    ~/Library/Keychains/login.keychain-db

# Add an internet password
$ security add-internet-password \
    -a "username" \
    -s "example.com" \
    -w "password" \
    -r "htps" \
    -l "Example Website" \
    ~/Library/Keychains/login.keychain-db

# Update existing password (use -U flag)
$ security add-generic-password \
    -a "username" \
    -s "service-name" \
    -w "new-password" \
    -U \
    ~/Library/Keychains/login.keychain-db
```

### Password Options

```bash
# -a : Account name
# -s : Service name
# -w : Password (or -w to read from stdin)
# -l : Label
# -c : Creator (4-char code)
# -C : Type (4-char code)
# -D : Kind description
# -r : Protocol (htps, http, etc.)
# -p : Path
# -P : Port
# -j : Comment
# -T : Application path for ACL (use "" for no apps)
# -A : Allow all applications to access
# -U : Update existing item
```

### Deleting Passwords

```bash
# Delete a generic password
$ security delete-generic-password -a "username" -s "service-name" \
    ~/Library/Keychains/login.keychain-db

# Delete an internet password
$ security delete-internet-password -a "username" -s "example.com" \
    ~/Library/Keychains/login.keychain-db

# Delete by label
$ security delete-generic-password -l "My Label" \
    ~/Library/Keychains/login.keychain-db
```

## Managing Certificates

### Viewing Certificates

```bash
# List all certificates in a keychain
$ security find-certificate -a ~/Library/Keychains/login.keychain-db

# List with details
$ security find-certificate -a -p ~/Library/Keychains/login.keychain-db
# Outputs in PEM format

# Find certificate by name
$ security find-certificate -c "Developer ID" -a

# Find certificate by email
$ security find-certificate -e "your@email.com" -a

# Show certificate details
$ security find-certificate -c "Developer ID" -p | openssl x509 -noout -text

# Count certificates
$ security find-certificate -a | grep -c "keychain:"
```

### Adding Certificates

```bash
# Add a certificate to keychain
$ security add-certificates ~/path/to/certificate.cer

# Add to specific keychain
$ security add-certificates -k ~/Library/Keychains/login.keychain-db \
    ~/path/to/certificate.cer

# Add as trusted root (system keychain)
$ sudo security add-trusted-cert -d -r trustRoot \
    -k /Library/Keychains/System.keychain \
    ~/path/to/ca-certificate.cer
```

### Managing Trust Settings

```bash
# Get trust settings for a certificate
$ security dump-trust-settings -d
# -d : Admin domain
# -s : System domain (read-only)

# Add trust setting
$ sudo security add-trusted-cert -d -r trustRoot \
    -p ssl -p basic \
    -k /Library/Keychains/System.keychain \
    ~/path/to/certificate.cer

# Remove trust setting
$ sudo security remove-trusted-cert -d ~/path/to/certificate.cer

# Verify certificate trust
$ security verify-cert -c ~/path/to/certificate.cer
```

### Exporting Certificates

```bash
# Export certificate to PEM
$ security find-certificate -c "Certificate Name" -p > cert.pem

# Export certificate and key as PKCS12
$ security export -k ~/Library/Keychains/login.keychain-db \
    -t identities \
    -f pkcs12 \
    -o identity.p12

# Export specific identity
$ security find-identity -v -p codesigning
# Note the SHA-1 hash
$ security export -k ~/Library/Keychains/login.keychain-db \
    -t identities \
    -f pkcs12 \
    -o identity.p12 \
    -P "export-password"
```

### Importing Certificates and Keys

```bash
# Import PKCS12 file
$ security import ~/path/to/identity.p12 \
    -k ~/Library/Keychains/login.keychain-db \
    -f pkcs12 \
    -P "password"

# Import PEM certificate
$ security import ~/path/to/certificate.pem \
    -k ~/Library/Keychains/login.keychain-db \
    -f pem

# Import with application access
$ security import ~/path/to/identity.p12 \
    -k ~/Library/Keychains/login.keychain-db \
    -f pkcs12 \
    -P "password" \
    -A  # Allow all applications
```

## Managing Identities (Certificate + Private Key)

```bash
# List all identities (cert + key pairs)
$ security find-identity -v
  1) ABC123... "Developer ID Application: Your Name (TEAM)"
  2) DEF456... "Apple Development: your@email.com (TEAM)"
     2 valid identities found

# List code signing identities
$ security find-identity -v -p codesigning

# List SSL identities
$ security find-identity -v -p ssl

# Available policies:
# basic, ssl, smime, eap, ipsec, ichat, codesigning, sysdefault, timestamping
```

## Managing Keys

```bash
# Generate a symmetric key
$ security generate-key -a AES -b 256 -l "My Encryption Key"

# List keys
$ security dump-keychain ~/Library/Keychains/login.keychain-db | grep -A5 "class: key"

# Import a key
$ security import ~/path/to/private.key \
    -k ~/Library/Keychains/login.keychain-db \
    -f pem \
    -t priv

# Export key
$ security export -k ~/Library/Keychains/login.keychain-db \
    -t privKeys \
    -f pem \
    -o private.pem
```

## Access Control Lists (ACLs)

Control which applications can access keychain items:

```bash
# View ACL for a keychain item
$ security dump-keychain -a ~/Library/Keychains/login.keychain-db | grep -A20 "access:"

# Add password with specific app access
$ security add-generic-password \
    -a "username" \
    -s "service" \
    -w "password" \
    -T /Applications/Safari.app/Contents/MacOS/Safari \
    ~/Library/Keychains/login.keychain-db

# Add with no app access (requires manual approval each time)
$ security add-generic-password \
    -a "username" \
    -s "service" \
    -w "password" \
    -T "" \
    ~/Library/Keychains/login.keychain-db

# Allow all applications (less secure)
$ security add-generic-password \
    -a "username" \
    -s "service" \
    -w "password" \
    -A \
    ~/Library/Keychains/login.keychain-db

# Set ACL partition list (for codesigning)
$ security set-key-partition-list -S apple-tool:,apple:,codesign: \
    -s -k "keychain-password" \
    ~/Library/Keychains/login.keychain-db
```

## Practical Scripts

### Store and Retrieve Secrets in Scripts

```bash
#!/bin/bash
# secret-manager.sh - Manage secrets via keychain

SERVICE="com.example.myapp"
KEYCHAIN="$HOME/Library/Keychains/login.keychain-db"

store_secret() {
    local name="$1"
    local value="$2"

    security add-generic-password \
        -a "$name" \
        -s "$SERVICE" \
        -w "$value" \
        -U \
        "$KEYCHAIN" 2>/dev/null

    if [[ $? -eq 0 ]]; then
        echo "Secret '$name' stored successfully"
    else
        echo "Failed to store secret"
        return 1
    fi
}

get_secret() {
    local name="$1"

    security find-generic-password \
        -a "$name" \
        -s "$SERVICE" \
        -w "$KEYCHAIN" 2>/dev/null
}

delete_secret() {
    local name="$1"

    security delete-generic-password \
        -a "$name" \
        -s "$SERVICE" \
        "$KEYCHAIN" 2>/dev/null

    if [[ $? -eq 0 ]]; then
        echo "Secret '$name' deleted"
    else
        echo "Secret not found"
        return 1
    fi
}

# Usage
case "$1" in
    set)
        store_secret "$2" "$3"
        ;;
    get)
        get_secret "$2"
        ;;
    delete)
        delete_secret "$2"
        ;;
    *)
        echo "Usage: $0 {set|get|delete} name [value]"
        exit 1
        ;;
esac
```

### CI/CD Keychain Setup

```bash
#!/bin/bash
# ci-keychain-setup.sh - Setup keychain for CI/CD builds

set -e

KEYCHAIN_NAME="build.keychain"
KEYCHAIN_PATH="$HOME/Library/Keychains/$KEYCHAIN_NAME"
KEYCHAIN_PASSWORD="${CI_KEYCHAIN_PASSWORD:-build}"

# Create temporary keychain
security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"

# Set keychain settings (don't lock automatically)
security set-keychain-settings -t 3600 -u "$KEYCHAIN_PATH"

# Add to search list
security list-keychains -d user -s "$KEYCHAIN_PATH" $(security list-keychains -d user | tr -d '"')

# Unlock keychain
security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"

# Import certificate (base64 encoded in environment variable)
echo "$SIGNING_CERTIFICATE_P12_BASE64" | base64 --decode > /tmp/cert.p12
security import /tmp/cert.p12 \
    -k "$KEYCHAIN_PATH" \
    -P "$SIGNING_CERTIFICATE_PASSWORD" \
    -A \
    -T /usr/bin/codesign \
    -T /usr/bin/security

rm /tmp/cert.p12

# Allow codesign to access key without prompt
security set-key-partition-list -S apple-tool:,apple:,codesign: \
    -s -k "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"

echo "CI keychain setup complete"
```

### Backup Keychain Items

```bash
#!/bin/bash
# backup-keychain.sh - Export keychain items (certificates only)

BACKUP_DIR="$HOME/keychain-backup-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Export certificates
security find-certificate -a -p > "$BACKUP_DIR/all-certificates.pem"

# Export identities (will prompt for keychain password)
security export \
    -k ~/Library/Keychains/login.keychain-db \
    -t identities \
    -f pkcs12 \
    -o "$BACKUP_DIR/identities.p12"

echo "Backup saved to $BACKUP_DIR"
echo "WARNING: Secure this backup - it contains sensitive cryptographic material"
```

### Certificate Expiration Check

```bash
#!/bin/bash
# check-cert-expiry.sh - Check certificate expiration dates

echo "=== Certificate Expiration Report ==="
echo

security find-certificate -a -p 2>/dev/null | \
awk '/-----BEGIN CERTIFICATE-----/{cert=""} {cert=cert$0"\n"} /-----END CERTIFICATE-----/{print cert}' | \
while read -r cert_block; do
    if [[ -n "$cert_block" ]]; then
        subject=$(echo "$cert_block" | openssl x509 -noout -subject 2>/dev/null | sed 's/subject=//')
        expiry=$(echo "$cert_block" | openssl x509 -noout -enddate 2>/dev/null | sed 's/notAfter=//')

        if [[ -n "$expiry" ]]; then
            expiry_epoch=$(date -j -f "%b %d %H:%M:%S %Y %Z" "$expiry" "+%s" 2>/dev/null)
            now_epoch=$(date "+%s")
            days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

            if [[ $days_left -lt 30 ]]; then
                status="WARNING"
            else
                status="OK"
            fi

            printf "%-10s %-80s %s (%d days)\n" "[$status]" "${subject:0:80}" "$expiry" "$days_left"
        fi
    fi
done
```

## Security Considerations

### Keychain Security Best Practices

1. **Lock keychain when not in use**
   ```bash
   $ security lock-keychain
   ```

2. **Set auto-lock timeout**
   ```bash
   $ security set-keychain-settings -t 300 ~/Library/Keychains/login.keychain-db
   ```

3. **Use specific application ACLs**
   ```bash
   $ security add-generic-password -T /path/to/specific/app -a user -s service -w pass
   ```

4. **Never put passwords directly in scripts**
   ```bash
   # Bad
   PASSWORD="hardcoded"

   # Good
   PASSWORD=$(security find-generic-password -s "myservice" -w)
   ```

5. **Create separate keychains for automation**
   ```bash
   $ security create-keychain -p "$SECURE_PASSWORD" ~/Library/Keychains/automation.keychain
   ```

## Summary

The `security` command provides complete Keychain access from the terminal:

| Task | Command |
|------|---------|
| List keychains | `security list-keychains` |
| Unlock keychain | `security unlock-keychain` |
| Find password | `security find-generic-password -s "service" -w` |
| Add password | `security add-generic-password -s "service" -a "user" -w "pass"` |
| List certificates | `security find-certificate -a` |
| Import certificate | `security import cert.p12 -k keychain` |
| List identities | `security find-identity -v -p codesigning` |

Key concepts:

- Login keychain stores user credentials
- System keychain stores system-wide items
- Passwords can be stored/retrieved without GUI
- ACLs control which apps can access items
- Always use keychain for secrets in scripts

```bash
# Quick reference
$ security list-keychains                                    # List keychains
$ security find-generic-password -s "service" -w             # Get password
$ security add-generic-password -s "service" -a "user" -w "pass"  # Store password
$ security find-identity -v -p codesigning                   # List signing identities
$ security unlock-keychain                                   # Unlock keychain
```

The Keychain provides secure credential storage that integrates with macOS authentication, making it the proper place to store secrets rather than configuration files or environment variables.
