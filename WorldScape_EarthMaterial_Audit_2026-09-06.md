# Audit du master material de la Terre face au quadtree GPU (2026-09-06)

Demande de Benja : aucune difference de rendu de surface entre le clipmap CPU et le quadtree GPU de WorldScape, puis
un audit du graphe de `M_EarthBase_OPT` (parent de `Mi_EarthMat2_OPT`, materiau de `WorldScapeRoot_1` dans
`L_Persistent_Universe`) et une proposition de refonte (materiau de 2019, 934 noeuds).

Sources : export T3D de `M_EarthBase_OPT`, `Mi_EarthMat2_OPT`, `MF_PlanetaryWeather`, `MF_PlanetaryDust`,
`MF_SLM_ColorMaskPlanet`, `MF_PBR` (script `t3d_matgraph.py`), code de `WorldScapeCore` et `WorldScapeGPUTerrain`,
export T3D d un anneau du clipmap CPU en cours de jeu (script `ws_cpu_mesh_stats.py`).

## 1. Ce que le materiau lit du maillage (et ce que chaque chemin lui donne)

| Entree du graphe | Usage dans M_EarthBase_OPT | Clipmap CPU | Quadtree GPU | Verdict |
|---|---|---|---|---|
| `VertexColor` RGB (R HeightNormalize, G Temperature, B Humidity) | 6 noeuds, reroutes VertexColorRED/GREEN/BLUE : Temperate, Humidity, Sand, SedimentLayer, Ceil(R + HeightShift), ShoreShift, neige, DistanceOpacity | `UWorldScapeLod::UpdateMesh` -> `UpdateMeshSections_LinearColor`, `bSRGBConversion = true` par defaut : **sRGB 8 bits** (`FLinearColor::ToFColorSRGB`) | `WSMeshGenerate.usf` : **lineaire** depuis le 2026-09-02 (fausse piste "CreateMeshSection_LinearColor est lineaire") | **CAUSE PRINCIPALE**, corrigee : port bit-exact de la table sRGB du moteur |
| `VertexColor` A (Hole) | `Normal = lerp(normale planete, VertexNormalWS, A)`, `1 - A` dans la visibilite | 0 hors trou | `Aux.a` = Hole, 0 hors trou | egal |
| `VertexNormalWS` | 7 noeuds : masque de pente `dot(N, normale planete) - 0.96` (VertexInterpolator), CliffNormal, VertexMask | normale geometrique, moyenne ponderee des 6 triangles voisins, sortante (mesure : dot radial median 0.9999, jamais nulle) | differences centrees a +-1 texel (NormalStepTexels), redressee vers le rayon | equivalent a densite egale ; pas la cause des falaises |
| `Absolute World Position` (18) | PlanetWorldPosition, normale planete, lat/long (atan2, asin) pour T_TrueEarth / EarthNormal / T_PackedEarth, toutes les projections `PlanetAlignedTexture` (28), SphereMask camera (12) | LocalToWorld en double + PreViewTranslation | position camera-relative reconstruite (TwoSum), `PreViewTranslation = -ViewOrigin` verifie pour les vues principales et les clipmaps VSM | egal |
| `Transform World -> Local` (7, vecteurs) | coordonnees des textures volume de bruit, GlobalTextureUV | rotation de la planete (composants attaches au TransformKeeper) | meme rotation (keeper mesh) | egal |
| `TextureSample` (38) | 0 en UV0 par defaut, toutes avec coordonnees explicites | UV0 d anneau | UV0 de tuile | sans effet |
| `PlanetPosistion` (DoubleVectorParameter, 8) | centre de la planete | defaut (0,0,-3.18e8) = position reelle de l acteur | idem | egal ; le C++ pose `"PlanetLocation"`, parametre inexistant (sans effet des deux cotes) |
| `RuntimeVirtualTextureSample` / `Output` / `Replace` | branche RVT | aucun volume RVT dans l Univers, aucun composant avec RuntimeVirtualTextures | idem, et le proxy GPU ne rend pas dans les RVT | canal mort des deux cotes |
| `DistanceToNearestSurface` (2), `PixelDepth` | sortie RVT seulement ou branche pendante | | | mort |
| `VertexInterpolator` (10) | pente, masques, UV de SlopeVariation, coordonnees de Noise1 | interpoles sur les triangles de l anneau | interpoles sur les triangles de la tuile | equivalent a densite egale |

