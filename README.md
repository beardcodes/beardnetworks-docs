# Beardnetworks Docs

A personal technical knowledge base and blog built with **Zensical** (by the creators of Material for MkDocs).

This repository contains guides, notes, tutorials, homelab documentation, self-hosting projects, security write-ups, and things I learn while building and maintaining my infrastructure.

## 🌐 Website

The documentation is published as a static website using **Zensical** and deployed automatically from this repository.

## 📚 Topics

Content currently includes:

* 🏠 Home Server & Homelab
* 🐳 Docker
* 🌐 Networking
* 🔐 Security
* 🧪 Hack The Box
* 🖥️ Self-Hosting
* ⚙️ DevOps
* 📝 Technical Blog Posts

More guides and notes will be added over time.

## 🗂 Repository Structure

```text
.
├── docs/
│   ├── assets/
│   ├── hack-the-box/
│   ├── home-server/
│   └── index.md
├── mkdocs.yml
├── requirements.txt            # MkDocs fallback path
├── requirements-zensical.txt   # Zensical build path
└── README.md
```

All website content is written in Markdown inside the `docs/` directory.

## 🚀 Run Locally

### 1. Clone the repository

```bash
git clone https://github.com/beardcodes/mkdocs.git
cd mkdocs
```

### 2. Create a virtual environment

```bash
python -m venv .venv
```

Activate it:

**Windows**

```bash
.venv\Scripts\activate
```

**Linux / macOS**

```bash
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements-zensical.txt
```

### 4. Start the dev server

```bash
zensical serve
```

Open:

```text
http://127.0.0.1:8000
```

The server automatically reloads when Markdown files are changed.

### Fallback: MkDocs

Zensical reads the same `mkdocs.yml`, so the original MkDocs toolchain still
builds this site. Use a **separate** virtual environment for it -- the two
requirement sets pin conflicting versions of `Markdown`, `Jinja2` and
`pymdown-extensions`.

```bash
pip install -r requirements.txt
mkdocs serve
```

## ✍️ Adding Content

Create Markdown files inside the appropriate folder:

```text
docs/home-server/
docs/hack-the-box/
```

For example:

```text
docs/home-server/docker-compose.md
```

Then add the page to the navigation inside `mkdocs.yml`.

```yaml
nav:
  - Home: index.md
  - Home Server:
      - Docker Compose: home-server/docker-compose.md
```

## 🏗 Build

Generate the static website with:

```bash
zensical build
```

The generated website will be placed inside:

```text
site/
```

The `site/` directory should not be committed to Git.

## ☁️ Deployment

The website can be deployed automatically using services such as:

* Cloudflare Pages
* GitHub Pages
* Netlify
* Vercel

The deployment service only needs to run:

```bash
pip install -r requirements-zensical.txt
zensical build
```

and publish the generated:

```text
site/
```

directory.

## 🛠 Built With

* [Zensical](https://zensical.org/)
* [MkDocs](https://www.mkdocs.org/) + [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) (fallback)
* Markdown
* GitHub

## 📄 License

Content in this repository is provided for educational and documentation purposes.

---

Built and maintained by **Beardnetworks**.
