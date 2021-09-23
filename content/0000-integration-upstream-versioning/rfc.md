# Upstream versioning for integrations (2021-09-16)

## Stakeholders

* **Recommenders:** Noah Fontes
* **Agreers:** Rick Lane
* **Performers:** *TBD*
* **Inputers:** Cortez Frazier, Kenaz Kwa, Kyle Terry
* **Deciders:** Kenaz Kwa, Rick Lane

## Problem

Right now, we basically arbitrarily increase versions of upstream software to
support our internal needs. In some cases, we do so in a way that would actually
break other users' integrations. For example, we upgraded the Terraform
integration version from 0.12 to 0.13 and then to 0.14 without any notice. Each
of these upgrades require code changes to the underlying Terraform modules that
might be applied, so a consumer of these integrations would be very startled to
find that their steps suddenly stop working.

## Summary

This RFC proposes to add a new field, `upstreams`, to the top-level integration
metadata file defined in [RFC 0006](../0006-integration-layout/rfc.md). This
field maps vendor versions of software used in images to customizations of those
images. For example, it might be used to set an `ARG` containing a specific
repository tag when using the Docker builder.

To leverage a particular upstream configuration to build an image with a matrix
of supported versions, we add a new field, `upstream`, to each image metadata
file. This field selects an upstream vendor defined in the integration metadata
file to apply to the current image.

## Motivation

By implementing upstream-based matrix builds defined in integration and
container image metadata, we can use a single image container definition to
provide multiple concurrent versions of upstream software. We can also introduce
new versions without disrupting existing users.

If we can produce versioning information in a readily consumable way, the burden
of selecting an appropriate version should be small for the user and they should
be happily guaranteed the consistency they need for years to come.

## Product-level explanation

### Top-level metadata

We extend `integration.yaml` as follows:

```yaml
# A list of upstream vendors used by this integration. Optional.
upstreams:
-
  # The name of the upstream vendor. Used to select this upstream in image
  # metadata files, but otherwise not exposed to end users. Required.
  name: terraform

  # The URL to this vendor's website. Optional.
  homepage: https://www.terraform.io

  # The URL to this vendor's source code. Optional. If the homepage is a GitHub
  # link, may be inferred automatically.
  source: git://github.com/hashicorp/terraform.git

  # A list of versions of the upstream software supported by this integration.
  # Versions are grouped by compatibility; that is, for example, for a vendor
  # that uses semantic versioning, it is not necessary to define a v1.1 and a
  # v1.2. Instead, just define a v1 and keep the version up to date within the
  # series as needed. Required.
  versions:
  -
    # The Relay version identifier, which must be usable as a Docker tag. For
    # images that use this upstream, this identifier will be prepended to the
    # generated tag for the image and separated with a dash. If there is no
    # generated tag (i.e., the tag name would have been "latest"), this name
    # becomes the tag.
    #
    # This name must be unique across all versions. If omitted or set to the
    # empty string, this version will be the default version. Optional.
    name: v1

    # The ranges from the vendor software this version supports expressed as a
    # semantic versioning constraint (à la NPM or Composer). Displayed in UI
    # widgets to help users select the right upstream version identifier.
    # Required.
    constraint: '>=1.0.0 <=1.0.6'

    # A channel override. If specified, this field may reduce (but not
    # increase) the stability of the integration channel for this particular
    # version. Useful for providing test builds of prerelease software.
    # Optional.
    channel: stable

    # Overlays match immutable fields of container image metadata and add or
    # replace mutable fields. In the broad sense, we call this operation a
    # "strategic merge" because the matching and merging logic is implied by
    # the semantics of the referenced fields.
    #
    # Overlays are applied in order, and all overlays that match are applied
    # (that is, there is no short-circuiting behavior).
    #
    # Optional.
    overlays:
    -
      # For example, the "build" field data is deeply merged.
      build:
        # Overlays can't possibly change the builder apiVersion or kind for the
        # image. So this is obviously a field only used for matching -- that
        # is, this overlay will only apply to container images that use the
        # Docker kind.
        apiVersion: build/v1
        kind: Docker

        # On the other hand, "args" is mutable and merged, so IMAGE will be
        # added to (or replace an existing value of) the args in any matched
        # container image.
        args:
          IMAGE: hashicorp/terraform:1.0.6
```

### Container images

For a given container image, we need to turn its metadata into a matrix of all
possible upstream versions it supports. Typically this will involve some amount
of post-processing to add or replace fields in the metadata (i.e., using
overlays). Possibilities include:

* Changing some aspect of the image builder
* Adding a version-specific example
* Producing a schema with version-specific components

#### Using an upstream version matrix

By default, container images are not mapped to a particular upstream in the
integration metadata, and will not produce a version matrix when built. To
select an upstream and compatible versions, we amend the common container image
metadata:

