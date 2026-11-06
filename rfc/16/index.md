---
authors:
  - name: Aafke Gros
    github: aafkegros
    orcid: 0000-0003-2459-9858
    affiliation: EMBL Heidelberg
    role: Corresponding Author
    date: "2026-11-05"
  - name: Jonas Windhager
    github: jwindhager
    orcid: 0000-0002-2111-5291
    affiliation: SciLifeLab / Uppsala University, Sweden
    role: Co-author
    date: "2026-11-05"
  - name: Suraj Pathak
    orcid: 0009-0002-1905-1943
    github: sspathak
    affiliation: University College London
    role: Co-author
    date: "2026-11-05"
# endorsers:
#   - name: Endorser 1
#     github: endorser1
#     orcid: 0000-0000-0000-0000
#     affiliation: Affiliation A
#     role: Endorser
#     date: "YYYY-MM-DD"
#     reference: https://example.com
# editors:
#   - name: Josh Moore
#     github: joshmoore
#     orcid: 0000-0000-0000-0000
#     affiliation: German BioImaging e.V.
#     role: Editor
#     date: "YYYY-MM-DD"
reference_pr: https://github.com/ome/ngff/pull/XXX
manual_status: D3
status_note: authors open PR
description: View settings
date: "2026-11-05"
---

# RFC-16: View settings

(rfcs:rfc16)=

Summary: Embedded, interoperable view settings for OME-Zarr.

## Status

Draft (D3). This document captures the working group's proposed direction.
The metadata structure and rendering semantics remain under discussion;
examples are illustrative, not a finalized schema. An editor and reference PR
still need to be recorded.

```{rfc-status}
```

```{toctree}
:hidden:
:maxdepth: 1
templates/index
```

## Overview

This RFC proposes embedding view settings in OME-Zarr so that viewers can share
how a dataset is presented. A view describes how dataset dimensions are used
for visualization, which non-spatial indices are selected, and how values are
mapped to colors. Cameras describe the spatial window and viewing direction.

The goal is the largest useful set of settings for which participating viewers
can agree on behavior. 

## Background

An image's stored values and coordinate system do not fully describe how to
present it. Users also choose contrast limits, colors, visible layers, and a
field of view. Recording these choices alongside the data would help users
reopen a useful presentation and share it across applications.

The 2026 OME-Zarr Hackathon Zurich working group identified use cases including
field-of-view-specific contrast, archive thumbnails, figure panels, multilayer
images, time series, and RGB images. These uses motivate a common description,
while the variety of viewer capabilities makes it necessary to distinguish
shared behavior from application-specific rendering choices.

This proposal relates to coordinate transformations in [RFC-3](../3/index.md),
coordinate spaces in [RFC-5](../5/index.md), and collections and extensions in
[RFC-8](../8/index.md). 

## Proposal

Define an embedded view description with the following components:

- A reference to a scene coordinate space.
- A mapping from scene dimensions to visualization roles.
- Current indices for animation and other indexable dimensions.
- Explicit value-to-color mappings and a background color.
- One or more cameras describing the spatial presentation.

The location of this metadata in OME-Zarr, its versioning, and the ability to
store multiple view configurations remain open. The sketches below illustrate
possible contents without selecting a storage location or final field names.

### Scope

The proposal covers shared view settings with defined behavior. It should leave
room for future viewer-specific extensions without defining an extension system
in this RFC. A viewer-specific setting could eventually refine a shared setting,
with the application maintaining the shared fallback.

Canvas layouts, keyframes, parametric colormaps, and Zarr-vectors are out of
scope. Current indices for non-spatial dimensions belong to the view rather
than to its camera. Cameras describe presentation and are not intended to define
regions of interest.

## Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in IETF RFC 2119.

The proposed design requirements are:

- Shared settings have defined rendering semantics; unrestricted strings with
  merely suggested meanings are insufficient.
- Color mappings include their values. A human-readable name does not replace
  the mapping with a lookup in an application-specific colormap registry.
- Visualization dimensions inherit units from the referenced scene dimensions.
- Scene dimension types do not restrict their visualization roles.

### Dimension mapping

Scene dimensions can be mapped to spatial display axes (`x`, `y`, `z`), an
overlapping role such as channels, an animation dimension, or other indexable
dimensions. Animation and indexable dimensions carry current selections.

