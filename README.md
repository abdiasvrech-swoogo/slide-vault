<p align="center">
  <img src="assets/logo/logo-full.svg" alt="Swoogo" width="200">
</p>

# 📊 Slide Vault

I hate reading walls of text in Notion. Slides are just easier to follow – they're visual, they keep things focused, and they communicate better. So I built this to turn ideas into presentations fast.

## The Setup

I tell an AI what I want to present, it generates the slides as a web page, and I push it live. No dragging around boxes, no fighting with templates. Just content to slides in minutes.

Everything lives in git and gets hosted automatically on GitHub Pages. Clean URLs, version controlled, easy to share.

## Why Slides Over Docs?

Simple: I'd rather skim 10 slides than read 10 pages. Slides force you to be concise and make it way easier to follow along. They're better for sharing ideas quickly and actually keeping people's attention.

Plus, I can update them on the fly if I need to fix something or add more context later.

## How to Use

Check out [The Presentation Master Prompt](templates/master-prompt.md) for the AI instructions I use to generate presentations. It includes the brand style, tone guidelines, and structure template.

## Project Structure

```
slide-vault/
├── assets/
│   ├── logo/             # logo-full.svg, logo-icon.png
│   └── brand/            # tokens.json (colors, typography)
├── presentations/        # Generated slide decks
│   └── {slug}/
│       ├── index.html    # The presentation
│       └── image.png     # Social preview
├── templates/
│   ├── master-prompt.md  # AI instructions for generating slides
│   └── skeleton.html     # Starter HTML with navigation engine
└── README.md
``` 
