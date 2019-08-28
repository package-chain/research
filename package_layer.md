## Package layer
The blockchain layer is extended with the concept of packages and repositories.
Cargo and npm are extended to work with the build and package layer.

### Data model
A repository is a named collection of packages. Repositories just like
packages can be versioned. If a repository does not publish versions it is
a rolling release.

```rust
struct Chain {
    repositories: HashMap<Name, Repository>,
    packages: HashMap<PackageId, Package>,
}

struct Repository {
    meta: HashMap<Version, Cid>,
    yanked: HashMap<Version, bool>,
    packages: HashMap<Name, PackageId>,
}

struct Package {
    meta: HashMap<Version, Cid>,
    yanked: HashMap<Version, bool>,
    src: HashMap<Version, Cid>,
}
```

### Package file
```toml
[package]
repository = "crates"
name = "hello"
version = "1.0.0"
license = "ISC"

meta = ["Cargo.toml", "deps/Cargo.toml"]

src = [
  "Cargo.toml",
  "src/main.rs",
  "src/lib.rs",
]

[meta-inputs]
nixpkgs:cargo-instantiate = "*"

[native-inputs]
nixpkgs:rustc = ">=1.36"
nixpkgs:cargo = "*"

[inputs]
crates:dep = "0.2"
```

### The cli interface
Package identifiers have the following format:
`{repo}:{version}/{pkg}:{version}`. Versions are optional.
All cli commands take an optional package identifier. If the identifier is not
specified a `pkg.toml` in the current working directory is used.

```sh
pkg fetch
pkg lock
pkg instantiate
pkg build
pkg install
pkg publish
pkg update
pkg uninstall
pkg yank
```

```sh
repo fetch
repo lock
repo instantiate
repo build
repo publish
repo yank
```

### Mirroring nixpkgs, crates.io and npm
While the case studies show how it should work, in practice we will have to
mirror the existing repositories, so that developers can start using it.

Automated tools must be developed to import packages on chain and sources into
ipfs.

### Continuous integration
If an input changes the package needs to be rebuilt. An automated mechanism
for listening for dependencies being updated, and rebuilding and republishing
the package is required. For repos if a repo package changes the repo lock
is recomputed and the repo rebuilt. If the build succeeds, a new repo version
is published.
