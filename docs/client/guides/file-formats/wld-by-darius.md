---
description: Version 1.0, by Darius
---

# WLD File Reference

## Overview

.WLD files are, usually binary, data files that store game objects in EverQuest. They are located in .S3D files. This is mostly a copy of Windcatcher's document with changes based on my understanding of WLD files currently. New information comes from a few sources:

    An overview of object types produced by the WLDCOM.EXE program that came packed with some versions of the Tanarus client - https://wld-doc.github.io/object-types/overview

    ASCII data files found in the chequip.s3d archive file, and some others, that was included in some versions of the EverQuest client. These have file extensions like .sps for texture info, .mdf for materials, .spk for animated models, .spm for non-animated models, etc..

    Decompilations of the code in various versions of the EverQuest client software. Much of this data is based on the TAKP/EQMac client, since that is the oldest one I have the ability to run.

Here's the list of the fragment types as surrently understood:

```
0x01 - DefaultPaletteFile
0x02 - UserData
0x03 - BMInfo
0x04 - SimpleSpriteDef
0x05 - SimpleSprite
0x06 - 2DSpriteDef
0x07 - 2DSprite
0x08 - 3DSpriteDef
0x09 - 3DSprite
0x0A - 4DSpriteDef
0x0B - 4DSprite
0x0C - ParticleSpriteDef
0x0D - ParticleSprite
0x0E - CompositeSpriteDef
0x0F - CompositeSprite
0x10 - HierarchicalSpriteDef
0x11 - HierarchicalSprite
0x12 - TrackDef
0x13 - Track
0x14 - ActorDef
0x15 - Actor
0x16 - Sphere
0x17 - PolyhedronDef
0x18 - Polyhedron
0x19 - SphereListDef
0x1A - SphereList
0x1B - LightDef
0x1C - Light
0x1D - PointLightOld
0x1E - PointLightOldDef
0x1F - SoundDef
0x20 - Sound
0x21 - WorldTree
0x22 - Region
0x23 - ActiveGeoRegion
0x24 - SkyRegion
0x25 - DirectionalLightOld
0x26 - BlitSpriteDef
0x27 - BlitSprite
0x28 - PointLight
0x29 - Zone
0x2A - AmbientLight
0x2B - DirectionalLight
0x2C - DMSpriteDef
0x2D - DMSprite
0x2E - DMTrackDef
0x2F - DMTrack
0x30 - MaterialDef
0x31 - MaterialPalette
0x32 - DmRGBTrackDef
0x33 - DmRGBTrack
0x34 - ParticleCloudDef
0x35 - GlobalAmbientLightDef
0x36 - DmSpriteDef2
0x37 - DmTrackDef2
```

## .WLD File Organization

Data in .WLD files is encoded little-endian, so 0x54503D02 will appear as "02 3D 50 54" in a hex editor. .WLD files can be broken into three sections:

* Header
* String hash
*   Fragments

    .WLD Header

### The .WLD header consists of seven DWORDs:

#### Magic: DWORD

This always contains 0x54503D02. It identifies the file as a .WLD file.

#### Version: DWORD

For old-format .WLD files, this always contains 0x00015500. For new-format .WLD files, this always contains 0x1000C800. Apparently, if the upper half of the DWORD is 0x1000, then it is "new world". Seen with Luclin-era zones and after. 

#### FragmentCount: DWORD

Contains the number of fragments in the .WLD file, minus 1 (that is, the highest fragment index, starting at 0).

#### RegionCount: DWORD

Contains the number of regions in the file. Should only be > 0 in zone files.

#### MaxFragmentSize: DWORD

Contains the size, in bytes, of the largest fragment in the file.

#### StringHashSize: DWORD

Contains the size of the string hash in bytes. All strings in .WLD files are XOR-encoded using the following rotating set of flags:

```
0x95, 0x3A, 0xC5, 0x2A, 0x95, 0x7A, 0x95, 0x6A
```

That is, the first byte is XOR’ed with 0x95, the second byte with 0x3A, and so on. The set repeats at the ninth byte. Repeating the operation decodes the string. Kudos to the original author of ZoneConverter for figuring this out.

The first byte of the string hash is always a junk value (actually encoded zero which results in 0x95) and is used for fragments that have no string name. Encoded strings therefore start at position 1 in the string hash. The string hash is nothing more than a bunch of null-terminated strings that have been concatenated together and encoded.

#### StringCount: DWORD

Contains the number of strings in the string hash. 

## Comprehensive Fragment Reference Introduction

There are two basic kinds of fragments: plain and reference. Plain fragments are pure data structures that begin with three 32-bit fields, whereas reference fragments also come with a 32-bit reference to another fragment. The ID contains the specific fragment type. This type determines what follows the fragment header data and whether the fragment is a plain or reference fragment.

### Basic fragments

Almost all fragments (plain and reference) begin with the following data:

#### Size: DWORD

Size of the fragment in bytes. All fragments are padded such that Size is evenly divisible by 4 and Size should reflect the padded value.

#### ID: DWORD

The fragment type. This will typically be a value in the range 0x03 to 0x37 and tells the file reader which specific kind of fragment it is. Some fragment types are plain fragments and some are reference fragments: the ID determines which.

#### NameReference: DWORD

Most fragment types will have a string name, which is stored in encoded form in the .WLD file’s string hash. The NameReference gives a way to retrieve the name. If the fragment has a string name, its NameReference should contain the negative value of the string’s index in the string hash. For example, if the string is at position 31 in the string hash, then NameReference should contain –31. A value of 0 means the fragment doesn't have a name reference.

The client code apparently treats this field as conditional. If it is > 0, it references a fragment directly through its index, or position, within the .WLD file, and if it is < 0 it is an actual name reference. It is almost always only zero or < 0 (negative). For more information on how the condition works, see the Reference entry.

There are a few fragment types that don't have a name reference, including fragment types 0x01 (DefaultPaletteFile), 0x02 (UserData), and 0x35 (GlobalAmbientLightDef). 

#### Reference fragments

Reference fragments contain this additional field:

#### Reference: DWORD

This can be either a string reference or a fragment reference. If it is a fragment reference it must reference a fragment that has already been loaded from the file. It can reference it using one of two ways:

1. By name
2. By fragment index

If Reference contains a value that is less than zero, the value is negated and 1 is subtracted from it. Then a null-terminated string is loaded from the string hash, starting at that position. Every fragment that has been loaded is checked to see if its name matches the string that was loaded. If a match is found, then the reference is considered to point to that fragment.

If a match is not found, then the reference is considered to point only to that string instead of to a fragment. There may be cases where certain fragments point to “magic” strings that cause the client to do something special. This is not confirmed. The Polyhedron reference for DMSpriteDef2 fragments sometimes have the value "-2" if the hex 0x10000 flag is set, but in practice it does not seem to matter what value in in the Polyhedron reference if DMSpriteDef2 flag 0x10000 is set.

If Reference contains a value greater than zero then it is considered to be a direct reference to another fragment, based on the order in which they were loaded from the file.

All fragments are padded to end on DWORD boundaries. The Size field above must reflect this padding.

## 0x01 — DefaultPaletteFile

### Notes

This fragment simply contains the name of the palette.bmp file that is present in a lot of .S3D files. I have never seen one in an actual EverQuest .WLD file, but it seems to be read by some of the clients, and can be added manually. Doesn't seem to have any effect in-game.

### Fields

#### NameLength: WORD

Contains the length of the filename in bytes, +1 (padded with a null character).

#### FileName: BYTEs

The filename. May have to be encoded like the string hash names. See the introduction above for a description of string coding.

## 0x02 — UserData

### Notes

This fragment contains a string. Have not seen it in Tanarus files, and not sure of its use. I have never seen one in an actual EverQuest .WLD file, it seems to be read by some of the clients, and can be added manually. Doesn't seem to have any effect in-game. UserData is a property of some other fragment types, though.

### Fields

#### Length: DWORD

Contains the length of the userdata in bytes.

#### Data: BYTEs

The userdata. May have to be encoded like the string hash names. See the introduction above for a description of string coding.

## 0x03 — BMInfo

### Notes

This fragment references one or more texture filenames (BM standing for BitMap?). WLDCOM.EXE variably calls this BMINFO or FRAME. If one texture is referenced, it will call it "FRAME" and if multiple, it will call it "BMINFO". BMInfo also seems to be what it is called in client code. Most of these fragments will only have one texture reference. 

Layered textures, like for Luclin player character models, and some terrain textures starting with Luclin, will have 2 texture references; one for a base texture, and one for a overlay. In these cases, the decoded names of the files may have extra information outside of the the actual filename that controls how they are used by the client. For instance, Luclin player character model files will will often have:

ELFCHSK01.DDS\
ELFCH0001.DDS_LAYER

The "_LAYER" texure (ELFCH0001.DDS) will be applied over the base texture (ELFCHSK01.DDS).

and for some terrain textures:

TWICLIFF02.DDS\
ROCK05D.DDS_DETAIL_4.000000

The "_DETAIL" texture (ROCK05D.DDS) will be applied over the base texture (TWICLIFF02.DDS) and a scale will be applied of (4.000000) to the "_DETAIL" texture's UVs.

Many Luclin outdoor zones have more than 2 texture files referenced by the same BMInfo fragment for a system that uses a base texture, a indexed-color bitmap as a mask for multiple textures simultaneously, and then the overlay textures, like this:

TWIBASE1C.DDS\
TWIBASE1CPAL.BMP\
1, 1, 2, GRASS2E.DDS\
2, 1, 3, GRASS2D.DDS\
3, 1, 1, SAND02A.DDS\
4, 1, 0, GRASS2E.DDS\
5, 1, 0, GRASS2D.DDS\
6, 1, 0, SAND02A.DDS

Here TWIBASE1C.DDS is a base texture. TWIBASE1CPAL.BMP is an indexed-color bitmap with only a few different colors. The palette bitmap looks roughly like the base texture. 

The first comma-separated number, before the overlay texture filenames, lists which indexed color in the palette bitmap that texture will appear at. In reality, the actual number can be changed and it does not affect how the texture will be applied, only their order dictates that. The overlay texture will appear semi transparent over the base texture in-game. 

The middle number affects the scaling of the UVs of the overlay texture. The higher the number, the smaller the tiles will be. 

The third number doesn't seem to do anything. The numbers can apparently only be integers.

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### TextureCount: DWORD

Contains the number of texture filenames in this fragment, minus 1. For instance, if there is only one texture reference, TextureCount would be 0. There will be a NameLength and then a FileName for TextureCount 0, and then another NameLength and FileName for each TextureCount > 0.

#### NameLength: WORD

Contains the length of the filename in bytes, +1 (padded with a null character).

#### FileName: BYTEs

The encoded filename, equal to the preceeding NameLength. See the introduction above for a description of string coding.

## 0x04 — SimpleSpriteDef

### Notes

A simple sprite is basically a reference to one or more BMInfo fragments as "frames". If it contains only one frame, it will be a simple texture, and if it contains multiple frames, it can be animated. 

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### Flags: DWORD

