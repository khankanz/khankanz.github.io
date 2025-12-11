# Zardar Khan — Personal Website

A high-impact personal website designed to convert visitors into connections.

## Structure

```
├── index.html          # Landing page (hero, proof points, featured work)
├── resume.html         # HTML resume with PDF download
├── blog.html           # Blog listing page
├── _config.yml         # Jekyll configuration
├── _layouts/
│   └── post.html       # Custom blog post layout
├── _posts/             # Your existing blog posts (copy from current repo)
└── assets/
    └── Zardar_Khan_Resume.pdf
```

## Deployment

1. Copy these files to your `khankanz.github.io` repository
2. Copy your existing `_posts/` folder
3. Commit and push
4. GitHub Pages will automatically build and deploy

## What Changed

**Before:** Generic Jekyll blog with post list
**After:** 
- Hero section with punchy positioning + availability indicator
- Proof strip with quantified achievements (700x, 17K, 3wk, 90%)
- Featured projects with impact metrics
- Clean blog listing with tags
- HTML resume + PDF download
- Consistent dark theme with technical aesthetic

## Customization

- **Colors:** Edit CSS variables in `:root` (accent color is `#4ade80`)
- **Content:** Update proof numbers, projects, and about section in `index.html`
- **Resume:** Edit `resume.html` directly, re-upload PDF to `/assets/`

## Notes

- All pages use inline CSS (no external dependencies except Google Fonts)
- Print-friendly resume (colors adjust for printing)
- Mobile responsive
- Jekyll will process the blog posts with the custom `post.html` layout
