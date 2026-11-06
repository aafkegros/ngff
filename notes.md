# Visualization RFC notes

## Scope

We want an embedded view setting in the OME-Zarr.

At least 3 (or more?) viewers will need to support settings to be included.

Question to address here: What is a maximal intersection of view settings that we can agree on?

### Out of scope

- Canvas
- Keyframes, non-spatial dimensions in cameras (current indices belong to view)
- parametric colormaps
- 

### Related work

OME-Zarr: 
- Collections & extensions (RFC-8)
- Thumbnails (no RFC yet, see also RFC-9 reviews/comments)

Zarr-vectors (out of scope)

### Viewer-specific settings --> extensions?

Viewers are high dimensional and a shared value list will not be complete.

- viewer setting traits - can we have a defined list and then a fallback? 
- could be done by having extension specific keys that viewers can override if they exist.
- ~~alternatively, free-text with suggested (i.e., specified) values~~ this would essentially decouple settings from expected behavior --> avoid
- prepare for a future of extensions but don't define an extension system


OME-NGFF.json
```
blending_mode: "additive"
```
OME-NGFF::MicroscopyNodes.json
```
blending_mode: "additive" # MiN maintains the fallback (not part of the extension)
blending_mode_microscopy_nodes_emmissive: 0.95
blending_mode_microscopy_nodes_scattering: 0.05
```
    
OME-specific Zarr convention for prototyping for the RFC?
We can try to get an experimental convention by the end of the week that we can iterate on over the next year - while other RFCs get defined. This will not be public.


## User stories

- Contrast specific to FOV
- Thumbnails in archives --> relevant to RFC-9?
- Figure panel spec
- Min/Max + contrast settings / histogram
- Multi-layer
- Dynamic/time-series images: viewer speed
- RGB


# Spec

Open questions:
- Split 2D/3D?
- Blending modes / compositing?
- Multiple view configs?
- how to do lut across channels instead of per channel? - e.g. time-trajectory 
- Channels at all!!
	- do we define channel/layer order??

    

### Coordinate spaces + RFC-3



Scene dimensions can be mapped **into** target dimensions for visualization that are constrained to:
- x
- y
- z (right handed?)
- overlapping ("channels")
- animation time ("play button")
	- Current timepoint
- indexable extra dimensions ("sliders")
	- Current index

RFC-5 Scene dimensions can be mapped to any of these. Visualization dimensions inherit the unit of that RFC-5 dimension. 
There is no required type-to-type mapping:
Example:
you can have an RFC-5 coordinate space of time, by x y z sampling, but map this to a kymograph with x and time axis and make z and y indexable.




### Camera(s), n-dimensional (incl. time)

Is a viewport a different thing than a camera?

Aafke is in favor of defining a camera as a rectangle in 2D or a frustum in 3D.

Jonas is unsure whether cameras (on top of the target dimensions mapping) are covered by the term "viewer settings". To him, viewer settings ensure reproducibility *given* a specific viewport. But not opposed to including it considering the use cases & happy to adopt eventually.

Avoid accidentally defining ROIs!
- We can state that Camera's are not meant to be ROIs in the spec. We cannot stop people from still using them as such.

```
# coordinates are given in the unit of the dimension that are inherited from the RFC-5 Scene

# 2D camera
{
	x: [500, 700],
	y: [10, 40]
}

# 3D camera (ortho)
{
    frontplane: {
        x: [500, 700],
        y: [10, 40],
        z: [10, 50]
    },
    backplane: {
        x: [500, 700],
        y: [10, 40],
        z: [10, 50]
    }
}

# 3D camera (perspective)
{
    frontplane: {
        x: [500, 700],
        y: [10, 40],
        z: [10, 50],
    },
    backplane: {
        x: [200, 600],
        y: [5, 30],
        z: [5, 40]
    }
}
```


From @jni: no backplane because its too easy to mess up and too hard to give infinity in
- maybe just focal length? 
- inf is ortho?

how do we annotate view direction

Do we save an xy number of screen pixels??


Multiple cameras 



### LUTs/Colormaps

maybe look at https://cmap-docs.readthedocs.io/en/stable/

user stories for colors:
- random colors + seed
    - labels
- value to color mapping
- boundary conditions 
- interpolation types
- non-equal spacing of stops
- rgb
- gamma?


