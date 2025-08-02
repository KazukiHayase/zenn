# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Zenn content repository containing technical articles and documentation. Zenn is a Japanese tech publishing platform. The repository uses Zenn CLI for content management and publishing.

## Common Commands

### Content Management
```bash
# Preview articles locally
pnpm preview

# Create a new article
pnpm new:article

# Format code and markdown
pnpm format
```

### Package Management
- Uses `pnpm` as the package manager (version 10.8.0+)
- Dependencies are managed in `package.json`

## Repository Structure

### Core Directories
- `articles/` - Published and draft Zenn articles in Markdown format
- `books/` - Zenn book content (if any)
- `images/` - Article images organized by article ID subdirectories
- `docs/` - Internal documentation including writing style guide
- `proposals/` - Draft content and conference proposals

### Key Files
- `docs/writing-style.md` - Comprehensive style guide for maintaining consistent writing tone and technical focus as a full-stack engineer
- `articles/outline.md` - Article ideas and outlines for AI-driven development workflows
- Article files use Zenn's slug-based naming convention (e.g., `042ab9ba855b77.md`)

## Writing and Content Guidelines

### Article Creation Process
1. **Planning**: Use outline files to structure article ideas, particularly focusing on AI-driven development workflows
2. **Writing Style**: Follow the established style guide in `docs/writing-style.md` which defines:
   - Technical expertise across full-stack development (React/TypeScript, Go, AWS, GraphQL, etc.)
   - Conversational "です・ます" tone with personal experience examples
   - Structure: Introduction (background/challenges) → Implementation → Conclusion
   - Characteristic phrases like "重宝しています", "〜していきます", "〜かなと思います"

### Content Focus Areas
Based on existing articles, the main technical areas covered include:
- GraphQL and Apollo Client implementation patterns
- React/TypeScript development with modern tooling
- Go backend development and AWS integration
- Developer productivity tools (Raycast, CLI tools, AI assistance)
- Claude Code and AI-driven development workflows

### Image Management
- Images are stored in `images/{article-id}/` subdirectories
- Includes screenshots, GIFs demonstrating functionality, and diagrams
- Use descriptive filenames for easy reference

## Development Workflow Integration

The repository supports AI-driven development workflows where Claude Code is integrated throughout the entire development process from requirements definition through QA and PR creation, as outlined in the development process documentation.