# rust-note

Source code reading notes for [rust-lang/rust](https://github.com/rust-lang/rust).

The upstream compiler source lives in [`./upstream`](./upstream) as a git submodule
(shallow clone). My reading notes live in [`./notes`](./notes).

## Layout

```
.
├── upstream/   # git submodule -> rust-lang/rust (shallow)
└── notes/      # source reading notes (see notes/README.md)
```

## Setup

After cloning this repository, initialize the submodule:

```sh
git submodule update --init --depth 1 upstream
```

### Updating the upstream snapshot

The submodule is a shallow clone pinned to a single commit. To move it forward:

```sh
cd upstream
git fetch --depth 1 origin master
git checkout FETCH_HEAD
cd ..
git add upstream
git commit -m "chore: bump upstream snapshot"
```

### Converting to a full clone (if history/blame is needed later)

```sh
cd upstream
git fetch --unshallow
```

## How we read

Notes are written collaboratively while reading. Each note captures a question,
the relevant code (with `path:line` references into `upstream/`), and the
explanation. See [`notes/README.md`](./notes/README.md) for the index.