```yaml
# The name of the upstream vendor to use for this step. Without further
# configuration, enables a matrix build configuration for each .
#
# The upstream field is part of the API contract of the container image, so if
# an integration author needs to change it, the container image version must
# also be incremented.
#
# Optional.
upstream: terraform

# Configures the build matrix for this container image. When a matrix field is
# provided, only the versions explicitly listed in the matrix will be built.
# Optional.
matrix:
-
  # Selects aspects of the upstream to use to compute the matrix. Required.
  upstream:
    # Selector for specific versions of the upstream. At least one version must
    # be specified. Required.
    versions:
    - v1
    - v0.15

  # An optional additional overlay to use. This overlay is applied after any
  # matched overlays defined by the upstream. There's no need for matching
  # behavior here, so the strategic merging approach will be used for all
  # fields. Optional.
  overlay:
    # Schema content is merged as if `allOf` were used in the JSON Schema
    # definition. Optional.
    schemas:
      spec:
        source: file
        file: spec-v1-v0.15.schema.json

    # Examples are appended.
    examples:
    - name: An example specific to v1 or v0.15
      content:
        apiVersion: v1
        kind: Step
        # ...
```

#### Overlays

To produce a matrix of versions from a single image metadata file, we introduce
the concept of overlays. Overlays are essentially combined matching and patching
operations. [Does it make sense to rigorously define when matching applies?] In
general, matching is used against fields that it wouldn't make sense for an
overlay to change, like kinds and names, while patching is used for everything
else.

By specifying overlays for a given upstream version, we can, for example, set
source Docker images to the correct repository and/or tag for each version
across all container images without having to modify each image metadata file.

#### Changes to image versioning

In RFC 0006, we introduced a version tuple for container images of `(<channel>,
<image-major-version>, <integration-version>)`, where typically the release
channel is assumed to be stable and the integration version automatically
selects the highest possible version, so end users only have to worry about
picking the container image version to retain compatibility.

When a container image leverages an upstream, we need to add a new item to the
version tuple, `<upstream-version>`. This component also has significance to the
end user (i.e., they will be deliberately choosing it in some UI). We expect
that users will only really care about the container image version when they
switch upstream versions or need support for a new feature.

We insert the upstraem version as the third element in the tuple to make the
full version tuple `(<channel>, <container-image-version>, <upstream-version>,
<integration-version>)`. Upstream versions are a property of the container image
version because the upstream is selected by the container image metadata.

#### Changes to image references

To support the addition of the upstream version as a component in the container
image version tuple, we amend the container image reference format.

This syntax is a significant departure from the original one described in RFC
0006. As we haven't implemented it yet, I'm comfortable making the call to
change it to something more readable with fewer special characters. [Are you?]

```
[<channel>/]<integration-name>/<image-name>[/<upstream-version>]@<image-major-version>[.<integration-version>]
```

Some possible representations of the Terraform apply step described above:

* `puppet/terraform/apply/v0.14@1`
* `terraform/apply/v1@3.v20210916`
* `alpha/terraform/apply/v1.1@3`

### Additional provider name reservations

Because we're using slashes as a common separator for each component of the
image reference, we further reserve the provider slugs `alpha`, `beta`, `rc`,
and `stable` to prevent ambiguity.

## Engineering-level explanation

### Building Docker images

Without a proper implementation of the `relay integration` CLI commands, the
complexity of manually interpreting and building images for an entire version
matrix is quite high.

The engineering-level effort requires at least the Integrations V2 engineering
phase of RFC 0006 to be complete. After it is in place, we extend the
`integration build` machinery to evaluate the build matrix and run individual
builds for each possible version. [I'm leaving this intentionally vague for the
moment because there's still a lot more to figure out just to get to a working
`integratoin build` command.]

We may also want to provide an API of some sort that can render a list of
container image metadata for each possible upstream version with all overlays
applied for other tools to consume. [Feedback here?]

### UI changes

We should be able to expose the version matrix for a given container image to
end users. In particular:

* The library needs to list available versions for an integration and specific
  container images
* The form-based image builder needs to allow a user to select versions to use

### Operational impact

Implementing this RFC results in more Docker images being produced and stored on
Docker Hub. It should have no impact to Relay infrastructure.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

[Describe the benefits, both to product and engineering, by implementing this
solution. Include a comparison of alternative designs as part of the narrative
if possible.]

### What is the impact of not doing this?

[...]

## Drawbacks

[Why should we _not_ include this change in the product? What specific risks are
associated with this design?]

## Success criteria

[What will we observe when this change is completely implemented? What product
metrics can we use to measure success? Can we integrate any learnings into
future processes?]

## Unresolved questions

* [Insert a list of unresolved questions and your strategy for addressing them
  before the change is complete. Include potential impact to the scope of work.]

## Future possibilities

[Does this change unlock other desired functionality? Will it be highly
extensible, very constrained, or somewhere in between?]
