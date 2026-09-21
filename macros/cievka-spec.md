# Parametrická cievka na kábel — špecifikácia pre agenta

Zadanie pre AI agenta, ktorý bude ďalej vyvíjať alebo opravovať model `cievka.scad`.
Dokument popisuje, **čo sa navrhuje, prečo sú rozhodnutia také aké sú, a čo sa nesmie pokaziť.**

---

## 1. Čo to je

Navíjacia cievka na kábel pre FDM 3D tlač. Kábel sa navíja do **vonkajšej obvodovej drážky**.
Na hornej aj spodnej čelnej ploche je **zapustený otočný kotúč**, ktorý sa dá nezávisle otáčať
a nesie **kapsu na konektor** plus kanál, ktorým sa ku kapse vedie koniec kábla.

Cievka nie je univerzálna. Pre každý typ kábla sa **generuje a tlačí nová** — model je
parametrický, nie modulárny. Vymeniteľné vložky, redukcie ani sada kotúčov nie sú súčasťou
zadania a nemajú sa zavádzať.

### Funkčný princíp

1. Kábel sa navinie do vonkajšej drážky.
2. Koniec kábla sa vyvedie ktorýmkoľvek zárezom v leme (je ich 8–16 po obvode), takže
   nezáleží na tom, kde presne navíjanie skončí.
3. Otočný kotúč sa natočí tak, aby ústie jeho kanála smerovalo k tomu zárezu.
4. Zvyšok kábla sa vedie kanálom a konektor zapadne do kapsy.

**Kľúčová myšlienka:** otočný kotúč odstraňuje požiadavku, aby konektor skončil na presnom
mieste. To je celý dôvod, prečo je kotúč otočný. Akákoľvek zmena návrhu, ktorá túto voľnosť
zruší, je regresia.

---

## 2. Zoznam dielov

| Diel | Počet | Poznámka |
|---|---|---|
| `half_a` | 1 | polovica tela s centrálnym čapom |
| `half_b` | 1 | polovica tela s dutinou pre čap |
| `disc` | 2 | otočný kotúč, oba rovnaké |

Polovice nie sú identické (líšia sa čapom/dutinou). Pokus urobiť ich identickými
(hermafroditný bajonet) bol zvážený a zamietnutý ako zbytočne komplikovaný.

---

## 3. Geometria a súradnice

Model je v `cievka.scad` (OpenSCAD).

- Os otáčania = **Z**.
- Pre polovicu platí `z = 0` v **deliacej rovine (pás)** a `z = Hh` na čelnej ploche lemu.
- Polovica B sa v zostave otáča `rotate([180,0,0])`.

### Profil tela

Vonkajší obrys je parabolický: `prof_r(z) = Rw + gd * (z / (gw/2))^2`

- pri `z = 0` je polomer `Rw` (pás) a dotyčnica je **zvislá**
- pri `z = gw/2` je polomer `Rf = Rw + gd` (lem)
- od `gw/2` po `Hh` je polomer konštantný `Rf` (valcový lem)

### Odvodené veličiny

```
gd      = cable_d * groove_depth_n      hĺbka drážky (radiálne)
gw      = cable_d * groove_width_n      šírka drážky (axiálne)
n_turns = floor(gd/cable_d) * floor(gw/cable_d) * pack_eff
hub_d   = cable_len / (n_turns * PI) - gd        (režim "length")
rec_d   = disc_t + clr_ax + ledge_h     hĺbka vybrania pre kotúč
ft      = rec_d + 2.5                   hrúbka lemu
Hh      = gw/2 + ft                     výška polovice
Rrec    = Rw - wall                     polomer vybrania
Rd      = Rrec - clr_rad                polomer kotúča
```

Minimálny priemer bubna je `max(8 * cable_d, 30)` — limituje ho polomer ohybu kábla, nie tlač.

---

## 4. Tlačová orientácia — neporušiteľné pravidlo

