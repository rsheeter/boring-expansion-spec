# `glyf`—Glyf Data Table Version 1: Variable Composites / Components

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The version number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined version of the `glyf` table is version 0. We propose version 1 of the `glyf` table to add multiple features. See full [proposal](glyf1.md) for all features. In this proposal we cover: _Variable Composites / Components_.

In `glyf` table format 0, there exist two types of glyphs: Simple glyphs which have a `numberOfContours` of >= 0, and Composite glyphs that have a `numberOfContours` of < 0, with a recommended value of -1.

In `glyf` table format 1, the old-style Simple and Composite glyphs exist as well. The old-style Composite glyphs shall always use a `numberOfContours` of -1. A new composite glyph type call VariableComposite will be introduced which uses `numberOfContours` of -2. A VariableComposite glyph has the ability to refer to _variable components_.


## Variable Composite Description

A Variable Composite glyph starts with the standard glyph header with a `numberOfContours` of -2, followed by a number of Variable Component records. Each Variable Component record is at least five bytes long.


## Variable Component Record


| type | name | notes |
|-|-|-|
| uint16 | flags | see below |
| uint8 | numAxes | Number of axes to follow |
| GlyphID16 or GlyphID24 | gid | This is a GlyphID16 if bit 12 of `flags` is clear, else GlyphID24 |
| uint8 or uint16 | axisIndices[numAxes] | This is a uint16 if bit 1 of `flags` is set, else a uint8 |
| F2DOT14 | axisValues[numAxes] | The axis value for each axis |
| FWORD | TranslateX | Optional, only present if bit 3 of `flags` is set |
| FWORD |  TranslateY | Optional, only present if bit 4 of `flags` is set |
| F4DOT12 | Rotation | Optional, only present if bit 5 of `flags` is set |
| F6DOT10 | ScaleX | Optional, only present if bit 6 of `flags` is set |
| F6DOT10 | ScaleY | Optional, only present if bit 7 of `flags` is set |
| F4DOT12 | SkewX | Optional, only present if bit 8 of `flags` is set |
| F4DOT12 | SkewY | Optional, only present if bit 9 of `flags` is set |
| FWORD | TCenterX | Optional, only present if bit 10 of `flags` is set |
| FWORD |  TCenterY | Optional, only present if bit 11 of `flags` is set |


### Variable Component Transformation

The transformation data consists of individual optional fields, which can be
used to construct a transformation matrix.

Transformation fields:

| name | default value |
|-|-|
| TranslateX | 0 |
| TranslateY | 0 |
| Rotation | 0 |
| ScaleX | 1 |
| ScaleY | 1 |
| SkewX | 0 |
| SkewY | 0 |
| TCenterX | 0 |
| TCenterY | 0 |

The `TCenterX` and `TCenterY` values represent the “center of transformation”.

Details of how to build a transformation matrix, as pseudo-Python code:

```python
# Using fontTools.misc.transform.Transform
t = Transform()  # Identity
t = t.translate(TranslateX + TCenterX, TranslateY + TCenterY)
t = t.rotate(Rotation * math.pi)
t = t.scale(ScaleX, ScaleY)
t = t.skew(-SkewX * math.pi, SkewY * math.pi)
t = t.translate(-TCenterX, -TCenterY)
```

### Variable Component Flags

| bit number | meaning |
|-|-|
| 0 | Use my metrics |
| 1 | axis indices are shorts (clear = bytes, set = shorts) |
| 2 | if ScaleY is missing: take value from ScaleX |
| 3 | have TranslateX |
| 4 | have TranslateY |
| 5 | have Rotation |
| 6 | have ScaleX |
| 7 | have ScaleY |
| 8 | have SkewX |
| 9 | have SkewY |
| 10 | have TCenterX |
| 11 | have TCenterY |
| 12 | gid is 24 bit |
| 13 | axis indices have variation |
| 14 | reset unspecified axes |
| 15 | (reserved, set to 0) |

## Processing

Variations of composite glyphs are processed this way: For each composite glyph, a vector of coordinate points is prepared, and its variation applied from the `gvar` table. The coordinate points for the composite glyph are a concatenation of those for each variable component in order. For the purposes of `gvar` delta IUP calculations, each point is considered to be in its own contour.

The coordinate points for each variable component consist of those for each axis value if flag bit 13 is set, represented as the X value of a coordinate point; followed by up to five points representing the transformation. The five possible points encode, in order, in their X,Y components, the following transformation components: (`TranslateX`,`TranslateY`), (`Rotation`,0), (`ScaleX`,`ScaleY`), (`SkewX`,`SkewY`), (`TCenterX`,`TCenterY`). Only the transformation components present according to the flag bits are encoded. For example, the point (`TranslateX`,`TranslateY`) is encoded if and only if either of flag 3 or 4 is set. If that point is encoded, the variations from it will be applied to the transformation components even if a specific component has its flag bit off. For example, if only flag 3 is set (`TranslateX`), any variations for `TranslateY` are still applied to the `TranslateY` of the transformation.

The component glyphs to be loaded use the coordinate values specified (with any variations applied if present). For any unspecified axis, the value used depends on flag bit 14. If the flag is set, then the normalized value zero is used. If the flag is clear the axis values from current glyph being processed (which itself might recursively come from the font or its own parent glyphs) are used.  For example, if the font variations have `wght`=.25 (normalized), and current glyph being processed is using `wght`=.5 because it was referenced from another VarComposite glyph itself, when referring to a component that does _not_ specify the `wght` axis, if flag bit 14 is set, then the value of `wght`=0 (default) will be used. If flag bit 14 is clear, `wght`=.5 (from current glyph) will be used.

**Note:** While it is the (undocumented?) behavior of the `glyf` table that glyphs loaded are shifted to align their LSB to that specified in the `hmtx` table, much like regular Composite glyphs, this does not apply to component glyphs being loaded as part of a VariableComposite glyph.