Densite de sommets : `Quadtree_ScreenSpaceErrorThresholdPixels = 0` -> seuil automatique
`2.75 x (hauteur d ecran / (2 tan(fov/2))) / 127` (14 px sur un viewport de 711 px), calibre pour reproduire la
densite du clipmap de meme LodResolution. Un anneau CPU a des cellules de `d / 64` (5 a 10 px), les tuiles GPU des
texels de 7 a 14 px : le GPU est legerement plus grossier, jamais plus fin.

## 2. Pourquoi un pas 8 bits fait basculer un biome

Un anneau CPU exporte pendant la session : R entre 185 et 217, p10 187, mediane 189 (0.741). Les seuils du materiau
sur R : `ShoreShift` 0.739 avec `ShoreShaprness` 190 (largeur de transition 0.005 = un pas de 1/255),
`Ceil(R + HeightShift)` avec `HeightShift` -0.742. La moitie du terrain visible est donc a +-2 pas des seuils et un
pas de R vaut environ 115 m d altitude (`NoiseIntensity` 2.2e6 cm). Toute difference d encodage d un seul pas deplace
la ligne de rivage, le sable et les zones de falaise sur des surfaces entieres. D ou le port bit-exact.

Residu connu : la texture aux du heightfield GPU est en fp16 (`PF_FloatRGBA`), pas de 0.00049 dans [0.5, 1) soit 8 %
d un pas sRGB8 ; environ 8 % des sommets situes pres d une frontiere de pas peuvent tomber d un cote different du CPU.
Parite complete : aux en `PF_A32B32G32R32F` (C++, memoire aux x2, budget de heightfields a recalculer) ou R derive de la
hauteur R32F dans le shader de maillage (`HeightNormalize = Height / NoiseIntensity` si la relation est exacte, a
verifier avec les volumes).

## 2b. Retour de Benja apres le port bit-exact : sommet OK, flancs KO (roche et strates), CPU OK partout

Verifie sans dependance a la face ou a la position : shader de maillage (normale par differences centrees, redressee
vers le rayon, meme resultat sur les 6 faces), vertex factory (position camera-relative TwoSum, rotation de la planete
appliquee aux positions ET aux normales, `PreViewTranslation = -ViewOrigin` aussi pour les clipmaps VSM), volumes et
trous du heightfield (rotation racine en quaternion double, `WS_ECEFToWorldD`), materiau (aucun ObjectPosition /
Bounds / LocalPosition / TexCoord, transforms World->Local vectoriels donc rotation seule). Le clipmap CPU n est
pas un cube-sphere : repere tangent au vecteur haut snappe (GridAngle 5 deg) ; la normale et les couleurs restent
independantes du repere.

Ce qui distingue les flancs : bases et plaines cotieres au niveau de la mer (R a +-2 pas des seuils 0.739 / 0.742)
et zone temperee (G, B au milieu des seuils de biome), alors que le sommet est froid et haut. Un ecart de moins d un
pas 8 bits y bascule des zones entieres, et la texture aux etait en fp16 : environ un sommet sur douze pres d une
frontiere tombait du mauvais cote, chaque sommet bascule entrainant ses 6 triangles (masques par sommet et
interpolation). Correctif ecrit, **a compiler** : aux en `PF_A32B32G32R32F` (`WSHeightfieldManager.cpp`, 2 lignes),
+284 Mo de VRAM sur la Terre (budget auto 426 -> 710 Mo, sous le plafond de 2 Go, meme nombre de tuiles).
Residu apres fp32 : evaluation du bruit en float sur GPU (1e-5 relatif, environ 0,3 % d un pas).

Mesure a faire si les flancs restent faux apres compilation (le pont CLIScape etait deconnecte a la fin de la
session) : validateur `bValidateIndirectInstancedNoise` sur une tuile de flanc (log `WS IndirectTerrain Noise
Validation`, diffs HeightNorm / Temp / Hum GPU vs CPU) et `ws.GPUTerrain.Indirect.ForceVertexColorView` 2 / 3 / 4 / 5
(R, G, B, A en couleur) sommet contre flanc.