**Každý diel sa tlačí celou čelnou plochou na podložku. Žiadne podpery.**

- Polovica tela: **lemom nadol**, čap smeruje hore.
- Kotúč: **plochou stranou nadol**, kanál a kapsa hore.

Z toho vyplývajú obmedzenia, ktoré musí agent dodržať pri každej zmene:

1. Smerom hore v tlači (= smerom k menšiemu `z` v súradniciach dielu) sa **obrys tela nesmie
   nikde rozširovať**. Parabolický profil to spĺňa; nahradenie iným profilom to môže pokaziť.
2. Vybranie pre kotúč ústi **na podložke** — je to dutina otvorená nadol, tlačí sa ako
   obyčajný otvor. Nie je to previs.
3. Každé miesto, kde sa dutina smerom hore v tlači rozširuje, musí mať **45° skosenie**.
   Týka sa to zádržného lemu vybrania a drážky pre zacvakávací prstenec.
4. Zárezy v leme ústia na podložke — tiež bez previsu.

Ak agent pridá akýkoľvek prvok, musí si overiť oba tieto smery. Chyba tu znamená podpery
vnútri rotačnej škáry, čo je neopraviteľné.

---

## 5. Mechanizmy

### 5.1 Uchytenie kotúča — bajonet

Tri výstupky (`lug_n`) na hornej hrane kotúča, tri vstupné zárezy v stene vybrania,
medzi nimi obvodový kanál. Kotúč sa vloží a pootočí o ~30°.

**Prečo bajonet a nie zacvaknutie:** žiadny prvok sa neohýba, takže niet čo unaviť ani
zlomiť. Axiálna sila ide cez plný materiál. Tolerancia je nekritická — kanál len musí byť
vyšší než výstupok.

Kritický rozmer: `lug_h + clr_rad <= lug_t + 2*clr_ax`, inak sa 45° skosenie zádržného lemu
nezmestí do výšky kanála. Pri zmene `lug_*` treba overiť.

Kotúč sa dá teoreticky vytiahnuť, ak sa natočí presne do zárezov. V praxi mu v tom bráni
prevlečený kábel. Ak by to prekážalo, riešením je plytký detent (výstupok v kanáli), nie
zmena princípu.

### 5.2 Spoj polovíc

Centrálny čap so zacvakávacím prstencom (`barb`), štyri pozdĺžne zárezy, aby mal čap kam
pružiť. Plus tri protirotačné kolíčky (`pin_count`), ktoré zároveň zaručia, že **zárezy
v leme oboch polovíc na seba sadnú** — bez nich by sa polovice pootočili a zárez by nebol
priechodný cez celú výšku lemu.

Spoj je zámerne jednosmerný. Cievka sa nemá rozoberať.

### 5.3 Kanál v kotúči

Generuje ho funkcia `cpt(t)` — oblúk od okraja kotúča (uhol −90°) ku kapse konektora
(uhol 0°, polomer `conn_l/2`), so súčasným zmenšovaním polomeru. Trasa sa skladá
reťazou `hull()` medzi valcami.

**Nesmie sa nahradiť lomenou čiarou.** Bežný kábel znesie polomer ohybu asi `4.5 × cable_d`;
ostrý zlom mu poškodí vodiče. Ak sa kanál skracuje alebo mení, treba skontrolovať, že
najmenší polomer trasy toto splní.

### 5.4 Zárezy v leme

Dva rezy na zárez:
- prierez lemom až do drážky (kadiaľ prejde kábel),
- plytká radiálna drážka po čelnej ploche (spojí zárez s vybraním kotúča).

Počet `notch_count` je kompromis: viac zárezov = menšie dotáčanie kotúča, ale slabší lem
práve v deliacej rovine, kde je už aj tak najtenší. Rozumné rozpätie 8–16.

### 5.5 Opierací prstenec

