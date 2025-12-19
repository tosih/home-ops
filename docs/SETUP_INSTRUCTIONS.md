# GitHub Pages Setup Instructions

## Enable GitHub Pages

To enable the documentation site, follow these steps:

1. **Go to your repository settings**
   - Navigate to `https://github.com/tosih/home-ops/settings/pages`

2. **Configure GitHub Pages**
   - Under "Build and deployment"
   - Source: **GitHub Actions**
   - Click **Save**

3. **Trigger the first deployment**
   ```bash
   git add -A
   git commit -m "docs: setup MkDocs documentation site"
   git push
   ```

4. **Wait for deployment**
   - Go to the **Actions** tab in your repository
   - Watch the "Deploy Documentation" workflow
   - Takes about 2-3 minutes

5. **Access your documentation**
   - Once deployed, visit: **https://tosih.github.io/home-ops**

## Local Development

To preview the documentation locally before pushing:

### Install MkDocs

```bash
pip install mkdocs-material
pip install mkdocs-git-revision-date-localized-plugin
```

### Serve locally

```bash
# Start development server
mkdocs serve

# Open browser to http://127.0.0.1:8000
```

### Build for production

```bash
# Build static site
mkdocs build

# Output will be in site/ directory
```

## Automatic Deployment

The documentation automatically rebuilds and deploys when you push changes to:

- `docs/**` - Any documentation files
- `mkdocs.yml` - Configuration file
- `.github/workflows/docs.yaml` - Workflow file

## Customization

### Update site info

Edit `mkdocs.yml`:

```yaml
site_name: Your Cluster Name
site_description: Your description
site_author: Your Name
site_url: https://yourusername.github.io/home-ops

repo_name: yourusername/home-ops
repo_url: https://github.com/yourusername/home-ops
```

### Add new pages

1. Create markdown file in `docs/` directory
2. Add to navigation in `mkdocs.yml`:

```yaml
nav:
  - Home: index.md
  - Your Section:
      - Your Page: section/your-page.md
```

### Change theme colors

Edit `mkdocs.yml`:

```yaml
theme:
  palette:
    primary: indigo  # blue, green, red, etc.
    accent: indigo
```

## Troubleshooting

### Build fails

Check the Actions tab for error details. Common issues:

- Invalid YAML in `mkdocs.yml`
- Missing markdown files referenced in nav
- Broken internal links

### Site not updating

- Check the Actions workflow completed successfully
- GitHub Pages can take 1-2 minutes to update after deployment
- Try a hard refresh (Ctrl+Shift+R)

### Local preview issues

```bash
# Clear cache and rebuild
rm -rf site/
mkdocs build

# Reinstall dependencies
pip install --force-reinstall mkdocs-material
```

## Features Included

✅ **Material Design** - Modern, responsive theme
✅ **Dark Mode** - Automatic light/dark switching
✅ **Search** - Full-text search built-in
✅ **Code Highlighting** - Syntax highlighting for code blocks
✅ **Git Info** - Last updated dates on pages
✅ **Mobile Friendly** - Responsive design
✅ **Navigation** - Tabbed navigation and table of contents

## Documentation Structure

```
docs/
├── index.md                          # Home page
├── getting-started/
│   ├── overview.md                   # Cluster overview
│   ├── prerequisites.md              # Required tools/accounts
│   ├── initial-setup.md              # Deployment guide
│   └── post-installation.md          # Post-install tasks and upgrades
└── reference/
    ├── architecture.md               # System architecture
    ├── authentication.md             # OIDC/SSO configuration
    └── tools.md                      # Tool reference
```

Add new sections as needed by creating directories and updating `mkdocs.yml`.
