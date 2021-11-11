# Design and architecture

## Design
The `image-rs` crate is a rustified and tailored version of [containers/image](https://github.com/containers/image),
to provide a small, simple, secure, lightweight and high performance OCI container image management library for
[Confidential Containers](https://github.com/confidential-containers).

The `image-rs` crate focuses on usage scenarios to run `Confidential Containers` on Kubernetes clusters with
[Kata Containers](https://katacontainers.io/). So it implements the
[CRI Image Service](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1alpha2/api.proto#L119)
interface to be easily integrated with the K8s ecosystem. Among the container image `Build`, `Ship` and `Run` stages,
it focuses on the `Run` stage instead of `Build` and `Ship`, that is only to provide interfaces to prepare container
images for running in a confidential environment. It may aslo be enhanced to support other confidential container
runtimes on demand.

The `image-rs` crate supports both encrypted and unencrypted container images since
[Encrypted Container Image](https://github.com/opencontainers/artifacts/pull/15)
is mandatory for many confidential container usage scenarios. It also supports image signature verification to ensure
integrity.

For confidential containers based on enhanced hardware virtualization technologies (TDX/SEV/PEF etc), there's no easy
way to share container images among pods/containers running on the same physical node. But it's time consuming to
download, verify, decrypt and extract container images for each pod/container. So the `image-rs` crate will explore
technologies and innovations to reduce image preparation time. Data decryption and decompression acceleration libraries
may be used to speed up image decryption and decompression. Image data on demand loading may also be adopted to reduce
container startup time.

### Goals
 * Support pulling OCI/docker v2 images from remote registries compatible with the OCI distribution spec.
 * Support downloading OCI images from local [OCI layout compatiable](https://github.com/opencontainers/image-spec/blob/main/image-layout.md) storage
 * Support OCI image decryption
 * Support OCI image signature verification
 * Support seamlessly integration with Kata Agent and Attestation Agent
 * Explore image decryption and decompression acceleration technologies
 * Explore image on demand loading technologies


### Non-Goals
 * Image push, content discovery/management operations for remote registries
 * Image encryption and signing
 * Image export to local storage
 * Image import from other service daemon or other local storage format


### Existing Solutions
 * [containers-image-proxy-rs](https://github.com/containers/containers-image-proxy-rs): a rust bindings for skopeo,
   which doesn't meet our requirements of small TCB and lightweight.
 * [oci-distribution](https://github.com/krustlet/oci-distribution): a crate from
   [krustlet](https://github.com/krustlet/krustlet) to run wasm application with K8S. Since wasm is platform agnostic
   and small code size, `oci-distribution` does not support manifest lists for different platforms and internal data
   struct is not friend to generic container workload.
 * [oci-registry-client](https://github.com/ecarrara/oci-registry-client): a very simple client for OCI image registries
   with limited features and latest commit is one and half years ago.

Above projects focus on image distribution and do not support image unpack and storage management features.


## Architecture
The following diagram demonstrates the overall architecture and how to
integrate with Kata Agent:
![Architecture](images/architecture.png)

The image_client module will provide the API and mainly cover image management,
image pulling and OnDemand image pulling.

For image pulling part, it will separate the container image as two parts:
metadata and data, for metadata it can be consumed by the image management
module and the data part can be decrypted by ocicrypt-rs if needed. After
image data is unpacked, snapshots module will prepare the rootfs together with
the container configuration and metadata, the bundles can be generated for the
runtime service to consume.


Due to data decryption and de-compression can be time consuming, image-rs will
support different acceleration libraries to accelerate these process like
Intel storage acceleration library:
[isa-l](https://github.com/intel/isa-l)
[isa-l_crypto](https://github.com/intel/isa-l_crypto).

The snapshot module can support different snapshotters which can support
image OnDemand pulling.

The image-spec module will define the [OCI image spec](https://github.com/opencontainers/image-spec)
to avoid the dependency of other heavy crates.


## Reference
[Nydus Image Service](https://github.com/dragonflyoss/image-service) for data on demand loading, data deduplication and encryption.
[stargz](https://github.com/containerd/stargz-snapshotter) for data on demand loading.
