# GitHub Setup Instructions

## Prerequisites

You need to create a new repository on GitHub before pushing.

## Step 1: Create GitHub Repository

1. Go to https://github.com/new
2. Repository name: `Flowise` (or `Flowise-F5`)
3. Description: `F5 AI Agent Workflow Design Hub - Built on Flowise`
4. Choose: **Public** or **Private** (your preference)
5. **Do NOT** initialize with README, .gitignore, or license (we already have these)
6. Click **Create repository**

## Step 2: Push to GitHub

The repository has already been prepared with all changes committed locally. To push:

```bash
cd /Users/br.lee/projects/Flowise

# Push to your repository
git push -u origin main
```

If you get authentication errors, you may need to:

### Option A: Using Personal Access Token (Recommended)

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token with `repo` scope
3. Copy the token
4. When prompted for password, paste the token instead

### Option B: Using SSH

```bash
# Generate SSH key if you don't have one
ssh-keygen -t ed25519 -C "your_email@example.com"

# Add to ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Copy public key
cat ~/.ssh/id_ed25519.pub
# Add this to GitHub Settings → SSH and GPG keys → New SSH key

# Update remote to use SSH
git remote set-url origin git@github.com:maximuslee1226/Flowise.git

# Push
git push -u origin main
```

## Step 3: Verify

After pushing, visit:
```
https://github.com/maximuslee1226/Flowise
```

You should see all your files including:
- DEPLOYMENT.md (deployment instructions)
- docker-compose-f5.yml
- F5 branding updates
- All customizations

## What's Excluded

The following are automatically excluded via `.gitignore` (privacy/security):
- `.env` files (environment variables)
- `.claude/` directory (Claude Code settings)
- `prompts.txt` (your prompts)
- `node_modules/` (dependencies)
- `build/` and `dist/` (compiled code)
- `logs/` (log files)
- Private keys and certificates

## Backup Archive

A compressed backup has been created at:
```
/Users/br.lee/projects/Flowise-F5-20251027.tar.gz
```

This archive contains the entire project (excluding node_modules, build files, and sensitive data) and can be:
- Stored as a backup
- Shared with team members
- Used for offline deployment

## Next Steps

1. Create the GitHub repository
2. Push your changes
3. Update the repository README if needed
4. Share the repository URL with your team
5. Follow DEPLOYMENT.md for deployment instructions

## Repository Structure

After pushing, your repository will contain:

```
Flowise/
├── DEPLOYMENT.md              # Comprehensive deployment guide
├── GITHUB_SETUP.md           # This file
├── README.md                 # Original Flowise README
├── Dockerfile                # Container build file
├── docker-compose-f5.yml     # Docker Compose configuration
├── packages/
│   ├── ui/                   # Frontend (F5 branded)
│   └── server/               # Backend
├── images/
│   └── f5-logo.png          # F5 branding assets
└── ... (other files)
```

## Troubleshooting

### Repository Not Found Error

If you get "repository not found":
1. Make sure you created the repository on GitHub first
2. Check the repository name matches: `https://github.com/maximuslee1226/Flowise`
3. Verify you have access to push to this repository

### Authentication Failed

If authentication fails:
1. Use a Personal Access Token instead of password
2. Or set up SSH authentication (recommended)
3. Make sure your GitHub account has push access

### Push Rejected

If push is rejected:
```bash
# If the remote has commits you don't have locally
git pull --rebase origin main
git push -u origin main
```

---

**Ready to push? Follow Step 1 and Step 2 above!**
