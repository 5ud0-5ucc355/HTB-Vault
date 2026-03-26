---
type: cheatsheet
tags:
  - tool/git
  - phase/workflow
status: stable
---

# 📑 Git: The Vault Sync Tool

> [!ABSTRACT] Summary
> Git is your "Save Game" system. It tracks changes locally on your Mac. GitHub is your "Cloud Backup."

---

## 🔄 The Daily "A-C-P" Loop
*Run these every time you finish a study session or add a new tool.*

### 1. Add (Stage) - "Hey Git, look at these files."
   ```bash
   git add .
   ```
### 2. Commit (Snapshot)

"Save a version to my Mac with this label."

```bash
git commit -m "Brief description of what you changed"
```

### 3. Push to the Cloud

```
git push origin main
```

### 4. Pro-Configuration

Run these once to make your Mac life easier.

Remember my Password (Keychain):

```bash
git config --global credential.helper osxkeychain
```

**Check what is waiting to be saved:**

```bash
git status
```

## 5. Vocabulary for Operators

- **Origin:** Your remote repository on GitHub.
    
- **Main:** Your primary work timeline.
    
- **Commit:** A permanent "Save Game" point.
    
- **Push:** Sending your local saves to the internet.
    
- **Pull:** Grabbing the internet's saves for your local machine.

