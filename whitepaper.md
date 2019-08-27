# Overview of the package chain project

- Package chain is the first truly decentralized software distribution
  mechanism. Software distributed with package chain is guaranteed to work on
  any computer running a package chain client.
- Package chain blures the line between package management and incremental
  build caching, leading to reduced compile times and higher developer
  productivity.
- Getting package managers right is hard. Package chain comes with a reference
  package manager, which is ideal for new and niche programming languages to
  easily create a customized package manager, taking advantage of the registry
  and distributed, reproducible and incremental build cache.
- Package chain is initially intends to support nix, cargo and npm
  packages, giving it a large potential user base.

# Technical concept

## File layer
A flexible and performant ipfs implementation optimized for the package
management use case.

### Block store
The block store is located at `/ipfs/blocks/{cid}`. A file or directory is
encoded using unixfsv2 to a list of blocks. Blocks can be added to the store
concurrently. A db is maintained listing references to other blocks. Each
block is represented by a file. After writing a block it is marked read only to
prevent accidental modification.

### File system
The block store is mounted as a read only fuse file system at `/ipfs/store`.
The filesystem provides `/ipfs/store/{cid}/{path}` for efficiently reading
unixfsv2 encoded directories. During reads the block store is checked for
consistency, optionally hash rewriting is performed to allow self referential
store paths.

### Pinning and garbage collection
Users can pin blocks by creating a symlink in `/ipfs/pins/per-user/{user}/{prog}/{cid}`
to `/ipfs/store/{cid}`. Pinning a store path prevents it from being garbage
collected. By adding a level of indirection to a users home directory with
auto links `/ipfs/pins/per-user/{user}/{prog}/auto/{hash(path)}` the user can
conviniently manage pins manually. The garbage collector removes dangling
symlinks and dead store paths. A lock file `/ipfs/pins/lock` and temporary pins
`/ipfs/pins/temp/{pid}` are used to add items to the store while the garbage
collector is running.

## Build layer
A nix daemon is implemented using the ipfs backend, the nix package manager
is ported and the hello package builds.

### Isolated builds
Kernel namespaces are used to isolate the build from the system. The `unshare`
rust library does the heavy lifting. File systems need to be mounted into the
container and possibly resources constrained through cgroups.

### User environments
A user environment is a hierarchy of symlinks that mirrors the union of the
directory hierarchies of the installed components. These are stored in
generations `/nix/profiles/per-user/{user}/generations/{id}` to enable atomic
upgrades and rollbacks. The current user environment is symlinked from
`/home/{user}/.nix_profile` to `/nix/profiles/per-user/{user}/current` which
in symlinks to the active generation.

## Network layer
The file layer is extended with efficient p2p block sharing.

### Peer and block discovery
A distributed hash table is used to locate peers that have a block. Local peers
discovered through mdns are optimistically assumed to have any block.

### Block exchange
The graphsync protocol is used to request all child blocks of a given cid. Over
the bitswap protocol blocks are requested and exchanged.

## Blockchain layer
The blockchain layer is responsible for creating proofs that a derivation
results in an output. The basic roles in this process are authority, publisher
and substituter. An authority is a participant in the block minting process.
This is not a role specific to this chain but is one provided by the underlying
blockchain framework. A publisher is any network participant that submits
derivations to the chain. A substituter builds the derivations and creates a
proof of build. Publishers and substituters are light clients that communicate
with authorities, they are not required to store the entire blockchain state.

### Proof of build
Proof of build works through a cryptographic commitment scheme, where a number
of substituters provide various easily verifiable properties of the output that
are required to match. Care must be taken to make sure that those differences
are build independent, in case of a non-reproducible build. A combination of
the following properties could lead to enough redundancy that they can't
be guessed and non determinism can be detected. An empirical study must be
performed to assess suitability.

- retained dependencies: every output of a derivation has a set of retained
  dependencies. These are unlikely to change due to build non determinism. If
  only the inputs change but not the builder, this property might be guessable.
