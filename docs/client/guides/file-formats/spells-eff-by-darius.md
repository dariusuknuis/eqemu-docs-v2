---
description: Version 1.0, by Darius
---

# Spells.eff File Reference

## Overview

Spells.eff is a binary file that contains the "classic" spell effects data for EverQuest. These effects can be enabled, at least up to the TAKP/EQMac client, by having spell effects on, and removing the spellsnew.eff and spellsnew.edd files from your EverQuest client files folder. The file is always 714,752 bytes, and contains 256 spell effects records that are 0xAE8 (2,792) bytes in size. These are indexed by their position in the file, and the SpellAffectIndex field in the spells_en or spellsnew table in a database refers to this zero-based index. Only about 50 of the records actually contain data.

For each record, there is an 0x8 byte "header" of some-type, and then three 0x3A0 (928) byte effects that represent the effect that happens around the caster, then the effect that travels between the caster and target, and then the effect that happens around the target. 

Each of those effects have up to 3 subeffects and up to 12 aftereffects. These are not split up into blocks, and each parameter will have multiple fields, each field referring to a subeffect or aftereffect, depending on the position of the field. Subeffects will be referred to as such:

SubEffect0 - This effect will appear from level 1, and after.\
SubEffect1 - This effect will appear from level 24, and after.\
SubEffect2 - This effect will appear from level 39, and after.

## Spell record entries (there are 256 of these):

#### Header: 0x8 BYTEs

This contains either 0 or 1, its purpose is currently unknown.

**Spell effect entries (there are 3 of these)**

***BlitSprite entries (there are 3 of these, one for each SubEffect)***

#### BlitSprite: 0x20 BYTEs

This is an uncoded string that always refers to 0x26 BlitSpriteDef WLD fragments with the name format "GEN%c%d0_SPRITE"; where %c is a single alphabetic character, and %d is an integer. So, "GENA00_SPRITE" is a valid name, but "GENZ_SPRITE" crashes the game, even if the BlitSprite exists. Most of the BlitSprites are contained in the "equip.s3d" files, like gequip.s3d, gequip5.s3d, etc...

There are 3 of these consecutively. One for each SubEffect.

#### Role: 0x20 BYTEs

This is an uncoded string that describes the type of effect entry. It can only contain "Source" or "Target". The effect that travels between the caster and target does not have a Role string.

***AttachmentType entries (there are 3 of these, one for each SubEffect)***

#### AttachmentType: signed DWORD

This controls what bone (known as DAGs in the WLD-based EverQuest code), in the skeleton of the mob, that the effect will be emitted from. Only certain EffectTypes use this value, as some are locked to certain DAGs or locations. The values work as follows:

0 - Emitter is the %sCH_DAG or %sCHEST_POINT_DAG, where %s is the model name. The effect will emit from the chest of the mob.\
1 - Emitter is the %sHEAD_POINT_DAG, where %s is the model name. The effect will emit from the head of the mob.\
2 - Emitter is the %sR_POINT_DAG, where %s is the model name. The effect will emit from the right hand of the mob.\
3 - Emitter is the %sL_POINT_DAG, where %s is the model name. The effect will emit from the left hand of the mob.\
4 - Emitter is the %sBO_R_DAG or %sBOFOOTR_DAG, where %s is the model name. The effect will emit from the right foot of the mob.\
5 - Emitter is the %sBO_L_DAG or %sBOFOOTL_DAG, where %s is the model name. The effect will emit from the left foot of the mob.

Any other value defaults to 0 (%sCH_DAG or %sCHEST_POINT_DAG). There are 3 of these consecutively. One for each SubEffect. 

***EffectType entries (there are 3 of these, one for each SubEffect)***

#### EffectType: signed DWORD

This controls a set of preset values for the SubEffect. These are all properties of a 0x34 ParticleCloudDef WLD fragment. If the SubEffect has a value > zero for the property, then it uses that value in the spell effect, otherwise it uses the values in the EffectType presets. Some of the properties are not able to be set in a SubEffect, so these values are always used. For a full description of the properties, please see the "WLD by Darius" doc. Here are the types:

Type 0\
	ParticleType: 3\
    PCloudFlags: 0x001, 0x100, 0x200, 0x400\
    Size: 500\
    Gravity: 0.0\
    SpawnNormal: 0.0x, 0.0y, -1.0z\
    BoxMin:\
    BoxMax:\
    SpawnType: 3\
	Duration: 1600\
    SpawnVelocity: 0.0x, 0.0y, -1.0z\
	SpawnRadius: 0.0\
	SpawnAngle: 20.0\
	Lifespan: 1000
	SpawnVelocityMulitplier 3.0
	SpawnRate: 20
	SpawnScale: 0.05
	Color: 
    BlitSpriteRef: "I_SNOWFLAKESPRITE"

Type 1\
	ParticleType: 3\
    PCloudFlags: 0x001, 0x100, 0x200, 0x400\
    Size: 2000\
    Gravity: 0.0\
    SpawnNormal: -1.0x, 0.0y, 0.0z\
    BoxMin:\
    BoxMax:\
    SpawnType: 3\
	Duration: 5000\
    SpawnVelocity: -1.0x, 0.0y, 0.0z\
	SpawnRadius: 0.0\
	SpawnAngle: 20.0\
	Lifespan: 2000\
	SpawnVelocityMulitplier 20.0\
	SpawnRate: 1\
	SpawnScale: 0.2\
	Color: \
    BlitSpriteRef: "I_SNOWFLAKESPRITE"