For example, a scene sampled over time and three spatial dimensions could be
shown as a kymograph: map scene `x` and time to display `x` and `y`, and make
scene `y` and `z` indexable. The mapping does not alter the scene's coordinate
space or units.

The representation of omitted dimensions, axis direction, origin, handedness,
and channel or layer order remains to be specified.

### Cameras

A camera consists of a sensor/window box and a view vector, with a focal length
for 3D views. The `sensor` field retains the box representation used in the
initial sketch: each axis has a two-element interval `[min, max]`. It describes
the spatial window to frame, in the mapped visualization coordinates, rather
than two planes whose corners must form a valid frustum.

For a 2D view, the box contains `x` and `y`. For a 3D view, it also contains `z`.
The box is axis-aligned in the visualization coordinate system; the view vector
specifies the direction of viewing through that window, toward the scene.
Its magnitude does not affect the view. Current timepoints and other index
selections remain properties of the view, outside the camera.

For a 3D perspective view, `focal_length` is a positive finite distance from the
projection center to the sensor plane. The proposed convention is
`focal_length: null` for orthographic projection, avoiding an infinity value in
JSON. A 2D camera omits `focal_length`. The focal length and spatial box need
commensurate units; conversion between mapped axis units must be resolved before
perspective projection can be specified completely.

The proposed validation constraints are:

- `sensor` contains exactly the spatial axes of the view, each with two finite
  numbers in increasing order.
- `view_vector` contains finite `x`, `y`, and `z` components in the display frame
  and is nonzero. For a 2D view, `z` denotes the display-plane normal rather
  than an additional scene dimension; the proposed direction is `[0, 0, -1]`.
- A 3D camera includes `focal_length`, either a positive finite number or `null`;
  a 2D camera omits it.

These constraints avoid validating relationships between front and back planes.
Interval ordering and the nonzero-vector check may require semantic validation
in addition to a JSON Schema.

The box and direction do not yet fully specify an arbitrarily rotated camera.
The convention for camera roll (or an explicit up vector), placement of the
sensor plane relative to the box, and fitting the box to the output aspect
ratio remain to be agreed. In particular, the box's `z` interval describes its
spatial extent, not a clipping-plane pair. Clipping behavior and output pixel
dimensions are also open. The first camera in the list is proposed as primary.

### Colors and contrast

The proposed color description separates the mapping of data values to a color
coordinate from the mapping of that coordinate to explicit colors. The first
mapping may be an identity mapping, which is useful for discrete label values.
Colors are proposed as hexadecimal RGBA values in sRGB.

The design needs to cover:

- Continuous mappings with potentially unevenly spaced stops.
- Discrete mappings for label values.
- Interpolation, including nearest-neighbor and linear behavior.
- Values above and below the mapping range, including explicit colors,
  periodic behavior, or mirrored behavior.
- Invalid values, gamma, and blending or compositing.
- Random label colors with a seed, and RGB presentation.

The ordering and exact meaning of these operations still need specification.
In particular, gamma placement, color interpolation space, and boundary behavior
must be agreed before a mapping is interoperable. Whether mappings attach to
channels, layers, or another object is also unresolved, including mappings
across channels such as time-trajectory coloring.

## Stakeholders

The authors are listed above. An editor and reviewers have not yet been assigned.
The proposal originated in the 2026 OME-Zarr Hackathon Zurich view-settings
working group.

Potential viewers for evaluation include AGAVE, ImageJ/Fiji, BigDataViewer,
BigVolumeViewer/BigVolumeBrowser, Microscopy Nodes, neuroglancer, napari, QuPath,
Webknossos, and Vizarr. This list does not imply participation or endorsement.

## Implementation

Start with an experimental convention to compare behavior across viewers and
iterate while related RFCs develop. The notes propose an initially non-public
prototype; a final convention, schema, and public examples remain to be prepared.

Implementers will need to resolve coordinate-space references, apply dimension
mappings and index selections, and evaluate the agreed color operations.
Camera support requires agreement on the remaining orientation and window-fitting
conventions.

## Drawbacks, risks, alternatives, and unknowns

Different viewers may support different rendering models. A broad shared set
could be difficult to implement, while a narrow set could omit important uses.
Viewer-specific refinements are a possible future direction, but their fallback
and override semantics are not defined here.

The main decisions still required are:

