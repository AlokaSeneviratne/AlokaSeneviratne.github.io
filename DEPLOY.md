# Deploying your portfolio to GitHub Pages (free)

## 1. Set your username
Open `index.html`, scroll to the `CONFIG` block near the bottom, and edit:

```js
githubUsername: "your-actual-username",
```

Also update the `links` array (email, GitHub, LinkedIn URLs).

## 2. Create the repo
On GitHub, create a new **public** repository named exactly:

```
your-actual-username.github.io
```

That special name makes GitHub serve it at `https://your-actual-username.github.io`.

## 3. Upload the file
Easiest way, no Git needed:
1. In the new repo, click **Add file > Upload files**
2. Drag in `index.html`
3. Commit

Or with Git:
```bash
git clone https://github.com/your-actual-username/your-actual-username.github.io.git
cd your-actual-username.github.io
cp /path/to/index.html .
git add . && git commit -m "Portfolio site" && git push
```

## 4. Done
Within a minute or two the site is live at
`https://your-actual-username.github.io`.

New projects appear automatically: any public repo you create shows up on the
site the next time someone loads the page. No rebuild, no redeploy.

## Customizing
Everything personal lives in two places:
- The `CONFIG` block in the script (username, links, repos to hide, sort order)
- The hero and About section text in the HTML body

Tips:
- Add **topics** to your GitHub repos (repo page > gear icon next to About).
  They show up as tags on the project cards.
- Write a one-line **description** for each repo. It becomes the card text.
- To hide a repo, add its name to `excludeRepos` in CONFIG.

## Notes
- The site uses the public GitHub API (60 requests/hour per visitor IP),
  which is far more than a portfolio needs.
- Forked and archived repos are hidden by default.