0x04 - If set, the CurrentFrame field will exist.\
0x08 - If set, the Sleep field will exist.\
0x10 - Always seems to be set, probably "HAVESKIPFRAMES", which allows the 0x40 flag to be read, but doesn't seem to be read by the client.\
0x40 - Toggle for "SKIPFRAMES". I have no idea what it does does.

#### FrameCount: DWORD

Contains the number of 0x03 BMInfo references.

#### CurrentFrame: DWORD

Exists if 0x04 flag is set. Doesn't seem to do anything.

#### Sleep: DWORD

Exists if 0x08 flag is set. It is the time in milliseconds between frames. If there are multiple frames and Sleep exists, then the simple sprite will be animated.

#### Frames: DWORDs

There will be FrameCount DWORDs that are always direct fragment index references to 0x03 BMInfo fragments. An animated simple sprite uses the order of these Frames as the order of the animation.

## 0x05 — SimpleSprite

### Notes

Instance of a SimpleSpriteDef. Fragments that need to use a SimpleSpriteDef will always reference them through these fragments. 

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### SpriteReference: DWORD

Reference to a 0x04 SimpleSpriteDef fragment. See "Basic fragments - Reference" for details.

#### Flags: DWORD

0x10 - Always seems to be set, probably "HAVESKIPFRAMES", which allows the 0x40 flag to be read.\
0x40 - Toggle for "SKIPFRAMES". I have no idea what it does does.

## 0x06 — 2DSpriteDef

### Notes

This fragment is rarely used. It describes objects that are purely two-dimensional in nature. Examples are coins and blood spatters.

### Fields

#### Flags: DWORD

Its purpose is unknown. The function of the known bits is as follows:

Bit 0 ........ If 1, Params3 exists. \
Bit 1 ........ If 1, Params4 exists. \
Bit 2 ........ If 1, Params5 exists. \
Bit 3 ........ If 1, Params6 exists. \
Bit 7 ........ If 1, Params2 exists.

#### SubSize1 : DWORD

Its purpose is unknown.

#### Size1 : DWORD

Its purpose is unknown.

#### Params1 : 2DWORDs

Its purpose is unknown, though I suspect it might be the object’s size.

#### Fragment : DWORD

Its purpose is unknown.

#### Params2 : FLOAT

Its purpose is unknown. It only exists if bit 7 of Flags is 1.

#### Params3 : 3 DWORDs

Their purpose is unknown. They only exist if bit 0 of Flags is 1.

#### Params4 : FLOAT

Its purpose is unknown. It only exists if bit 1 of Flags is 1.

#### Params5 : DWORD

Its purpose is unknown. It only exists if bit 2 of Flags is 1.

#### Params6 : DWORD

Its purpose is unknown, though it typically contains 100. It only exists if bit 3 of Flags is 1. Data1 entries (there are Size1 of these):

#### Data6Params1 : DWORD

Its purpose is unknown. It typically contains 512 (0x200).

#### Data6Flags : DWORD

The most significant bit of this field (0x80000000) is a flag of some sort. The other bits constitute another size field which we shall call Data6Size here.

Data6Data entries (there are Data6Size of these):

#### Data6DataParams1 : DWORD

Its purpose is unknown. It typically contains 64 (0x40).

#### Data6DataFragments : SubSize1 DWORDs

These point to one or more 0x03 Texture Bitmap Name fragments (one if the object is static or more than one if it has an animated texture, such as blood from a weapon strike).

#### Params7Params1 : DWORD

Its purpose is unknown.

#### Params7Flags : DWORD

Its purpose is unknown. The function of the known bits is as follows:

Bit 0 ........ If 1, Params7Params2 exists. \
Bit 1 ........ If 1, Params7Params3 exists. \
Bit 2 ........ If 1, Params7Params4 exists. \
Bit 3 ........ If 1, Params7Fragment exists. \
Bit 4 ........ If 1, Params7Matrix exists.\
Bit 5 ........ If 1, Params7Size and Params7Data exist.

#### Params7Params2 : DWORD

Its purpose is unknown. Only exists if bit 0 of Params7Flags is 1.

#### Params7Params3 : FLOAT

Its purpose is unknown. Only exists if bit 1 of Params7Flags is 1.

#### Params7Params4 : FLOAT

Its purpose is unknown. Only exists if bit 2 of Params7Flags is 1.

#### Params7Fragment : DWORD

Its purpose is unknown. Only exists if bit 3 of Params7Flags is 1.

#### Params7Matrix : 9 DWORDs

Its purpose is unknown, though it looks like some sort of transformation matrix. Only exists if bit 4 of Params7Flags is 1.

#### Params7Size : DWORD

Its purpose is unknown. Only exists if bit 5 of Params7Flags is 1.

#### Params7Data : (Params7Size * 2) DWORDs

Their purpose is unknown. Only exists if bit 5 of Params7Flags is 1.

## 0x07 — 2DSprite

Reference points to a 0x06 Two-dimensional Object fragment. 

### Fields

#### Flags : DWORD

Its purpose is unknown, but it always seems to contain 0.

## 0x08 — 3DSpriteDef

### Notes

This type of fragment was originally also used for 3D mob models in Tanarus, but seems to be only used for the "CAMERA_DUMMY" object that the player actor references since the earliest versions of the EverQuest client. Changing the values of the various fields doesn't seem to do much.

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details. In main zone files, the name of this fragment always seems to be "CAMERA_DUMMY".

#### Flags: DWORD

0x01 - If set, CenterOffset exists.\
0x02 - If set, BoundingRadius exists.\
0x20 - This is a toggle for the normals for lighting to be set from face normals (if not set) or smooth shaded normals (if set), for the 3d meshes that this type of fragment was originally used for. Called "ENABLEGOURAUD2" in WLDCOM.EXE printout.\
0x40 - If set, 4 Normal FLOATs will exist at the end of each BspNode.

#### VertexCount: DWORD

The is the number of vertices in the "mesh". Vertices are first defined globally for the mesh and then referenced by the BspNodes by index. In EverQuest it is always the same 4 vertices.

#### BspNodeCount: DWORD

Number of BspNodes that are detailed further down. There is always only 1 in this fragment in EverQuest, but in Tanarus this could have many nodes, and each one seemed to be a piece of the mesh with a unique material.

#### SphereListRef: DWORD

There is a reference to a 0x1a SphereList fragment, which in turn references a 0x19 SphereListDef fragment. This was apparently a collision volume in Tanarus, but doesn't seem to be used in any EverQuest files.

#### CenterOffset X: FLOAT

This field and the next two fields only exist if the 0x01 flag is set. They never exist in this type of fragment in EverQuest, but in Tanarus, many models have this set. Mostly likely adjusts the position of the model. In WLDCOM.EXE printouts, it is formatted exactly like other CenterOffset values, so this is surely the X axis component.

#### CenterOffset Y: FLOAT

Only exists if the 0x01 flag is set. This is similar to CenterOffset X but references the Y axis.

#### CenterOffset Z: FLOAT

Only exists if the 0x01 flag is set. This is similar to CenterOffset X but references the Z axis.

#### BoundingRadius: FLOAT

This field only exists if the 0x02 flag is set. If this works anything like in EverQuest for other mesh types, it most likely was the greatest distance any vertex could be from the center of the mesh. It never exists for the fragment type in EverQuest, but in Tanarus many models had this set.

**Vertex entries (there are VertexCount of these):**

#### Vertex X: FLOAT

X component of the vertex position.

#### Vertex Y: FLOAT

Y component of the vertex position.

#### Vertex Z: FLOAT

Z component of the vertex position.

**BSP Node fields (there are BspNodeCount of all these fields potentially, but practically only ever one set):**

#### VertexIndexCount: DWORD

This is the number of vertices in the BSP node itself, and dictates the number of VertexIndex entries farther down.

#### RenderMethod: DWORD

This is the same RenderMethod field that is detailed in the 0x30 MaterialDef fragment. It is only ever zero in this fragment type in EverQuest, which is fully transparent. Changing the this value doesn't seem to do anything. Since the 0x30 MaterialDef fragment is far more common, and RenderMethod seems to be functional in that fragment type, please refer to that entry for an explanation of the field.

#### RenderInfoFlags: DWORD

This contains numerous different flags for controlling if various RenderInfo fields exist or not. Many of these RenderInfo fields seem similar to the fields in the 0x30 MaterialDef fragment, but they are not optional in the 0x30 MaterialDef fragment type.

0x01 - If set, the Pen value exists.\
0x02 - If set, Brightness exists.\
0x04 - If set, ScaledAmbient exists.\
0x08 - If set, the SimpleSpriteReference exists.\
0x10 - If set, there will be 3 UVOrigin FLOATs, then 3 U-Axis FLOATs, then 3 V-Axis FLOATs. Otherwise, these 9 FLOATs will not exist.\
0x20 - If set, there will be a UVCount DWORD, followed by a UVCount number of UV FLOAT pairs. Otherwise, these fields will not exist.\
0x40 - This seems to be the same flag that causes the 0x36 MaterialDef fragment to become a two-sided material if it is set.

#### RenderInfoPen: DWORD

This field only exists if the 0x01 RenderInfoFlag is set. This is apparently a reference to an index in a color palette. There is no color data in the actual value. This is always "11" in this fragment in EverQuest.

#### RenderInfoBrightness: FLOAT

This field only exists if the 0x02 RenderInfoFlag is set. Brightness is sometimes referred to as "constant intensity" in some versions of the client. This is never present in this fragment type in EverQuest, but I have seen it in Tanarus models.

#### RenderInfoScaledAmbient: FLOAT

This field only exists if the 0x04 RenderInfoFlag is set. This is never present in this fragment type in EverQuest, but I have seen it in Tanarus models. ScaledAmbient also exsits on 0x30 MaterialDef fragments, but unsure of what it does there as well.

#### RenderInfoSimpleSpriteReference: DWORD

This field only exists if the 0x08 RenderInfoFlag is set. It can contain a reference to a 0x05 SimpleSprite reference fragment, which in turn references a 0x04 SimpleSpriteDef fragment. Seems to contain the texture references for Tanarus 3D mob models, but in EverQuest, this field never exists in this fragment type.

#### RenderInfoUVOrigin X: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Not sure of its purpose, but it seems to set up the UV coordinate system, along with the other RenderInfo UV fields. Seems to be the X coordinate of the UVOrigin. This field never exists in this fragment type in EverQuest.

#### RenderInfoUVOrigin Y: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Seems to be the Y coordinate of the UVOrigin. This field never exists in this fragment type in EverQuest.

#### RenderInfoUVOrigin Z: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Seems to be the Z coordinate of the UVOrigin. This field never exists in this fragment type in EverQuest.

#### RenderInfoUAxis [0]: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Possibly part of the vector that the U-Axis of the UV coordinate system sits along, along with the other 2 RenderInfoUAxis fields. This field never exists in this fragment type in EverQuest.

#### RenderInfoUAxis [1]: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Possibly part of the vector that the U-Axis of the UV coordinate system sits along, along with the other 2 RenderInfoUAxis fields. This field never exists in this fragment type in EverQuest.

#### RenderInfoUAxis [2]: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Possibly part of the vector that the U-Axis of the UV coordinate system sits along, along with the other 2 RenderInfoUAxis fields. This field never exists in this fragment type in EverQuest.

