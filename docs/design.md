# Design and architecture

## Design
Image-rs is a library providing OCI container image management in a light,
secure and fast way, with quite some ideas from [containerd](https://github.com/containerd/containerd)
and [containers/image](https://github.com/containers/image) but using Rust.

One key usage case for image-rs is [confidential containers](https://github.com/confidential-containers)
which requires image download within pod. With this, image-rs has
to be light and fast. Then for the feature set part, it focusing on
pod/container runtime stage and does not cover the stage of container
image development and shipment. For the API part, image-rs provides
[CRI image service](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1alpha2/api.proto#L119)
compatible API.

image-rs will support both encrypted and unencrypted container image format since
[encrypted container image](https://github.com/opencontainers/artifacts/pull/15)
is mandatory in many confidential container scenarios, and leverages
[ocicrypt-rs](https://github.com/containers/ocicrypt-rs) rust crate for decryption.

A stretch goal is to improve the container cold startup speed by supporting
OnDemand or lazy image pulling. These features already implemented by
several projects like [nydus](https://github.com/dragonflyoss/image-service),
[stargz](https://github.com/containerd/stargz-snapshotter)
and supported by containerd. But how to combine OnDemand pulling and
OnDemand decryption together, still have lots of gaps to do like
define related container image metadata and data format.

### Goals
 * Support OCI image pulling from remote registries
 * Support OCI image loading from local [OCI layout compatiable](https://github.com/opencontainers/image-spec/blob/main/image-layout.md) storage
 * Support OCI image decryption
 * Support OCI image signing verification
 * Support OCI image acceleration for decryption and de-compression
 * Support OnDemand image pulling
 * Can be seamlessly integrated with Kata Agent and Attestation Agent

### Non-Goals
 * Image push, content discovery/management operations for remote registries
 * Image encryption and signing
 * Image export to local storage
 * Image import from other service daemon or other local storage format

### Existing solutions
 * [containers-image-proxy-rs](https://github.com/containers/containers-image-proxy-rs)
 * [oci-distribution](https://github.com/krustlet/oci-distribution)
 * [oci-registry-client](https://github.com/ecarrara/oci-registry-client)

containers-image-proxy-rs is a rust bindings for skopeo which did not meet our
light requirement. oci-distribution is a crate for [krustlet](https://github.com/krustlet/krustlet)
which is a rust implemented kubelet for running wasm. Since wasm is
platform agnostic and small code size, oci-distribution did not support
manifest lists for different platforms and internal data struct is not friend
to generic container workload. oci-registry-client is a very simple client for
OCI image registries with limited features and latest commit is one and
half years ago. These projects also do not support image unpack and
storage management features.

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
