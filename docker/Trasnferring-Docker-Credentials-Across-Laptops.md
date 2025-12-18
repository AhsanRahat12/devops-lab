# Transferring Docker Credentials Between Laptops

## Background

Docker Desktop for Linux requires `pass` (password manager) with GPG encryption before you can sign in. This isn't optional - Docker won't let you login without it.

From the official Docker docs:
> Docker Desktop for Linux relies on pass to store credentials in GPG-encrypted files. Before signing in to Docker Desktop with your Docker ID, you must initialize pass.

When I set up my second laptop, I transferred my existing GPG key instead of creating a new one. At the time, I thought I needed the same GPG key to use the same Docker account (turns out that's not true - you can create a new GPG key on each laptop and still use the same Docker account). But transferring the key works fine and keeps the same GPG passphrase across my machines.

**Important clarification**: "Same setup" doesn't mean you need the same GPG key. You can have the exact same Docker setup (pass + GPG encryption) with different GPG keys on each laptop. The GPG key is just for encrypting your password locally - it has nothing to do with your Docker Hub account. I transferred mine to use the same GPG passphrase across laptops, but that's a personal preference, not a requirement.

---

## Part 1: Export from Old Laptop

### Get Your GPG Key ID
```bash
gpg --list-secret-keys
```
Copy the long string after `sec` - that's your key ID.

### Export and Encrypt
```bash
# Export keys
gpg --export-secret-keys <YOUR_KEY_ID> > private.key
gpg --export <YOUR_KEY_ID> > public.key

# Create encrypted archive
tar czf - private.key public.key | gpg --symmetric --cipher-algo AES256 -o gpg-keys.tar.gz.gpg

# Clean up
rm private.key public.key
```

Email the `gpg-keys.tar.gz.gpg` file to yourself.

---

## Part 2: Set Up New Laptop

### Install Dependencies
```bash
paru -S docker-desktop docker-credential-pass-bin
sudo pacman -S pass gnupg
```

### Start Docker Desktop
```bash
systemctl --user enable docker-desktop
systemctl --user start docker-desktop
```

### Import Keys
```bash
cd ~/Downloads

# Decrypt and extract
gpg --decrypt gpg-keys.tar.gz.gpg | tar xzf -

# Import keys
gpg --import public.key
gpg --import private.key

# Trust your key
gpg --edit-key <YOUR_KEY_ID>
# Type: trust → 5 → y → quit

# Clean up
rm private.key public.key
```

### Configure Docker to Use Pass
```bash
# Initialize pass with your transferred key
pass init <YOUR_KEY_ID>

# Configure Docker
mkdir -p ~/.docker
echo '{"credsStore": "pass"}' > ~/.docker/config.json

# Restart Docker
systemctl --user restart docker-desktop
```

Wait 15-20 seconds, then login:
```bash
docker login -u <YOUR_DOCKER_USERNAME>
```

Done! Your credentials are now encrypted with GPG.

---

## Alternative: Create New GPG Key Instead

You don't actually need to transfer the key. You could create a fresh GPG key on each laptop:

```bash
# On new laptop
gpg --generate-key
# Enter your name and email

# Copy the GPG ID from the output, then:
pass init <NEW_GPG_ID>

# Configure Docker
mkdir -p ~/.docker
echo '{"credsStore": "pass"}' > ~/.docker/config.json

# Login
docker login -u <YOUR_DOCKER_USERNAME>
```

This works just as well - you can use the same Docker account with different GPG keys on different machines. The GPG key just encrypts your local password storage, it's not tied to your Docker account.

---

## Quick Troubleshooting

**DNS errors?**
```bash
sudo nano /etc/resolv.conf
# Add: nameserver 8.8.8.8
systemctl --user restart docker-desktop
```

**GPG password prompts?** This is normal - it's decrypting your credentials.

**Verify it works:**
```bash
docker info | grep Username
```

---

## Why This Setup?

Docker Desktop on Arch requires this GPG/pass setup for security. It stores your Docker Hub password encrypted instead of in plaintext. The GPG key is what encrypts/decrypts your password - you'll be prompted for your GPG passphrase when Docker needs to access your credentials.

**Key Point**: The GPG key and your Docker Hub account are completely separate:
- **Docker Hub account** = Your username/password on hub.docker.com (stored in the cloud)
- **GPG key** = Just encrypts your password storage on your local laptop

You can use the same Docker Hub account on multiple laptops with:
- Same GPG key on all laptops (what I did - same passphrase everywhere)
- Different GPG keys per laptop (more secure, but more passphrases to remember)
- No GPG at all on some laptops (less secure - password in plaintext)

The "setup" refers to how your local machine stores passwords, not which Docker account you're logged into. I transferred my key for convenience (one passphrase to remember), but it's not required to use the same Docker account.