- Whether to separate 2D and 3D views, and the remaining camera orientation,
  unit conversion, window-fitting, and clipping conventions.
- Metadata placement, references, defaults, and multiple view configurations.
- Blending and compositing semantics, including layer order.
- Channel identity, RGB representation, and color-mapping attachment.
- Axis direction, handedness, and coordinate selection semantics.
- Required viewer participation and handling of unsupported settings.

Free-text settings with only suggested values were considered and set aside
because they would decouple stored settings from expected behavior.

## Performance

Time-series playback is a motivating use case. Evaluation should determine
whether restoring a view or applying color mappings introduces significant
loading or interaction delays. No playback speed requirement is defined yet.

## Compatibility

The relationship to existing OMERO transitional visualization metadata remains
to be specified. This draft does not define a migration or compatibility layer.

## Testing

Once the shared semantics are agreed, evaluate representative views in at least
three viewers. Compare dimension selections, contrast and color results, and,
camera presentation. Shared examples should cover continuous
intensities, discrete labels, RGB, and a kymograph.

## Tutorials and examples

The following provisional example combines dimension mapping, two cameras,
a discrete label map, and a continuous intensity map. It preserves the proposed
structure from the working notes, expressed as valid JSON. Field names,
attachment of color mappings to data, and rendering semantics remain open.
The coordinate-space identifier is a placeholder.

Dimensions can be omitted when not shown. In this sketch, `time` and `indexable`
include current selections, and the first camera is primary. Both cameras use
a sensor/window box and view vector. The first illustrates perspective
projection and the second orthographic projection; the geometry conventions
described above remain provisional.

```json
{
  "views": [
    {
      "background_color": "#00000000",
      "handedness": "right-handed",
      "coordinate_space": "id-of-RFC-5-coordinate-space",
      "dimensions": {
        "x": "x",
        "y": "y",
        "z": "t",
        "time": {
          "u": 1
        },
        "overlapping": "airyscan_sample",
        "indexable": [
          {
            "c": 0
          },
          {
            "sample": 54
          }
        ]
      },
      "cameras": [
        {
          "sensor": {
            "x": [500, 700],
            "y": [10, 40],
            "z": [10, 50]
          },
          "view_vector": {"x": 0, "y": 0, "z": -1},
          "focal_length": 100
        },
        {
          "sensor": {
            "x": [310, 700],
            "y": [10, 40],
            "z": [10, 50]
          },
          "view_vector": {"x": 0, "y": 0, "z": -1},
          "focal_length": null
        }
      ],
      "colors": [
        {
          "over_boundary": "periodic",
          "under_boundary": "#00000000",
          "bad_mapping": "#FF0000FF",
          "blending_mode": "additive",
          "mapping": "unique",
          "gamma": 1,
          "value_to_cspace": "identity",
          "cspace_to_color": [
            [1, "#000000FF"],
            [2, "#FFFFFFFF"],
            [115, "#FFFFFFFF"]
          ],
          "name": "labelmap"
        },
        {
          "over_boundary": "periodic",
          "under_boundary": "#00000000",
          "bad_mapping": "#FF0000FF",
          "blending_mode": "additive",
          "mapping": "linear",
          "gamma": 1,
          "value_to_cspace": [
            [13, 0.0],
            [255, 1.0]
          ],
          "cspace_to_color": [
            [0.0, "#000000FF"],
            [1.0, "#FFFFFFFF"]
          ],
          "name": "black to white"
        }
      ]
    }
  ]
}
```

The color mappings include explicit values; their names are descriptive.
`value_to_cspace` can be `"identity"` for direct mapping of label values or a
list of pairs for the first stage of a continuous mapping. Both mapping stages
use arrays of pairs.

The notes also consider `mirror` and explicit colors for boundary behavior,
`screen` and `multiply` for blending, and `nearest-neighbor` for interpolation.
These alternatives and their exact semantics remain to be specified. Public
example datasets remain to be selected.

## UI/UX

Users should be able to recover the agreed presentation in participating
viewers. Decisions about selecting among multiple saved views and communicating
unsupported settings remain open. Archive thumbnails and figure panels are
motivating uses, but this RFC does not define their canvas layout.

## Future possibilities

A future extension mechanism could describe viewer-specific rendering controls
alongside shared fallback settings. Thumbnail workflows may consume saved views
once their relationship to the thumbnail proposal is established.