#### RenderInfoVAxis [0]: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Possibly part of the vector that the V-Axis of the UV coordinate system sits along, along with the other 2 RenderInfoVAxis fields. This field never exists in this fragment type in EverQuest.

#### RenderInfoVAxis [1]: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Possibly part of the vector that the V-Axis of the UV coordinate system sits along, along with the other 2 RenderInfoVAxis fields. This field never exists in this fragment type in EverQuest.

#### RenderInfoVAxis [2]: FLOAT

This field only exists if the 0x10 RenderInfoFlag is set. Possibly part of the vector that the V-Axis of the UV coordinate system sits along, along with the other 2 RenderInfoVAxis fields. This field never exists in this fragment type in EverQuest.

#### UVCount: DWORD

This field only exists if the 0x20 RenderInfoFlag is set. It contains the number of UV entries that follows. 

**UV coordinate entries (there are UVCount of these)**

#### UV U: FLOAT

Only exists if the 0x20 RenderInfoFlag is set, and UVCount is > 0. It is the U component of the texture UV.

#### UV V: FLOAT

Only exists if the 0x20 RenderInfoFlag is set, and UVCount is > 0. It is the V component of the texture UV.

#### Normal A: FLOAT

Exists, for each BSP node, if the 0x40 Flags value is set. This is the A component of the BSP node normal. Not sure what it does. In a regular BSP tree, it describes the plane that separates the fronttree and backtree, when traversing the tree. Values A, B, and C are a vector that is the plane normal, and D is the distance the plane from the origin. 

#### Normal B: FLOAT

Exists, for each BSP node, if the 0x40 Flags value is set. This is the B component of the BSP node normal.

#### Normal C: FLOAT

Exists, for each BSP node, if the 0x40 Flags value is set. This is the C component of the BSP node normal.

#### Normal D: FLOAT

Exists, for each BSP node, if the 0x40 Flags value is set. This is the D component of the BSP node normal.

## 0x09 — 3DSprite

### Notes

Reference fragment for the 0x08 3DSpriteDef fragment. In EverQuest, this is referenced only by the PLAYER ActorDef. 

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details. Always zero for this fragment type.

#### 3DSpriteDefRef: DWORD

Reference to a 0x08 3DspriteDef fragment.

#### Flags: DWORD

Always 0. Doesn't seem to be read by the client.

## 0x10 — HierarchicalSpriteDef

### Notes

This fragment describes a skeleton for an entire animated model, and is used for mob models, and placed objects. Basically, only the parent-child relationship between the bones of the skeleton are stored in this fragment, and their positions and movements are stored in the 0x12 TrackDef fragments. Bones are called DAGs in the EverQuest client code with respect to .WLD files.

For each piece there is a 0x13 Mob Skeleton Piece Track Reference fragment, which references one 0x12 Mob Skeleton Piece Track fragment. Each 0x12 fragment defines how that piece is rotated and/or shifted relative to its parent piece.

### Fields

#### Flags : DWORD

Bit 0 ........ If 1, Params1[0..2] fields exist.\
Bit 1 ........ If 1, Params2 exists.\
Bit 9 ........ If 1, Size2, Fragment3, and Data3 fields exist.

#### Size1 : DWORD

Number of track reference entries (see below)

#### Fragment : DWORD

Optionally points to a 0x18 Polygon Animation Reference? fragment.

#### Params1[0] : DWORD

Unknown purpose. Only exists if bit 0 of Flags is 1.

#### Params1[1] : DWORD

Unknown purpose. Only exists if bit 0 of Flags is 1.

#### Params1[2] : DWORD

Unknown purpose. Only exists if bit 0 of Flags is 1.

#### Params2 : FLOAT

Unknown purpose.

Entries (there are Size1 of them):

#### Entry1NameReference : DWORD

This seems to refer to the name of either this or another 0x10 fragment. It seems that at least one name reference points to the name of this fragment.

#### Entry1Flags : DWORD

Usually zero.

#### Entry1Fragment1 : DWORD

Reference to a 0x13 Mob Skeleton Piece Track Reference fragment.

Important: animated models generally only reference a basic set of fragments necessary to render the model but not animate it. There will generally be other sets of 0x13 fragments where each set corresponds to a different animation of the model. Software reading .WLD files must use the name of the first 0x13 fragment referenced by the 0x10 Skeleton Track Set to discover any other animation sets. The first fragment of any alternate animation set will have the same name as the first 0x13 fragment, with an additional prefix. All other 0x13 fragments in that same set will likewise correspond to their counterparts in the basic animation set. Different animation sets will have different prefixes (e.g. “C01” for one combat animation, “C02” for another combat animation, etc.). All alternate animation sets for a particular model generally immediately follow the 0x10 Skeleton Track Set fragment (with the 0x11 Skeleton Track Set Reference immediately following those). I don’t know if this is a necessary arrangement.

#### Entry1Fragment2 : DWORD

Sometimes refers to a 0x2D Mesh Reference fragment.

#### Entry1Size : DWORD

Tells how many Entry1Data entries there are.

#### Entry1Data : DWORDs

Each of these contains the index of the next piece in the skeleton tree. A Skeleton Track Set is a hierarchical tree of pieces in the skeleton. It generally starts with a central “stem” and branches out to a skeleton’s extremities. For instance, the first entry might be the stem; that entry might point to the pelvis entry; the pelvis entry might point to the left thigh, right thigh, and chest entries; and those entries would each point to other parts of the skeleton. The exact topography of the tree depends upon the overall structure of the skeleton. The proper way to use a Skeleton Track Set fragment is to start with the first entry and recursively walk the tree by following each entry’s Entry1Data field to other connected pieces.

It’s also worth noting that, although an entry might reference a 0x13 Mob Skeleton Piece Track Reference fragment in its EntityFragment1 field, that does not mean it will be valid for rendering (see the 0x12 Mob Skeleton Piece Track fragment for more information). Many model skeletons apparently contain extraneous pieces that have an unknown purpose, though I suspect that they are for determining attachment points for weapons and shields and are otherwise not meant to be rendered. These pieces are generally not referenced by the 0x36 Mesh fragments that the skeleton indirectly references (via 0x2D Mesh Reference fragments).

#### Size2 : DWORD

Tells how many Fragment3 and Data3 entries there are. This field only exists if the proper bit in the Flags field is set.

#### Fragment3 : DWORDs

There are Size2 of these. This field only exists if the proper bit in the Flags field is set. These entries generally point to 0x2D Mesh Reference fragments and outline all of the meshes in the animated model. For example, there might be a mesh for a model’s body and another one for the head.

#### Data3 : DWORDs

There are Size2 of these. It’s unknown what they typically contain. This field only exists if the proper bit in the Flags field is set.

## 0x11 — Skeleton Track Set Reference — REFERENCE

Reference points to a 0x10 Skeleton Track Set fragment. 

### Fields

#### Params1 : DWORD

Apparently must be zero.

## 0x12 — Mob Skeleton Piece Track — PLAIN

### Notes

This fragment describes how a skeleton piece is shifted or rotated relative to its parent piece. The overall skeleton is contained in a 0x10 Skeleton Track Set fragment and is structured as a hierarchical tree (see that fragment for information on how skeletons are structured). The 0x12 fragment contains information on how that particular skeleton piece is rotated and/or shifted relative to its parent piece.

Rotation and shifting information is contained as a series of fractions. The fragment contains one denominator value for rotation and another for translation (X, Y, Z, shift). It contains one numerator each for X, Y, Z rotation and shift, for a total of eight values. For rotation, the resulting value should be multiplied by Pi / 2 radians (i.e. 1 corresponds to 90 degrees, 2 corresponds to 180 degrees, etc.).

### Fields

For rendering polygons, the X, Y, Z rotation and shift information in this fragment should be taken into account by adding them to the rotation and shift values passed from the parent piece (that is, rotation and shift are cumulative). However, before adding the shift values, the X, Y, and Z shift values should first be rotated according to the parent’s rotation values. The rotation values in this fragment represent the orientation of this piece relative to the parent so calculating its starting position should **not** take its own rotation into account. Software rendering a skeleton piece should perform the following steps in this order:

* Calculate the X, Y, and Z shift values from this fragment
* Rotate the shift values according to the rotation values from the parent piece
* Add the shift values to the shift values from the parent piece
* Calculate the X, Y, and Z rotation values from this fragment
* Add the rotation values to the rotation values from the parent piece
*   Adjust the vertices for this piece by rotating them using the new rotation values and then shifting them by the new

    shift values (or save the rotation and shift values for this piece to be looked up later on when rendering)
* Process the next piece in the tree with the new rotation and shift values
*   When all pieces have been processed, render all meshes in the model, using either the adjusted vertex values

    (more efficient) or looking up the corresponding piece for each vertex and adjusting the vertex values according to the adjusted rotation and shift values calculated above (less efficient).

#### Flags : DWORD

Bit 3 ........ If 1, Data2 exists (though I’m not at all sure about this since I have yet to see an example). It could instead mean that the rotation and shift entries are unsigned DWORDs or it could mean that they’re 32-bit FLOATs.

#### Size : DWORD

Tells how many Data1 and Data2 entries there are.

#### RotateDenominator : SIGNED WORD (signed 16-bit value)

This represents the denominator for the piece’s X, Y, and Z rotation values. It’s vital to note that it is possible to encounter situations where this value is zero. I have seen this for pieces with no vertices or polygons and in this case rotation should be ignored (just use the existing rotation value as passed from the parent piece). My belief is that such pieces represent attachment points for weapons or items (e.g. shields) and otherwise don’t represent a part of the model to be rendered.

#### RotateXNumerator : SIGNED WORD (signed 16-bit value)

The numerator for rotation about the X axis.

#### RotateYNumerator : SIGNED WORD (signed 16-bit value)

The numerator for rotation about the Y axis.

#### RotateZNumerator : SIGNED WORD (signed 16-bit value)

The numerator for rotation about the Z axis.

#### ShiftXNumerator : SIGNED WORD (signed 16-bit value)

The numerator for translation along the X axis.

#### ShiftYNumerator : SIGNED WORD (signed 16-bit value)

The numerator for translation along the Y axis.

#### ShiftZNumerator : SIGNED WORD (signed 16-bit value)

The numerator for translation along the Z axis.

#### ShiftDenominator : SIGNED WORD (signed 16-bit value)

The denominator for the piece X, Y, and Z shift values. Like the rotation denominator, software should check to see if this is zero and ignore translation in that case.

#### Data2 : 4 DWORDs

There are (4 x Size) DWORDs here. Their purpose is unknown. This field exists only if the proper bit in Flags is set. It’s possible that this is a bogus field and really just represents the above fields in some sort of 32-bit form.

## 0x13 — Mob Skeleton Piece Track Reference — REFERENCE

Reference points to a 0x12 Mob Skeleton Piece Track fragment. 

### Fields

#### Flags : DWORD

Bit 0 ........ If 1, Params1 exists. \
Bit 2 ........ Usually set to 1.

#### Params1 : DWORD

Unknown purpose. It’s usually set to 1000, but I’ve also seen it set to 100. My guess is that it might have to do with animation speed.

## 0x14 — Static or Animated Model Reference/Player Info — PLAIN

