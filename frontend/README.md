# Frontend Submodule Repository

This is the **frontend repository**. In this project structure, the frontend is added to the parent monorepo using a **Git submodule**.

---

## 🔗 How It Works

- The parent repository contains a **pointer** to the latest commit in this frontend repo.
- This repo contains **only** the frontend source code (e.g., React, Vue, etc.).
- During deployment, the parent repo updates the submodule and rebuilds the frontend container if changes are detected.

---

## ⚙️ Development

You can work on the frontend independently in this repo. The changes will be reflected in the parent repo after updating the submodule reference with:

```bash
git submodule update --remote frontend
git add frontend
git commit -m "Update frontend to latest commit"
```