```

Colors are hex RGBA in sRGB (IEC 61966-2-1)

colors: {
	
	over_boundary : periodic #mirror, periodic, color
	under_boundary : #00000000
	bad_mapping : #FF0000FF
	blending_mode: "additive", # screen, multiply etc
	mapping : "linear", # nearest-neighbor, linear 
	gamma : 1,
	value_to_cspace : { # can also be "identity" if you don't want dual transforms.
		[13, 0.0],
		[255, 1.0]
	},
	cspace_to_color : {
		[0.0, #000000FF], 
		[1.0, #FFFFFFFF],
	}
	name: "black to white"

}

colors: {
	
	over_boundary : periodic #mirror, periodic, color
	under_boundary : #00000000
	bad_mapping : #FF0000FF
	blending_mode: "additive", # screen, multiply etc
	mapping : "unique", # nearest-neighbor, linear 
	gamma : 1,
	value_to_cspace : "identity",
	cspace_to_color : {
		[1, #000000FF], 
		[2, #FFFFFFFF],
		[115, #FFFFFFFF],
	}
	name: "labelmap"

}


```

## View generic

how do we do origin location / axis direction

```
views : [
	{
		background_color : #00000000,
		handedness : right-handed,
		coordinate_space: id-of-RFC-5-coordinate-space
		dimensions : { # you can leave out when not shown
			x : x
			y : y
			z : t
			time : {u : 1} # current index named
			overlapping : airyscan_sample
			indexable : [
				{c : 0},
				{sample : 54}
			]
		},
		cameras : [ # first is primary
			{
				sensor: {
					x: [500, 700],
					y: [10, 40],
					z: [10, 50],
				},
				backplane: { 
					x: [200, 600],
					y: [5, 30],
					z: [5, 40]
				}
				
			},
			{
				sensor: {
					x: [310, 700],
					y: [10, 40],
					z: [10, 50],
				},
				backplane: {
					x: [200, 600],
					y: [5, 30],
					z: [5, 40]
				}
				
			}		
		],
		colors : [
			{
	
				over_boundary : periodic #mirror, periodic, color
				under_boundary : #00000000
				bad_mapping : #FF0000FF
				blending_mode: "additive", # screen, multiply etc
				mapping : "unique", # nearest-neighbor, linear 
				gamma : 1,
				value_to_cspace : "identity",
				cspace_to_color : {
					[1, #000000FF], 
					[2, #FFFFFFFF],
					[115, #FFFFFFFF],
				}
				name: "labelmap"

			},
			{
	
				over_boundary : periodic #mirror, periodic, color
				under_boundary : #00000000
				bad_mapping : #FF0000FF
				blending_mode: "additive", # screen, multiply etc
				mapping : "linear", # nearest-neighbor, linear 
				gamma : 1,
				value_to_cspace : { # can also be "identity" if you don't want dual transforms.
					[13, 0.0],
					[255, 1.0]
				},
				cspace_to_color : {
					[0.0, #000000FF], 
					[1.0, #FFFFFFFF],
				}
				name: "black to white"

			}

		]
	}
]
	
	
	
}


```


# imagesc post

2026 OME-Zarr Hackathon Zurich - view settings

Hi all, we had a working group working on a draft 




\








random work document things:

For each channel: (Do you need to have the channel ID? Does the channel data type matter?) 
- Color
    - Current OMERO (transitional metadata) specification is color# and "inverted" boolean
    - Is there a pre-defined list of colormaps consistent between applications? e.g. plasma and viridis vs color: 0000FF and inverted: false
    	- yes this exists - will look this up later, but i'm not a big fan of by-name lookup (aafke)
        - is this language specific?
- Min / max

... does this overlap too much with RFC-7?
... is RGB 3 channel? :D - shouldnt be but currently is

Types:
- generative mapping f(x) -> y   NOT SUPPORTED
- 
- named colormaps by string lookup    WE ONLY HAVE AN ARBITRARY NAME, BUT ALWAYS INCLUDE THE VALUES
- discrete mapping (string, img value -> color)

- include a name



## Viewers

Selection of reference viewers to evaluate?

### AGAVE

### ImageJ/Fiji

#### BigDataViewer

#### BigVolumeViewer / BigVolumeBrowser

### Microscopy Nodes (Blender)

### neuroglancer

### napari

### QuPath

### Webknossos

### Vizarr