### Notes

When this fragment is used in a main zone file, the name of the fragment seems to always be PLAYER_1.

### Fields

#### Flags : DWORD

Bit 0 ........ If 1, Params1 exists.\
Bit 1 ........ If 1, Params2 exists.\
Bit 7 ........ If 0, Fragment2 must contain 0.

#### Fragment1 : DWORD

This isn’t really a fragment reference but a string reference. It points to a “magic” string. When this fragment is used in main zone files the string is “FLYCAMCALLBACK”. When used as a placeable object reference, the string is “SPRITECALLBACK”. When creating a 0x14 fragment this is currently accomplished by creating a fragment reference, setting the fragment to null, and setting the reference name to the magic string.

#### Size1 : DWORD

Tells how many entries there are (see below).

#### Size2 : DWORD

Tells how many Fragment3 entries there are (see below):

#### Fragment2 : DWORD

Unknown purpose.

#### Params1 : DWORD

This seems to always contain 0. It seems to only be used in main zone files.

#### Params2 : 7 DWORDs

These seem to always contain zeroes. They seem to only be used in main zone files. Entries (there are Size1 of these):

#### Entry1Size : DWORD

Tells how many Entry1Data DATAPAIRs there are.

#### Entry1Data : DATAPAIRs

Unknown purpose.

#### Fragment3: DWORDs

There are Size2 fragment references here. These references can point to several different kinds of fragments. In main zone files, there seems to be only one entry, which points to a 0x09 Camera Reference fragment. When this is instead a static object reference, the entry points to either a 0x2D Mesh Reference fragment. If this is an animated (mob) object reference, it points to a 0x11 Skeleton Track Set Reference fragment. This also has been seen to point to a 0x07 Two-dimensional Object Reference fragment (e.g. coins and blood spots).

#### Size3 : DWORD

Tells how many bytes are in the Name3 field.

#### Name3 : BYTEs

An encoded string. It’s purpose and possible values are unknown.

## 0x15 — Object Location — REFERENCE

When used in main zone files, the reference points to a 0x14 Player Info fragment. When used for static (placeable) objects, the reference is a string reference (not a fragment reference) and points to a “magic” string. It typically contains the name of the object with “_ACTORDEF” appended to the end.

### Fields

#### Flags : DWORD

Typically 0x2E when used in main zone files and 0x32E when used for placeable objects.

#### Fragment1 : DWORD

When used in main zone files, points to a 0x16 fragment. When used for placeable objects, seems to always contain 0. This might be due to the difference in the Flags value.

#### X : FLOAT

When used in main zone files, contains the minimum X value of the entire zone. When used for placeable objects, contains the X value of the object’s location.

#### Y : FLOAT

When used in main zone files, contains the minimum Y value of the entire zone. When used for placeable objects, contains the Y value of the object’s location.

#### Z : FLOAT

When used in main zone files, contains the minimum Z value of the entire zone. When used for placeable objects, contains the Z value of the object’s location.

#### RotateZ : FLOAT

When used in main zone files, typically contains 0. When used for placeable objects, contains a value describing rotation around the Z axis, scaled as Degrees x (512 / 360).

#### RotateY : FLOAT

When used in main zone files, typically contains 0. When used for placeable objects, contains a value describing rotation around the Y axis, scaled as Degrees x (512 / 360).

#### RotateX : FLOAT

When used in main zone files, typically contains 0. When used for placeable objects, contains a value describing rotation around the X axis, scaled as Degrees x (512 / 360).

#### Params1[3] : FLOAT

Typically contains 0 (though might be more significant for placeable objects).

#### ScaleY : FLOAT

When used in main zone files, typically contains 0.5. When used for placeable objects, contains the object’s scaling factor in the Y direction (e.g. 2.0 would make the object twice as big in the Y direction).

#### ScaleX : FLOAT

When used in main zone files, typically contains 0.5. When used for placeable objects, contains the object’s scaling factor in the X direction (e.g. 2.0 would make the object twice as big in the X direction).

#### Fragment2 : DWORD

When used in main zone files, typically contains 0 (might be related to the Flags value). When used for placeable objects, points to a 0x33 Vertex Color Reference fragment.

#### Params2 : DWORD

Typically contains 30 when used in main zone files and 0 when used for placeable objects. This field only exists if Fragment2 points to a fragment.

## 0x16 — Zone Unknown — PLAIN

### Fields

## 0x17 — Polygon Animation? — PLAIN

### Fields

#### Params1 : FLOAT

Typically contains 0.1. This fragment’s purpose is unknown.

#### Flags : DWORD

Bit 0 ........ If 0, Params2 must be 1.0.

#### Size1 : DWORD

Tells how many entries there are (see below).

#### Size2 : DWORD

Tells how many entries there are (see below).

#### Params1 : FLOAT

Unknown purpose.

#### Params2 : FLOAT

Usually 1.

Entries (there are Size1 of these):

#### Entry1X : FLOAT

Unknown purpose.

#### Entry1Y : FLOAT

Unknown purpose.

#### Entry1Z : FLOAT

Unknown purpose. Entries (there are Size2 of these):

#### Entry2Size : DWORD

Tells how many DWORDs there are in Entry2Data.

#### Entry2Data : DWORDs

There are Entry2Size of these. These appear to be indices into the X, Y, Z entries above.

## 0x18 — Polygon Animation Reference? — REFERENCE

Reference points to a 0x17 Polygon Animation? fragment. 

### Fields

#### Flags : DWORD

Bit 0 ........ If 1, Params1 exists.

#### Params1 : FLOAT

Unknown purpose.

## 0x1B — Light Source — PLAIN

When used in main zone files, the name of this fragment is typically DEFAULT_LIGHTDEF. 

### Fields

#### Flags : DWORD

Bit 1 ........ Typically 1 when dealing with placed light sources.\
Bit 2 ........ Typically 1.\
Bit 3 ........ Typically 1 when dealing with placed light sources. If Bit 4 is 1 then Params3b only exists if this bit is also 1 (not sure about this).\
Bit 4 ........ If 0, Params3a exists but Params3b and Params4[0..3] don’t exist. Otherwise, Params3a doesn’t exist but Params3b and Params4[0..3] do exist. This flag seems to determine whether the light is just a simple white light or a light with its own color values.

#### Params2 : DWORD

Typically contains 1.

#### Params3a : FLOAT

Typically contains 1.

#### Params3b : DWORD

Typically contains 200 (attenuation?).

#### Params4[0] : FLOAT

Typically contains 1.

#### Params4[1] : FLOAT

Light red component, scaled from 0 (no red component) to 1 (100% red).

#### Params4[2] : FLOAT

Light green component, scaled from 0 (no green component) to 1 (100% green).

#### Params4[3] : FLOAT

Light blue component, scaled from 0 (no blue component) to 1 (100% blue).

## 0x1C — Light Source Reference — REFERENCE

Reference points to a 0x1B Light Source fragment. 

### Fields:

#### Flags : DWORD

Typically contains zero.

## 0x21 — BSP Tree — PLAIN

### Fields:

#### Size1 : DWORD

Entries (there are Size1 of them):

#### Entry1NormalX : FLOAT

X component of the normal to the split plane.

#### Entry1NormalY : FLOAT

Y component of the normal to the split plane.

#### Entry1NormalZ : FLOAT

Z component of the normal to the split plane.

#### Entry1SplitDistance : FLOAT

Distance from the splitting plane to the origin (0, 0, 0) in x-y-z-space. With the above fields, the splitting plane is represented in Hessian Normal Form.

#### Entry1RegionID : DWORD

If this is a leaf node, contains the index of the 0x22 BSP Region fragment that this refers to (with the lowest index being 1). Otherwise contains zero.

#### Entry1Node1 : DWORD

If this is not a leaf node, contains the index of the entry in the tree corresponding to everything in the remaining area on one side of the splitting plane (with the lowest index containing zero). Otherwise contains zero.

#### Entry1Node2 : DWORD

If this is not a leaf node, contains the index of the entry in the tree corresponding to everything in the remaining area on the other side of the splitting plane (with the lowest index containing zero). Otherwise contains zero.

## 0x22 — BSP Region — PLAIN

### Fields

#### Flags : DWORD

Typically contains 0x181 for regions that contain polygons and 0x81 for regions that are empty.

Bit 5 ........ If 1, then the Data6Data consists of WORDs.\
Bit 7 ........ If 1, then the Data6Data consists of BYTEs (the usual).

#### Fragment1 : DWORD

It is unknown what this references. It usually doesn’t reference anything.

#### Size1 : DWORD

Tells how many bytes are in the Data1 field.

#### Size2 : DWORD

Tells how many bytes are in the Data2 field.

#### Params1 : DWORD

Typically contains zero. It’s purpose is unknown.

#### Size3 : DWORD

Tells how many Data3 entries there are (usually none).

#### Size4 : DWORD

Tells how many Data4 entries there are (usually none).

#### Params2 : DWORD

Typically contains zero. It’s purpose is unknown.

#### Size5 : DWORD

Tells how many Data5 entries there are (usually only 1).

#### Size6 : DWORD

Tells how many Data6 entries there are (usually only 1).

#### Data1 : BYTEs

According to the ZoneConverter source there are 12 * Size1 bytes here. Their format is unknown, for lack of sample data to figure it out.

#### Data2 : BYTEs

According to the ZoneConverter source there are 8 * Size2 bytes here. Their format is unknown, for lack of sample data to figure it out.

Data3 entries (There are Size3 of these):

#### Data3Flags : DWORD

Bit 1 ........ If 1, then the Data3Params1[0..2] and Data3Params2 fields exist.

#### Data3Size1 : DWORD

Tells how many Data3Data1 DWORDs there are.

#### Data3Data1 : DWORDs

There are Data3Size1 DWORDs. Their purpose is unknown.

#### Data3Params1[0] : DWORD

Unknown purpose. Only exists if Data3Flags Bit 1 is set to 1.

#### Data3Params1[1] : DWORD

Unknown purpose. Only exists if Data3Flags Bit 1 is set to 1.

#### Data3Params1[2] : DWORD

Unknown purpose. Only exists if Data3Flags Bit 1 is set to 1.

#### Data3Params2 : DWORD

Unknown purpose. Only exists if Data3Flags Bit 1 is set to 1. Data4 entries (There are Size4 of these):

#### Data4Flags : DWORD

Bit 2 ........ If 1, then the Data4Size1 and Data4Data1 fields exist.

#### Data4Params1 : DWORD

Unknown purpose.

#### Data4Type : DWORD

This seems to determine whether Data4Params2a and/or Data4Params2b exist.

#### Data4Params2a : DWORD

Unknown purpose. Only exists if Data4Type is greater than 7.

#### Data4Params2b : DWORD

Unknown purpose. Only exists if Data4Type is one of the following: 0x0A, 0x0B, 0x0C (though I’m not at all sure about this, due to a lack of sample data).

#### Data4NameSize : DWORD

Tells the number of bytes in the Data4Name field.

#### Data4Name : BYTEs

Contains an encoded string. This field is Data4NameSize bytes long. Data5 entries (There are Size5 of these):

#### Data5Params1[0] : DWORD

Unknown purpose. Typically contains zero.

