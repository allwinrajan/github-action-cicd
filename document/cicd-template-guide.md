# CI/CD Template: Static Site to S3

Use this every time you deploy a new React/Vite/static app to S3.
The pipeline logic is written once. A new project only changes **settings**, not code.

---

# How the template is built

- `.github/workflows/reusable-s3-static-site.yml` is the **engine**
  - Build, Test, Deploy to S3 (OIDC login, no keys)
  - **Never edit this per project**
- `.github/workflows/ci-cd.yml` is the **caller**
  - Short file, only project settings
  - **This is the only workflow file you touch**
- GitHub **Variables** hold the AWS settings
  - `AWS_REGION`, `S3_BUCKET`, `AWS_ROLE_ARN`
  - Different per repo, so no AWS values are hardcoded in code

```
push to main
   │
   ▼
ci-cd.yml (settings)  ──calls──►  reusable-s3-static-site.yml
                                    Build ─► Test ─► Deploy to S3
                                                       │
                                     OIDC login ◄──────┘
                                     (role trusts only this repo + main)
```

---

# What runs when

| Event | Build | Test | Deploy |
|---|---|---|---|
| Push to `main` | ✅ | ✅ | ✅ |
| Pull request to `main` | ✅ | ✅ | ❌ |
| Run workflow on `main` | ✅ | ✅ | ✅ |
| Run workflow on other branch | ✅ | ✅ | ❌ |

---

# Settings you can change in `ci-cd.yml`

| Setting | Default | Change when |
|---|---|---|
| `app-dir` | `.` | app is in a subfolder, e.g. `frontend` |
| `node-version` | `20` | app needs another Node version |
| `build-command` | `npm run build` | different build script |
| `test-command` | `npm test` | different test script |
| `build-dir` | `dist` | Create React App uses `build` |

---

# Make this repo a GitHub template (one time)

- GitHub → this repo → **Settings** → **General**
- Tick **Template repository**
- Now the repo page shows a green **Use this template** button

---

# New project checklist

# 1. Create the repo from the template
- GitHub → this repo → **Use this template** → **Create a new repository**
- Name: e.g. `my-new-site`
- Clone it and replace the app code with your app (keep the `.github` folder)

# 2. Create the S3 bucket
- Follow `s3-static-site-deploy.md` **steps 1 to 7** with the new bucket name

# 3. Create the IAM role
- Follow `s3-static-site-deploy.md` **steps 12 to 15**
- Use the **new repo name** in step 12
- Use the **new bucket name** in step 14
- Skip step 11 (GitHub provider already exists in your AWS account)

# 4. Add the 3 variables in the new repo
- New repo → **Settings** → **Secrets and variables** → **Actions** → **Variables**
- `AWS_REGION` = `ap-south-1`
- `S3_BUCKET` = new bucket name
- `AWS_ROLE_ARN` = new role ARN

# 5. Adjust settings (only if needed)
- Open `.github/workflows/ci-cd.yml`
- Change `name:` at the top
- Change `app-dir` / `build-dir` / commands if your app is different

# 6. Push and check
- Command:
```bash
git add .
git commit -m "Setup CI/CD"
git push origin main
```
- GitHub → **Actions** → all 3 jobs green
- Click the run → **Summary** shows the website link

---

# Fill-in sheet (copy for each project)

| Item | Value |
|---|---|
| Repo | `allwinrajan/________` |
| Bucket | `________` |
| Region | `ap-south-1` |
| Role name | `github-actions-________` |
| Role ARN | `arn:aws:iam::____________:role/________` |
| Website URL | `http://________.s3-website.ap-south-1.amazonaws.com` |

---

# Current project (filled)

| Item | Value |
|---|---|
| Repo | `allwinrajan/github-action-cicd` |
| Bucket | `dk-site-cicd` |
| Region | `ap-south-1` |
| Role name | `github-actions-s3-deploy` |
| Website URL | `http://dk-site-cicd.s3-website.ap-south-1.amazonaws.com` |

---

# Rules to keep it secure
- One **role per repo**: never reuse a role across repos
- One **bucket per site**
- Never add AWS access keys to GitHub Secrets. OIDC replaces them
- Trust policy must name the exact repo + `main` (no `*`)
- Don't edit the reusable file per project. If you improve it, every project gets the change

---

# Troubleshooting
- **Deploy job skipped**: you are on a PR or not on `main`. That is expected
- **"S3_BUCKET variable is not set"**: add the variables (checklist step 4)
- **Not authorized to perform sts:AssumeRoleWithWebIdentity**: role trust has the wrong repo name (checklist step 3)
- **AccessDenied on upload**: role policy has the wrong bucket name
- **npm ci fails: no package-lock.json**: wrong `app-dir`, or run `npm install` once locally and commit `package-lock.json`
- **Deploy uploads nothing / 404**: wrong `build-dir` (`dist` vs `build`)
