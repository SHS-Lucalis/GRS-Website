---
title: Adding Weapons
description: 
published: true
date: 2026-09-26T00:54:55.252Z
tags: 
editor: markdown
dateCreated: 2026-09-26T00:54:55.252Z
---

# Adding weapons — from a bought pack to a Denied Ground gun (2026-09-25)
How a weapon gets into the game, what every weapon needs, and the two ways to make one: the automatic way used for packs we buy (the kit weapon seeder) and the by-hand way in the editor, which produces exactly the same assets. The Infima Modern Sniper (/Game/InfimaGames/ModernSniper) is the worked example; it was set up with the seeder on 2026-09-25 and is the first pack weapon in the project.

1. Who owns what

In the current game the FPS kit owns the weapon: firing, ammo, recoil, first- and third-person animation, attachments, sounds and the weapon smith all run in the kit's Blueprints. Denied Ground never reimplements a weapon; it reads the kit's two tables and drives them:

The kit's	Denied Ground's use
DT_WeaponSelection (one row per weapon)	The loadout screen, the deploy screen, AI loadouts, the trader, loot, the searchable bodies. A row automatically becomes an inventory item KitWeapon:<Row> and a magazine item KitMagazine:<Row>.
DT_Attachments (one row per optic, laser, suppressor, stock, grip, handguard)	The attachment pickers in the loadout screen and AI loadouts; CompatibleWeapons decides which weapons show it.
The weapon Blueprint (a child of BP_WeaponBase)	Spawned by the kit's C_WeaponManager when a body is possessed; DG only chooses which class through the kit's own controller functions.

So "adding a weapon" means adding a kit weapon. Everything on the DG side follows from the row: nothing has to be registered with DG itself, except an AI loadout if the AI should carry it (section 6).

The UDGWeaponDefinition / manifest system in DeniedGroundWeapons (the JSON manifests under Plugins/DeniedGroundWeapons/Manifests) is the older modular renderer from the ReadyToPlay era. On the kit pawn it is switched off (bBuildModularWeapon = false); leave it alone for kit weapons.

2. Anatomy of a kit weapon

Everything a working weapon consists of, with the values the KAR98K uses. A new weapon copies a template weapon — the kit weapon it behaves most like — and changes only what differs.

2.1 The weapon Blueprint

A child of the template weapon (BP_KAR98K, BP_M4A1, BP_AK74Light, BP_S1897, BP_1911), which is itself a child of BP_WeaponBase. Deriving from the template rather than from BP_WeaponBase keeps its fire mode, bolt or magazine reload logic, sounds, recoil and animations.

Component / variable	What it is	KAR98K
FPMesh (skeletal)	The weapon the owner sees, under SceneWeapon, which the kit snaps to the arms socket FPSocketAttach. Its relative location and rotation are the grip fit.	SKM_Kar98, socket Socket_Kar98K on the arms
TPMesh (skeletal)	The weapon everyone else sees, snapped to TPSocketAttach on the body. Holstered at HolsterAttachSocket.	SKM_Kar98
GameName	Name shown in the HUD.	KAR98K
UIImage	HUD icon brush.	T_kar98k
First / Third Person Animation Data	DA_FPAnimations / DA_TPAnimations instances: the arm and body clips for idle, fire, reload, holster, sprint, and the blend spaces. Authored per template weapon on the kit's arms and body skeletons, so a new weapon that plays like the template keeps them.	DA_KAR98FirstPersonAnimations, DA_KAR98_ThirdPersonAnimations
Weapon Animation Data	DA_WeaponAnimations instance: clips played on the weapon mesh itself (bolt, magazine, charging handle). These are bound to the template's weapon skeleton, so a new mesh needs its own instance (empty until clips exist for it).	DA_KAR98WeaponAnimations
Recoil Data, Weapon Effects Data	Recoil curves; muzzle flash, tracer, shell, sounds. Usually kept from the template.	DA_KAR98Recoil, DA_KAR98Effects
DamageType, BulletDamageType	Damage class and projectile class.	DT_KAR98K_Damage, BP_Kar98K_Bullet
Damage, FireRate, MagazineCapacity, CurrentAmmo, AmmoGainedOnPickup, BulletVelocity, WeaponRange, FireModes, AmmoType, ADSFov, AimInSpeed / AimOutSpeed, DistanceFromSight, LengthOfWeapon	Handling numbers.	inherited
Weapon Axis Y Positive?	The mesh points along +Y instead of +X (Infima meshes do).	false
2.2 Sockets on the weapon skeleton

The kit attaches everything by socket name on the weapon's skeleton asset (USkeleton), so a new mesh needs the same names. Positions are per mesh.