#### Data5Params1[1] : DWORD

Unknown purpose. Typically contains zero.

#### Data5Params1[2] : DWORD

Unknown purpose. Typically contains zero.

#### Data5Params2 : DWORD

Unknown purpose. Typically contains zero.

#### Data5Params3 : DWORD

Unknown purpose. Typically contains 1.

#### Data5Params4 : DWORD

Unknown purpose. Typically contains zero.

#### Data5Params5 : DWORD

Unknown purpose. Typically contains zero. Data6 entries (There are Size6 of these):

#### Data6Size1 : WORD

Tells the number of entries in the Data6Data field.

#### Data6Data : Either BYTEs or WORDs

This is a complicated field. It contains run-length-encoded data that tells the client which regions are “nearby”. The purpose appears to be so that the client can determine which mobs in the zone have to have their Z coordinates checked, so that they will fall to the ground (or until they land on something). Since it’s expensive to do this, it makes sense to only do it for regions that are visible to the player instead of doing it for all mobs in the entire zone (repeatedly).

I’ve only encountered data where the stream is a list of BYTEs instead of WORDs. The following discussion describes RLE encoding a BYTE stream.

The idea here is to form a sorted list of all region IDs that are within a certain distance, and then write that list as an RLE-encoded stream to save space. The procedure is as follows:

1. Set an initial region ID value to zero.
2. If this region ID is not present in the (sorted) list, skip forward to the first one that is in the list. Write something to the stream that tells it how many IDs were skipped.
3. Form a block of consecutive IDs that are in the list and write something to the stream that tells the client that there are this many IDs that are in the list.
4. If there are more region IDs in the list, go back to step 2.

When writing to the stream, either one or three bytes are written:

|             |                                                                                    |
| ----------- | ---------------------------------------------------------------------------------- |
| 0x00..0x3E  | skip forward by this many region IDs                                               |
| 0x3F, WORD  | skip forward by the amount given in the following 16-bit WORD                      |
|  0x40..0x7F | skip forward based on bits 3..5, then include the number of IDs based on bits 0..2 |
| 0x80..0xBF  | include the number of IDs based on bits 3..5, then skip forward based on bits 0..2 |
| 0xC0..0xFE  | subtracting 0xC0, this many region IDs are nearby                                  |
| 0xFF, WORD  |                                                                                    |

It should be noted that the values in the range 0x40..0xBF allow skipping and including of no more than seven IDs at a time. Also, they are not necessary to encode a region list: they merely allow better compression.

#### Size7 : DWORD

Tells how many bytes are in the Name7 field.

#### Name7 : BYTEs

An encoded string. It’s purpose and possible values are unknown.

#### Fragment2 : DWORD

It is unknown what this references. It usually doesn’t reference anything.

#### Fragment3 : DWORD

If there are any polygons in this region, then this region points to a 0x36 Mesh fragment that contains only those polygons. That mesh must contain all geometry information contained within the volume that this region represents and nothing that lies outside that volume.

## 0x28 — Light Info — REFERENCE

Reference points to a 0x1C Light Source Reference fragment. 

### Fields

#### Flags : DWORD

Typically contains 256 (0x100).

#### X : FLOAT

X component of the light location.

#### Y : FLOAT

Y component of the light location.

#### Z : FLOAT

Z component of the light location.

#### Radius : FLOAT

Contains the light radius.

## 0x29 — Region Flag — PLAIN

### Notes

This fragment lets you flag certain regions (as defined by 0x22 BSP Region fragments) in a particular way. The flagging is done by setting the name of this fragment to a particular “magic” value. The possible values are:

WT_ZONE ................................................ Flag all regions in the list as underwater regions.\
LA_ZONE ................................................. Flag all regions in the list as lava regions.\
DRP_ZONE .............................................. Flag all regions in the list as PvP regions.\
DRNTP##########_ZONE............. Flag all regions in the list as zone point regions. The ####’s are actually numbers and hyphens that somehow tell the client the zone destination. This method of setting zone points may or may not be obsolete.

### Fields

#### Flags : DWORD

Typically contains zero.

#### Size1 : DWORD

Tells how many region IDs follow.

#### Regions : DWORDs

There are Size1 DWORDs here. Each isn’t a fragment reference per se, but the ID of a 0x22 BSP region fragment. For example, if there are 100 0x22 BSP Region fragments, then the possible values are in the range 0-99. This constitutes a list of regions that are to be flagged in the particular way.

#### Size2 : DWORD

Tells how many bytes follow in the Data2 field.

#### Data2 : BYTEs

An encoded string. An alternate way of using this fragment is to call this fragment Z####_ZONE, where #### is a four- digit number starting with zero. Then Data2 would contain a “magic” string that told the client what was special about the included regions (e.g. WTN__01521000000000000000000000___000000000000). This field is padded with nulls to make it end on a DWORD boundary.

## 0x2A — Ambient Light — REFERENCE

Reference points to a 0x1C Light Source Reference fragment. 

### Fields

#### Flags : DWORD

Typically contains zero.

#### Size1 : DWORD

Tells how many region IDs follow.

#### Regions : DWORDs

There are Size1 DWORDs here. Each isn’t a fragment reference per se, but the ID of a 0x22 BSP region fragment. For example, if there are 100 0x22 BSP Region fragments, then the possible values are in the range 0-99. This constitutes a list of regions that have the ambient lighting given by the 0x1C fragment that this fragment references.

## 0x2C — Alternate Mesh — PLAIN

### Notes

This fragment is rarely seen. It is very similar to the 0x36 Mesh fragment. I believe that this might have been the original type and was later replaced by the 0x36 Mesh fragment. I’ve only seen one example of this fragment so far so the information here is uncertain.

### Fields

#### Flags : DWORD

Typically contains 0x00001803. The meaning of the known bits is believed to be as follows:

Bit 0 ........ If 1, then CenterX, CenterY, and CenterZ are valid. Otherwise they must contain zero. \
Bit 1 ........ If 1, then Params2 is valid. Otherwise it must contain zero.\
Bit 9 ........ If 1, then the Size8 field and Data8 entries exist.\
Bit 11 ...... If 1, then the PolygonTexCount field and PolygonTex entries exist.\
Bit 12 ...... If 1, then the VertexTexCount field and VertexTex entries exist.\
Bit 13 ...... If 1, then the Params3[] fields exist.

#### VertexCount : DWORD

Tells how many vertices there are in the mesh. Normally this is three times the number of polygons, but this is by no means necessary as polygons can share vertices. However, sharing vertices degrades the ability to use vertex normals to make a mesh look more rounded (with shading).

#### TexCoordsCount : DWORD

Tells how many texture coordinate pairs there are in the mesh. This should equal the number of vertices in the mesh. Presumably this could contain zero if none of the polygons have textures mapped to them (but why would anyone do that?)

#### NormalsCount : DWORD

Tells how many vertex normal entries there are in the mesh. This should equal the number of vertices in the mesh. Presumably this could contain zero if vertices should use polygon normals instead, but I haven’t tried it (vertex normals are preferable anyway).

#### Size4 : DWORD

Its purpose is unknown (though if the pattern with the 0x36 fragment holds then it should contain color information).

#### PolygonsCount : DWORD

Tells how many polygons there are in the mesh.

#### Size6 : WORD

This seems to only be used when dealing with animated (mob) models. It tells how many entries are in the Data6 area.

#### VertexPieceCount : WORD

This seems to only be used when dealing with animated (mob) models. It tells how many VertexPiece entries there are. Vertices are grouped together by skeleton piece in this case and VertexPiece entries tell the client how many vertices are in each piece. It’s possible that there could be more pieces in the skeleton than are in the meshes it references. Extra pieces have no polygons or vertices and I suspect they are there to define attachment points for objects (e.g. weapons or shields).

#### Fragment1 : DWORD

References a 0x31 Texture List fragment. It tells the client which textures this mesh uses. For zone meshes, a single 0x31 fragment should be built that contains all the textures used in the entire zone. For placeable objects, there should be a 0x31 fragment that references only those textures used in that particular object.

#### Fragment2 : DWORD

Its purpose is unknown.

#### Fragment3 : DWORD

Its purpose is unknown.

#### CenterX : FLOAT

This seems to define the center of the model along the X axis and is used for positioning (I think).

#### CenterY : FLOAT

This is similar to CenterX but references the Y axis.

#### CenterZ : FLOAT

This is similar to CenterX but references the Z axis.

#### Params2 : DWORD or FLOAT (not sure)

Its purpose is unknown.

Vertex entries (there are VertexCount of these):

#### X : FLOAT

X component of the vertex position.

#### Y : FLOAT

Y component of the vertex position.

#### Z : FLOAT

Z component of the vertex position.

Texture coordinate entries (there are TexCoordsCount of these)

#### TX : FLOAT

Contains a 32-bit floating-point texture value ranging from 0 to 1. This represents the horizontal position along a texture bitmap.

#### TZ : FLOAT

Contains a 32-bit floating-point texture value ranging from 0 to 1. This represents the vertical position along a texture bitmap.

Vertex normal entries (there are NormalsCount of these)

#### NX : FLOAT

Contains a 32-bit floating-point number representing the X component of the vertex normal.

#### NY : FLOAT

Contains a 32-bit floating-point number representing the Y component of the vertex normal.

#### NZ : FLOAT

Contains a 32-bit floating-point number representing the Z component of the vertex normal. Data4 entries (there are Size4 of these)

#### Data4Data : DWORD

Its purpose is unknown.

Polygon entries (there are PolygonsCount of these)

#### PolygonFlag : WORD

Normally contains 0x004B for polygons.

#### PolygonData : 4 WORDs

Usually contain zero. Their purpose is unknown.

#### Vertex1 : WORD

Index of the polygon’s first vertex.

#### Vertex2 : WORD

Index of the polygon’s second vertex.

#### Vertex3 : WORD

Index of the polygon’s third vertex. Data6 entries (there are Size9 of these)

#### Data6Type : DWORD

The purpose of this field is unknown, but it seems to control whether VertexIndex1, VertexIndex2, and Offset exist. It can only contain values in the range 1 to 4. It looks like the Data9 entries are broken up into blocks, where each block is terminated by an entry where Data9Type is 4.

#### VertexIndex : DWORD

This seems to reference one of the vertex entries. This field only exists if Data6Type contains a value in the range 1 to 3.

#### Offset : FLOAT

If Data6Type contains 4, then this field exists instead of VertexIndex (this field only exists if Data6Type contains 4). Its purpose is unknown. Data6 entries seem to be sorted by this value.

#### Data6Param1 : WORD

The purpose of this field is unknown. It seems to only contain values in the range 0 to 2.

#### Data6Param2 : WORD

The purpose of this field is unknown. VertexPiece entries (there are VertexPieceCount of these)

#### VertexCount : WORD

Number of vertices in the skeleton piece.

#### PieceIndex : WORD

This is the index of the piece according to the 0x10 Skeleton Track Set fragment. The very first piece (index 0) is usually not referenced here as it is usually just a “stem” starting point for the skeleton. Only those pieces referenced here in the mesh should actually be rendered. Any other pieces in the skeleton contain no vertices or polygons and have other purposes.

#### Size8 : DWORD

