# my-first-repos
its new start
📦 quickhash-cli/
├─ README.
├─ LICENSE
├─ .gitign
├─ pyproject.tom
├─ src/
│  └─ quickhas
│     ├─ __ini
│     └─ cli.p
├─ test
│  └─ test_cli.p
└─ .github
   └─ workflows
      └─ ci.yml

=======================
README.md
========================
# QuickHash CLI

A tiny Pthon CLI to hash files or folders using `md5`, `sha1`, `sha256`, or `blake2b`, with pretty output. Includes CI, tests, and MIT license.

## Install
```bash
pip install -e .[dev]
```

## Usage
```bash
# Hash a file
quickhash file path/to/file --alg sha256

# Hash all files under a directory (recursively)
quickhash dir ./data --alg md5 --recursive

# Verify hashes from a checksum file
quickhash verify checksums.txt --alg sha256
```

## Examples
```bash
quickhash file README.md --alg sha256
quickhash dir . --alg blake2b --recursive
quickhash verify sample.sha256 --alg sha256
```

## Development
```bash
python -m venv .venv && source .venv/bin/activate
pip install -e .[dev]
pytest -q
flake8
```

## License
MIT

========================
LICENSE
========================
MIT License

Copyright (c) 2025 <Your Name>

Permission is hereby granted, free of charge, to any person obtaining a copy
...

========================
.gitignore
========================
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
build/
dist/
*.egg-info/
.eggs/
.vscode/
.idea/
.DS_Store
Thumbs.db

========================
pyproject.toml
========================
[build-system]
requires = ["hatchling>=1.25.0"]
build-backend = "hatchling.build"

[project]
name = "quickhash-cli"
version = "0.1.0"
description = "Tiny CLI to hash files/folders with pretty output"
readme = "README.md"
requires-python = ">=3.10"
authors = [
  { name = "Your Name", email = "you@example.com" }
]
license = { text = "MIT" }
dependencies = [
  "typer>=0.12",
  "rich>=13.0",
]

[project.scripts]
quickhash = "quickhash.cli:main"

[project.optional-dependencies]
dev = [
  "pytest>=8.0",
  "flake8>=7.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/quickhash"]

[tool.flake8]
max-line-length = 100

========================
src/quickhash/__init__.py
========================
__all__ = ["__version__"]
__version__ = "0.1.0"

========================
src/quickhash/cli.py
========================
from __future__ import annotations
import hashlib
import os
from pathlib import Path
from typing import Iterable, Optional

import typer
from rich.console import Console
from rich.progress import Progress

app = typer.Typer(add_completion=False, help="QuickHash: hash files & folders with pretty output")
console = Console()

ALG_MAP = {
    "md5": hashlib.md5,
    "sha1": hashlib.sha1,
    "sha256": hashlib.sha256,
    "blake2b": hashlib.blake2b,
}

def iter_files(root: Path, recursive: bool) -> Iterable[Path]:
    if root.is_file():
        yield root
        return
    if recursive:
        for dirpath, _, filenames in os.walk(root):
            for name in filenames:
                yield Path(dirpath) / name
    else:
        for p in root.iterdir():
            if p.is_file():
                yield p

def hash_file(path: Path, alg: str, chunk_size: int = 1024 * 1024) -> str:
    h = ALG_MAP[alg]()
    with path.open("rb") as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            h.update(chunk)
    return h.hexdigest()

@app.command()
def file(
    path: Path = typer.Argument(..., exists=True, file_okay=True, dir_okay=False, readable=True),
    alg: str = typer.Option("sha256", "--alg", case_sensitive=False, help=f"Algorithm: {', '.join(ALG_MAP)}"),
):
    """Hash a single FILE."""
    alg = alg.lower()
    if alg not in ALG_MAP:
        raise typer.BadParameter(f"Unsupported algorithm: {alg}")
    digest = hash_file(path, alg)
    console.print(f"[bold]Algorithm:[/bold] {alg}")
    console.print(f"[bold]File:[/bold] {path}")
    console.print(f"[bold]Hash:[/bold] {digest}")

@app.command()
def dir(
    path: Path = typer.Argument(Path("."), exists=True, file_okay=False, dir_okay=True, readable=True),
    alg: str = typer.Option("sha256", "--alg", case_sensitive=False),
    recursive: bool = typer.Option(False, "--recursive", "-r", help="Recurse into subdirectories"),
):
    """Hash all files in a DIRECTORY."""
    alg = alg.lower()
    if alg not in ALG_MAP:
        raise typer.BadParameter(f"Unsupported algorithm: {alg}")

    files = list(iter_files(path, recursive))
    if not files:
        console.print("[yellow]No files found.[/yellow]")
        raise typer.Exit(code=1)

    with Progress() as progress:
        task = progress.add_task("Hashing", total=len(files))
        for fp in files:
            try:
                digest = hash_file(fp, alg)
                console.print(f"{alg}  {digest}  {fp}")
            except Exception as e:
                console.print(f"[red]Error:[/red] {fp}: {e}")
            finally:
                progress.advance(task)

@app.command()
def verify(
    checksum_file: Path = typer.Argument(..., exists=True, file_okay=True, dir_okay=False, readable=True),
    alg: Optional[str] = typer.Option(None, "--alg", case_sensitive=False, help="Required if file lines omit the alg prefix"),
):
    """Verify hashes from a checksum FILE."""
    if alg is not None:
        alg = alg.lower()
        if alg not in ALG_MAP:
            raise typer.BadParameter(f"Unsupported algorithm: {alg}")

    ok, bad, total = 0, 0, 0
    for line in checksum_file.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        parts = line.split()
        if len(parts) < 2:
            console.print(f"[yellow]Skipping:[/yellow] {line}")
            continue
        if len(parts) == 2:
            if not alg:
                raise typer.BadParameter("--alg is required when lines omit the algorithm")
            expected, path = parts
            this_alg = alg
        else:
            this_alg, expected, path = parts[0].lower(), parts[1], " ".join(parts[2:])
            if this_alg not in ALG_MAP:
                console.print(f"[yellow]Skipping (unknown alg):[/yellow] {line}")
                continue

        total += 1
        fp = Path(path)
        if not fp.exists():
            console.print(f"[red]Missing:[/red] {fp}")
            bad += 1
            continue
        actual = hash_file(fp, this_alg)
        if actual == expected:
            console.print(f"[green]OK[/green] {this_alg}  {fp}")
            ok += 1
        else:
            console.print(f"[red]FAIL[/red] {this_alg}  {fp}")
            bad += 1

    console.print(f"\n[bold]Summary:[/bold] {ok} OK, {bad} FAIL, {total} total")
    raise typer.Exit(code=0 if bad == 0 else 1)

def main():
    app()

if __name__ == "__main__":
    main()

========================
tests/test_cli.py
========================
import subprocess
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]

def run_cmd(args):
    return subprocess.run(args, capture_output=True, text=True)

def test_cli_help():
    r = run_cmd([sys.executable, "-m", "quickhash", "--help"])
    assert r.returncode == 0
    assert "QuickHash" in r.stdout

def test_hash_file(tmp_path):
    sample = tmp_path / "sample.txt"
    sample.write_text("hello")
    r = run_cmd(["quickhash", "file", str(sample), "--alg", "sha256"])
    assert r.returncode == 0
    assert "sha256" in r.stdout.lower()

========================
.github/workflows/ci.yml
========================
name: CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install
        run: |
          python -m pip install --upgrade pip
          pip install -e .[dev]
      - name: Lint
        run: flake8
      - name: Tests
        run: pytest -q
