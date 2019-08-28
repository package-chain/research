# The verified build chain project
- Build chain is the first truly decentralized software distribution mechanism.
  Binaries distributed with package chain are guaranteed to work on any computer
  running a build chain client.
- Binaries are verified on chain. There is a trustless link between the sources,
  the binary and the build process.
- Build chain is a decentralised and distributed backend for package managers,
  build caches and ci systems.

# Problem statement
Developer often face two significant issues hampering their productivity.

1. Long build times of large projects can significantlly cause lost
   productivity. Developers are most productive when they have short
   compile-test cycles.
2. Deployment of large codebases in uncontrolled environments can
   be an annoying and painstaking process. This happens when the developer
   envionment does not match the deployment environment.

# Existing approaches to these problems

## Nix package manager
The nix package manager (Eelco Dolstra PhD 2006) was a break through, offering
a completely new way of thinking about and performin deployment of software.
Unlike prevous solutions, complete and correct binary deployment to any system
running a linux kernel or mac os was possible.
Saddly it has not found much adoption outside of the nix and guix projects. The
nix deployment system fails to integrate with existing developer workflows,
finding only niche application in the deployment of curated collections of
software maintained by the nix and guix communities. These curated collections
are expensive and time consuming to maintain and the build, storage and
bandwidth costs are beared by the curators. It also suffers from centralisation
of trust. Users must trust that the centralised manifest and the build
infrastructure have not been breached.

## ccache and distcc
Ccache and sccache are distributed build caches. They work generally by hashing
the inputs to a compiler and storing the outputs in a cache. If the compiler is
invoked with the same inputs the result is retrieved from the cache.
This mostly useful for batch compilers running that compile individual files. A
more modern approach is for compilers to support incremental compilation,
reducing the performance gain of such build caches, since they are very coarse
grained. Distcc distributes a batch of coarse grained compiler invokations to
multiple machines. This approach also suffers from being coarse grained working
at the level of files.

## IPFS
IPFS (Juan Benet 2014) is a decentralised and distributed content addressed
block store. It is a novel synthesis of distributed hash tables and merkle
dags and combined with a new incentivised block exchange protocol to create a self
certified, deduplicated and content addressed block store. Blocks are used to
fetch data from untrusted peers, by imposing an upper bound on their size. This
prevents a malicious peer from continuously sending a stream of data claiming
it is going to match a hash.
For distributing binaries using the nix deployment model, pure content
addressing is insuficcient. A binary may and often does contain self references.
For the distribution of binaries the ipfs implementation must keep track of
those self references and transparently rewrite them when accessing a block.
There are also implementation issues, due to different focus of the ipfs team.
The fuse file system is slow, effort was spent instead on an ipfs http gateway,
and the development and specification of a complex rpc and cli interface. Using
a fuse implementation and an existing static web server would have reduced or
eliminated the need for those things.


# Decentralised and distributed build, cache and distribution system

## File layer

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
The file layer is extended with efficient p2p block sharing. The network shim
is replaced with an actual implementation. A pubsub network is used to
broadcast successful builds.

### Peer and block discovery
A distributed hash table is used to locate peers that have a block. Local peers
discovered through mdns are optimistically assumed to have any block.

### Block exchange
The graphsync protocol is used to request all child blocks of a given cid. Over
the bitswap protocol blocks are requested and exchanged.

### Derivation to output mapping
The build layer broadcasts built derivations via a pubsub protocol. Builders
maintain a set of trusted builders.


# Eliminating trust

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

## Build Chain Token (BCT)
The token account is a core component of build chain. It serves multiple
purposes. It is used to reward good behaviour and punish bad behaviour. This
is essential for build chain to function correctly. It serves as a payment
system for consumption of build chain resources. This is essential to build
chain's security and availability properties. Finally the build chain token
is used to sustain the developement of build chain.

## Initial coin distribution

# Integration with existing ecosystems

## Distributed nix
The nix package manager should be easily portable to a new backend. Storage and
bandwidth is provided by nix users, trustless verified binaries can be
distributed.

## Distributing binaries with cargo
Cargo already supports emiting a build plan. A tool is written to convert that
build plan into a set of derivations, that can be built and published using
package chain.

## Integration with other chains
The build layer can use golem to offload builds and file coin to replicate the
results.

# Future
For adding traceability from packages and version control to derivations
two future additions are proposed.

## Package registry
To trace a derivation back to a package definition a generic package registry
is required. To make a package registry generic we can delegate creating
derivations to a language specific tool. The package registry should support
multiple repositories that are versionable. Versioning a repository is like
creating a multi package lock file for the entire repository, similar to what
nix does today manually in the sources. Not versioning a repository is the
equivalent of a rolling deployment. Some notes on a possible implementation
can be found in `package_layer.md`.

## Version control
With the current system there is no method of tracing ipfs source releases back
to a version control system. There is a technical limitation that needs to be
overcome. Current version control systems like git allow having objects of
arbitrary size making encoding it into ipfs blocks problematic. A version
control system suitable for ipfs should be designed. A method of associating
entries in an incremental build cache with commits could improve build times
further.

# Team
We are not a random collection of people, but a strong team with a proven track
record of delivering.

## Developers
David Craven, Lead Software Engineer, co-founder

Experience packaging software for functional package managers. Contributed to
substrate - a blockchain framework, guix - a functional package manager and
implemented an ipfs node.

