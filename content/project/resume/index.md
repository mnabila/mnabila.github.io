+++
draft = false
date = '2026-03-15T09:58:37+07:00'
title = 'Resume'
type = 'project'
description = 'A minimal, warm-toned Hugo theme for personal resume and portfolio sites — built with Tailwind CSS v4, structured front matter, taxonomy-driven tech stacks, and SEO-first design.'
image = ''
repository = 'https://github.com/mnabila/resume'
languages = ['html', 'css', 'javascript']
tools = ['hugo', 'tailwindcss']
+++

Most Hugo themes for portfolios and resumes try to do everything. They ship with dozens of configuration options, multiple layout variants, and enough CSS to style a small framework. The result is a theme you spend more time configuring than actually writing content for.

This theme takes the opposite approach. It is a single-purpose Hugo theme designed around one workflow: define your resume data in structured front matter, write your projects and blog posts in markdown, and let the theme handle the rest. No page builder, no widget system, no theme options panel — just content and layout.

## Problem Background

I needed a personal site that served two purposes: a resume that I could point recruiters and colleagues to, and a portfolio where I could write about projects and technical topics. The requirements were specific:

- **Resume as structured data** — experience, education, and skills should be defined as front matter fields, not free-form markdown. This ensures consistent formatting and enables programmatic rendering (sorting by date, displaying technology badges, generating structured data for SEO)
- **Project and blog support** — individual pages with proper typography, syntax highlighting, table of contents, and metadata display
- **Cross-referencing via taxonomies** — clicking a technology like "Go" or "Docker" should show all projects and skills that use it, not just a tag cloud
- **Performance and SEO** — the site should load fast, rank well, and produce valid structured data for search engines
- **Minimal build tooling** — no webpack, no Vite, no complex build chain. Hugo and npm should be the only requirements

Existing Hugo themes either focused on blogging (with resume as an afterthought) or on resume display (with no blog support). The ones that tried to do both were over-engineered — dozens of config parameters, complex partial hierarchies, and CSS frameworks that required their own learning curve.

## Solution Overview

I built a Hugo theme from scratch that treats the homepage as a single-scroll resume composed from structured content sections, with full multi-page support for projects, blog posts, and skills. The visual design uses a warm, cream-toned palette that feels more like a printed document than a typical developer portfolio.

**Tech stack:** Hugo (>= 0.146.0), Tailwind CSS v4, vanilla JavaScript, Google Fonts (Fraunces, Manrope, JetBrains Mono)

**My role:** Sole developer — design, layout architecture, CSS system, SEO implementation, and accessibility

## System Architecture

The theme follows Hugo's standard layout structure with clear separation between content types:

```
resume/
├── archetypes/              # Content scaffolding templates
│   ├── blog.md
│   ├── education.md
│   ├── experience.md
│   ├── project.md
│   └── skill.md
├── assets/
│   ├── css/main.css         # Tailwind CSS v4 with custom prose styles
│   └── js/main.js           # IntersectionObserver scroll animations
├── layouts/
│   ├── _partials/
│   │   ├── head.html        # SEO, fonts, Open Graph, JSON-LD
│   │   ├── header.html      # Responsive nav with hamburger menu
│   │   ├── footer.html
│   │   ├── about.html       # Resume: about section
│   │   ├── experience.html  # Resume: work history
│   │   ├── education.html   # Resume: education
│   │   ├── skill.html       # Resume: skills grid
│   │   ├── project.html     # Resume: project previews
│   │   ├── blog.html        # Resume: blog previews
│   │   ├── pagination.html  # Reusable paginator component
│   │   ├── card/            # Blog and project card partials
│   │   ├── head/            # CSS and JS asset pipelines
│   │   └── icon/            # Inline SVG icon partials
│   ├── _markup/
│   │   ├── render-codeblock-mermaid.html  # Mermaid diagram support
│   │   └── render-image.html              # Lazy loading images
│   ├── blog/                # Blog list + detail layouts
│   ├── project/             # Project list + detail layouts
│   ├── skill/               # Skill list + detail layouts
│   ├── baseof.html          # Base template shell
│   ├── home.html            # Homepage resume composition
│   ├── taxonomy.html        # Taxonomy listing with previews
│   └── term.html            # Term page with type-aware cards
├── exampleSite/             # Complete working demo site
├── package.json             # Tailwind CSS v4 dependency
└── static/                  # Static assets
```