Type 2\
	ParticleType: 3\
    PCloudFlags: 0x001, 0x100, 0x200, 0x400\
    Size: 1000\
    Gravity: 0.0\
    SpawnNormal: 0.0x, 0.0y, -1.0z\
    BoxMin:\
    BoxMax:\
    SpawnType: 1\
	Duration: 1600\
    SpawnVelocity: 0.0x, 0.0y, 0.0z\
	SpawnRadius: 5.0\
	SpawnAngle: 0.0\
	Lifespan: 1000\
	SpawnVelocityMulitplier 3.0\
	SpawnRate: 2\
	SpawnScale: 0.05\
	Color:\
    BlitSpriteRef: "I_SNOWFLAKESPRITE"

Type 3\
	ParticleType: 3\
    PCloudFlags: 0x001, 0x100, 0x200, 0x400\
    Size: 1000\
    Gravity: 0.0\
    SpawnNormal: 0.0x, 0.0y, -1.0z\
    BoxMin:\
    BoxMax:\
    SpawnType: 4\
	Duration: 5000\
    SpawnVelocity: 0.0x, 0.0y, -1.0z\
	SpawnRadius: 20.0\
	SpawnAngle: 0.0\
	Lifespan: 1000\
	SpawnVelocityMulitplier 7.0\
	SpawnRate: 2\
	SpawnScale: 0.2\
	Color: \
    BlitSpriteRef: "I_SNOWFLAKESPRITE"

Type 4\
	ParticleType: 3\
    PCloudFlags: 0x001, 0x100, 0x200, 0x400\
    Size: 1000\
    Gravity: 5.0\
    SpawnNormal: 0.0x, 0.0y, 1.0z\
    BoxMin:\
    BoxMax:\
    SpawnType: 4\
	Duration: 5000\
    SpawnVelocity: 0.0x, 0.0y, 1.0z\
	SpawnRadius: 4.0\
	SpawnAngle: 0.0\
	Lifespan: 1000\
	SpawnVelocityMulitplier 7.0\
	SpawnRate: 2\
	SpawnScale: 0.2\
	Color: \
    BlitSpriteRef: "I_SNOWFLAKESPRITE"

Type 5\
	ParticleType: 3\
    PCloudFlags: 0x001, 0x100, 0x200, 0x400\
    Size: 1000\
    Gravity: 5.0\
    SpawnNormal: 0.0x, 0.0y, 1.0z\
    BoxMin:\
    BoxMax:\
    SpawnType: 2\
	Duration: 5000\
    SpawnVelocity: 0.0x, 0.0y, 1.0z\
	SpawnRadius: 5.0\
	SpawnAngle: 0.0\
	Lifespan: 1000\
	SpawnVelocityMulitplier 7.0\
	SpawnRate: 2\
	SpawnScale: 0.2\
	Color: \
    BlitSpriteRef: "I_SNOWFLAKESPRITE"
    
There are 3 of these consecutively. One for each SubEffect.

***AfterEffectSprite entries (there are 12 of these, one for each AfterEffect)***

#### AfterEffectSprite: 0x20 BYTEs

placeholder. There are 12 of these consecutively. One for each AfterEffect.

#### EffectMode: signed DWORD

Placeholder.

#### SoundReference: signed DWORD

Placeholder.

***ColorBGRA entries (there are 3 sets of these, one for each SubEffect)***

#### ColorBGRA [B]: BYTE

Placeholder.

#### ColorBGRA [G]: BYTE

Placeholder.

#### ColorBGRA [R]: BYTE

Placeholder.

#### ColorBGRA [A]: BYTE

Placeholder.

***Gravity entries (there are 3 of these, one for each SubEffect)***

#### Gravity: FLOAT

Placeholder.

***SpawnNormal entries (there are 3 sets of these, one for each SubEffect)***

#### SpawnNormal [X]: FLOAT

Placeholder.

#### SpawnNormal [Y]: FLOAT

Placeholder.

#### SpawnNormal [Z]: FLOAT

Placeholder.

***SpawnRadius entries (there are 3 of these, one for each SubEffect)***

#### SpawnRadius: FLOAT

Placeholder.

***SpawnAngle entries (there are 3 of these, one for each SubEffect)***

#### SpawnAngle: FLOAT

Placeholder.

***Lifespan entries (there are 3 of these, one for each SubEffect)***

#### Lifespan: unsigned DWORD

Placeholder.

***SpawnVelocity entries (there are 3 of these, one for each SubEffect)***

#### SpawnVelocity: FLOAT

Placeholder.

***SpawnRate entries (there are 3 of these, one for each SubEffect)***

#### SpawnRate: unsigned DWORD

Placeholder.

***SpawnScale entries (there are 3 of these, one for each SubEffect)***

#### SpawnScale: FLOAT

Placeholder.

***ColorBGR entries (there are 12 sets of these, one for each AfterEffect)***

#### ColorBGR [B]: BYTE

Placeholder.

#### ColorBGR [G]: BYTE

Placeholder.

#### ColorBGR [R]: BYTE

Placeholder.

***SpriteID entries (there are 12 of these, one for each AfterEffect)***

#### SpriteID: signed DWORD

Placeholder.

***AngleRangeA entries (there are 12 of these, one for each AfterEffect)***

#### AngleRangeA: signed WORD

Placeholder.

***AngleRangeB entries (there are 12 of these, one for each AfterEffect)***

#### AngleRangeB: signed WORD

Placeholder.

***AfterEffectRadius entries (there are 12 of these, one for each AfterEffect)***

#### AfterEffectRadius: FLOAT

Placeholder.

***AfterEffectType entries (there are 12 of these, one for each AfterEffect)***

#### AfterEffectType: signed WORD

Placeholder.

***AfterEffectScale entries (there are 12 of these, one for each AfterEffect)***

#### AfterEffectScale: FLOAT

Placeholder.