## 2c. CAUSE MESUREE des flancs : la vertex factory GPU ne remplissait pas le bloc LWC de l etage vertex

Mesures sur la zone de Benja (base sur un flanc, editeur ouvert) : validateur du plugin 77 tuiles PASSED, ecarts
GPU / CPU de 1e-7 sur hauteur, temperature, humidite, couleurs de vertex exactes (`MaxDiffToAux` 0) ; normale de la
sphere forcee dans le shader de maillage : rendu inchange ; Virtual Shadow Maps coupees : rendu inchange. Les
sommets sont donc identiques au CPU et ce n est pas l ombrage.

Contrat moteur (UE 5.7, `MaterialTemplate.ush`) : a l etage VERTEX, `GetWorldPosition`, `GetWorldCameraOrigin`,
`GetPreViewTranslation`, `GetInstanceToWorld` et `GetWorldToInstance` lisent `Parameters.LWCData`, un bloc
`FMaterialLWCData` que chaque vertex factory remplit en fin de `GetMaterialVertexParameters` par
`Result.LWCData = MakeMaterialLWCData(Result);` (LocalVertexFactory.ush:777, Landscape, GpuSkin, GeometryCache,
particules). `WSGPUTerrainVertexFactory.ush` ne le faisait pas : bloc initialise a zero. `M_EarthBase_OPT` evalue ses
masques dans 10 `VertexInterpolator` : le vecteur haut par sommet, `normalize(WorldPos - PlanetPosistion)`, devenait
`normalize(0 - (0, 0, -3.18e8))` = +Z monde, exact seulement pres de l origine du monde (le sommet de la Terre) et
faux ailleurs : `dot(normale, haut) < 0.96` partout, masque de pente sature, falaise et sediments sur tout le flanc.
L etage pixel etait correct (`CalcMaterialParametersEx` remplit le bloc cote moteur), d ou une vue orbitale plausible.

Fix (shader seul) : une ligne dans `GetMaterialVertexParameters` de `WSGPUTerrainVertexFactory.ush`, recompilation
par `recompileshaders changed` (shaders materiau de la VF, plus long qu un shader global). Regle a retenir : toute
vertex factory custom du projet doit etre diffee contre `LocalVertexFactory.ush` a chaque montee de version moteur ;
le compilateur ne signale pas un champ de struct oublie.

## 3. Etat des corrections

- `WSMeshGenerate.usf` : `WSPackColorToFColorSRGB` (table `stb_fp32_to_srgb8_tab4` du moteur, alpha
  `(uint)(a * 255 + 0.5)`). Recompile par `recompileshaders changed`, actif.
- `WSGPUTerrainVertexFactory.h` : commentaires du contrat (RGB sRGB, alpha lineaire).
- `WSGPUTerrainRenderer.cpp` : le validateur compare `MaxDiffToAux` a `FLinearColor(...).ToFColor(true)`. **Non compile**
  (log seulement, aucun effet tant que Benja ne rebuild pas).
- `WorldScape_Plugin_Documentation.md` : paragraphe du quadtree mis a jour.
- `WSHeightfieldManager.cpp` : texture aux en fp32 (`PF_A32B32G32R32F`, 16 octets par texel). Compile par Benja le 06/09.
- `WSGPUTerrainVertexFactory.ush` : `Result.LWCData = MakeMaterialLWCData(Result);` (cause des flancs). Recompile, actif,
  valide visuellement par Benja le 06/09 (sable de desert revenu sur la base du flanc).
  Attention : `FWSTerrainVertexFactory::ShouldCompilePermutation` accepte tous les materiaux du projet, donc toute
  modification de ce fichier suivie de `recompileshaders changed` retraduit tous les materiaux du jeu (56 s ici, 97 %
  de hits DDC, terrain en materiau par defaut pendant la compilation). A restreindre (flag d usage sur les materiaux
  terrain) pour le confort en editeur et le cook.

## 4. Proposition de refonte du materiau (a valider avant tout travail)

