---
layout: post
title: "Git Local Ignore: Beyond .gitignore"
date: 2025-05-23
categories: [git, development, productivity]
tags: [git, gitignore, version-control, development-workflow, local-config]
---

While working on projects, I often need to create files for testing or experimentation that I don't want to push to the remote repository. At the same time, I don't want to add them to .gitignore, as that could introduce unintended changes for others or even for myself when cloning the project on another machine.

<!--more-->

Sometimes I need to create temporary files like `test_spider.py` or `sample.dev` to help with debugging or prototyping. These files aren't meant to be part of the codebase, and I don't want to commit them or include them in `.gitignore`, since that could cause unnecessary changes for others or even for myself when cloning the project later.

## The Solution: .git/info/exclude

To solve this, I use `.git/info/exclude`. It works like a local `.gitignore` that applies only to my machine and is not tracked by Git, any changes I make stay local and don't affect anyone else. This makes it ideal for keeping my personal or temporary files out of version control without touching the shared ignore rules.

## How It Works

The `.git/info/exclude` file follows the same syntax and patterns as `.gitignore`, but with a crucial difference: it's stored in the `.git` directory, which means it's local to your repository clone and never gets committed or shared with others.

## Example Usage

In my case, I added a single line to `.git/info/exclude`:

```
*gitignore.*
```

Now, any file matching that pattern; like `test_run_gitignore.py` is automatically ignored by Git. Clean, simple, and effective.

## When to Use .git/info/exclude

Use this approach when you have:

- **Personal debugging files**: Scripts or logs you create for troubleshooting
- **Local configuration overrides**: Settings specific to your development environment
- **Temporary test files**: Files created during experimentation that shouldn't be shared
- **Personal notes or documentation**: Files that are useful to you but not relevant to the project
- **IDE-specific files**: When you don't want to modify the shared `.gitignore` for your editor preferences

## Comparison with Other Ignore Methods

| Method | Scope | Tracked by Git | Use Case |
|--------|-------|----------------|----------|
| `.gitignore` | Repository-wide | Yes | Files that should always be ignored by everyone |
| `.git/info/exclude` | Local only | No | Personal files specific to your workflow |
| Global gitignore | All repositories | No | Personal patterns across all projects |

## Best Practices

1. **Keep it personal**: Only add patterns that are specific to your workflow
2. **Document patterns**: Add comments to explain complex patterns
3. **Regular cleanup**: Periodically review and clean up outdated patterns
4. **Share knowledge**: If you find a pattern that would benefit everyone, consider adding it to the shared `.gitignore`

## Example Patterns

Here are some useful patterns for `.git/info/exclude`:

```
# Personal debugging files
debug_*
test_*
*.local

# Personal notes
NOTES.md
TODO.personal

# Temporary files
*.tmp
*.temp
scratch.*

# Personal configuration
.vscode/settings.json
.idea/workspace.xml
```

## Conclusion

The `.git/info/exclude` file is a powerful but often overlooked Git feature that allows you to maintain personal ignore patterns without affecting your team's workflow. It's the perfect solution for keeping your repository clean while preserving the flexibility to create personal files for testing and experimentation.