The homepage (`home.html`) composes the resume by sequentially rendering partials — about, experience, education, skill, project, blog — each separated by dashed `<hr>` dividers. This design means adding or removing a resume section is a one-line change in the template.

Each content type has its own layout directory with specialized templates:

| Content Type | List Page | Detail Page | Sidebar |
|---|---|---|---|
| Blog | Paginated cards | Article with prose styling | Table of contents |
| Project | Paginated cards | Article with prose styling | TOC + tech stack + repo link |
| Skill | Paginated cards | Article with prose styling | Languages and tools inline |

Resume sections (experience, education) are rendered only on the homepage — they do not have standalone list or detail pages because they exist as structured front matter, not long-form content.

## Key Features

- **Structured front matter for resume data** — experience entries define role, period, location, work mode, technologies, and tasks as front matter fields rather than markdown. The theme renders these programmatically with consistent formatting, sorted by `period_begin` descending
- **Taxonomy-driven tech stacks** — languages and tools are Hugo taxonomies, not just display strings. Each language and tool automatically gets its own listing page showing all projects and skills that reference it, enabling cross-referencing without manual linking
- **SEO with adaptive JSON-LD** — structured data adapts per content type: `Person` schema on the homepage, `BlogPosting` on blog articles, `CreativeWork` on projects. Full Open Graph and Twitter Card support on every page
- **Tailwind CSS v4 with Hugo native integration** — uses Hugo's `css.TailwindCSS` pipe instead of PostCSS, with PurgeCSS driven by `hugo_stats.json`. No separate build step needed
- **Lazy Mermaid diagram loading** — Mermaid.js is loaded from CDN only when a page contains a mermaid code block, using Hugo's `.Store` mechanism. Pages without diagrams pay zero cost
- **Scroll reveal animations** — subtle `translateY` + `opacity` animations triggered by IntersectionObserver, with automatic fallback for users who prefer reduced motion (`prefers-reduced-motion`)
- **Asset integrity in production** — CSS and JS are fingerprinted with SRI (Subresource Integrity) hashes in production builds, while development uses plain URLs for faster iteration
- **Responsive mobile-first layout** — hamburger navigation on mobile with vanilla JavaScript toggle. Single-column on mobile, multi-column grids and sidebars on desktop
- **Custom Chroma syntax highlighting** — monochromatic pastel color scheme tuned to match the warm-toned design, replacing the default Chroma colors
- **Accessibility** — skip-to-content link, semantic HTML, ARIA labels, and visible focus states throughout

## Technical Challenges and Solutions

**Tailwind CSS v4 integration with Hugo.** Tailwind CSS v4 introduced a new standalone architecture that dropped the PostCSS dependency. Hugo added native support via the `css.TailwindCSS` pipe, but the integration with PurgeCSS required a specific setup: Hugo generates a `hugo_stats.json` file listing all HTML classes used across the site, which is then mounted as an asset and referenced as `@source` in the CSS file. This ensures unused Tailwind utilities are stripped in production without a separate PurgeCSS configuration. The `hugo_stats.json` file is also configured as a cache buster, so CSS rebuilds automatically when new utility classes appear in templates:

```css
@import "tailwindcss";
@source "hugo_stats.json";
```

**Adaptive JSON-LD structured data.** Different page types require different schema.org types. The `head.html` partial uses Hugo's page context to determine the current content type and renders the appropriate JSON-LD block — `Person` for the homepage (with social links and job title), `BlogPosting` for blog articles (with author, date, description), and `CreativeWork` for projects (with repository URL and technologies). This is handled with conditional blocks in a single partial rather than separate templates per type, keeping the SEO logic centralized.