- size of the build output: Build non determinism can influence the size of the
  build output. An example would be a cpu supporting simd instructions or
  another form of cpu extension, and the substituters getting different output
  sizes. One possible solution would be to require substituters to have not
  only a compatible architecture but the exact same cpu model.
- file layout of the output: This is unlikely to change due to build non
  determinism.
- build log: Build parallelism can cause the build log to be out order

### Preventing collusion
Substituters are randomly selected based on a VRF. This makes hard for one
substituter to pose as multiple and only build the output derivation once.

### Preventing modification
To make sure that the substituter does not have time to modify the build in
any malicious way and assuming most substituters are honest, be take the
first finished substituter as the base build time measured in number of blocks.
and require that the other substituters finish in `f(build_time_base)`.

### Incentivising substituters
A basic incentivisation scheme would be to reward substituters by minting
tokens according to `payout(base_build_time)`. This basic scheme has some
downsides.

1. Incentivises substituters to delay reporting their results to increase their
payout.

Payout must reward the first substituter to commit more than the last. An
exponential payout function would incentivise substituters to build and commit
the derivation as quickly as possible. We end up with the following payout
function: `payout(rewardable_build_time(base_build_time, rank))`.

2. The inflation rate is unpredictable.

We introduce a fixed inflation rate. From the fixed inflation rate we calculate
a constant amount of tokens minted per block. We reward substituters
proportional to their rewardable build times.

### Publishing a derivation
1. Publisher sends a transaction to an authority. The transaction contains
the cid of the derivation, a sealed commitment to reveal the hash and the proof
of build. A small transaction fee is payed and an amount of funds are locked.
2. The authority minting the block fetches the derivation and verifies it is a
valid derivation. To prevent collusion it then randomly selects n substituters
out of the registered substituters using a VRF and includes the transaction in
the block including the selected substituters and the proof of the VRF.
3. All selected substituters build the derivation and send a transaction
containing a sealed commitment to reveal the hash and the proof of build. After
the first substituter commits all other substituters must commit within maximum
f(t).
4. After all substituters have commited, all substituters and the publisher
reveal their proofs. The proofs must be revealed within time t.
5. The proof of build is checked. Substituters that provided an invalid proof
are slashed. The remaining substituters are rewarded proportional to their
rewardable build time.

### Substituting a derivation
The chain is querried for the revealed hashes and fetches the package. The
package is checked to match the proof of build.

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

## Future
There are numerous possible extensions and applications.

### Support more platforms
While we initially intend to focus on linux users as nix has shown there are
no technical reasons why it cannot work on mac os. Windows support can take
advantage of the windows subsystem for linux (wsl). An android port is also
a possibility, but would require a larger effort due to nixpkgs collection
not been ported yet.

### Service management
As nixos and nixops have showed, package managers naturally extend to service
management and service deployment. Integrating package chain into nixos would
be a great application for package chain.

### Integration with other chains
Package chain can benefit from integration with golem to offload builds. The
file coin chain can be used for replication of packages.

### Build management
Support for low level build management to improve distributed build caching.
The rust compiler handles incremental builds internally. Distributing the
incremental build cache may improve build times further.

### Version control
All inputs are encoded in unixfsv2. There are some technical issues to
distributing git repositories on ipfs, due to git objects possibly being larger
than the size of an ipfs block. To improve tracing the heritage of a binary,
linking the tarball release to a version control commit is desirable. Ideally
this is achieved by designing a version control system that is distributable on
through ipfs.

# Economical concept

## Package Chain Token (PCT)
The token account is a core component of package chain. It serves multiple
purposes. It is used to reward good behaviour and punish bad behaviour. This
is essential for package chain to function correctly. It serves as a payment
system for consumption of package chain resources. This is essential for
package chain's security and availability, by preventing spam and DoS attacks.
Finally the package chain token is used to sustain the developement of package
chain.

## Initial coin distribution


# Team
We are not a random collection of people, but a strong team with a proven track
record of delivering.

## Developers
David Craven, Lead Software Engineer, co-founder
Experience packaging software for functional package managers. Contributed to
substrate - a blockchain framework, guix - a functional package manager and
implemented an ipfs node.

