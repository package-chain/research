# package-chain
> **deprecated**: see [github](https://github.com/package-chain/)

- Package chain is the first truly decentralized software distribution mechanism.
  Binaries distributed with build chain are guaranteed to work on any computer
  running a build chain client.
- Binaries are verified on chain. There is a trustless link between the sources,
  the binary and the build process.
- Build chain is a decentralised and distributed backend for package managers,
  build caches and ci systems.

# Problem statement
There are several problems with existing build and deployment systems.

1. Large codebases have long build times which result in significant amounts of
   lost productivity.
2. Deployment of complex codebases in uncontrolled environments can
   be an annoying and painstaking process. This happens when the developer
   envionment does not match the deployment environment.
3. Developers carry the cost of storage and bandwidth required to store and
   distribute binaries.
4. Users are required to trust that the developers build infrastructure has not
   been compromised.

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

## ccache
Ccache and sccache are distributed build caches. They work generally by hashing
the inputs to a compiler and storing the outputs in a cache. If the compiler is
invoked with the same inputs the result is retrieved from the cache.

## distcc
Distcc works by simulating the compiler. When the build system invokes distcc it
forwards the arguments and inputs to a remote host. The remote host executes the
real compiler and returns the results.

## IPFS
IPFS (Juan Benet 2014) is a decentralised and distributed content addressed
block store. It is a novel synthesis of distributed hash tables and merkle
dags and combined with a new incentivised block exchange protocol to create a self
certified, deduplicated and content addressed block store. Blocks are used to
fetch data from untrusted peers, by imposing an upper bound on their size. This
prevents a malicious peer from continuously sending a stream of data claiming
it is going to match a hash.
For distributing binaries using the nix deployment model, pure content
addressing is insuficcient. A binary can contain self references. For the
distribution of binaries the ipfs implementation must keep track of those self
references and transparently rewrite them when accessing a block.
There are also implementation issues, due to different focus of the ipfs team.
The fuse file system is slow, effort was spent instead on an ipfs http gateway,
and the development and specification of a complex rpc and cli interface. Using
a fuse implementation and an existing static web server would have reduced or
eliminated the need for those things.


# Decentralised and distributed backend for building and distributing software
By synthesis of nix, ccache, distcc and ipfs with blockchain technology we can
solve these issues.

## Storage layer
A globally shared distributed decentralised file system.

### Block store
The block store is located at `/ipfs/blocks/{cid}`. A file or directory is
encoded using unixfsv2 to a list of blocks. Blocks can be added to the store
concurrently. Each block is represented by a file. After writing a block it is
marked read only to prevent accidental modification. To speed up queries a db
of block references is maintained.

### File system
The block store is mounted as a read only fuse file system at `/ipfs/store`.
The filesystem provides different views of the block store. Each view is a
plugin to the fuse file system. `/ipfs/store/{plugin}/{cid}/{path}`. A block,
unixfsv2 views allow inspecting raw block contents and the decoded files and
directories. During reads the block store is checked for consistency.

### Pinning and garbage collection
Users can pin blocks by creating a symlink in `/ipfs/pins/per-user/{user}/{prog}/{cid}`
to `/ipfs/store/{cid}`. Pinning a store path prevents it from being garbage
collected. By adding a level of indirection to a users home directory with
auto links `/ipfs/pins/per-user/{user}/{prog}/auto/{hash(path)}` the user can
conviniently manage pins manually. The garbage collector removes dangling
symlinks and dead store paths. A lock file `/ipfs/pins/lock` and temporary pins
`/ipfs/pins/temp/{pid}` are used to add items to the store while the garbage
collector is running.

### CRDT db
TODO: Required for building a shared ledger

### Peer and block discovery
A distributed hash table is used to locate peers that have a block. Peers
that have a block are optimistically assumed to have all blocks that are linked
from that block. Local peers discovered through mdns are optimistically assumed
to have any block.

### Block exchange
The bitswap protocol is used to request and exchange blocks. To promote
cooperation in a large population with high turnover, asymetric interests and
zero-cost identities we use use the following techniques:

1. discriminating server selection
2. shared history
3. maxflow based subjective reputation
4. adaptive stranger policy
5. short term history

To prevent DoS attacks we impose an upper size limit for blocks. This allows
continuous verifying of the received data.

### Syncing block graphs
graphsync


## Compute layer

### Derivations
A derivation is the description of a computation. It fully describes all inputs
to a computation, the command, all environment variables and command line
arguments. A derivation is executed in isolated kernel namespaces. Declared
inputs and file systems are mounted into the namespace. This allows executing
untrusted derivations. When a derivation is untrusted, cgroups are used to
constrain it's resource consumption. An upper bound on the runtime is used to
ensure that the derivation terminates.

### Verified computation
Verified computation works by offloading a derivation to multiple randomly
selected peers. The peers publish `hash(nonce, hash(output))`. After all
peers publish their commitments, they publish the nonce and the `hash(output)`.
If they agree the result is stored on-chain. Based on the reported cpu usage,
their efficiency ratios are calculated. Similar projects such as golem show
that this is possible. The difference is the concept of derivations, the use of
containers and a strong focus on the target use case of build tasks and
public computation.


## Build layer

### Meta derivations
A meta derivation can create new derivations. This is achived by mounting a
unix domain socket in the container, which allows communication with the
compute layer. This is useful for wrapping commands like `gcc` or `rustc`
to offload them to the compute layer. Untrusted meta derivations should not be
allowed to offload derivations, only substitute locally cached outputs. 

### File system views
An app view and a webapp view are implemented. The app view performs hash
rewriting to allow self referential store paths, and the webapp view is the
subset of unixfsv2 directories that are marked with `webapp=true`. This allows
for a static webserver to automatically serve the files locally. In addition
a browser plugin can list installed webapps.

### User environments
A user environment is a hierarchy of symlinks that mirrors the union of the
directory hierarchies of the installed components. These are stored in
generations `/nix/profiles/per-user/{user}/generations/{id}` to enable atomic
upgrades and rollbacks. The current user environment is symlinked from
`/home/{user}/.nix_profile` to `/nix/profiles/per-user/{user}/current` which
in symlinks to the active generation.

### Verified binaries
An open research question is how to concatenate the proofs of the computation
layer to achive a proof that an output is the result of a meta derivation. This
would allow one click installs of binaries and webapps.


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

# References
- [0] The purely functional software deployment model
- [1] Robust Incentive Techniques for Peer-to-Peer Networks
