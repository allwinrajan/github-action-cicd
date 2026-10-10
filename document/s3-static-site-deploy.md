# Deploy React App to AWS S3 using GitHub Actions (AWS Console Guide)

Reusable template. For a new project, repeat everything with a new bucket name and repo name.

How it works:
- **S3 bucket** hosts the website files (public can only read)
- **GitHub Actions** builds, tests and uploads `dist/` to the bucket on every push to `main`
- **OIDC login**: GitHub gets a short-lived token from AWS, so no AWS keys are saved anywhere
- **IAM role** can only be used by **this repo + main branch**, and can only touch **this one bucket**

Values used in this guide (change for your project):
- Bucket name: `dk-site-cicd`
- Region: `Asia Pacific (Mumbai) ap-south-1`
- GitHub repo: `allwinrajan/github-action-cicd`
- Role name: `github-actions-s3-deploy`

---

# Part A: Create S3 bucket

# 1. Open S3
- Login to https://console.aws.amazon.com
- Top right: select region **Asia Pacific (Mumbai)**
- Search bar: type **S3** → click **S3**
- Click **Create bucket**

# 2. Bucket settings
- Bucket type: **General purpose**
- Bucket name: `dk-site-cicd` (must be unique worldwide, lowercase)
- Object Ownership: **ACLs disabled (recommended)**

# 3. Block Public Access settings
- Uncheck **Block all public access**
- Keep these 2 **checked**:
  - Block public access to buckets and objects granted through new ACLs
  - Block public access to buckets and objects granted through any ACLs
- Keep these 2 **unchecked**:
  - Block public access ... through new public bucket or access point policies
  - Block public and cross-account access ... through any public bucket or access point policies
- Tick the warning box **I acknowledge that the current settings might result in this bucket and the objects within becoming public**

# 4. Finish creating
- Bucket Versioning: **Enable** (lets you restore old files if a bad deploy happens)
- Default encryption: leave **SSE-S3**
- Click **Create bucket**

---

# Part B: Enable static website hosting

# 5. Turn on website hosting
- Click your bucket name
- Go to **Properties** tab
- Scroll to the bottom → **Static website hosting** → **Edit**
- Static website hosting: **Enable**
- Hosting type: **Host a static website**
- Index document: `index.html`
- Error document: `index.html`
- Click **Save changes**

# 6. Copy the website URL
- **Properties** tab → bottom → **Static website hosting**
- Copy **Bucket website endpoint**
  - Looks like `http://dk-site-cicd.s3-website.ap-south-1.amazonaws.com`
http://dk-site-cicd.s3-website.ap-south-1.amazonaws.com/

---

# Part C: Allow public read only

