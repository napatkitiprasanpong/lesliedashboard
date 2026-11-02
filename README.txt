LESLIE PROJECT DASHBOARD - GitHub website
=========================================
Your website (this index.html) lives on GitHub Pages.
Your data and PDFs live in a SEPARATE PRIVATE repo. Both phone and PC read/write it.

STEP 1 - Put the website online
  1. github.com > New repository > name: leslie-dashboard (Public is fine - it has no data).
  2. "uploading an existing file" > drag index.html in > Commit.
  3. Settings > Pages > Source: "Deploy from a branch", Branch: main, folder: / (root) > Save.
  4. After ~1 minute your site is at  https://YOUR-USERNAME.github.io/leslie-dashboard/

STEP 2 - Make the private data repo
  1. New repository > name: leslie-data > choose PRIVATE > tick "Add a README file" > Create.
     (Never make this one public. It holds your tasks, money and PDFs.)

STEP 3 - Make an access token (a password for the app)
  1. GitHub > your photo > Settings > Developer settings > Personal access tokens > Fine-grained tokens > Generate new token.
  2. Name: leslie-dashboard. Expiration: 1 year.
  3. Repository access: "Only select repositories" > leslie-data.
  4. Permissions > Repository permissions > Contents: Read and write.
  5. Generate, then copy the token (starts with github_pat_). You only see it once.

STEP 4 - Connect each device (PC and phone)
  1. Open your site > Dashboard > "GitHub sync".
  2. Repo: YOUR-USERNAME/leslie-data    Token: paste it    Branch: main  > Connect.
  3. Repeat on the phone. Phone: add the site to your home screen
     (iPhone Safari: Share > Add to Home Screen. Android Chrome: menu > Add to Home screen).

HOW SYNC WORKS
  - Data is saved to leslie-data.json in the private repo a few seconds after each change,
    when you open the site, when you come back to it, and every minute while it is open.
  - Changes from both devices are merged, so nothing is overwritten.
  - PDFs (File tab) are stored in the repo's files/ folder. Open = view, Download = save to your device. Max 25 MB each.
  - The token is saved only in that browser on that device. Do not share it or put it in the website repo.
  - Everything is also in the repo's history on GitHub, so you can always get old versions back.
