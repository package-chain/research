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

# Roadmap

## File layer
A flexible and performant ipfs implementation optimized for the package
management use case.

### Block store
The block store is located at `/ipfs/blocks/{cid}`. A file or directory is
encoded using unixfsv2 to a list of blocks. Blocks can be added to the store
concurrently with a db for bookkeeping references to other blocks. Each block
is represented by a file. After writing a block it is marked read only to
prevent accidental modification.

### File system
The block store is mounted as a read only fuse file system at `/ipfs/store`.
The filesystem provides `/ipfs/store/{cid}/path` for efficiently reading
unixfsv2 encoded directories. During reads the block store is checked for
consistency, optionally hash remapping is performed to allow self referential
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
The nix daemon is implemented using the ipfs backend, the nix package manager
is ported and the hello package builds.

### Isolated builds
TODO

### User environments
A user environment is a hierarchy of symlinks that mirrors the union of the
directory hierarchies of the installed components. These are stored in
generations `/nix/profiles/per-user/{user}/generations/{id}` to enable atomic
upgrades and rollbacks. The current user environment is symlinked from
`/home/{user}/.nix_profile` to `/nix/profiles/per-user/{user}/current` which
in symlinks to the active generation.

## Network layer
The file layer is extended with efficient p2p block sharing. The build layer is
extended with a blockchain to map build inputs to build outputs.

### Peer and block discovery
A distributed hash table is used to locate peers that have a block. Local peers
discovered through mdns are optimistically assumed to have any block.

### Block exchange
The graphsync protocol is used to request all child blocks of a given cid. Over
the bitswap protocol blocks are requested and exchanged.

## Blockchain layer
Substituters publish a mapping of equivalence classes to outputs and
participate in an incentivised voting mechanism to determine which builds are
reproducible.

### Voting mechanism
TODO

### Non-reproducible builds
A mapping for non-reproducible builds is found by checking if:
- a trusted substituter has published a mapping
- the package maintainer has published a mapping
- the signer of the release tarball or commit has published a mapping
If no substitution was found the package is built locally.


```md
## Package layer
TODO:
Requires further research and planning. Preliminary objectives:

### Package registry
TODO:
on-chain package metadata and dependency graph

### Locking graphs
TODO:
language package manager: single package lock file.
distro package manager: multi package lock file.
```

## Future
There are numerous possible extensions and applications.

### Integration with other chains
Package chain can benefit from integration with golem to offload builds. The
file coin chain can be used for replication of packages.

### Service management
As nixos and nixops have showed, package managers naturally extend to service
management and service deployment.

### Build management
Support for low level build management enables support for incremental builds
and distributed build caching. Specifically companies can benefit from this
feature over mdns, leading to a reduction in build times.

# Funding
TODO

# Team
We are not a random collection of people, but a strong team with a proven track
record of delivering.

## Developers
David Craven, Lead Software Engineer, co-founder
Experience packaging software for functional package managers. Contributed to
substrate - a blockchain framework, guix - a functional package manager and
implemented an ipfs node.

