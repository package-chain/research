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

### Derivation to output mapping
The build layer broadcasts built derivations via a pubsub protocol. Builders
maintain a set of trusted builders.

## Blockchain layer
The blockchain layer is responsible for creating proofs that an output is the
result of a derivation. The basic roles in this process are authority, publisher
and validator. An authority is a participant in the block minting process.
This is not a role specific to this chain but is one provided by the underlying
blockchain framework. A publisher is any network participant that submits a
claim, that an output is a valid output of a derivation. A validator builds a
derivation when requested by the authority and provides the result.

### Validation
This claim is verified in an iterated probabilistic prisoners dilema game using
the following algorithm:

```rust
struct Chain {
    /// Registered validators
    validators: Vec<Validator>,
    /// Probability that result is validated again
    p_continue: f32,
}

impl Chain {
    pub fn claim(&self, derivation: Cid, output: Cid) {
        let result = self.validate(derivation, output);
        self.apply_incentives(result);
    }
    
    fn validate(&self, derivation: Cid, output: Cid) -> Result {
        let validator = self.validators[random(self.validators.len())];
        let output2 = validator.build(derivation);
        if output == output2 {
            if random() > self.p_continue {
                self.validate(derivation, output)
            } else {
                Result::Valid
            }
        } else {
            if self.predict(output, output2) == Prediction::NonDeterminism {
                if random() > p_continue {
                    self.validate(derivation, output)
                } else {
                    Result::Valid
                }
            } else {
                Result::Misbehaviour
            }
        }
    }

    fn predict(output1: Cid, output2: Cid) -> Prediction;
    
    fn apply_incentives(&self);
}

enum Result {
    Valid,
    Misbehaviour,
}

enum Prediction {
    NonDeterminism,
    Misbehaviour,
}
```

### Prediction market
To determine if the differences between build output are due to non determinism
or a malicious actor a prediction market is consulted. Inspecting the results
should reveal the cause.

### Incentivising validators
A basic incentivisation scheme would be to reward substituters by minting
tokens according to `payout(build_time)`. This basic scheme has some downsides.

1. Incentivises substituters to delay reporting their results to increase their
payout.
2. The inflation rate is unpredictable.

To correct for this we keep track of validators efficiency factor, and introduce
a fixed inflation rate. We define the average efficiency factor to be 1. We take
the self reported build time `u_i` and compute the average build time `u_a`. Then
we calculate the efficiency factors `r_i` of each validator with `r_i = u_i / u_a`.
The average build time is added as the validators rewardable build time to the
current payment period. At the end of the payment period a constant amount of
tokens are minted according to the set inflation rate and are distributed
to the validators proportionally to the rewardable build times. If a validation
yields misbehaviour then all previous validators and the publisher are slashed.
The publisher is slashed for trying to distribute malware and the previous
validators are slashed for either claiming they built the derivation or copying
the output and purposefully faking non determinism, by for example manipulating
a time stamp.

### Publishing a derivation
1. Publisher sends a claim to an authority. A small transaction fee is payed
and an amount of funds are locked.
2. The authority minting the block includes the transaction and using a VRF
randomly assigns a validator from the validator pool. 
3. The validator builds the derivation and reports the output and the build
time to the chain.
4. The validation algorithm is run to determine if the validation is
sufficient. If no goes back to step 2.
5. After the validation algorithm returns, the rewardable build times are
computed and added to the payment period. If the result is misbehaviour, then
the publisher and all previous validators are slashed. This

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

