---
layout: post
published: true
title: Potion Craft deal cost composition
tags: colony coder
date: 2022-01-06 13:54:15 +0300
edit: 2022-01-08 01:08:71 +0300
---

1. TOC
{:toc}

## Manifest

This is gonna be a guide for [Potion Craft: Alchemist Simulator] focusing on dissecting math which the game is using to determine the deal cost with customers.

## Legal matter

This blog, its administrator and writer is not affiliated with Potion Craft: Alchemist Simulator, its publisher or developer. Reserved trademarks belong to their holders. Yada-yada.

## TL;DR

Made a [calculator on Desmos]!
![screenshot-desmos]
That's how it's supposed to look with the semi-graphical selector.
Zoom and drag around if you can't see it.
You can inspect the formulae and data used in them under collapsible folders.
Current version (5) doesn't account for conflicts and multiple effects of the same tier.

## Deal example

I'll start by jumping straight into applying what will be discussed in organized detail later.

Suppose you're an alchemist with a fully maxed out trading talent…
![screenshot-talent]
And suppose you have a popularity level of 8…
![screenshot-popularity]
In comes a guard requesting…
> I want to rub my sword with something that will make it more deadly.

To understand which effects he wants you can either consult [requests] appendix or try figuring it out yourself using common sense.

He's going to be fighting with this sword he coats in your potion, which means he needs offensive effects. But the two most valuable offensive effects aren't applicable, because the sword he presumably wields is made of metal and acid would readily dissolve metal, while also, naturally, even though he's not gonna strike at himself, the force of an explosion travels through air and at this range would be dangeroud to the wielder, let alone risk cracking the blade.

If we do consult the [listing of requests] — indeed, it's all the offensive effects except acid and explosion he wants. *It's as if I knew this in advance and applied post-rationalization.*

That leaves us with lightning as the most valuable effect applicable to be dominant in our potion.
Trailing just behind is fire to be the second effect in our potion.
You can check nominal values of effects in [effects] listing.
We brew a potion with a tier III lightning effect and a tier II fire effect.

Now we offer it to the customer and the game reviews our potion the following way…

The lightning effect at tier III is appraised as its nominal: `250`.
The fire effect at tier II is appraised at `0.7` of its nominal value: `175 * 0.7 ≈ 122.5`.
You can check how tiers affect value of effects in [effect tiers] listing.
*There are no conflicts between these, so we don't have to worry about degrading effective tiers.*
*You can read more about effect [compatibility] later.*
We sum these tier-adjusted values of effects and truncate them to an integer: `floor(250 + 122.5) = 372`.

Thus we arrived at the nominal value of the potion but now it's time for commercial adjustments.

Our trading talent at tier X puts the customer's greed at `2`.
Yes, it's a bit confusing, read more in [trading talent tiers] listing.
And our popularity level at 8 gives us a potion cost bonus of `1.3` (+20%).
The [popularity boost] is more straightforward.
Those two modifiers are applied to the potion's value in the following way: `floor(372/2 * 1.3) = 241`

Let's compare with the game…
![screenshot-deal]
All that remains is selling as is or haggling, although I recommend against haggling for cheap potions like that, since you receive less popularity for deals you've haggled for.
I'll not go deep into mechanics of haggling but AFAIK most you can haggle for is `1.5` at fully maxed talent. *Nominal haggling potential is 20% and you gain up to 30% more from the skill.*

## Research method

Reverse-engineered the game.
[dnSpy] 6.1.8 for initial disassembly.
[AssetStudio] 0.16.21 for exporting scripts and other assets.
Still, some values are loaded over others in obscure ways so I had to debug the game in runtime to document and double-check some things. There are many red herrings.
Patched the game into a debug version to attach [dnSpy] as a debugger and read the memory at breakpoints. And even in memory, some variable assignments remained untraceable.
So then I patched some more to fool it into thinking it's running inside Unity Editor in order to enable the debug mode built into the game for quicker documentation.
Correlated all these sources of data into the guide you're currently reading.

Also wrote a shoddy little ruby script to compile the desperate metadata files into the [requests] JSON.

## Mechanics and data

Let's go into details…

### Effects

There are 23 effects in the game.

#### Nominal values

Nominal values of effects roughly correlate with their distance from base solvent (currently, only water is fully implemented).

##### Offensive

* Poison          `125`
* Frost           `150`
* Fire            `175`
* Lightning       `250`
* Explosion       `300`
* Acid            `550`

##### Restorative

* Healing         `100`
* Mana            `200`
* StoneSkin       `300`

##### Buff

* Light           `150`
* SharpVision     `275`
* Bounce          `350`
* Berserker       `425`
* Charm           `525`
* Libido          `525`
* Hallucinations  `600`
* Invisibility    `700`
* Fly             `800`

##### Sedative

* Sleep           `200`
* SlowDown        `250`

##### Fertilizer

* Growth          `225`
* Crop            `275`

##### Special

* Necromancy      `1400`

#### Tiers

It may be a bit counter-intuitive how tiers apply.
The [nominal values of effects][effects] correspond to Tier III, while lower ones diminish the effect value when multiplied by their corresponding factors.

* Tier I    `0.4`
* Tier II   `0.7`
* Tier III  `1`

#### Compatibility

Not all [categories of of effects][effects] are to be mixed.
By mixing incompatible effects, you degrade their effective tiers.

Sometimes both fire and frost are applicable as in the [example] above but mixing them is ill-advised.

Other times, it's a bit more complex: according to [requests] appendix, sometimes a peasant may ask for a potion that would allow them to cross a river.
They'll accept frost for freezing a path, levitation for flying over it and bounce to scale it in a hop.
But frost is an offensive effect and conflicts with pretty much all other categories.
Levitation is worth much more nominal value than bounce.
So you're best off making Levitation III + Bounce II potion.

Terrible mixes (both effects lose a tier):
* Fire with frost.
* Offensive with restorative or fertilizers.
* Sedative with buffs or fertilizers.
* Necromancy with anything.

Bad mixes (one of the effects loses a tier):
* Offensive with buffs or sedative.
* Fertilizers with restorative or buffs.

Surprisingly, for compatibility purposes:
Hallucinogen counts as buff but mixes well rather than terrible with sedative.
Stoneskin counts as restorative but mixes badly rather than terrible with sedative.