Cahier des charges = ce que la version actuelle fait (section 1 et inventaire `t3d/reroutes_backtrace.txt`) :
34 reroutes nommees (Temperate, Humidity, Sand, GrassColor, CliffColor / CliffNormal, SedimentLayer, VertexMask,
SlopeVariation, MacroVariation, VariationMask, FlowMapMask, SkySpaceMask, MaskNear/Far, CameraSphereMask,
DistanceOpacity, AddDust, Wet, UnderWater, GlobalTextureUV, Planet normal, Roughness, Specular, Normal, BaseColor),
28 projections `PlanetAlignedTexture` (herbe, sable, falaise, rivage, roche desert, sediment, pied de colline : chacune
en 2 ou 3 echelles pres / loin / espace), 10 textures volume de bruit, 12 SphereMask de distance camera, 5 switches
statiques, environ 150 parametres dont une dizaine morts (`qsdqsdsqd`, `Roclteter`, `TesterRadius`, RVT, wet par
distance field), Substrate slab en sortie.

Architecture proposee :

1. **Garder le master comme coquille** : memes noms et types de parametres, memes switches statiques, memes textures
   en `TextureObjectParameter`, meme sortie Substrate. `Mi_EarthMat2_OPT` et les autres instances (Mi_EarthMat,
   Mi_EarthMat1 a 4, FAR, DEV, INST_WPO, lunes) restent valides sans re-reglage.
2. **Le calcul dans 3 ou 4 Material Functions qui encapsulent chacune un noeud Custom HLSL** avec un fichier d include
   du plugin (`Plugins/WorldScape/Shaders/.../WS_PlanetSurface.ush`, le repertoire est deja mappe pour les shaders
   globaux) :
   - `WS_PlanetFrame` : centre, rayon, vecteur haut, altitude, latitude et longitude en double (LWC) ;
   - `WS_BiomeMasks` : temperature, humidite, neige, sable, rivage, sediments, pente, macro-variation, avec les
     formules actuelles reproduites exactement (memes seuils, memes contrastes) pour conserver la distribution des
     biomes ;
   - `WS_LayerSampler` : la projection planet-aligned et le choix d echelle pres / loin / espace en un seul
     echantillon par couche, avec branchement dynamique pour ne pas echantillonner une couche dont le masque est nul
     (aujourd hui jusqu a 84 echantillons par pixel) ;
   - `WS_Compose` : melange des couches, roughness, specular, normale (macro `EarthNormal` + details), dust, weather.
3. **Corriger les precisions a la source** : l altitude, le rivage et les sediments a partir de la position monde en
   double (`length(P - centre) - rayon`) et non du canal R 8 bits (115 m par pas) ; garder R pour la couleur seulement.
   Temperature et humidite restent en 8 bits (champs doux).
4. **Nettoyage** : supprimer la branche RVT (aucun volume), `DistanceToNearestSurface`, les parametres morts, les
   doublons (VertexColor x6, WorldPosition x18, CameraPosition x10).
5. **Modernisation, sans toucher la distribution** : melange par hauteur des couches (height lerp) au lieu de lerp
   plats, une seule variation macro partagee, textures ORMH deja presentes (T_Grass_Near_ORMH), roughness et specular
   coherents entre pres / loin, neige avec normale, sable humide pres de l eau, optionnellement un `Texture2DArray`
   par famille pour reduire les samplers.

Methode : (a) construire `M_EarthBase_V3` en asset NEUF a cote de l ancien, (b) instance de test creee par copie des
parametres de `Mi_EarthMat2_OPT`, (c) A/B sur cameras fixes (sol, 1 km, orbite) dans les deux modes de maillage,
(d) statistiques du shader avant / apres (instructions base pass, samplers), (e) bascule de la Terre seulement sur
validation de Benja, anciens assets conserves.

Risques : noeuds Custom + LWC (types `FDFVector3` en entree, a valider sur 5.7), moins lisible pour un artiste, pas de
preview par noeud, permutations de switches statiques inchangees, contrat 8 bits sRGB du sommet a garder tant que le
CPU l ecrit (ou a changer d un bloc CPU + GPU + materiau vers du lineaire 16 bits).

Questions ouvertes pour Benja : garder le contrat 8 bits sRGB ou passer tout le pipeline en lineaire 16 bits ;
reproduire la distribution des biomes a l identique ou accepter des retouches ; qui utilise encore Mi_EarthMat,
Mi_EarthMat1 a 4, FAR, DEV, INST_WPO (a verifier par `get_asset_dependencies` avant la bascule).