**Lazy loading Mermaid from CDN.** Loading Mermaid.js on every page would add unnecessary weight to pages that do not use diagrams. The solution uses Hugo's render hook system: a custom `render-codeblock-mermaid.html` hook converts mermaid code blocks into `<div class="mermaid">` elements and sets a `.Store` flag. In `baseof.html`, the Mermaid script and initialization are conditionally included only when that flag is set. The Mermaid theme itself is customized with warm-toned colors (cream backgrounds, muted borders) to match the site's visual identity.

**Content-type-aware taxonomy pages.** Hugo taxonomies by default render all tagged content with the same template. But this theme needs different card layouts: blog posts should show date and tags, while projects should show languages and tools. The `term.html` template dynamically selects between blog card and project card partials based on the taxonomy name — `language` and `tool` taxonomies render project cards, everything else renders blog cards. This makes the taxonomy pages visually consistent with their respective section pages.

**Consistent design language across code rendering.** The default Chroma syntax highlighting colors clash with the warm-toned design. A custom monochromatic color scheme was built from scratch, using muted golds, browns, and soft greens that complement the cream backgrounds. This extends to Mermaid diagrams, which are also themed with matching colors — an easily overlooked detail that significantly improves visual cohesion.

## Lessons Learned

**Structured front matter beats free-form content for resume data.** Making experience, education, and skills purely data-driven (with rendering handled by templates) was the most impactful design decision. It enforces consistency, enables sorting and filtering, and makes it trivial to add new entries — just create a markdown file with the right front matter fields.

**Hugo's asset pipeline is underrated.** By using Hugo's native `css.TailwindCSS` and `js.Build` pipes, the entire build process requires zero external build tools beyond npm for installing Tailwind. No webpack config, no Vite setup, no PostCSS pipeline — Hugo handles fingerprinting, SRI hashes, and minification out of the box. This keeps the development experience fast and the deployment simple.

**Taxonomies are more powerful than tags.** Treating languages and tools as first-class Hugo taxonomies instead of simple string arrays unlocked automatic cross-referencing pages. A visitor can click "Go" on any project and see every project and skill that uses Go. This emerged naturally from Hugo's taxonomy system without any custom logic.

**Design consistency requires intentional effort at every layer.** Matching the color palette across prose typography, Chroma syntax highlighting, Mermaid diagrams, and UI components required deliberate work. Each layer has its own configuration format and override mechanism. The result is a cohesive visual identity, but it was not free — every new rendering layer (code blocks, diagrams, tables) needed its own color tuning pass.

**Accessibility defaults should be baked in, not bolted on.** Adding skip-to-content links, semantic HTML, ARIA labels, `prefers-reduced-motion` checks, and focus states from the beginning was far easier than retrofitting them later. These are one-time decisions during initial development that benefit every page automatically.

## Conclusion

This theme powers [mnabila.com](https://mnabila.com) and is designed to be forked and adapted. Getting started requires Hugo (>= 0.146.0) and Node.js:

```bash
# Add as a submodule to your Hugo site
git submodule add https://github.com/mnabila/resume themes/resume

# Install Tailwind CSS
npm install --prefix themes/resume

# Run development server
hugo server -D

# Build for production
hugo --gc --minify
```

The theme includes a complete `exampleSite/` with sample content for every section — experience, education, skills, projects, and blog posts — along with a GitHub Actions workflow for deployment to GitHub Pages.

| Content Type | Archetype Command |
|---|---|
| Blog post | `hugo new blog/post-title/index.md` |
| Project | `hugo new project/project-name/index.md` |
| Skill | `hugo new skill/role-name.md` |
| Experience | `hugo new experience/company-name.md` |
| Education | `hugo new education/institution-name.md` |

The full source is available on [GitHub](https://github.com/mnabila/resume). It is a theme, not a starter — bring your own content and make it yours.