{% details Original `EffectsCompatibilityData` file ripped from the game's assets. %}
```
                                                                                                
    Main \ Secondary    Acid    Poison  Fire    Explosion   Frost   Lightning   Healing Mana    Berserker   Charm   Libido  SharpVision Bounce  Fly Invisibility    Hallucinations  Light   SlowDown    Sleep   StoneSkin   Crop    Growth  Necromancy
    Acid    =   1   1   1   1   1   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   0   0
    Poison  1   =   1   1   1   1   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   0   0
    Fire    1   1   =   1   0   1   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   0   0
    Explosion   1   1   1   =   1   1   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   0   0
    Frost   1   1   0   1   =   1   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   0   0
    Lightning   1   1   1   1   1   =   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   0   0
    Healing 0   0   0   0   0   0   =   1   1   1   1   1   1   1   1   1   1   1   1   1   0   0   0
    Mana    0   0   0   0   0   0   1   =   1   1   1   1   1   1   1   1   1   1   1   1   0   0   0
    Berserker   0   0   0   0   0   0   1   1   =   1   1   1   1   1   1   1   1   0   0   1   0   0   0
    Charm   0   0   0   0   0   0   1   1   1   =   1   1   1   1   1   1   1   0   0   1   0   0   0
    Libido  0   0   0   0   0   0   1   1   1   1   =   1   1   1   1   1   1   0   0   1   0   0   0
    SharpVision 0   0   0   0   0   0   1   1   1   1   1   =   1   1   1   1   1   0   0   1   0   0   0
    Bounce  0   0   0   0   0   0   1   1   1   1   1   1   =   1   1   1   1   0   0   1   0   0   0
    Fly 0   0   0   0   0   0   1   1   1   1   1   1   1   =   1   1   1   0   0   1   0   0   0
    Invisibility    0   0   0   0   0   0   1   1   1   1   1   1   1   1   =   1   1   0   0   1   0   0   0
    Hallucinations  0   0   0   0   0   0   1   1   1   1   1   1   1   1   1   =   1   1   1   1   0   0   0
    Light   0   0   0   0   0   0   1   1   1   1   1   1   1   1   1   1   =   0   0   1   0   0   0
    SlowDown    0   0   0   0   0   0   1   1   0   0   0   0   0   0   0   1   0   =   1   1   0   0   0
    Sleep   0   0   0   0   0   0   1   1   0   0   0   0   0   0   0   1   0   1   =   1   0   0   0
    StoneSkin   0   0   0   0   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   =   0   0   0
    Crop    0   0   0   0   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   1   =   1   0
    Growth  0   0   0   0   0   0   1   1   1   1   1   1   1   1   1   1   1   0   0   1   1   =   0
    Necromancy  0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   0   =
```
{% enddetails %}

### Trading talent

Trading talent is applied in a weird way.
Under the hood it decreases the denominator of the potion's cost by `0.2` per talent tier starting at `4` nominally.

* 0     `4`
* I     `3.8`
* II    `3.6`
* III   `3.4`
* IV    `3.2`
* V     `3`
* VI    `2.8`
* VII   `2.6`
* VIII  `2.4`
* IX    `2.2`
* X     `2`

### Popularity boost

The effect of popularity is exposed in GUI of the game quite straightforwardly and applied in the end.
When and by how much it increases is more murky…
You start at a bonus factor of `1` and then at popularity tiers 1, 5, 7, 10, 13 and 15 your bonus additively increases by 10% each, but per every following tier from then on you get a further 1% additive increase instead.

* 0            = `1`     (+0%)
* 1   + `0.1`  = `1.1`   (+10%)
…
* 5   + `0.1`  = `1.2`   (+20%)
…
* 7   + `0.1`  = `1.3`   (+30%)
…
* 10  + `0.1`  = `1.4`   (+40%)
…
* 13  + `0.1`  = `1.5`   (+50%)
…
* 15  + `0.1`  = `1.6`   (+60%)
* 16+ + `0.01` = `1.61`  (+61%)
…
* 999 + `0.01` = `11.43` (+1043%)

I believe, all tiers after 15 are designated as "paragon" tiers and continue up to 999.

### Requests

To fulfill a request, you need to supply at least one of the desired [effects] in a potion you offer.
Combining desired effects improves the deal value.
Desired effects may be [conflicting][compatibility]. Conflicts diminish the deal value.
When no effect is requested — all apply. This is relevant to merchants but out of the scope here.
You do not get any credit for effects the customer wasn't seeking, but they don't hurt either, unless conflicting.

{% details Full JSON of requests. %}
```json
[
  {
    "m_Name": "MonsterHunter_1_HealSwallow",
    "description": "Hello, alchemist. Help me replenish my supply of potions. I had some with me, but the hunt turned out... wild, and I had to drink every last one. I need a potion that accelerates wound healing. Preferably not very toxic.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "MonsterHunter_2_FireSamum",
    "description": "Hello again, alchemist. The residents of a nearby village asked me to deal with the ghouls that have been terrorizing them. Apparently, they have a nest somewhere near the village. To destroy it, I need a bomb, or something combustible at least. Will you help me?",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Explosion"
    ]
  },
  {
    "m_Name": "MonsterHunter_3_SpecterOil",
    "description": "Hello, alchemist. I’m going on a ghost hunt soon. This particular undead is invulnerable to steel, but fortunately not to magic. Give me a potion that imbues my blade with magical power, and I will toss you a coin.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Lightning",
      "Slow"
    ]
  },
  {
    "m_Name": "Headacke",
    "description": "I’ve had a headache for three days. I can’t sleep at night... Can you help me?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal",
      "Sleep"
    ]
  },
  {
    "m_Name": "HealAfterRogues",
    "description": "Some bandits attacked me. I fended them off, but they stuck me. The wound isn’t deep, but it hurts like crazy. Do you have any kind of healing potion?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "PoisonRats",
    "description": "Rats have infested my barn. I need to poison them...",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "RaiseDeadElixir",
    "description": "I need an elixir that can... raise the dead, let’s say. There aren’t many who can make such a thing, and even fewer who would dare. But you’re up to it, right? Can I count on you?",
    "karmaReward": -30,
    "desiredEffects": [
      "Necromancy"
    ]
  },
  {
    "m_Name": "LavaGiant",
    "description": "I have a lava giant to slay. Surely you have something to aid me in battle.",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost",
      "Heal",
      "Berserker"
    ]
  },
  {
    "m_Name": "HideFromBandits",
    "description": "A band of cutthroats is looking for me! I think they’ve already tracked me down. I need to hide quick! Disappear, you know what I mean?",
    "karmaReward": 3,
    "desiredEffects": [
      "Invisibility"
    ]
  },
  {
    "m_Name": "FindInvisible",
    "description": "We’re looking for someone, and we think he’s using some kind of cloaking magic...\nYou must have a potion to help us track him, right?",
    "karmaReward": 3,
    "desiredEffects": [
      "Vision"
    ]
  },
  {
    "m_Name": "NightHerbalism",
    "description": "I want to enter the forest at night to gather some herbs.\nI won’t be able to see anything in the dark, but my hands will be full of herbs, so I won’t be able to carry a lantern.",
    "karmaReward": 3,
    "desiredEffects": [
      "Light",
      "Vision"
    ]
  },
  {
    "m_Name": "DaggerPoison",
    "description": "I want to dip my blade in your strongest poison. My enemies will writhe in agony... yes...",
    "karmaReward": -30,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "SleepWorries",
    "description": "I’ve had insomnia and intense anxiety for a week. I can’t relax for a minute.\nMaybe you can help me?",
    "karmaReward": 3,
    "desiredEffects": [
      "Sleep",
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "StrongWound",
    "description": "Hello! I was badly injured in battle recently. I think I need some medicine, or things could go south.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "GoBerserk",
    "description": "I’m going to battle, and I need something to cope with the fear and pain! I want to be consumed with rage!",
    "karmaReward": 3,
    "desiredEffects": [
      "Berserker"
    ]
  },
  {
    "m_Name": "HuntSlowDown",
    "description": "I’m going hunting, and I need to dip my arrows in something to slow down my prey without spoiling the meat.",
    "karmaReward": 3,
    "desiredEffects": [
      "Slow",
      "Sleep"
    ]
  },
  {
    "m_Name": "Stoneskin",
    "description": "I heard there were potions that can make your skin hard as a rock. Is it true? Do you have any?",
    "karmaReward": 3,
    "desiredEffects": [
      "StoneSkin"
    ]
  },
  {
    "m_Name": "BanditRaidBuff",
    "description": "Our gang is going raiding soon, and my buddy told me a few potions will make you invincible in battle.\nWell, can you brew anything like that?",
    "karmaReward": -20,
    "desiredEffects": [
      "Berserker",
      "StoneSkin"
    ]
  },
  {
    "m_Name": "HunterSight",
    "description": "I’m a hunter, but my vision is getting worse and worse lately. I can’t even see where I’m shooting sometimes.\nCan you help me with my eyes?",
    "karmaReward": 3,
    "desiredEffects": [
      "Vision"
    ]
  },
  {
    "m_Name": "LockedChest",
    "description": "I have a chest, but the key to it is... I lost it. I need to pick the lock somehow.",
    "karmaReward": -20,
    "desiredEffects": [
      "Explosion",
      "Acid"
    ]
  },
  {
    "m_Name": "UnlockDoorSilently",
    "description": "I need to break into a room unnoticed. I have no problem being stealthy, but the door has a really stubborn lock, and I can't blow it, too loud.\nGot any ideas?",
    "karmaReward": -20,
    "desiredEffects": [
      "Acid"
    ]
  },
  {
    "m_Name": "MineLightSource",
    "description": "I work in a mine, and we can’t use lanterns now because we exposed a gas spring that will explode if it touches flame.\nI need to light my way somehow. Any ideas?",
    "karmaReward": 3,
    "desiredEffects": [
      "Light",
      "Vision"
    ]
  },
  {
    "m_Name": "MineExplosion",
    "description": "I work in a mine, and we came across some really hard rock. We need to blast it away. Got anything explosive?",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion"
    ]
  },
  {
    "m_Name": "PassOutGuard",
    "description": "There’s a guard who’s making it really hard for me to get where I need to go. I want to put something in his food so he'll be out of commission for a short while. But I don’t want to kill him.",
    "karmaReward": -20,
    "desiredEffects": [
      "Sleep",
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "ThunderHammer",
    "description": "I want my hammer to have thunderous might! Like a storm laying waste to my enemies! May the gods witness my epic battles!",
    "karmaReward": 3,
    "desiredEffects": [
      "Lightning",
      "Explosion"
    ]
  },
  {
    "m_Name": "SecretOfNecromancy",
    "description": "The dead don’t talk too much... The dead don’t get jealous... The dead don’t betray...\nA select few have managed to learn secrets that ordinary people will never know. We can learn those secrets together, but first I need an elixir to get the process going...",
    "karmaReward": -30,
    "desiredEffects": [
      "Necromancy"
    ]
  },
  {
    "m_Name": "CropsAreEaten",
    "description": "Weevils have eaten nearly our entire harvest. What should we do?",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth",
      "CropHarvest"
    ]
  },
  {
    "m_Name": "NoApplesOnTrees",
    "description": "Our apple trees just won’t bear fruit. Maybe we can water them with something special?",
    "karmaReward": 3,
    "desiredEffects": [
      "CropHarvest"
    ]
  },
  {
    "m_Name": "RegrowForest",
    "description": "Lumberjacks cut down our woods. The new trees will take years to grow... Do you have anything to speed up their growth?",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth"
    ]
  },
  {
    "m_Name": "WifeLowLibido",
    "description": "My wife... We’re doing well, except, um... Let’s just say we’d like to give each other more attention in a certain way.\nI don’t have any problems there, but she’s just never in the mood.\nCan you come up with anything? Some kind of tincture, for instance?",
    "karmaReward": 3,
    "desiredEffects": [
      "Libido"
    ]
  },
  {
    "m_Name": "HelpWomanLover",
    "description": "Hello, potions merchant! I'll get right to business – I'm a great lover of women, but work and stress... Anyway, when things get interesting, I have... problems.\nYou're a master of your trade, right? Got anything I can rub on my sword? Catch my drift?",
    "karmaReward": 3,
    "desiredEffects": [
      "Libido"
    ]
  },
  {
    "m_Name": "CharmLady",
    "description": "There’s this girl... I’m in love with her, but she won’t even look at me! I need to make her love me, understand?",
    "karmaReward": -10,
    "desiredEffects": [
      "Charm"
    ]
  },
  {
    "m_Name": "InvisibilityInCave",
    "description": "I need to get into a cave unnoticed... Do you have a potion for that?",
    "karmaReward": 3,
    "desiredEffects": [
      "Invisibility"
    ]
  },
  {
    "m_Name": "QuarryExplosion",
    "description": "I work in a quarry. Sometimes it’s easier to just blow up a rock than work it for hours with a pick.\nGot anything that goes bang?",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion"
    ]
  },
  {
    "m_Name": "TunnelExplosion",
    "description": "We’re building a tunnel, and we need to blast away some mountain rock. Do you have any kind of bomb?",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion"
    ]
  },
  {
    "m_Name": "FireSword",
    "description": "I want to imbue my sword with fire to smite my enemies with tongues of flame!",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "IgniteWithFirePotion",
    "description": "I need to start a fire and it has to work. That’s why I’m here. A fire potion is the best option in this situation.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "IgniteWetWood",
    "description": "My firewood at home is all wet. I can’t light it. Have anything to help me get it lit?",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "PassDeepRiver",
    "description": "I need to cross a deep river. Can you suggest anything?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost",
      "Bounce",
      "Levitation"
    ]
  },
  {
    "m_Name": "ArtistTrip",
    "description": "Hello. I’m a creative person, you know, and my most creative thinking happens when I’m asleep. But I can’t just fall asleep whenever I want to! What should I do?",
    "karmaReward": 3,
    "desiredEffects": [
      "Sleep",
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "RelaxAfterCrusade",
    "description": "I just came from a harsh campaign. I want to relax...",
    "karmaReward": 3,
    "desiredEffects": [
      "Hallucinogen",
      "Libido"
    ]
  },
  {
    "m_Name": "ThrowingBattlePotion",
    "description": "I need a potion that I can throw at enemies in battle.",
    "karmaReward": 3,
    "desiredEffects": [
      "Acid",
      "Fire",
      "Frost",
      "Poison",
      "Lightning",
      "Explosion"
    ]
  },
  {
    "m_Name": "ThrowingFirePotion",
    "description": "I need a fire potion that I can toss into a lair of monsters.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "ThrowingFrostPotion",
    "description": "I need a potion that can freeze an enemy.",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "ThrowingLightningPotion",
    "description": "I need a potion that can strike an enemy with lightning.",
    "karmaReward": 3,
    "desiredEffects": [
      "Lightning"
    ]
  },
  {
    "m_Name": "HealHandRash",
    "description": "I have some kind of rash on my arm. Do you have any healing ointment?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "BrokenLegAfterJump",
    "description": "I tried to jump over a ravine and broke my leg... They say it will take a long time to heal. Can we speed up the process somehow?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "ToothPain",
    "description": "My tooth has been hurting for some time. Now the pain is unbearable. Is there anything you can do?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "ManaDepleted",
    "description": "My mana is running low... I need to restore my power.",
    "karmaReward": 3,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "BattleMageRestore",
    "description": "I’m a battle mage. Various potions come in handy after battle to restore my mana. I need some extras on hand just in case.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal",
      "Mana"
    ]
  },
  {
    "m_Name": "MagicalPowers",
    "description": "I need a potion that grants magic powers.",
    "karmaReward": 3,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "GrowGardenTree",
    "description": "I planted a tree in the garden, but it’s growing so slowly. Is there any way to speed up the process?",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth"
    ]
  },
  {
    "m_Name": "FightFrostDragon",
    "description": "Soon I must face an ice dragon in battle. I heard potions might help me achieve victory.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Explosion",
      "Heal",
      "StoneSkin",
      "Lightning",
      "Acid",
      "Poison"
    ]
  },
  {
    "m_Name": "SprayPosionToKillBeetles",
    "description": "Pests have infested my home. I want to spray a poisonous cloud there and leave for a month.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "JumpOverPit",
    "description": "I need to leap over an enormous crevice, but I’m not much of a jumper. No one could jump a gap that wide. But I heard potions can give you magic powers. Can you come up with something?",
    "karmaReward": 3,
    "desiredEffects": [
      "Bounce",
      "Levitation"
    ]
  },
  {
    "m_Name": "FireArrows",
    "description": "I need fire arrows to set enemy buildings on fire from a distance. But I only have normal arrows...",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "FrostArrows",
    "description": "I need ice arrows. I heard they were highly effective.",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "SwordEnhancementOil",
    "description": "I want to rub my sword with something that will make it more deadly.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Frost",
      "Poison",
      "Lightning"
    ]
  },
  {
    "m_Name": "ThrowingAcidPotion",
    "description": "I need a vial of acid I can throw.",
    "karmaReward": 3,
    "desiredEffects": [
      "Acid"
    ]
  },
  {
    "m_Name": "SpikeEnhancementOil",
    "description": "I want to rub my spear with a battle potion.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Frost",
      "Poison",
      "Lightning"
    ]
  },
  {
    "m_Name": "WoundFesters",
    "description": "I was injured recently, and it looks like the wound is starting to fester. What should I do?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "NoCropsAfterFire",
    "description": "Our whole harvest was destroyed in a fire. If we don’t restore our fields, we’ll face a famine...",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth",
      "CropHarvest"
    ]
  },
  {
    "m_Name": "GuardBadEyeSight",
    "description": "I’m a guard. A big part of my job is spotting troublemakers, but my eyesight is going. Can you help me somehow?",
    "karmaReward": 3,
    "desiredEffects": [
      "Vision"
    ]
  },
  {
    "m_Name": "SneakNearMonster",
    "description": "I need to get past a monster... I can’t go around it, only over, but I can’t fly!\nAnd as soon as I get close, it’ll see me and gobble me up.",
    "karmaReward": 3,
    "desiredEffects": [
      "Levitation",
      "Invisibility"
    ]
  },
  {
    "m_Name": "BuffForCrusade",
    "description": "We have a raid soon, and I think the slaughter will be fierce. I’m afraid I won’t get out alive. I’d like to increase my chances a bit.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal",
      "Berserker",
      "StoneSkin"
    ]
  },
  {
    "m_Name": "HallucinationMushrooms",
    "description": "I ate a mushroom recently and, wow... It was interesting. There were so many colors and flowers swirling around me! But I can’t remember what the mushroom looked like. You must be an expert in that sort of thing.\nCan you make me a broth from those mushrooms?",
    "karmaReward": 0,
    "desiredEffects": [
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "GoDownTheMountain",
    "description": "I need to get down a mountain swiftly and safely. Surely you can think of something?",
    "karmaReward": 3,
    "desiredEffects": [
      "Levitation"
    ]
  },
  {
    "m_Name": "StickyTrap",
    "description": "I want to make a trap that make trespassers stick to it and won’t let them go.\nI’ll make the trap mechanism, but I need a potion to make it sticky.",
    "karmaReward": 3,
    "desiredEffects": [
      "Slow"
    ]
  },
  {
    "m_Name": "PoisonArrows",
    "description": "Hello. My request is simple – I need poison arrows. I have the arrows, but the poison is up to you.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "ClimbTower",
    "description": "I need to climb a tall tower, but I have no idea how. Is there a magic potion to help me pull it off?",
    "karmaReward": 3,
    "desiredEffects": [
      "Levitation",
      "Bounce"
    ]
  },
  {
    "m_Name": "HealCutFingers",
    "description": "Greetings. I work at the smithy making weapons. I spend all day sharpening swords and daggers. I’ve honed hundreds of blades now, and I know what I’m doing, only I’m a bit clumsy, and I cut myself all the time. I need a potion to heal my injured fingers faster, or I won’t be able to work.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "HealThornsCut",
    "description": "Hello! I was recently gathering berries in the forest, but I was careless and fell right into a thorny thicket... I got all the thorns out of me but was left with deep scrapes that hurt quite a bit... Can you advise anything?",
    "comment": "No metadata found!"
  },
  {
    "m_Name": "PoisonMouse",
    "description": "Hello. Mice have infested my hut, even though I have a cat! The cat just lies on the cabinets and watches the mice do whatever they please. So, I need a vial of rat poison. It’ll be easier to poison those rodents than wait for the stupid cat’s help.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "HealFathersBack",
    "description": "My father and I were heading to market, but his back went out on the way. We’re almost there, but he can’t get out of the cart! Do you have any healing ointment to help the old man straighten his back?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "PoisonCropBugs",
    "description": "Good afternoon. We have trouble in our village. Invasive beetles have taken up in our wheat fields! They’re eating our harvest, and they’re doing it fast! We need to poison them all, or our harvest is completely doomed.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "HealHusbandHand",
    "description": "Hello. My silly husband managed to pinch his hand in the gate yesterday. Now, instead of working, he’s lying on the bench supposedly waiting for his hand to heal. Do you have anything to mend the hand so the bum can get back to work?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "HealNightWatch",
    "description": "Hello! I need a healing potion that is guaranteed to help me if an enemy sword suddenly stabs me in the side. And that's not the worst thing that can happen to you on the night watch... Well, can you suggest anything?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "HealUpsetStomach",
    "description": "I had some strange soup for lunch today, and now I don’t feel so good. Oof... Do you have anything for a troubled gut? Ugh... No more gastronomic adventures...",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "IceKeepFruitsFresh",
    "description": "Hello. I’m a fruit merchant, and I often have to transport goods over great distances. Some fruits tend to spoil on the road due to heat. My friend told me I could pour a frost potion in the bottom of my cart to keep the fruits fresh for days. Do you have a potion like that?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "PoisonBarabashka",
    "description": "A tiny but very troublesome creature has gotten into my home. I haven’t spotted it yet, but it already ate its way through my pickles and pantry. And now every night it eats the porridge I leave for the morning. I want to poison the porridge to finally put an end to the evil sprite.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireForCooking",
    "description": "Hello. I lost my flint somewhere, and now I can’t light a fire. My husband is coming home with friends tonight, and I need to cook them some dinner. Do you have anything to help me light a fire in the hearth?",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "IceColdAle",
    "description": "Good day! I’m going on a long crusade soon, and what better to support a fighting spirit than a flask of ale? The problem is the sun will heat up my flask. I think, if I add a drop of frost potion to it, the ale will stay ice cold even in the summer’s heat. Sound right to you?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "HealBumpOnTheHead",
    "description": "I was just walking down the street minding my own business when a flowerpot fell out of nowhere and bonked me right on the head! Good thing I’ve always had a thick skull, or I’d be out cold. But I’ve got a nice big bump on my head now. What can I rub on it to make it go away?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "PoisonMonsterHunt",
    "description": "I’m not the one to challenge monsters head to head, I’m the one who slays them and doesn’t end up dead! Sometimes it’s easier to shoot a monster with a poison arrow and wait a bit than get up close. On that note, I need a good poison. The daredevils can have the rest.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireExWifeRevenge",
    "description": "My husband recently left me for another woman. I eventually found their love nest. I can’t let him be happy with someone else. I want him to feel the fire of my jealousy. I want their home and their feelings to go down in flames.",
    "karmaReward": -30,
    "desiredEffects": [
      "Fire",
      "Explosion"
    ]
  },
  {
    "m_Name": "HealHousewifeFootBlisters",
    "description": "I have so many cares and worries every day. I wash, cook, shop, and look after the children... I can spend all day on my feet and not sit down to rest once! Because of that, my feet are covered in calluses and corns. Perhaps you have a remedy for them?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "HealArrowWound",
    "description": "Alchemist, give me a healing potion. A bandit shot me through the shoulder with an arrow yesterday. The wound isn’t serious, but I can’t lie around for a whole month. I need to get back to work soon.",
    "comment": "No metadata found!"
  },
  {
    "m_Name": "LightTreasureDiggers",
    "description": "My friends and I found a map with a red cross on it, and telling by the layout of the map, the place is not too far from here. It must be a buried treasure! We can make it to the spot by nightfall, but we’ll need a light source to see where to dig.",
    "karmaReward": 3,
    "desiredEffects": [
      "Light",
      "Fire",
      "Vision"
    ]
  },
  {
    "m_Name": "HealKidsLeg",
    "description": "Please help! My son was playing with friends in the forest and twisted his ankle jumping from a tree. Please help heal my little boy’s leg!",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "PoisonNeighborsCattle",
    "description": "The neighbor’s livestock has gotten into the habit of eating the vegetables out of my garden! I’m tired of chasing sheep and goats away every night. I want to spice my vegetables up with something special, so my neighbor has fewer livestock, and I have fewer problems!",
    "karmaReward": -20,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireScareOffWolves",
    "description": "I often patrol the forest roads at night. It would be no big deal if not for the wolves. I saw their glowing eyes in the darkness last time. I need a fire potion to scare the beasts away if they try to attack.",
    "comment": "No metadata found!"
  },
  {
    "m_Name": "FireCookWildBoar",
    "description": "Well met! Me and the guys are going hog hunting! I’m sure we’ll nab at least a couple. We’ll sell one at the market, but we want to roast the other one right in the forest. We need something to get the fire going, or our plans will fall through.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "IceForTavernDrinks",
    "description": "Good afternoon! The owner of the nearby tavern sent me. He needs a potion that can turn water to ice. Buying ice delivered from the mountains is very expensive, but our patrons love cold beer and ale. Help us out, and we’ll treat you to a fresh draught next time you stop by!",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "LightToAmbushBandits",
    "description": "Greetings. Our unit is making ready to attack a camp of bandits tomorrow. We want to ambush them at night when their leader is there. We need something that can light up the area and blind the bandits, so we can catch them off guard.",
    "karmaReward": 3,
    "desiredEffects": [
      "Light",
      "Explosion"
    ]
  },
  {
    "m_Name": "LightAfraidOfCandles",
    "description": "Hello. A very strange guest came to our hotel. He says he has a great fear of fire and even refuses to use candles. However, he still demands light in his room. Do you have anything that can replace candles?",
    "karmaReward": 3,
    "desiredEffects": [
      "Light"
    ]
  },
  {
    "m_Name": "ExplosionClearPassage",
    "description": "The road from my village to town passes through the cliff tunnel. When I went to town, everything was fine, but on the way home I found the tunnel blocked! Now a barrel of gunpowder would come in handy... Do you happen to have that?",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion"
    ]
  },
  {
    "m_Name": "ManaPotionForBattleMage",
    "description": "Hi there. I was heading to my unit’s rendezvous point when I saw your shop. I thought I’d stop in and buy a potion for our battle mage. He’s been complaining recently that his magical powers have waned. I heard there was a potion that helps mages to recover. Do you have one?",
    "karmaReward": 3,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "LightningKillGhoul",
    "description": "I’m hunting an ancient ghoul. Arrows bounce off it, and it feels neither heat nor cold. People say it’s immortal, but I saw it run into its lair to escape a storm. It’s afraid of thunder and lightning! Give me a potion that harnesses that element, and I swear I can destroy this monster.",
    "karmaReward": 3,
    "desiredEffects": [
      "Lightning"
    ]
  },
  {
    "m_Name": "HeaICow",
    "description": "Hello! The cattle in our village are sick. One neighbor already lost a goat, and another lost a sheep. Now my cow has fallen ill. Got anything to help my beloved bovine feel better? I feel bad for her, and the farm needs milk.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "PoisonCockroaches",
    "description": "Cockroaches have infested my cellar. They’re running all over my food stores and getting into bags of grain. It’s terrible! I can’t live like this! Give me something that will get rid of those buggers once and for all.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireBurnWitchHouse",
    "description": "There’s a house in our village where a witch once lived. She died a long time ago, and no one has lived in the house since. But lately terrible screams can be heard coming from it at night. Now the villagers want to burn the house down, but not with ordinary fire. We heard enchanted fire is more reliable.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Explosion"
    ]
  },
  {
    "m_Name": "IceCrossRiver",
    "description": "My home lies on one side of a river, but the rest of my village is on the other. Every time I need to go to town, I make a long detour to the bridge. I’m sick of it, and I want to try freezing the river, so I can pass quickly over the ice to the other side. Do you think it’ll work?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost",
      "Bounce",
      "Levitation"
    ]
  },
  {
    "m_Name": "LightVillageFestival",
    "description": "Our village is preparing for a local holiday. There will be dancing, reveling, and mountains of food! We also want to hang lanterns all over town that will burn for several days until the celebration ends. Do you have anything that will burn longer than oil or candles?",
    "karmaReward": 3,
    "desiredEffects": [
      "Light",
      "Fire"
    ]
  },
  {
    "m_Name": "ManaVoodooСurse",
    "description": "Hello, good alchemist. I need a potion that restores mana. I put curses on people for a living, and that, as you may have guessed, can be taxing. Be a dear and help me, or I’ll put a curse on you, too!",
    "comment": "No metadata found!"
  },
  {
    "m_Name": "ManaForHealer",
    "description": "Yesterday there was an intense skirmish outside of town between the guardsmen and bandits. There were many wounded, and us healers tended to them all night. We managed to save everyone, but spent a lot of energy, so I was looking for something to restore my magical powers.",
    "karmaReward": 3,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "ExplosionMonsterInWell",
    "description": "I have a well on my property, and scary sounds started coming from it recently! My neighbors and I listened and listened to those sounds, and we believe it's a monster trying to climb out of it. So, I decided to throw a bomb in the well to kill the monster and collapse the well – in case there’s more than one creature down there.",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion",
      "Fire"
    ]
  },
  {
    "m_Name": "LightningKillGolem",
    "description": "I’m an adventurer, and I recently came across some ancient catacombs. I think their depths may contain great treasures. How do I know? Because an iron golem is blocking the way! The problem is no fire, poison, or frost will take it down. I need something else...",
    "karmaReward": 3,
    "desiredEffects": [
      "Lightning",
      "Explosion",
      "Acid"
    ]
  },
  {
    "m_Name": "VisionInvisibleWizard",
    "description": "There are rumors of a sorcerer nearby with a strange scar on his forehead. My unit was ordered to arrest him. But the sorcerer has an invisibility cloak. I need a potion that will expose him, or we’ll never catch him.",
    "karmaReward": 3,
    "desiredEffects": [
      "Vision"
    ]
  },
  {
    "m_Name": "GrowthGrandmasDream",
    "description": "My granny always wanted apple trees and pear trees in her yard. I planted some, but they’ll take a long time to grow, of course, and my granny isn’t getting any younger... Perhaps you have some magic fertilizer to make them grow like in that fairy tale about magic beans?",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth"
    ]
  },
  {
    "m_Name": "SleepDeprivedSoldier",
    "description": "I got really unlucky with my fellow soldiers in the barracks. After lights out, they all start snoring like a herd of swines! I don’t know how long it’s been since I had a decent night’s sleep... Do you have a sleeping potion?",
    "karmaReward": 3,
    "desiredEffects": [
      "Sleep",
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "HealWoodmansLeg",
    "description": "Hello! A log fell on a lumberjack at the sawmill. He’s alive, but his leg is in bad shape. Got anything to help it heal?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "PoisonAnts",
    "description": "A massive anthill appeared near my house and now ants are everywhere: on the floor, on the walls, on the ceiling, even in the food! It’s unbearable! I need something to get rid of those annoying bugs, and fast.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireJealousFarmer",
    "description": "Why does my neighbor always have a bounteous harvest and fat cattle, while I can’t grow anything, not to say all my animals died out by the end of the winter? How is that fair? It isn’t. You must have some fire potions, right? I need one to send my neighbor some fiery justice!",
    "karmaReward": -20,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "HealBruisesAfterRobbery",
    "description": "Three robbers attacked me yesterday. They wanted my money, but couldn’t find any, so they gave me a good beating instead. But I did have money on me! I just hid it in a safe place. Though, now I have to spend the money on a healing potion. My whole body aches...",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "SleepWifeFraud",
    "description": "Hello! I need a sleeping potion for my husband. He needs to sleep deep tonight. I don’t want him to catch my lover and me vanishing into the night with all the money and valuables. I think that would really upset him. Don’t you think?",
    "karmaReward": -20,
    "desiredEffects": [
      "Sleep"
    ]
  },
  {
    "m_Name": "ManaFortuneteller",
    "description": "Hello, alchemist. Do you have a potion that restores magical powers? You might not be able to tell, but I’m a skilled fortuneteller. I get many visitors, but sometimes I need to have some rest. Help me, and fortune is sure to smile on you!",
    "karmaReward": 3,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "PoisonUnfaithfulHusband",
    "description": "My husband is cheating on me. I know everything, but I’m playing dumb. Very soon I will take my revenge. Our wedding anniversary is coming up. We’re having a romantic dinner for two as always. I already bought a good wine. Now I just need a good poison.",
    "karmaReward": -30,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "HealHusbandsFever",
    "description": "My husband has been ill for some time now, but recently he got even worse. Now he can’t even get up. He has a fever and sweats. Do you have any medicine for my poor husband?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal",
      "Frost"
    ]
  },
  {
    "m_Name": "GrowthBalconyGarden",
    "description": "I recently decided to grow a small garden on the balcony. I already bought and arranged all the plants I want, but they’ll take forever to grow, and I want a nice garden right now. Do you have a way to make my flowers grow faster?",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth"
    ]
  },
  {
    "m_Name": "HealKnifeWounds",
    "description": "I was recently in a tussle with bandits, and someone disarmed me, so I had to grab one of their knives with my bare hand. As you can see, I won the fight but was left with some deep wounds that need healing.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "SlowTakingPrisoners",
    "description": "Us soldiers don’t always have to kill robbers and bandits. Sometimes a living criminal brings a greater reward. And a potion that can immobilize them really comes in handy there. And I need exactly this kind of potion.",
    "karmaReward": 3,
    "desiredEffects": [
      "Slow"
    ]
  },
  {
    "m_Name": "PoisonKillBigBandit",
    "description": "I found him — the bandit who killed my buddies. He’s huge and covered in armor from head to toe that no sword can pierce. But I don’t need a sword. I’ll dip an arrow in poison and shoot him right through his visor. His death will be slow and painful...",
    "karmaReward": 0,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireMountainIceElemental",
    "description": "Hello! I was summoned to deal with an ice elemental tormenting a mountain village. An ordinary sword is useless against elementals, but if you can give me a fire potion, I’ll coat my blade with it and take that elemental down.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "IceSaveCropsFromElemental",
    "description": "What a coincidence! Command sent me to fight an elemental too. But this one is a fire elemental. He’s gotten into the habit of strolling through the wheat fields lately and starting fires. Do you have anything that can cool him down?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "HealCaravanAmbush",
    "description": "Be a friend and give me a healing remedy on the stronger side. Some guys and I were escorting a trade caravan yesterday. Then we got attacked by bandits... I was stabbed with a knife and fell out of the cart onto the road. I’m lucky to be alive... Now I need to recover and catch up with the caravan.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "BattlePotion1",
    "description": "I’m patrolling a dangerous forest road tonight. They say dark forces frequently assail travelers on it. I don’t think that’s true, but... If you happen to have a battle potion, I would feel better about it.",
    "karmaReward": 0,
    "desiredEffects": [
      "Fire",
      "Poison",
      "Frost",
      "Lightning"
    ]
  },
  {
    "m_Name": "BattlePotion2",
    "description": "Greetings! I’m dealing with a gang of thugs today. I need a potion that not only makes my sword strike true but also imbues it with magic power. What kind of magic? Well, I trust your judgment.",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Poison",
      "Frost",
      "Lightning"
    ]
  },
  {
    "m_Name": "BattlePotion3",
    "description": "All the guys in my unit have enchanted swords, and I’ve always envied them for it. I was hoping to enchant my sword somehow too. With fire perhaps? Or lightning? Or should I just rub poison on it? But what if I cut myself... What should I choose?",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire",
      "Poison",
      "Frost",
      "Lightning"
    ]
  },
  {
    "m_Name": "RustedDoor",
    "description": "The metal door from my barn rusted, and now I can't get inside. I need a potion that will help me open that damn door.",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion",
      "Acid"
    ]
  },
  {
    "m_Name": "PoisonFoxMenace",
    "description": "The foxes are getting into my chickens! If they keep this up, I’ll be left with a coop and a rooster. I decided to scatter some poison bait around.",
    "karmaReward": 3,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "FireBurnStumps",
    "description": "I finished digging up stumps on my field and was planning to burn them to fertilize the soil with ash. But then, as luck would have it, it started raining today, and all the stumps got soaked! I need to plow the field this week. Help me out!",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "ExplosionAttackOnBanditCamp",
    "description": "Hello, Alchemist! Today the city guards are going to ambush some bandits. I need an exploding potion. I’ll throw it when those scum are sitting around their campfire.",
    "karmaReward": 3,
    "desiredEffects": [
      "Explosion"
    ]
  },
  {
    "m_Name": "FireballWizard",
    "description": "Greetings. I'm preparing to battle a wizard that can throw fireballs. I need to somehow reinforce my shield, so it doesn't melt at the worst possible moment.",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost",
      "StoneSkin"
    ]
  },
  {
    "m_Name": "MermanLake",
    "description": "Children who wander into the forest to play by the lake have started going missing of late. The villagers are certain it's a merman. I've been asked to deal with the beast. I need to reinforce my sword to defeat this scum. I heard mermen flee at the sound of thunder...",
    "karmaReward": 3,
    "desiredEffects": [
      "Lightning"
    ]
  },
  {
    "m_Name": "HealLittlePuppy",
    "description": "Yesterday I found a stray puppy outside. When I brought it home, I saw that its back leg was injured. Do you have a healing cream or medicine for my pet?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "ManaHealerWoman",
    "description": "Hello, Alchemist. I'm a healer, and I need a potion to restore my magical powers. Can you help me?",
    "karmaReward": 3,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "BerserkerNeedStrength",
    "description": "Alchemist, I need a potion that gives me strength and sends me into a battle frenzy, so I can fight like ten men. Do you have something like that?",
    "karmaReward": 3,
    "desiredEffects": [
      "Berserker"
    ]
  },
  {
    "m_Name": "CharmLoveSpell",
    "description": "I have a crush on a young man. He doesn't feel the same, and that drives me crazy. I need a potion that will help me win his heart.",
    "karmaReward": -10,
    "desiredEffects": [
      "Charm"
    ]
  },
  {
    "m_Name": "LibidoLoveGod",
    "description": "Hi. So, I went to the brothel... but let's just say I wasn't up to the mark. The lady even laughed in my face... Now I plan to show up again and show her what a man I truly am. Can you give me a potion that makes me a god of love?",
    "karmaReward": 3,
    "desiredEffects": [
      "Libido"
    ]
  },
  {
    "m_Name": "VisionInvisibleMonster",
    "description": "I was asked to deal with a monster that's been attacking travelers at night. They say it's invisible. I don't know if that's true, but I could use a potion that allows me to see through invisibility.",
    "karmaReward": 3,
    "desiredEffects": [
      "Vision"
    ]
  },
  {
    "m_Name": "CropsAndWitch",
    "description": "A nasty witch has gotten into the habit of flying over the wheat fields and scaring the peasants. That's fine, she can fly, but she casts her spells and ruins the crops! Anyway, I need a potion so I can catch her in the air and scold her.",
    "karmaReward": 3,
    "desiredEffects": [
      "Levitation",
      "Bounce"
    ]
  },
  {
    "m_Name": "InvisibilityDateWithLover",
    "description": "Tomorrow I’m seeing a... friend. But my husband is suspicious and won’t let me out of his sight. He’s so jealous! I need a potion that will allow me to slip out of the house unnoticed.",
    "karmaReward": -10,
    "desiredEffects": [
      "Invisibility"
    ]
  },
  {
    "m_Name": "HallucinationWantedAlive",
    "description": "Our squad has orders to capture a group of bandits and bring them to town where they will be tried for their crimes. Give me a potion that will help me stupefy the bandits. Then we can tie them up without too much resistance.",
    "karmaReward": 3,
    "desiredEffects": [
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "LightPaintHouse",
    "description": "I wanted to paint my house to make it more unique. I thought it would be cool to paint the walls with glow-in-the-dark paint! Do you have a potion that makes paint glow in the dark?",
    "karmaReward": 3,
    "desiredEffects": [
      "Light"
    ]
  },
  {
    "m_Name": "SlowDownCrazyChikens",
    "description": "My chickens have gone crazy! They can’t stand still, constantly running around the yard. When I need to cook chicken, I can’t even catch one. Can I put something in their food to slow them down?",
    "karmaReward": 3,
    "desiredEffects": [
      "Slow"
    ]
  },
  {
    "m_Name": "SleepSpookedChildren",
    "description": "My children can’t sleep at night. They’re constantly worried about monsters behind the window or under the bed. It doesn’t matter how much I try to calm them down. Do you have a sleeping potion for my little ones?",
    "karmaReward": 3,
    "desiredEffects": [
      "Sleep"
    ]
  },
  {
    "m_Name": "CropAppleTrees",
    "description": "My apple trees produced very few apples this year. What will I sell at market now? Do you have a potion that will produce a better apple harvest?",
    "karmaReward": 3,
    "desiredEffects": [
      "CropHarvest"
    ]
  },
  {
    "m_Name": "GrowthBerriesInGarden",
    "description": "I planted berries in my garden. Now I can’t wait for them to grow and bear fruit. Do you have anything that will help them grow faster?",
    "karmaReward": 3,
    "desiredEffects": [
      "Growth",
      "CropHarvest"
    ]
  },
  {
    "m_Name": "Rogue_InvisibilityDarkwing",
    "description": "I am the terror that flaps in the night! I am the deadly blade that always hits its mark. I am the invisible hand of fate. Well, to be honest, I’m not that invisible... Maybe you can help me with that?",
    "karmaReward": -20,
    "desiredEffects": [
      "Invisibility"
    ]
  },
  {
    "m_Name": "Rogue_OpenStolenChest",
    "description": "Recently, I managed to get my hands on a chest. But it has a very intricate lock that I can’t open. And the chest itself is so strong that even a sledge won’t smash it! Can you help me open it?",
    "karmaReward": -20,
    "desiredEffects": [
      "Acid",
      "Explosion"
    ]
  },
  {
    "m_Name": "Rogue_ExplosionIsArt",
    "description": "I love explosions! They are a true art form! Yesterday I was caught in a downpour and now my bombs are all useless. I’ll go crazy if I don’t blow something up today!",
    "karmaReward": -30,
    "desiredEffects": [
      "Explosion"
    ]
  },
  {
    "m_Name": "Rogue_GrowthGardenAccident",
    "description": "Someone had... an accident in the garden. The sight is not for the faint of heart. I need the flowers and grass to grow rapidly and help me hide the mess...",
    "karmaReward": -30,
    "desiredEffects": [
      "Growth"
    ]
  },
  {
    "m_Name": "Rogue_UnsafeWaterpool",
    "description": "The person I need to, let’s say, “take care of” goes to the bathhouse a lot. They find the steam quite good for their health. Do you have anything to add to a bucket of water to make the steam less... healthy?",
    "karmaReward": -30,
    "desiredEffects": [
      "Poison",
      "Acid",
      "Explosion"
    ]
  },
  {
    "m_Name": "Brewer_1_HealBurnedTongue",
    "description": "Good afternoon! Yesterday at a friend's party I burned my tongue on some hot soup. Now I can’t taste a thing, and I’m a brewer! I need to be able to check the quality of my product. My tongue is useless now. Do you have a healing tincture of some kind?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "Brewer_2_FireBeer",
    "description": "Greetings! I had a brilliant idea yesterday: to brew a beer that warms your bones even on the coldest days. People would pay a pretty penny for it in the northern provinces! The recipe is almost ready, but for the beer to always be warm, I need something like liquid fire... Do you have anything like that?",
    "karmaReward": 3,
    "desiredEffects": [
      "Fire"
    ]
  },
  {
    "m_Name": "Brewer_3_FrostBeer",
    "description": "Hello! As you know, good beer should be enjoyed cold. But it's hard to keep it that way in the summer or in southern climes. I want to brew a beer that will always stay cold! Do you have a potion that can help?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "Brewer_4_LibidoBeer",
    "description": "Greetings! Madam Twerks, the owner of the local brothel, came to me and asked me to come up with a special beer for her establishment that helps her clients feel liberated and awaken their “inner beast,” hehe. I have the beer, but I don’t know about “inner beasts.” Do you know anything that could make it... wilder?",
    "karmaReward": 3,
    "desiredEffects": [
      "Libido"
    ]
  },
  {
    "m_Name": "Brewer_5_SleepBeer",
    "description": "Hello! I need your help. The local tavernkeeper asked me to come up with a special beer for especially rowdy guests. They drink too much and accost other customers. The tavernkeeper wants to be able to give them some kind of sedative beverage on the house... Any ideas?",
    "karmaReward": 3,
    "desiredEffects": [
      "Sleep"
    ]
  },
  {
    "m_Name": "EvilDude_1_Poison",
    "description": "I need a strong and fast-acting poison. I’d appreciate it if you saved your questions and kept our little deal a secret. Your silence will be generously rewarded, of course.",
    "karmaReward": -30,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "EvilDude_2_Poison",
    "description": "Greetings. I need more poison. I don’t think I need to remind you of the terms of our deal.",
    "karmaReward": -30,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "EvilDude_3_Acid",
    "description": "Greetings. Today I need acid. Don’t let me down.",
    "karmaReward": -30,
    "desiredEffects": [
      "Acid"
    ]
  },
  {
    "m_Name": "EvilDude_4_Hallucinations",
    "description": "Greetings. I need a potion that confuses the senses and clouds the mind.",
    "karmaReward": -30,
    "desiredEffects": [
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "EvilDude_5_Invisibility",
    "description": "Greetings. I need a potion that allows me to hide from the eyes of unwanted witnesses. It’s a difficult task, but I’m sure you’ll manage it.",
    "karmaReward": -30,
    "desiredEffects": [
      "Invisibility"
    ]
  },
  {
    "m_Name": "Fisherman_1_HealHand",
    "description": "Greetings! I recently went fishing at a lake in the forest. I cast my line out and immediately got a bite! I reeled it in and tried to get it off the hook, but that fish had some teeth! It bit me on the palm and flopped back in the water. Now my hand is all swollen. How can I fish now?",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "Fisherman_2_LightBait",
    "description": "Hello Alchemist! How is business? I heard about a rare fish spotted nearby. It comes out at night and only eats fireflies. Do you follow? If you can give me some kind of glowing potion, I’ll make some bait just for that fish and catch it.",
    "karmaReward": 3,
    "desiredEffects": [
      "Light"
    ]
  },
  {
    "m_Name": "Fisherman_3_FrostDragonFish",
    "description": "Hey there! I’m trying to catch a mean fish... Believe it or not, the little thing breathes fire. It melted my whole rod last time! Could you give me a frost potion for my bait? If the fish swallows it, maybe its heat will back off a bit, right?",
    "karmaReward": 3,
    "desiredEffects": [
      "Frost"
    ]
  },
  {
    "m_Name": "Fisherman_4_SlowDownCrazyFish",
    "description": "Hi! Listen, there's a crazed fish in the area. All the fishermen know it. As soon as it takes the bait, it goes mad and swims like hell. No line can hold it! I want to soak my bait in a potion that will slow the fish down. Please help!",
    "karmaReward": 3,
    "desiredEffects": [
      "Slow",
      "Sleep",
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "Fisherman_5_FlyShyFish",
    "description": "Hey there! I’ll let you in on a secret: there’s a spot on the river nearby with the tastiest fish you’ve ever had in your life! Only problem is they’re awfully skittish. As soon as they smell your boat, they’re gone! Yep, it’s hard to get close enough. Except by air... If only there were a potion that makes people float over the water!",
    "karmaReward": 3,
    "desiredEffects": [
      "Levitation"
    ]
  },
  {
    "m_Name": "Fisherman_6_MightyFish",
    "description": "Hello! I came for your help. I made a bet with guys that I could catch this one fish. I would do it, but it's going to hurt... It can tear the rod out of your hands or pull a fisherman right into the water. I won't manage without one of your potions.",
    "karmaReward": 3,
    "desiredEffects": [
      "Berserker",
      "Sleep",
      "Slow",
      "Hallucinogen"
    ]
  },
  {
    "m_Name": "Guard_1_HealShoulder",
    "description": "Alchemist, give me a healing potion. A bandit shot me through the shoulder with an arrow yesterday. The wound isn’t serious, but I can’t lie around for a whole month. I need to get back to work soon.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "Guard_2_HealHead",
    "description": "Alchemist, I need another healing potion. Yesterday I was protecting some peasants from bandits. I almost had them... But one of them crept up behind me and knocked me out. Luckily, the peasants managed to flee, and none were hurt. But I got a bad concussion. My head won’t stop pounding.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "Guard_3_HealHip",
    "description": "Alchemist, I need a healing potion. I was chasing a thief yesterday. When I tried to grab him, he got me in the hip with a dagger. It's not a mortal wound, but I'm limping a bit now. I need to recover quickly and get back to work protecting the town.",
    "karmaReward": 3,
    "desiredEffects": [
      "Heal"
    ]
  },
  {
    "m_Name": "Guard_4_StoneSkin",
    "description": "Hello, Alchemist. I'm always getting into tight situations in the service – getting shot at by robbers or stabbed by bandits. I have a tough job, but I defend the residents of this town, and that’s what counts. Anyway, I heard about a potion that makes your skin hard as a rock. Do you have anything like that? I could definitely use it.",
    "karmaReward": 3,
    "desiredEffects": [
      "StoneSkin"
    ]
  },
  {
    "m_Name": "Witch_1_PoisonWitchRecipe",
    "description": "Well hello. I heard there was a new alchemist in town, so I came to say hi. I’m the local sorceress. Or witch if you like. That’s what the locals call me. Anyway, I didn’t just come in out of curiosity. Do you happen to carry poison in your shop?",
    "karmaReward": 0,
    "desiredEffects": [
      "Poison"
    ]
  },
  {
    "m_Name": "Witch_2_ManaRecovery",
    "description": "Are you well, Alchemist? No one has put a curse on you? *Laughs* I’m quite tired today, even laughing is a burden... You understand, sorcery is hard work! Do you have a potion that restores mana by chance?",
    "karmaReward": 0,
    "desiredEffects": [
      "Mana"
    ]
  },
  {
    "m_Name": "Witch_3_GrowthMushrooms",
    "description": "Hello, Alchemist! I need special mushrooms for a certain ritual. They usually appear around this time, but they haven't come out yet. I need them right now, so I was hoping to speed up their growth. Do you have a potion for that?",
    "karmaReward": 0,
    "desiredEffects": [
      "Growth"
    ]
  },
  {
    "m_Name": "Witch_4_FlyEnchantBroom",
    "description": "Hello, Alchemist. Very soon, a coven of witches will be held in a distant place, and I want to take part. But first I need to enchant my broom so it can fly... Usually witches enchant brooms themselves, but I don't have time. Be a dear and help me.",
    "karmaReward": 0,
    "desiredEffects": [
      "Levitation"
    ]
  },
  {
    "m_Name": "GenerousCustomer",
    "description": "Now I will buy all the potions!",
    "karmaReward": 0,
    "desiredEffects": [
      "Acid"
    ]
  }
]
```
{% enddetails %}{:.wrap-code}

[calculator on Desmos]: https://www.desmos.com/calculator/wi6kmo3pz0
[Potion Craft: Alchemist Simulator]: https://store.steampowered.com/app/1210320/Potion_Craft_Alchemist_Simulator/
[dnSpy]: https://github.com/dnSpy/dnSpy
[AssetStudio]: https://github.com/Perfare/AssetStudio
[screenshot-talent]: <images/Desktop Screenshot 2022.01.06 - 14.40.01.18.png>
[screenshot-popularity]: <images/Desktop Screenshot 2022.01.06 - 14.40.58.38.png>
[screenshot-deal]: <images/Desktop Screenshot 2022.01.06 - 15.53.34.58.png>
[screenshot-desmos]: <images/Desktop Screenshot 2022.01.08 - 00.45.18.49.png>
[example]: <#deal-example>
[effects]: <#effects>
[effect tiers]: <#tiers>
[compatibility]: <#compatibility>
[trading talent tiers]: <#trading-talent>
[popularity boost]: <#popularity-boost>
[requests]: <#requests>
