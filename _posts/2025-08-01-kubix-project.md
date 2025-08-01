---
title: "Kubix: A Smarter CLI Wrapper for kubectl"
classes: wide
header:
  teaser: /assets/images/posts/kubix-logo.png
  overlay_image: /assets/images/posts/kubix-logo.png
  overlay_filter: 0.3
ribbon: DarkSlateGray
excerpt: "Kubix is a Rust-powered CLI wrapper around kubectl that simplifies your Kubernetes workflow with smart aliases and powerful UX."
description: "Introducing Kubix - a lightweight, user-friendly CLI built in Rust to boost kubectl productivity. Learn why and how it was built, and how to use it."
categories:
  - Projects
  - Kubernetes
tags:
  - Rust
  - Kubernetes
  - CLI
  - Open Source
  - Dev Tools
  - kubectl
toc: true
toc_sticky: true
toc_label: "Kubix Guide"
toc_icon: "terminal"
---

# Meet Kubix 🧩

Kubix is a smart CLI wrapper around `kubectl`, written in *Rust*, that helps you run common Kubernetes commands faster and more intuitively.

👉 [GitHub Repo](https://github.com/orez-rj/kubix/tree/main)

## Why I Built It

I wanted to learn Rust. I also wanted to streamline my everyday work with `kubectl`, which often involves typing long repetitive commands and switching contexts or namespaces.

Sure, there are existing scripts and even tools like `kubectl ai`, but building *yet another wrapper* gave me the hands-on experience I needed, and a way to tailor the tool exactly to how I work. No better way to learn a language than solving a problem you care about.

## What Kubix Does

Kubix acts as a shortcut layer on top of `kubectl`. It offers:

- **Faster, cleaner commands** for common tasks (e.g., `kubix pods` instead of `kubectl get pods`)
- **Context-aware behavior** for easier switching between clusters and namespaces
- **Aliased commands** for logs, exec, describe, delete, and more
- **Powerful CLI UX** thanks to Rust's blazing performance and the [`clap`](https://docs.rs/clap/latest/clap/) framework
- **Extensibility** - future support planned for fuzzy search, natural language (LLM) inputs, and command history

Here’s a taste:

```bash
# List all pods
kubix pods

# Execute into a pod
kubix exec my-app

# View logs
kubix logs my-app
```

### Beautiful CLI Output (Yes, It Sparks Joy)

Kubix isn’t just fast - it’s also pleasant to use.

Using the excellent [`tabled`](https://docs.rs/tabled/) crate, Kubix formats Kubernetes objects (like pods, contexts, etc.) into clean, readable tables that make your terminal feel more like a dashboard.

Example:

```text
+------+---------------------+-------------+
| NAME |     NAMESPACE      |     AGE     |
+------+---------------------+-------------+
| web  | default             | 3h 21m ago  |
| api  | production-backend  | 7m 42s ago  |
+------+---------------------+-------------+
```

There’s also interactive selection using pattern matching and fuzzy input.

```bash
kubix exec
```

This command will show a numbered list of pods matching your query, and let you select one easily by number or name:

```text
Which pod do you want to exec into?

[0] web-5f68d7b9fd-xk4zb
[1] api-57c4b66dc5-tkp8n
[2] db-66ccfcbf55-72vvn

Type number or pattern:
> 1
```

This kind of UX makes `kubectl` feel almost... friendly. 😄

### Automated Cross-Platform Releases

One thing I really wanted was to make Kubix **zero-effort to install** on any system.

To achieve that, I set up a fully automated GitHub Action that:

* Detects version bumps via `cargo release`
* Builds static binaries for Linux, macOS, and Windows
* Attaches them to a new GitHub release
* Updates the install script

All of this happens on push to `main` after merging `dev`, so cutting a new release is as easy as:

```bash
cargo release minor
```

Kubix is ready-to-go for anyone, anywhere - no Rust toolchain required. Just download, drop it in your `$PATH`, and start kubing like a pro.

## Under the Hood

Kubix is built using:

* 🦀 **Rust** - For performance and safety
* 🧼 **Clap** - To parse and structure CLI commands cleanly
* 📦 **cargo-release + GitHub Actions** - For versioned multi-platform releases
* 💻 **Cross-platform support** - Tested on Linux and macOS (Windows support planned)

If you want to understand how it works or contribute, check out the source: [orez-rj/kubix](https://github.com/orez-rj/kubix)

## How to Install

Download the latest binary from the [Releases](https://github.com/orez-rj/kubix/releases) page or install with `cargo`:

```bash
cargo install kubix
```

Or use the installer script:

```bash
curl -sSL https://raw.githubusercontent.com/orez-rj/kubix/main/install.sh | bash
```

## Roadmap

* [ ] Add natural language support (`kubix exec to shell in frontend pod`)
* [ ] Regex matching pod names
* [ ] Improve Windows support
* [ ] Optional LLM integration

## Final Thoughts

Kubix is my way of learning Rust while scratching a real itch in my Kubernetes workflow. It’s still growing, and contributions are welcome!

> I’ll probably never stop tweaking it-but that’s half the fun.

---

🧑‍💻 **Project by:** [Or Ezra (orez-rj)](https://github.com/orez-rj)
📁 **Repo:** [github.com/orez-rj/kubix](https://github.com/orez-rj/kubix)