Socket	Used for
Muzzle	Muzzle flash, tracer and the shot's trace origin.
ShellEject	Casing ejection.
Base_Muzzle, Base_Handle, Base_Grip, Base_Stock, Base_FrontSight_1, Base_RearSight_1	Factory parts from the row (DefaultMuzzle, DefaultHandGuard, DefaultPistolGrip, DefaultStock, DefaultFrontSight, DefaultRearSight).
Base_<Attachment> — Base_Trijicon, Base_Reflex, Base_Holographic, Base_PK01, Base_Vortex, Base_AimPoint, Base_DotSight, Base_Laser, Base_Flashlight, Base_PointShoot, Base_Supressor, Base_M4Supressor, Base_WeightedBrake, …	One per attachment row: the row's AttachToSocketName. A weapon takes an attachment only if it is in the row's CompatibleWeapons and has that socket.
2.3 The DT_WeaponSelection row (S_WeaponInfo)
Field	Meaning
WeaponName	Shown in the pickers.
WeaponMesh	Skeletal mesh for the menu previews.
WeaponSelect	The weapon Blueprint class.
WeaponType	Primary, Secondary or Melee.
CompatiblePlayersClasses	Kit classes allowed to take it.
DefaultStock / DefaultPistolGrip / DefaultHandGuard / DefaultMuzzle / DefaultFrontSight / DefaultRearSight	Factory parts: attachment Blueprints the kit fits at the Base_* sockets above. None is allowed.
WeaponImage	Picker icon.
Offset / WeaponRotation	How the kit's own loadout character holds it in the customisation map.

The row name is the weapon's id everywhere in DG (KitWeapon:KAR98K, the AI loadout dropdowns, the logs). A row name containing SNIPER, KAR98 or SVD makes AI carrying it Marksmen; 1897, SHOTGUN or SMG makes them Aggressive (UDGCombatBrainComponent).

2.4 A DT_Attachments row (S_AttachmentDetails)
Field	Meaning
AttachmentName / ShortenedAttachmentName / AttachmentThumbnail	Picker text and icon.
AttachmentType	Sight, Alternate Sight (canted), Muzzle Attachment, Handguard, Rail (lasers, lights), Pistol Grip, Stock. One of each per weapon.
AttachmentBP	A child of BP_SightBase / BP_Sight / BP_Ironsight, BP_SupressorBase, BP_BaseLaser, BP_BaseHandGuard, BP_PistolGripBase, BP_StockBase. Sights carry an Aimpoint arrow the camera lines up with when aiming.
AttachToSocketName	The Base_* socket on the weapon skeleton.
BaseIndex	For parts mounted on another part (a canted sight on a mount): index of that part in the starting list; -1 otherwise.
CompatibleWeapons	Weapon classes that list it.
3. The automatic way: the kit weapon seeder

FDGKitWeaponSeeder (DeniedGroundWeaponsEditor/Private/DGKitWeaponSeeder.cpp) holds a spec per bought pack and, when the editor starts (and from the DG Weapons toolset, SeedKitWeapons(true)), creates whatever is missing. It never changes an asset, row or socket that already exists, so everything tuned by hand afterwards — grip fit, aim points, socket positions, numbers — stays.

For one spec it makes:

Sockets on the pack skeleton. Muzzle and Base_Muzzle at the barrel bone, positioned by the default barrel mesh's own Muzzle socket; ShellEject at the ejection bone; Base_Handle, Base_Grip, Base_RearSight(_1), Base_Scope at the handguard, grip, rear-sight and optic bones; Base_FrontSight(_1) where the handguard mesh's Sight_Front socket sits; and one socket per kit attachment the template weapons take — sights at the optic bone, rail items where the handguard's Laser socket sits, muzzle devices at the muzzle. Read the pack's own part sockets, so they land where the pack's demo Blueprints put the parts.
DA_<Id>_WeaponAnimations — an empty weapon animation set (section 5.3).
Iron sights — BP_DG_<Id>_FrontSight / _RearSight, children of the template's sight Blueprints showing the pack's sight meshes.
BP_DG_<Id> — the weapon: a child of the template weapon with FPMesh and TPMesh set to the pack's receiver, a DG Kit Weapon Parts component listing the loose pieces (barrel, handguard, magazine) on their bones, GameName, magazine size and the Y-forward flag.
The DT_WeaponSelection row — a copy of the template's row with name, mesh, class and the two iron sights set; other factory parts cleared.
Compatibility — the weapon added to CompatibleWeapons of every sight, canted sight, rail and muzzle row the template weapons take (stocks, grips and handguards are shaped for one receiver and are left out).
The pack's own attachments — a row and a Blueprint each: the suppressor (a child of the kit's M4 suppressor with the pack mesh, at Base_Muzzle) and the scope (a child of the first sight the template takes, at Base_Scope).

The Output Log lists everything it made under LogDGKitSeed; the same text comes back from the toolset call. To add another pack, add a spec to Specs() — the fields are the bones and meshes of the pack and the kit weapon to copy — and restart the editor.

