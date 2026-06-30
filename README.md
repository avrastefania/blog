# chickenprophecy

A Jekyll blog for Hack The Box walkthroughs. Built to run on GitHub Pages with zero config.

## Posting a new walkthrough

1. Create a file in `_posts/` named `YYYY-MM-DD-htb-BOXNAME.md`
   (the date in the filename is what orders it).
2. At the top, add front matter:

   ```
   ---
   title: "HTB: BoxName"
   date: 2026-07-05
   tags: [hackthebox, htb-boxname, nmap, sqli, easy]
   ---
   ```

3. Write the body in Markdown below the `---`. Use `## Heading` for sections,
   triple-backticks for code blocks, and `![alt](/assets/img/pic.png)` for images.
4. Commit and push. The home page and HTB Labs page list it automatically —
   you never edit those by hand.

Look at `_posts/2026-06-30-htb-shoppy.md` as a working template.

## Images

Put screenshots in `assets/img/` and reference them as `/assets/img/name.png`.

## Editing the About page

Edit `about.html` — swap the links and rewrite the bio.

## Deploying to GitHub Pages

1. Create a repo. Two options:
   - **User site**: name the repo `chickenprophecy.github.io`. Site lives at
     `https://chickenprophecy.github.io`. Leave `baseurl: ""` in `_config.yml`.
   - **Project site**: any repo name, e.g. `htb-writeups`. Site lives at
     `https://chickenprophecy.github.io/htb-writeups`. Set
     `baseurl: "/htb-writeups"` in `_config.yml`.
2. Push all these files to the repo.
3. In the repo: Settings → Pages → Source → "Deploy from a branch" → branch
   `main`, folder `/ (root)` → Save.
4. Wait ~1 minute. Your site is live at the URL above.

## Running locally (optional)

If you install Ruby + Jekyll:

```
bundle install
bundle exec jekyll serve
```

Then open http://localhost:4000