Its purpose is unknown. This field only exists if bit 9 of Flags is 1. Data8 entries (there are Size8 of these)

#### Data8Data : DWORD

Its purpose is unknown. This field only exists if bit 9 of Flags is 1.

#### PolygonTexCount : DWORD

Tells how many PolygonTex entries there are. Polygons are grouped together by texture and PolygonTex entries tell the client how many polygons there are that use a particular texture. This field only exists if bit 11 of Flags is 1.

PolygonTex entries (there are PolygonTexCount of these)

#### PolygonCount : WORD

Number of polygons that use the same texture. All polygon entries are sorted by texture index so that polygons that use the same texture are together. This field only exists if bit 11 of Flags is 1.

#### TextureIndex : WORD

The index of the texture that the polygons use, according to the 0x31Texture List fragment that this fragment references. This field only exists if bit 11 of Flags is 1.

#### VertexTexCount : DWORD

Tells how many VertexTex entries there are. Vertices are grouped together by texture and VertexTex entries tell the client how many vertices there are that use a particular texture. This field only exists if bit 12 of Flags is 1.

VertexTex entries (there are VertexTexCount of these)

#### VertexCount : WORD

Number of vertices that use the same texture. Vertex entries, like polygon entries, are sorted by texture index so that vertices that use the same texture are together. This field only exists if bit 12 of Flags is 1.

#### TextureIndex : WORD

The index of the texture that the vertices use, according to the 0x31Texture List fragment that this fragment references. This field only exists if bit 12 of Flags is 1.

#### Params3[0] : DWORD

Its purpose is unknown. This field only exists if bit 13 of Flags is 1.

#### Params3[1] : DWORD

Its purpose is unknown. This field only exists if bit 13 of Flags is 1.

#### Params3[2] : DWORD

Its purpose is unknown. This field only exists if bit 13 of Flags is 1.

## 0x2D — Mesh Reference — REFERENCE

Reference points to either a 0x36 Mesh or 0x2C Alternate Mesh fragment. 

### Fields

#### Params1 : DWORD

Apparently must be zero.

## 0x2F — Mesh Animated Vertices Reference — REFERENCE

Reference points to a 0x37 Mesh Animated Vertices fragment. 

### Fields

#### Flags : DWORD

Typically contains zero.

## 0x30 — MaterialDef

### Notes

This fragment defines a material that is used with meshes. It contains texture references, and shader settings. If multiple files load a material with the same name, one will overwrite the other.

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### Flags: DWORD

0x01 - If this is set, the material will double-sided (also known as two-sided).\
0x02 - If this is set, the two UVShiftPerMs FLOATs will exist.

#### RenderMethod: DWORD

This value contains a mix of single-bit flags and multi bit fields that control properties of the material shader. Some of them seem to be deprecated, even in the earlier versions of the client, but are decribed for completeness.

Bits   0-1: Drawstyle - Seems to be mostly non-functional, except in some versions of the client, a value of 2 here will cause culling of geometry near the center of the camera frustrum. Some sources, like the WORLDCOM.EXE app, translate a value of 2 in this field to "WIREFRAME" and 3 to "SOLIDFILL".\
Bits   2-4: Lighting - Seems to be mostly non-functional. WORLDCOM.EXE translates a value of 0 in this field to "ZEROINTENSITY", 2 to "CONSTANT", 4 to "AMBIENT", and 5 to "SCALEDAMBIENT".\
Bits   5-6: Shading - Seems to be mostly non-functional. WORLDCOM.EXE translates a value of 2 in this field to "GOURAUD1", and 3 to "GOURAUD2".\
Bit      7: 0x00000080 (Masked Transparency) - If this is set, the material will use alpha transparency in textures that have an alpha channel, or in indexed-color bitmaps that have the first 2 bytes of the pixel data the same color as the 0-index color. In that case, all the pixels of that color in the bitmap will show 100% transparent.\
Bits  8-15: Texture - If any of these bits are set, the material will use the texture that the SimpleSprite reference ultimately points to.\
Bits 16-19: Alpha Blend Opacity - This is the percentage opacity of the material if the Alpha Blend flag is set. The value of these 4 bits are basically 0-15 and the opacity can be calculated as field/16 * 100.\
Bit     20: 0x00100000 (Additive flag) - If the Alpha Blend flag is set, this flag being set will cause the alpha blend to be additive, and it will also disable fog on the material, turn on the masked transparency of the 0x80 flag, and it will increase the opacity approximately 6.25%.\
Bit 	21: 0x00200000 - This flag seems to have originally be used to allow some type of dynamic lighting, but the value seems to be overwritten by the client, and not used from the RenderMethod.\
Bit    	24: 0x01000000 (Alpha Blend flag) - If this is set it will in effect apply an alpha transparency to the whole material with a percent opacity set in the Alpha Blend Opacity field. It will also disable the masked transparency, unless the 0x200000 flag is set.\
Bit 	30: 0x40000000 - This flag seems to have originally be used to force use of pre-baked lighting like vertex colors, but the value seems to be overwritten by the client, and not used from the RenderMethod.\
Bit     31: 0x80000000 (Userdefined flag) - If this is set, the rest of the bits of the RenderMethod are not read as individual fields or flags. The entire DWORD is compared to a table of prebuilt RenderMethods, and those values are used for the material. For instance, a value of 0x80000017 will be translated to values of 0x011B0507, being used in-game, which is Additive Alpha Blend transparency of 68.75% opacity that is textured.

A value of 0x00000000 will be a fully transparent material.

#### RGBPen: DWORD

The field contains RGB (or possibly BGR) color values, but I have not found how or where it is applied.

#### Brightness: FLOAT

Sometimes referred to as ConstantIntensity in client code. It is not clear what this does. Usually 0.

#### ScaledAmbient: FLOAT

I've seen this have values of 0, .75, and 1.0 most commonly. Also not clear what it does.

#### SimpleSpriteRef: DWORD

References a 0x05 SimpleSprite fragment, which in turn references a 0x04 SimpleSpriteDef fragment. This may be empty if the RenderMethod does not use a texture.

#### UVShiftPerMs U: FLOAT

Only exists if the 0x02 flag is set. Always contains 0.0. The two UVShiftPerMs values are used in the functions that control the display of the sky, but those functions also overwrite the values in the material. If you disable this overwrite, you can manually set the scrolling for the sky textures. This is the U value of the UV shift per millisecond. 

#### UVShiftPerMs V: FLOAT

Only exists if the 0x02 flag is set. This is the V value of the UV shift per millisecond.

## 0x31 — MaterialPalette

### Notes

This fragment is basically a list of materials that a mesh can reference. The mesh will reference this type of fragment directly.

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

### Fields

#### Flags: DWORD

Must contain zero. It's not read by the client code.

#### MaterialCount: DWORD

Tells how many 0x30 MaterialDef fragment references this fragment contains.

#### MaterialRefs: DWORDs

There are MaterialCount fragment references. Each refers to a 0x30 MaterialDef fragment by fragment index.

## 0x32 — DmRGBTrackDef

### Notes

This fragment is typically referenced 0x33 DmRGBTrack reference fragment which in turn is referenced by a 0x15 Actor fragment, which used for object placement in a zone. It is used to set pre-baked lighting for objects. It can also be animated, but I have never seen it used like that.

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### Flags: DWORD

0x01 - Apparently this lets the Alpha value of the RGBAFrames be used. Have not really tested it.

#### ColorCount: DWORD

Tells how many color values are in each RGBAFrame. It should be equal to the number of vertices in the placeable object, as contained in its 0x36 DmSpriteDef2 fragment.

#### FrameCount: DWORD

The is the number of RGBAFrames.

#### Sleep: DWORD

This is the number of milliseconds between RGBAFrames, if FrameCount is > 1.

#### Data4 : DWORD

Typically contains 0. Its purpose is unknown.

#### RGBAFrame: DWORDs

This contains an RGBA color value for each vertex in a mesh that references it. It applies vertex colors to the mesh. If FrameCount is > 1, there will be FrameCount RGBAFrame DWORDs, and the colors will animate. Each byte of the DWORD is a 0-255 value and the actual order is BGRA, or Blue, Green, Red,  and Alpha. The alpha value is overall brightness.

## 0x33 — DmRGBTrack

### Notes

This is the reference fragment for a 0x32 DmRGBTrackDef fragment.

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### TrackReference: DWORD

Reference to a 0x32 DmRGBTrackDef fragment. See "Basic fragments - Reference" for details.

#### Flags: DWORD

This may actually be a float that is hard coded to 1.0 by the client.

## 0x34 — ParticleCloudDef

### Notes

This represent a particle cloud that is generated when older spell effects are cast, and the older spell effects are enabled. They can also be attached to models to generate particle effects, like seen on Epic weapons. Typically they are attached to bones (also known as dags) in a model. 

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### Flags: DWORD

0x01 - If this is set, the 6 SpawnBox FLOATs will exist.\
0x02 - If this is set, the 6 Box FLOATs will exist.\
0x04 - If this is set, BlitSpriteRef DWORD will be used.

#### ParticleType: DWORD

Valid values for this seem to be 1-4:

Type 1: Simple one pixel point particle. I have not found a way to affect the appearance of these.\
Type 2: One pixel wide particles with tails. I have not found a way to affect the appearance of these.\
Type 3: Regular camera-facing blit sprite particles. This is the only ParticleType I have seen set on this type of fragment.\
Type 4: Axis aligned blit that sits on the XY plane. These are hard to see unless you are looking up or down on them. Otherwise, they look like the type 3 particles.

#### SpawnType: DWORD

Valid values for this seem to be 0-4:

Type 0: BOX - Particles will spawn randomly in the volume defined by the 6 SpawnBox FLOATs. Particles affected by XYZ Gravity and SpawnVelocity.\
Type 1: SPHERE - Particles will travel from the emitter randomly in any direction. Speed of particles is affected by SpawnVelocityMultiplier, but not the XYZ SpawnVelocity. Affected by XYZ Gravity. Affected by SpawnRadius.\
Type 2: PLANE - Particles will travel from the emitter in a random direction along one plane based on the SpawnVelocity. No movement with SpawnVelocityMultiplier alone. Affected by XYZ Gravity. Affected by SpawnRadius.\
Type 3: STREAM - Particles will travel from the emitter in one direction based on the SpawnVelocity. No movement with SpawnVelocityMultiplier alone. Affected by XYZ Gravity. Affected by SpawnAngle.\
Type 4: NONE - Particles will travel from the emitter in one direction based on the SpawnVelocity. No movement with SpawnVelocityMultiplier alone. Affected by XYZ Gravity.

#### PCloudFlags: DWORD

These are additional flags that control properties of the particles generated for the particle cloud. The names come from some versions of the client code. 