DG Kit Weapon Parts

UDGKitWeaponPartsComponent (DeniedGroundWeapons/Public/DGKitWeaponParts.h) completes a weapon whose art is a receiver plus loose static meshes, which is how Infima and most Fab packs ship. Put it on any kit weapon Blueprint and list the parts: Name, Mesh, Socket (bone or socket on the weapon mesh, or on the parent part's mesh), Parent (the Name of an earlier part it sits on), Offset, and whether it shows in first and third person. When the weapon spawns it attaches a copy of each part to FPMesh and to TPMesh and keeps them following each mesh's visibility rules (the owner alone sees the first-person set, everyone else the third-person set, the kit's view toggles are honoured, and a bot's removed first-person mesh takes its parts with it). Cosmetic only, nothing on a dedicated server; the loadout preview draws the same list, so the menu shows the whole gun.

Attachments are not parts: an optic, laser or suppressor still goes through DT_Attachments, because the kit's aiming, laser and sound logic lives in those Blueprints.

4. The by-hand way (the same result in the editor)

For a pack the seeder does not know, or to see what it does:

Import the pack into the project (Fab → Add to project, or copy the pack's Content folder in). Note the receiver skeletal mesh, its skeleton, and which meshes are loose parts.
Sockets. Open the skeleton asset (Skeleton Tree → right-click a bone → Add Socket) and add the names in 2.2, on the bones they belong to. Use the pack's own part sockets as the positions: the barrel mesh's Muzzle socket is where Muzzle goes.
Weapon animation data. Right-click → Miscellaneous → Data Asset → DA_WeaponAnimations; leave every clip empty for now.
Iron sights. Right-click the template's front and rear sight Blueprints (BP_FrontSight_Kar98K, BP_RearSight_Kar98K) → Create Child Blueprint Class; set the StaticMesh component's mesh to the pack's sight meshes; move the rear sight's Aimpoint arrow to the notch.
The weapon. Right-click the template weapon (BP_KAR98K) → Create Child Blueprint Class → BP_DG_<Name>. In the child: FPMesh and TPMesh → the pack receiver; add a DG Kit Weapon Parts component and list barrel, handguard, magazine on their bones; set GameName, MagazineCapacity, CurrentAmmo, AmmoGainedOnPickup, Weapon Animation Data (step 3), and tick Weapon Axis Y Positive? for a Y-forward mesh. Leave FPMesh's transform for step 7.
The row. Open DT_WeaponSelection, duplicate the template's row, rename it, and set WeaponName, WeaponMesh, WeaponSelect and the two sight classes; clear the other Default* parts.
Fit the grip in play (section 5.1).
Attachments (optional): add the weapon class to CompatibleWeapons of the rows it should take, and add a Base_* socket per row on the skeleton; for the pack's own optics and suppressors, child the nearest kit attachment Blueprint, swap the mesh, add a row.
AI: section 6.
5. The Modern Sniper (MSR) — what exists and what to tune

Made by the seeder on the first editor start after the 2026-09-25 build, under Content/DeniedGround/Weapons/Kit/ModernSniper: BP_DG_ModernSniper (child of BP_KAR98K, receiver SK_Sniper_Frame, parts SM_Sniper_Barrel_Default, SM_Sniper_Handguard_Default, SM_Sniper_Magazine on the Barrel, Handguard and Magazine bones), BP_DG_ModernSniper_FrontSight, BP_DG_ModernSniper_RearSight, BP_DG_ModernSniper_Suppressor, BP_DG_ModernSniper_Scope, DA_ModernSniper_WeaponAnimations; row ModernSniper ("MSR") in DT_WeaponSelection; rows ModernSniper_Suppressor and ModernSniper_Scope in DT_Attachments; sockets on SKEL_Sniper (frame bones: Barrel, Bolt, Bullet_Chambered, Eject_Casing, Grip, Handguard, Magazine, Magazine_Release, Scope, Sight_Rear, Trigger). Ten rounds, bolt action, KAR98K arms, sounds and recoil. Backups of the two tables and the skeleton before the seeder touched them are in _backups/modern_sniper/Content.

It is playable at once; four things are tuned by eye, in this order:

5.1 The grip (every new mesh needs this)

A pack mesh's origin is wherever its author put it, so the first spawn is usually rotated or off the hand. In play, holding the weapon, open the console:

Command	Does
DG.KitWeapon.Print	Shows the FPMesh and TPMesh relative transforms.
DG.KitWeapon.Preset (or Preset N)	Steps the first-person mesh through the 24 axis-aligned orientations; find the one that points forward and upright.
DG.KitWeapon.Nudge X Y Z [Pitch Yaw Roll]	Adds to the first-person mesh's relative transform, in cm and degrees.
DG.KitWeapon.NudgeTP …	The same for the third-person mesh.
DG.KitWeapon.Reset	Back to what the Blueprint has.
DG.KitWeapon.Save	Writes both transforms into the weapon Blueprint (editor build). Applies to the next spawn and to every player.

Fit the receiver to the right hand in the fire animation first, then check the left hand on the handguard and the third-person body.

5.2 Sockets and aim points

Open SKEL_Sniper and look at the sockets the seeder made: Muzzle should sit at the barrel tip (flash and tracers start there), the Base_* optics at the top rail, Base_Laser on the handguard's side rail, Base_Muzzle at the barrel tip. Drag any that are off. Then aim down the sights: if the sight picture is off, open BP_DG_ModernSniper_RearSight (and _Scope) and move the Aimpoint arrow to the notch or the eyepiece centre — the kit lines the camera up with that arrow.

5.3 Weapon animations

DA_ModernSniper_WeaponAnimations is empty, so the bolt and magazine do not move yet; the arms still cycle the bolt and the weapon fires, reloads and aims. To animate the weapon itself, make clips on SKEL_Sniper (Sequencer or Control Rig, or import from the pack author) for Weapon Fire Animation (bolt cycle), Weapon Reload Animation / Reload Empty (magazine out and in), Weapon Bolt Open, Equip, Inspect, and set them in the data asset. The KAR98 clips cannot be reused: they drive its own bones (w_bolt, w_clip_bullet_*).

5.4 Icon

UIImage and the row's WeaponImage still show the KAR98 icon (the pack ships none). Render one (a 256×128 PNG of the assembled rifle on a transparent background), import it, and set both.

6. Giving it to the AI

AI carry what their loadout lists. Open a DA_AILoadout_* in Content/DeniedGround/AI/Loadouts and add a Primaries entry with Weapon = ModernSniper (and, if wanted, Attachments = ModernSniper_Scope) and a weight. The row name contains "Sniper", so the AI that draw it become Marksmen: long range, crouched, short accurate bursts. Squad spawners and Scenario Commander units use the same loadouts.

7. Checklist for any new weapon
Loadout screen: it is listed, the preview shows the whole gun (parts included), the attachment pickers offer the rows it takes.
Deploy: it spawns in hand at the fitted grip; a second client sees the third-person mesh with its parts; nothing floats at the arms socket for other players.
Fire: flash and tracer at the muzzle; hits register; the HUD name and ammo are right.
Reload and aim: the animations play; the sight picture lines up; the scope and suppressor fit and the suppressor quiets the shot.
Drop and loot: it appears on searchable bodies and in the trader as KitWeapon:<Row>; picking it up rearms.
AI: a loadout carrying it spawns armed with it and shoots.
Saved/Logs/DeniedGround.log: LogDGKitSeed lists what was made, and no LogDGKitParts warnings.
8. Troubleshooting
Symptom	Cause and fix
Weapon spawns rotated or off the hand	Grip not fitted: 5.1.
Receiver shows, no barrel / magazine	The parts component's meshes did not load or the bones are misnamed; check the Parts list against the skeleton's bone names. On a dedicated server nothing is drawn by design.
Parts visible to other players at the owner's camera	The FP parts are not mirroring FPMesh; DG.KitWeapon.Print on the other client and check FPMesh has OnlyOwnerSee (kit default).
Muzzle flash at the receiver	Muzzle socket at the wrong place or missing on the skeleton.
An attachment does not show for the weapon	Not in the row's CompatibleWeapons, or the skeleton lacks the row's AttachToSocketName.
Attachment floats away from the gun	Socket present but misplaced: move it in the skeleton editor.
Sight picture off when aiming	Aimpoint arrow on the sight Blueprint: 5.2.
Seeder made nothing	LogDGKitSeed says what was missing: the pack mesh, the template class, or the kit tables. Existing assets are skipped on purpose; delete what you want remade.
Weapon Axis Y Positive? wrong	Shots go sideways: toggle it on the weapon Blueprint.
9. Files
DeniedGroundWeapons/Public|Private/DGKitWeaponParts.* — the parts component and the DG.KitWeapon.Nudge / NudgeTP / Preset / Print / Reset commands.
DeniedGroundWeapons/DGKitWeapons.* — CurrentWeaponOf (the held kit weapon).
DeniedGroundWeapons/DGWeaponPreviewActor.*, DeniedGroundFrontend/DGFrontEndWidget.cpp — the loadout preview draws the parts.
DeniedGroundWeaponsEditor/Private/DGKitWeaponSeeder.* — the seeder and its pack specs; DGKitWeaponFitSaver.* — DG.KitWeapon.Save; DGWeaponsToolset — SeedKitWeapons; the module dumps the kit's weapon Blueprints and both tables to Saved/DGDumps at start.