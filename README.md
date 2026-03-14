# MNABILA

Personal portfolio and resume website built with [Hugo](https://gohugo.io/) and deployed on [GitHub Pages](https://pages.github.com/).

**Live site**: [mnabila.com](https://mnabila.com)

## Tech Stack

- **Static Site Generator**: Hugo (v0.148.2+, extended)
- **CSS**: Tailwind CSS v4
- **Deployment**: GitHub Actions → GitHub Pages
- **Theme**: Custom `resume` theme

## Content

| Section    | Description                                    |
| ---------- | ---------------------------------------------- |
| About      | Professional summary and social links          |
| Experience | Work history with roles and technologies       |
| Education  | Academic background                            |
| Skills     | Technical skills grouped by role               |
| Projects   | Personal and professional project showcases    |
| Blog       | Technical articles on Linux and dev tooling    |

## Getting Started

### Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) >= 0.146.0
- [Node.js](https://nodejs.org/) >= 22
- [Go](https://go.dev/) >= 1.24

### Setup

```bash
git clone --recurse-submodules https://github.com/mnabila/mnabila.github.io.git
cd mnabila.github.io
npm install
```

### Development

```bash
npm run dev
```

Starts a local dev server with drafts enabled at `http://localhost:1313`.

### Build

```bash
npm run build
```

Generates the production site with garbage collection and minification.

## Adding Content

Hugo archetypes are available for each content type:

```bash
hugo new content blog/my-new-post.md
hugo new content experience/company-name.md
hugo new content project/project-name.md
hugo new content education/school-name.md
hugo new content skill/skill-category.md
```

## Project Structure

```
.
├── archetypes/        # Content templates
├── assets/            # Site-level assets (processed by Hugo Pipes)
├── content/           # Markdown content (blog, projects, experience, etc.)
├── static/            # Static files served as-is
├── themes/resume/     # Custom Hugo theme
├── hugo.toml          # Hugo configuration
└── package.json       # Node.js dependencies (Tailwind CSS)
```

## License

This project is dual-licensed:

- **Code** (theme, config, workflows): [MIT License](LICENSE)
- **Content** (blog posts, resume, project descriptions): [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)