# 7. Add bucket policy
- Go to **Permissions** tab
- **Bucket policy** → **Edit**
- Paste this (change the bucket name):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::dk-site-cicd/*"
    }
  ]
}
```
- Click **Save changes**
- Bucket now shows a red **Publicly accessible** tag. That is expected for a website.
- People can only **read** files. Nobody can upload, delete or list files.

---

# Part D: Manual test (make sure bucket works)

# 8. Build app on your laptop
- Command:
```bash
npm ci
npm run build
```

# 9. Upload files
- Bucket → **Objects** tab → **Upload**
- Open the `dist` folder and drag **what is inside it** (`index.html` + `assets` folder), not the `dist` folder itself
- Click **Upload**

# 10. Open the site
- Open the website URL from step 6
- You should see the Hello World page

---

# Part E: Let GitHub log in to AWS securely (OIDC)

# 11. Add GitHub as identity provider (one time per AWS account)
- Search bar: **IAM** → click **IAM**
- Left menu → **Identity providers** → **Add provider**
- Provider type: **OpenID Connect**
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`
- Click **Add provider**
- If it says it already exists, skip this step

# 12. Create the deploy role
- IAM → left menu → **Roles** → **Create role**
- Trusted entity type: **Web identity**
- Identity provider: `token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`
- GitHub organization: `allwinrajan`
- GitHub repository: `github-action-cicd`
- GitHub branch: `main`
- Click **Next**
- Add permissions page: **do not select anything** → **Next**
- Role name: `github-actions-s3-deploy`
- Click **Create role**

# 13. Fix the trust policy (IMPORTANT)
GitHub now sends the `sub` with owner ID + repo ID, so the wizard's value does not match.

- First push will fail at **Login to AWS (OIDC)**, which is expected
- Open the failed run → **Debug OIDC claims (on login failure)** → copy the `sub` value
  - Looks like `repo:allwinrajan@163723809/github-action-cicd@1412116653:ref:refs/heads/main`
- IAM → Roles → `github-actions-s3-deploy` → **Trust relationships** → **Edit trust policy**
- Paste (change account ID + `sub`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::152924643789:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:allwinrajan@163723809/github-action-cicd@1412116653:ref:refs/heads/main"
        }
      }
    }
  ]
}
```
- Click **Update policy** → GitHub → **Re-run all jobs**
- No `*` anywhere. This means **only main branch of this exact repo** can use the role
- The IDs also block a deleted-and-recreated repo with the same name

# 14. Give role access to this bucket only
- Same role → **Permissions** tab
- **Add permissions** → **Create inline policy**
- Click **JSON** → delete everything → paste (change the bucket name in 2 places):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListThisBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::dk-site-cicd"
    },
    {
      "Sid": "WriteThisBucket",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::dk-site-cicd/*"
    }
  ]
}
```
- Click **Next**
- Policy name: `s3-deploy-dk-site-cicd`
- Click **Create policy**

# 15. Copy role ARN
- Role **Summary** at the top → copy **ARN**
  - Looks like `arn:aws:iam::123456789012:role/github-actions-s3-deploy`

---

# Part F: Connect GitHub repo to AWS

# 16. Add repository variables
- GitHub → your repo → **Settings**
- Left menu → **Secrets and variables** → **Actions**
- Click **Variables** tab → **New repository variable**
- Add these 3:

| Name | Value |
|---|---|
| `AWS_ROLE_ARN` | role ARN from step 15 |
| `S3_BUCKET` | `dk-site-cicd` |
| `AWS_REGION` | `ap-south-1` |

- These are not secret. There are no AWS keys to hide because OIDC is used.

# 17. Pipeline files (already done in this repo)
- `.github/workflows/ci-cd.yml`: project settings only
- `.github/workflows/reusable-s3-static-site.yml`: Build, Test, Deploy to S3 (OIDC login, `aws s3 sync --delete`, no-cache `index.html`)
- Details: `cicd-template-guide.md`

---

# Part G: Run the pipeline

# 18. Push code
- Command:
```bash
git add .
git commit -m "Deploy to S3"
git push origin main
```
- Or run it by hand: GitHub → **Actions** → **CI/CD - Hello World React** → **Run workflow** → branch `main`

# 19. Watch it
- GitHub → **Actions** tab → click the latest run
- You should see **Build → Test → Deploy to S3**, all green
- Open **Deploy to S3** → **Upload files to S3**. The last line prints the website URL

# 20. Check S3 got updated
- S3 → bucket → **Objects** tab → click refresh
- **Last modified** time = time of the pipeline run

# 21. Prove auto-update works
- Edit text in `src/App.jsx` → also update `src/App.test.jsx` if the test checks that text
- Push to `main`
- After the pipeline is green → open site → **Ctrl + F5**
- New text shows, so the pipeline is updating the bucket

---

# Security checklist
- No AWS access keys stored in GitHub
- Role usable only by `allwinrajan/github-action-cicd` on `main` (step 13)
- Role can only read/write **one bucket** (step 14)
- Pull requests and other branches cannot deploy
- Public visitors can only **read** files
- ACLs disabled + ACL public access blocked
- Versioning on, so you can roll back a bad deploy
- S3 website is **HTTP only**. For HTTPS + custom domain, add CloudFront later.

---

# Reuse for a new project
- Follow the checklist in `cicd-template-guide.md`

---

# Troubleshooting
- **403 Forbidden** on website: check step 3 (last 2 boxes unchecked) and step 7 (bucket policy)
- **404 Not Found**: files were uploaded inside a `dist/` folder. Upload the contents of `dist`, not the folder
- **Not authorized to perform sts:AssumeRoleWithWebIdentity**: trust policy `sub` doesn't match. Copy `sub` from the **Debug OIDC claims** step into step 13
- **Could not load credentials**: `AWS_ROLE_ARN` variable missing or misspelled (step 16)
- **AccessDenied** during upload: bucket name in step 14 doesn't match the `S3_BUCKET` variable
- **Old page still showing**: Ctrl + F5

---

# Rollback a bad deploy
- S3 → bucket → **Objects** → turn on **Show versions**
- Delete the latest version of `index.html`, so the previous version becomes current
- Or just `git revert` the bad commit and push

---

# Cleanup (delete everything)
- S3 → bucket → **Empty** → type `permanently delete` → **Empty**
- S3 → select bucket → **Delete** → type bucket name → **Delete bucket**
- IAM → **Roles** → `github-actions-s3-deploy` → **Delete**
- GitHub → Settings → Secrets and variables → Actions → Variables → delete the 3 variables