0x00001 - FREE: I don't know what this does.\
0x00002 - COLLISION: I don't know what this does.\
0x00004 - RESPAWN: I don't know what this does.\
0x00008 - VIEWRELX: This will make particles disappear. Not sure what it actually should do.\
0x00010 - VIEWRELY: This will make particles disappear. Not sure what it actually should do.\
0x00020 - VIEWRELZ: Particles will change position on the Z axis depending on the angle of the camera?\
0x00040 - VIEWWARP: This will make particles disappear. Not sure what it actually should do.\
0x00080 - BROWNIAN: This will introduce a brownian motion-like random jiggle to the particles. Doesn't work in the RoF2 client, but does work in the TAKP-EQMac client.\
0x00100 - FADE: Particle will become more transparent from when it spawns until it reaches its lifespan, when it will become completely transparent.\
0x00200 - BOUNDINGBOX: I don't know what this does.\
0x00400 - UPDATE_BBOX: Seems to be required to display any particles.\
0x00800 - POINTGRAVITY: I don't know what this does.\
0x01000 - GRAVITY: I don't know what this does.\
0x02000 - FREEDEF: I don't know what this does.\
0x04000 - OBJECTRELATIVE: Particles move and rotate with the object they are being emitted from. XYZ coordinates are also relative to the object.\
0x08000 - PARENTOBJRELATIVE: Particles move and rotate with the parent object of the object that the particles are being emitted from. XYZ coordinates are also relative to the parent of the object.\
0x10000 - SPAWNSCALERELATIVE: Particles will scale with the model if it is resized in the DB or with an effect that resizes the model. Otherwise they will not scale.\ 
0x20000 - HIDEWITHSPAWNOBJECT: This flag is set on every particle cloud I have seen, removing the flag does not seem to do anything.

#### Size: DWORD

The max number of particles that can be displayed by the emitter at once. 

#### GravityMultiplier: FLOAT

Gravity is a movement property similar to SpawnVelocity. Just setting the GravityMultiplier alone will not make particles move. You must also set one of the XYZ Gravity values as well. If PCloudFlags 0x4000 or 0x8000 are set, then the gravity coordinate system is local to the object or object parent. Having negative values will cause particles to "fall" in the opposite direction.

#### Gravity X: FLOAT

How fast particles will "fall" in the X direction.

#### Gravity Y: FLOAT

How fast particles will "fall" in the Y direction.

#### Gravity Z: FLOAT

How fast particles will "fall" in the Z direction.

#### Duration: DWORD

Time, in milliseconds, that the particle cloud will generate new particles. It will stop after this time. Probably only useful is spell effects.

#### SpawnRadius: FLOAT

This seems to be the distance from the emitter's origin that the particles will spawn. Only affects particles with SPHERE or PLANE SpawnType type.

#### SpawnAngle: FLOAT

This seems to be the maximum angle that particles will travel randomly in a cone from the emitter. A value of 90.0 seems to be the maximum and greater values won't affect the appearance of the particles. Requires that SpawnVelocityMultiplier and at least one SpawnVelocity value is > 0.0. Only affects particles with STREAM SpawnType type.

#### Lifespan: DWORD

The amount of time, in milliseconds, that the particle will exist.

#### SpawnVelocityMultiplier: FLOAT

This value will multiply all 3 of the following SpawnVelocity values. If PCloudFlags 0x4000 or 0x8000 are set, then the coordinate system is local to the object or object parent. Setting to negative will cause particles to reverse direction.

#### SpawnVelocity X: FLOAT

How fast the particle moves in the X axis.

#### SpawnVelocity Y: FLOAT

How fast the particle moves in the Y axis.

#### SpawnVelocity Z: FLOAT

How fast the particle moves in the Z axis.

#### SpawnRate: DWORD

The time, in milliseconds, between the spawns of particles from the emitter. If an emitter has a Size of 5, and a SpawnRate of 100, it will spawn a particle every 100 milliseconds until it reaches 5 particles. Then it will stop emitting particles until one of the particles reaches its Lifespan.

#### SpawnScale: FLOAT

The size of the particles. Only seems to affect the blit type particles (Types 3 and 4).

#### Tint: DWORD

The color of the particle. If the particle is not greyscale, adjusting this will not have much effect. You can't make a blue blitsprite red by adjusting the color. Each byte of the field is a 0-255 color values in BGRA order.

#### SpawnBoxMin X: FLOAT

Exists if the 0x01 Flags value is set. This and the other 5 SpawnBox values create a volume that particles with SpawnType "BOX" will randomly spawn in. This is the X component of the bounding box minimum.

#### SpawnBoxMin Y: FLOAT

Exists if the 0x01 Flags value is set. This and the other 5 SpawnBox values create a volume that particles with SpawnType "BOX" will randomly spawn in. This is the Y component of the bounding box minimum.

#### SpawnBoxMin Z: FLOAT

Exists if the 0x01 Flags value is set. This and the other 5 SpawnBox values create a volume that particles with SpawnType "BOX" will randomly spawn in. This is the Z component of the bounding box minimum.

#### SpawnBoxMax X: FLOAT

Exists if the 0x01 Flags value is set. This and the other 5 SpawnBox values create a volume that particles with SpawnType "BOX" will randomly spawn in. This is the X component of the bounding box maximum.

#### SpawnBoxMax Y: FLOAT

Exists if the 0x01 Flags value is set. This and the other 5 SpawnBox values create a volume that particles with SpawnType "BOX" will randomly spawn in. This is the Y component of the bounding box maximum.

#### SpawnBoxMax Z: FLOAT

Exists if the 0x01 Flags value is set. This and the other 5 SpawnBox values create a volume that particles with SpawnType "BOX" will randomly spawn in. This is the Z component of the bounding box maximum.

#### BoxMin X: FLOAT

Exists if the 0x02 Flags value is set.

#### BoxMin Y: FLOAT

Exists if the 0x02 Flags value is set.

#### BoxMin Z: FLOAT

Exists if the 0x02 Flags value is set.

#### BoxMax X: FLOAT

Exists if the 0x02 Flags value is set.

#### BoxMax Y: FLOAT

Exists if the 0x02 Flags value is set.

#### BoxMax Z: FLOAT

Exists if the 0x02 Flags value is set.

#### BlitSpriteRef: DWORD

References a 0x26 BlitSpriteDef fragment directly. It does not use the 0x27 BlitSprite reference fragment. This will be the image that the particles will show up as, if the 0x04 flag value is set, and the ParticleType is 3 or 4.

## 0x35 — GlobalAmbientLightDef

### Notes

This fragment is a color value for the global lighting in a zone. 

### Fields

#### Color B: BYTE

Blue component of the global ambient light.

#### Color G: BYTE

Green component of the global ambient light.

#### Color R: BYTE

Red component of the global ambient light.

#### Color A: BYTE

Alpha component of the global ambient light.

## 0x36 — DmSpriteDef2

### Notes

This is the more common mesh fragment, the other being 0x2C DmSpriteDef. 

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### Flags: DWORD

Mobs DmSpriteDef2 meshes tend to have 0x00003 for flags. Objects often have 0x14003, and sometimes 0x10003. Terrain usually has 0x18003, and rarely 0x10003 (Chardok).

0x00001 - If this is set, the CenterOffset values will be used, otherwise, the client will set CenterOffset to 0.0x, 0.0y, 0.0z.\
0x00002 - If this is set, the BoundingRadius value will be used, otherwise, the client will set BoundingRadius to 1.0.\
0x02000 - If this is set, "Params2" will be used. I am not sure what Params2 does when this is set or not. I have never seen it set on a DmSpriteDef2 mesh.\
0x04000 - If this is set, the BoundingBox values will be used, otherwise, I believe the client will make the bounding box from the AABB of the DmSpriteDef2 mesh.\
0x08000 - If this is set, the vertex color alpha value will be used. Effectively this turns off vertex colors on a terrain mesh if not set.\
0x10000 - From the way this flag is used, it seems it should toggle collision on objects and terrain meshes. In the TAKP/EQMac client, it only toggles collison on objects, and has no effect on terrain. 

#### MaterialPaletteRef: DWORD

References a 0x31 MaterialPalette fragment. It tells the client which materials this mesh uses. Typically all the DmSpriteDef2 terrain meshes in a zone will reference the same material palette. Object or mob DmSpriteDef2 meshes from different "Actors" can technically reference the same material palette, but often have their own material palettes. Though, I believe different DmSpriteDef2 meshes that are part of the same Actor object must all use the same material palette. 

#### DmTrackRef: DWORD

References a 0x2F DmTrack fragment, which is the reference fragment for a 0x2E DmTrackDef, or a 0x37 DmTrackDef2 fragment. 0x2E may not be targeted by this reference by any DmSpriteDefe meshes. DmSpriteDef2 meshes that have this reference will have vertex animation, like some plants in Kunark jungles, or flags in Qeynos. 

#### DmRGBTrackRef: DWORD

According to the client code, this should be a reference to a 0x33 DmRGBTrack fragment, which is the reference fragment for a 0x32 DmRGBTrackDef fragment. I have never seen it used in a DmSpriteDef2 mesh, and my attempts to add it never saw any effect in-game.

#### PolyhedronRef: DWORD

References a 0x18 Polyhedron fragment, which is the reference fragment for a 0x17 PolyhedronDef fragment. Will usually be 0 on mobs, notable exceptions being the ships that take players between the different continents in the game. On DmSpriteDef2 meshes with flag 0x10000 set, the PolyhedronRef will usually be -2. This actually doesn't seem to be a magic number, and in the TAKP/EQMac client, it seem to just be ignored if the flag is set. 

#### CenterOffset X: FLOAT

For zone meshes this typically contains the X coordinate of the center of the mesh. This allows vertex coordinates in the mesh to be relative to the center instead of having absolute coordinates. This is important for preserving precision when encoding vertex coordinate values. For placeable objects this seems to define where the vertices will lie relative to the object’s local origin. This seems to allow placeable objects to be created that lie at some distance from their position as given in a 0x15 Object Location fragment (why one would do this is a mystery, though).

#### CenterOffset Y: FLOAT

This is similar to CenterX but references the Y axis.

#### CenterOffset Z: FLOAT

This is similar to CenterX but references the Z axis.

#### Params2[0]: FLOAT

It is unknown what Params2 actually does. It may be an offset for lighting on the mesh. Some objects that are light sources, like torches and braziers, have values for this. You can see some slight change in lighting in an object by adjusting Params2 if you remove the DmRGBtrack reference from an object. It doesn't seem to matter if the 0x2000 is set or not, even though some versions of the client code seem to use Params2 only if the flag is set.

This first value may be the X component.

#### Params2[1]: FLOAT

Mostly unknown, but this may be the Y component.

#### Params2[2]: FLOAT

Mostly unknown, but this may be the Z component.

#### BoundingRadius: FLOAT

This is the max distance from the CenterOffset that any vertex in the mesh can sit. If the 0x2 flag is not set, this value will be defaulted to 1.0, apparently.

#### BoundingBoxMin X: FLOAT

The bounding box is a volume that seems to be used for things like hitboxes, and objects that have special interactions, like ladders. If the 0x4000 is not set, the bounding box will apparently be calculated from the AABB of the DmSpriteDef2 mesh itself.

This is the Min X coordinate.

#### BoundingBoxMin Y: FLOAT

This is the Min Y coordinate.

#### BoundingBoxMin Z: FLOAT

This is the Min Z coordinate.

#### BoundingBoxMax X: FLOAT

This is the Max X coordinate.

#### BoundingBoxMax Y: FLOAT

This is the Max Y coordinate.

#### BoundingBoxMax Z: FLOAT