Kotúč stojí na úzkom prstenci šírky ~2 mm blízko obvodu, nie celou plochou. Znižuje trenie
a bráni tomu, aby sa kotúč v strede nadvihoval. Je na **kotúči**, nie na dne vybrania — na
dne vybrania by to bol previs.

Dôsledok: kotúč sa dotýka podložky len týmto prstencom. Pri tlači použiť brim.

---

## 6. Vôle

| Parameter | Hodnota | Kde |
|---|---|---|
| `clr_ax` | 0.3 | axiálne — kotúč vs. vybranie, kanál bajonetu |
| `clr_rad` | 0.4 | radiálne — kotúč vs. stena, čap vs. dutina |

Sú nastavené pre 0.4 mm trysku a 0.2 mm vrstvu. Pri 0.6 mm tryske treba zväčšiť.

**V slicri vypnúť** horizontal expansion / elephant foot compensation a ironing — inak sa
vôle zaslepia.

---

## 7. Parametre, ktoré mení používateľ

Vždy len sekcie 1–4 v hlavičke súboru. Sekcie 5–7 (spoj, bajonet, vôle) sú konštanty
doladené raz na danú tlačiareň.

Dva režimy veľkosti:
- `size_mode = "length"` — zadá sa `cable_len`, priemer bubna sa dopočíta
- `size_mode = "diameter"` — zadá sa `hub_d_input` priamo

Model po prepočte vypíše cez `echo()` priemer bubna, vonkajší priemer, výšku, počet závitov,
skutočnú kapacitu a priemer kotúča. Tento výpis sa musí zachovať — je to jediná spätná väzba,
ktorú používateľ pred tlačou má.

---

## 8. Známe nedokončené veci

1. **Kapsa na konektor** je len kváder otvorený nahor (`conn_l`, `conn_w`, `conn_h`). Pre
   konkrétny konektor ju treba dokresliť, vrátane podkosenia, ktoré konektor zamkne.
   Toto je jediná časť, ktorá sa nedá parametrizovať naslepo.
2. **Odľahčenie ťahu** — za vstupom kábla do kanála chýba kľukatina alebo výstupok, o ktorý
   by sa kábel zaprel. Bez toho ťah pôsobí priamo na konektor.
3. **Detent kotúča** — kotúč sa zatiaľ točí voľne, takže sa môže samovoľne pootočiť.
   Pružný jazýček v tele + plytké zuby po obvode kotúča by to vyriešili.
4. **Kľúčová dierka v zárezoch** — zárezy sú teraz priechodné, kábel v nich nie je zaistený.
   Zúženie hrdla na ~0.8× priemeru kábla by ho udržalo, ale treba overiť, či hrany
   neprerežú izoláciu.

---

## 9. Ako overovať zmeny

Pri každej úprave geometrie:

1. Vyrenderovať `part = "assembly"` a vizuálne skontrolovať, či sa diely prekrývajú.
2. Skontrolovať konzolu — varovania o malom kotúči a o vybraní zasahujúcom do pásu.
3. Overiť obe podmienky z kapitoly 4 (žiadne rozširovanie smerom hore v tlači, 45° skosenia).
4. Pred tlačou celej cievky testovať len rozhranie: jeden kotúč a polovicu tela zrezanú
   v slicri na prvých ~8 mm. Overí to bajonet aj vôle za 15 minút namiesto 6 hodín.

---

## 10. Čo nerobiť

- Nezavádzať vymeniteľné vložky ani „univerzálny" kotúč. Zadanie je opačné — nová cievka
  na každý kábel.
- Nenahrádzať bajonet pružným zacvaknutím. Pružný prvok sa unaví a praskne.
- Nemeniť deliacu rovinu z pása inam. Pás je rovina symetrie a škáru tam prekryje kábel.
- Neprechádzať na FreeCAD. Model je celý rotačný a odvodený z pár čísel; pri parametrickej
  zmene priemeru by sa v histórii rozbili skice (Topological Naming Problem).
- Nepridávať nič, čo vyžaduje podpery vnútri rotačnej škáry.