This is the Max Z coordinate.

#### VertexCount: WORD

Tells how many vertices there are in the mesh. It is a uint16, so that gives the maximum number of vertices in a mesh at 65,535.

#### UVCount: WORD

Tells how many UVs there are in the mesh. This usually equals the number of vertices in the mesh. Some zones have border wall meshes that have a simple transparent material and no UVs. If the .WLD Version value is 0x1000C800 (or at least the upper half of the DWORD is 0x1000), then these will be FLOATs, otherwise they will be SIGNED WORDs. UVs wrap/repeat if the value is greater than 1 or -1. For example, a UV coordinate of 1.5 is equivalent to 0.5. Coordinates are a percent of dimensions of the texture, so in a 256x512 texture, 0.5 is somewhere around pixel 128 in the U coordinate and 256 in the V coordinate. 

#### NormalsCount: WORD

Tells how many vertex normal entries there are in the mesh. This should equal the number of vertices in the mesh.

#### ColorCount: WORD

Tells how many vertex color entries are in the mesh. This should equal the number of vertices in the mesh, or zero if the mesh does not use vertex colors. Meshes do not require color entries to work. Zone terrain will usually be the only DmSpriteDef2 meshes that have these.

#### FaceCount: WORD

Tells how many triangle polygons there are in the mesh. All faces in a DmSpriteDef2 mesh must be a triangle. A vertex can be part of multiple faces, but all the faces it is part of must use the same material. 

#### VertexGroupCount: WORD

The is the number of vertex groups in the mesh and are present in bone animated models. The entries are referenced by bones (known as dags in some EverQuest documentation) of a 0x10 HierarchicalSpriteDef that references this DmSpriteDef2 mesh fragment. They are referenced by vertex index order in which they appear (starting at 0). I believe the vertices have to be grouped so that all the vertices that are references by a dag have to be contiguous. So you cannot have 2 vertex groups that reference the same dag. 

#### FaceMaterialGroupCount: WORD

Tells how many face material groups there are. Faces are grouped together by the material that they use. The groups must be contiguous in the face index order. This means that 2 different groups can use the same material. A face can only belong to one group, though.

#### VertexMaterialGroupCount: WORD

Tells how many vertex material groups there are. Vertices are also group by the material that is used by the face, or faces, that the vertex belongs to. Again, like the face material groups, these are contiguous in the vertex index order. A vertex may belong to multiple faces, but all the faces it belongs to must share the same material, so a vertex will only ever belong to one group, though multiple groups might use the same material, if the vertices are non-continguous in the vertex order. Vertex material groups seem to be used chiefly for controlling tinting.

#### MeshOpCount: WORD

Tells how many MeshOps there are. MeshOps are a method for LOD. They are used if "Level of Detail" is toggled to "ON" in some clients. They are basically a list of instructions of how to decimate the orignal mesh, based on distance from the mesh. Only mob models really have MeshOps. I am not sure they work on object or zone terrain DmSpriteDef2 meshes. Depending on the MeshOpType field, the first 4 bytes of an entry will either be 2 successive WORDs (MeshOpType 1 to 3) or a FLOAT (MeshOpType 4). The types will be detailed futher in the MeshOpType description further below.

#### FloatingPointScale: WORD

This allows vertex coordinates to be stored as integral values instead of floating-point values, without losing precision based on mesh size. Vertex coordinate values are divided by 2^FloatingPointScale for in-game values.

**Vertex entries (there are VertexCount of these)**

#### Vertex X: SIGNED WORD (signed 16-bit value)

X component of the vertex position.

#### Vertex Y: SIGNED WORD (signed 16-bit value)

Y component of the vertex position.

#### Vertex Z: SIGNED WORD (signed 16-bit value)

Z component of the vertex position. 

**UV coordinate entries (there are UVCount of these)**

#### UV U: SIGNED WORD (old-format file) or FLOAT (new-format file)

In old-format .WLD files, this contains a signed 16-bit UV value that are divided by 256 to get float values. In new-format .WLD files this is a float32. It is the U component of the texture UV.

#### UV V: SIGNED WORD (old-format file) or FLOAT (new-format file)

In old-format .WLD files, this contains a signed 16-bit UV value that are divided by 256 to get float values. In new-format .WLD files this is a float32. It is the V component of the texture UV.

**Vertex normal entries (there are NormalsCount of these)**

#### Normal X: SIGNED BYTE

Contains a signed byte representing the X component of the vertex normal, scaled such that –127 represents –1 and 127 represents 1.

#### Normal Y: SIGNED BYTE

Contains a signed byte representing the Y component of the vertex normal, scaled such that –127 represents –1 and 127 represents 1.

#### Normal Z: SIGNED BYTE

Contains a signed byte representing the Z component of the vertex normal, scaled such that –127 represents –1 and 127 represents 1.

**Vertex color entries (there are ColorCount of these)**

#### Color: DWORD

This contains an BGRA color value for each vertex in the mesh. It specifies the additional color to be applied to the vertex, as if that vertex has been illuminated by a nearby light source. The A value is a alpha value that is combined with dymanic light sources. There is a maximum value to the light, so a high value will start bright, and not change much with a dynamic light source. Each byte of the DWORD is one unsigned color value (0-255).

**Polygon entries (there are FaceCount of these)**

#### FaceFlags: WORD

The only known and probably possible flag for a DmSpriteDef2 face is 0x10, which lets the face be "passable", meaning, it doesn't have any collision. This flag can be applied to mobs mesh faces in some cases, but usually to object or zone terrain mesh faces.

#### Vertex 1: WORD

Index of the face's first vertex.

#### Vertex 2: WORD

Index of the face's second vertex.

#### Vertex 3: WORD

Index of the face's third vertex.

**VertexGroup entries (there are VertexGroupCount of these)**

#### VertexCount: WORD

Number of vertices in the vertex group. These indicies of the vertices in a group is assumed as them being in order of the vertex index. The first vertex group starts with the first vertex in the vertex index. So if the first VertexCount is 10, then it contains vertices 0-9. If the next VertexCount is 5, then it contains vertices 10-14, and so on...

#### DagIndex: WORD

This is the index of the bone (known as dags in some EverQuest documention) in the 0x10 HierarchicalSpriteDef that references this DmSpriteDef2 mesh fragment.

**FaceMaterialGroup entries (there are FaceMaterialGroupCount of these)**

#### FaceCount: WORD

Number of faces that use the same material. These indicies of the faces in a group is assumed as them being in order of the face index. The first face material group starts with the first face in the face index. So if the first FaceCount is 10, then it contains faces 0-9. If the next FaceCount is 5, then it contains faces 10-14, and so on...

#### MaterialIndex: WORD

The index of the material that the faces use, according to the 0x31 MaterialPalette fragment contained in this DmSpriteDef2's MaterialPaletteRef. Starts with 0.

VertexMaterialGroup entries (there are VertexMaterialGroupCount of these)

#### VertexCount: WORD

Number of vertices that use the same material. These indicies of the vertices in a group is assumed as them being in order of the vertex index. The first vertex group starts with the first vertex in the vertex index. So if the first VertexCount is 10, then it contains vertices 0-9. If the next VertexCount is 5, then it contains vertices 10-14, and so on... The chief difference between the groups for VertexGroups and VertexMaterialGroups, is that more than one VertexMaterialGroup can reference the same material, while 2 VertexGroups cannot reference the same dag.

#### MaterialIndex: WORD

The index of the material that the faces use, according to the 0x31 MaterialPalette fragment contained in this DmSpriteDef2's MaterialPaletteRef. Again, starts with 0.

**MeshOp entries (there are MeshOpCount of these)**

#### Index1: WORD (If MeshOpType is 1 to 3)

If MeshOpType is 1 or 2, this is the index of a face in the mesh. If MeshOpType is 3, this is the index of a vertex in the mesh. If MeshOpType is 4, this is instead the bottom half of the FLOAT Offset value described below. 

#### Index2: WORD (If MeshOpType is 1 to 3)

If MeshOpType is 1, this is the index of a vertex in the mesh. If MeshOpType is 2 or 3, this value is usually zero and not used. If MeshOpType is 4, this is instead the top half of the FLOAT Offset value described below.

#### Offset: FLOAT (If MeshOpType is 4)

If MeshOpType 4, this field is a FLOAT, see MeshOpType for usage.

#### FaceVertexIndex: WORD

If MeshOpType is not one, this should be zero, and is not used anyway. If MeshOpType is 1, then this can be 0 to 2, and represent a vertex of a face. 0 is the first vertex of a face, 1 is the second vertex of a face, and 2 is the 3rd vertex of a face. These are the vertices in the order they are listed in the face entries.

#### MeshOpType: WORD

This can contain values from 1 to 4. The types work as follows:

Type 1 (Corner Swap) - Index1 is a face index for a face in the mesh. FaceVertexIndex is the vertex of one of the vertices within that face. Index2 is the vertex index for a vertex in the mesh, and this operation will swap the corner of the face from the FaceVertexIndex vertex to the Index2 vertex.

Type 2 (Face Delete) - Index1 is a face index for a face in the mesh. All the other values will typically be zero. This operation deletes the face.

Type 3 (Vertex Delete) - Index1 is a vertex index for a vertex in the mesh. All the other values will typically be zero. This operation deletes the vertex.

Type 4 (Threshold Distance) - Offset represents a distance where all the MeshOps before it will be enacted. I am not sure how it translates into in-game units. All of the entries after a type 4 MeshOp until the next type 4 MeshOp will be enacted at the distance of that next type 4 MeshOp.

## 0x37 — DmTrackDef2

### Notes

This fragment contains sets of vertex values to be substituted for the vertex values in a mesh for a vertex animation. For example, if a mesh has 50 vertices then this fragment will have one or more sets of 50 vertices, one set for each animation frame. The vertex values in this fragment will then be used instead of the vertex values in the mesh as the client cycles through the animation frames. This is typically referenced through a 0x2F DMTrack fragment by a 0x36 DmSpriteDef2 fragment.

### Fields

#### NameReference: DWORD

Standard name reference. See "Basic fragments - NameReference" for details.

#### Flags: DWORD

Typically contains zero. Does not seem to be read by the client.

#### VertexCount: WORD

Should be equal to the number of vertices in the mesh that ultimately references it.

#### FrameCount: WORD

The number of animation frames.

#### Sleep: WORD

This is the number of milliseconds between frames of the animation.

#### Param2: WORD

Typically contains zero. Its purpose is unknown.

#### FPScale : WORD

This works in exactly the same way as the Scale field in the 0x36 DmSpriteDef2 fragment. By dividing the vertex positional values by 2^FPScale, you get the actual vertex positions.

**Frame entries (there are FrameCount of these)**\
**Vertex entries (there are VertexCount of these)**

#### Vertex X: SIGNED WORD (signed 16-bit value)

X component of the vertex position, divided by 2^FPScale.

#### Vertex Y: SIGNED WORD (signed 16-bit value)

Y component of the vertex position, divided by 2^FPScale.

#### Vertex Z: SIGNED WORD (signed 16-bit value)

Z component of the vertex position, divided by 2^FPScale.

#### Size6: WORD

Typically contains zero. Its purpose is unknown.
