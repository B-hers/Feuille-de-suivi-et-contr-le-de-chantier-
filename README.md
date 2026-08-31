# Rapport de Chantier — We Green Energy

*README technique du dépôt. Sert de mémoire de contexte pour reprendre le développement — avec Claude, Cursor, ou tout autre outil. Dernière mise à jour : 25 août 2026.*

---

## 1. Vue d'ensemble

**Ce qu'est l'application** : un fichier HTML unique combinant deux outils métier de We Green Energy (installateur solaire) :

- **We Green** — rapport de chantier photovoltaïque (onglets DC / AC / Admin / SAV / Maintenance), utilisé par les équipes techniques pendant/après l'installation.
- **We Cover** — rapport de chantier pour l'isolation de façade (ETICS) et la toiture, filiale/activité sœur.

Les deux vivent dans le **même fichier HTML**, servis par la **même page GitHub Pages**, mais sont **isolés en code** l'un de l'autre.

**Écosystème plus large** (fichiers séparés, non couverts par ce document) :
- **Briefing de Chantier** (aussi appelé "Feuille de Chantier") — rempli par le PM *avant* le chantier, pour préparer l'intervention.
- **Rapport de Chantier** (ce fichier) — rempli par l'équipe technique *pendant/après* le chantier, pour documenter le travail réalisé (photos, checklists, avancement).

**Déploiement** : GitHub Pages, `https://b-hers.github.io/Rapport-de-chantier-SAV-Maintenance/`
**Stack** : React 18 + Babel standalone (compilation dans le navigateur, **aucun build step**), déployé comme fichier `.html` unique. Pas de npm, pas de bundler.

---

## 2. Architecture technique

### 2.1 Structure du fichier

```
<script type="text/babel">
  ... code partagé (login, styles, constantes GREEN/BORDER/etc.) ...
  ... code We Green (composants, App We Green) ...          ← AVANT l'IIFE

  var WCApp;
  (function(){
    ... code We Cover (composants, App We Cover) ...         ← DANS l'IIFE, isolé
    WCApp = App;   // expose We Cover au routeur
  })();

  function detectEntity() { ... }   // lit ?ref=WC-... ou ?app=wcover
  function WCRouter() { return entity === "wcover" ? <WCApp/> : <App/>; }
  ReactDOM.createRoot(...).render(<WCRouter/>);
</script>
```

**Pourquoi l'IIFE ?** We Cover a son propre composant `App`, sa propre `PreviewDoc`, son propre `HistoryPanel`, `CloturJourneeBlock`, etc. — des noms de fonctions identiques à ceux de We Green mais avec des implémentations différentes. Sans isolation, `function App()` déclarée deux fois au même niveau global casserait tout. L'IIFE crée un scope fermé où We Cover peut redéfinir ces noms sans collision.

**Composants réellement partagés** (déclarés hors IIFE, donc accessibles aux deux via la chaîne de portée JS) :
- `PhotoZone` — gestion photos (upload, annotation, réordonnancement, compression)
- Constantes de style : `GREEN`, `GREEN_LIGHT`, `GREEN_MID`, `BORDER`, `WHITE`, `MUTED`, `BG`, `CARD`, `SH`, `LBL`
- L'écran de connexion (login screen, sélecteur sous-traitant/WGE)
- `SUPA_URL` / `SUPA_KEY` (le projet Supabase principal)

### 2.2 Comment savoir si un bout de code est We Green ou We Cover

```bash
grep -n "^var WCApp;\|^WCApp = App;" fichier.html
# donne les deux numéros de ligne délimitant l'IIFE We Cover
```

Tout ce qui est **avant** `var WCApp;` ou **après** `WCApp = App;` (dans le routeur final) = **We Green**.
Tout ce qui est **entre les deux** = **We Cover**.

Ces numéros de ligne bougent à chaque édition — toujours les re-vérifier avant de chercher "où est telle fonction".

### 2.3 Routage — quel mode s'affiche

Dans `detectEntity()` :
- `?ref=WC-...` ou `?ref=wc-...` (préfixe insensible à la casse) → mode We Cover
- `?app=wcover` → mode We Cover (indépendamment du ref)
- `?app=wgreen` ou par défaut → mode We Green

Pour construire un lien vers un rapport We Cover depuis l'extérieur : toujours ajouter `&app=wcover` en plus du `ref=`, pour ne pas dépendre du préfixe de la référence.

### 2.4 Outil de validation syntaxique

Comme il n'y a pas de build step, une erreur de syntaxe JSX casse **toute la page** (écran blanc) sans message clair pour l'utilisateur final. Un script Node local permet de valider chaque bloc `<script type="text/babel">` avant livraison :

```js
// check_syntax.js
const fs = require("fs");
const babel = require("@babel/core");
const html = fs.readFileSync(process.argv[2], "utf8");
const re = /<script type="text\/babel">([\s\S]*?)<\/script>/g;
let m, idx = 0, hadError = false;
while ((m = re.exec(html))) {
  idx++;
  try {
    babel.transformSync(m[1], { presets: ["@babel/preset-react"], sourceType: "script" });
    console.log("Block " + idx + ": OK");
  } catch (e) { hadError = true; console.log("Block " + idx + ": ERROR — " + e.message); }
}
process.exit(hadError ? 1 : 0);
```

```bash
npm install --no-audit --no-fund @babel/core @babel/preset-react
node check_syntax.js fichier.html
```

**À lancer après chaque modification**, sans exception. Plusieurs bugs de production ont été causés par des blocs JSX mal fermés lors de réécritures (fonctions supprimées par accident, div non refermées).

---

## 3. Modèle de données

### 3.1 Supabase — projets et tables

| Projet Supabase | URL | Usage |
|---|---|---|
| Principal (We Green + We Cover) | `SUPA_URL` (voir le code — clé publishable, sans risque à exposer côté client) | `feuilles_controle`, `versions`, `audit_log` |
| Briefing de Chantier | `BRIEFING_URL` (différent, autre projet Supabase) | `feuilles_chantier` |

**Tables principales (projet principal)** :
- **`feuilles_controle`** — table unique pour les deux apps. Colonnes : `id` (auto), `ref`, `label`, `data` (JSONB), `updated_at`.
  - We Green : plusieurs lignes possibles par `ref` (multi-feuilles / sections), `label` = nom de la feuille.
  - We Cover : **une seule ligne active par `ref`**, `label = "wecover"`. L'`id` de cette ligne est suivi côté app dans un `useRef` (`rowIdRef`) pour savoir s'il faut `POST` (créer) ou `PATCH` (mettre à jour).
- **`versions`** — table générique de snapshots/historique. `ref` (string libre), `data`, `user_email`, `user_name`, `label`, `created_at`.
  - We Green (contrôle qualité) : `ref = "<ref>_ctrl_<sheetId>"`
  - We Cover : `ref = "<ref>_wc"`
  - **Convention** : suffixer le `ref` pour partitionner logiquement une table partagée plutôt que créer une table dédiée. Utile pour ne pas dépendre de la création manuelle d'une nouvelle table Supabase à chaque nouvelle fonctionnalité.
- **`audit_log`** — même principe, mêmes conventions de `ref` suffixé.

**⚠️ Piège déjà rencontré** : des tables *dédiées* (`rapports_wecover`, `rapports_wecover_versions`, `rapports_wecover_audit`) avaient été référencées dans le code sans jamais être créées côté Supabase. Toutes les sauvegardes échouaient silencieusement (`try/catch` avalant l'erreur), avec un statut "sauvegarde locale" permanent. **Solution retenue : ne plus créer de tables dédiées — toujours réutiliser `feuilles_controle`/`versions`/`audit_log` avec un `label` ou un suffixe de `ref` distinctif.** C'est plus robuste (la table existe forcément, puisqu'elle est déjà utilisée en production par We Green) et évite ce genre de bug.

### 3.2 Structure des données We Cover (`data`)

```js
{
  schemaVersion: 2,
  societe: "wecover",
  ref: "wc-S04702",
  meta: {
    client, adresseChantier, dateDebut, dateFinPrevue,
    pm, chefChantier, commandeRef,
    nom_tache   // capturé depuis ?task= dans l'URL, persisté ensuite
  },
  typeChantier: { facade: bool, toiture: bool },
  facade: { sections: { <key>: { quantite, notes, photos: [] } }, notes },
  toitures: [ { id, nom, type: "pente"|"plate", sections: {...}, notes } ],
  journeesCloturees: [ { id, date, heure, auteur, note } ],
  remarques: ""
}
```

**Modèle de section simplifié (depuis juin 2026)** : chaque section = `{ quantite, notes, photos }`. Pas de notion de "passage" daté ni de % d'avancement manuel (ancien modèle, migré automatiquement — voir §4).

Chaque photo : `{ name, url, type, addedAt, uploadedBy, drawingUrl?, textObjects? }` — `addedAt` et `uploadedBy` sont capturés **automatiquement** à l'ajout par le composant `PhotoZone` partagé (fallback : `window._wgeCurrentUser` → `localStorage.wge_user`).

### 3.3 Structure des données du Briefing (`feuilles_chantier`)

**Deux générations de schéma coexistent** — toujours prévoir un fallback :

```js
// Nouvelle structure (actuelle)
data.sites[].pvInstallations[].tech.{ panneaux, panneaux2, onduleurs, batterie, borne }
data.sites[].pvInstallations[].strings[]   // { onduleur, count, orient, orientLibre }
data.sites[].electrical.{ amperageDisj, compteur, reseau }

// Ancienne structure (fallback, chantiers créés avant la migration)
data.installations[].tech.{ ...mêmes champs, mais électrique inclus dans tech }
data.client.{ nom, adresse }
data.weGreen.{ manager }
```

Toute lecture du Briefing doit gérer les deux (voir le pattern dans `prefillFromChantier` ou l'import AS-Built pour un exemple de code robuste).

---

## 4. Pièges connus & bugs déjà résolus

*Pour éviter de re-découvrir les mêmes problèmes.*

| Symptôme | Cause | Fix |
|---|---|---|
| Sauvegardes We Cover bloquées en "local" | Constante `WC_TABLE` référencée mais jamais déclarée | Ne plus utiliser de table dédiée — voir §3.1 |
| Photos perdues chez les sous-traitants (mobile) | Aucun flush de sauvegarde quand l'app passe en arrière-plan (systématique en prenant une photo — ouverture de l'appareil photo natif). Safari iOS peut suspendre l'onglet avant la fin du debounce. | Handlers `visibilitychange` / `pagehide` / `beforeunload` qui forcent la sauvegarde immédiate + mutex anti-concurrence + debounce réduit (1.2s) |
| Boutons du lightbox non cliquables si section verrouillée | `.locked-section button:not([data-lightbox])` a une spécificité CSS plus élevée que l'override `pointer-events:all` du lightbox | Ajouter l'attribut `data-lightbox="true"` sur les boutons du lightbox (sort de la portée du sélecteur `:not()`, plus fiable qu'une bataille de spécificité) |
| `<Composant ref={...} />` ne reçoit jamais la prop | `ref` est un nom réservé en React (intercepté pour les refs DOM), jamais transmis comme prop normale à un composant fonction classique | Toujours nommer autrement (`refChantier`, `sheetRef`, etc.) |
| Un style hérité disparaît sur un breakpoint | `{ ...CARD, padding: isMobile ? "compact" : undefined }` — `undefined` écrase la valeur héritée du spread au lieu de la laisser telle quelle | Utiliser `CARD.padding` explicitement plutôt que `undefined` |
| Nom de PM "faux" après correction d'une coquille | Le renommage dans la liste des PM (`PM_DATA`) ne corrige pas rétroactivement les rapports déjà sauvegardés avec l'ancien nom stocké dans `data.meta.pm` | Ajouter une migration dans `normalize()` qui corrige la valeur silencieusement au chargement |
| Composant qui semble être la bonne référence mais ne fait rien | Du code mort peut exister dans le fichier (ex: `InstallationBlock`, jamais monté nulle part) — probablement un vestige de fusion antérieure avec un autre fichier | Toujours vérifier avec `grep -n "<NomDuComposant"` qu'un composant est réellement utilisé avant de s'y fier comme référence |
| Deux fonctions du même nom, laquelle éditer ? | We Green et We Cover ont chacun leur propre `PreviewDoc`, `HistoryPanel`, `CloturJourneeBlock` (même nom, implémentations différentes, dans des scopes séparés par l'IIFE) | Toujours vérifier le numéro de ligne par rapport aux bornes de l'IIFE (§2.2) avant d'éditer |

---

## 5. Conventions & bonnes pratiques de développement

1. **Rétrocompatibilité non négociable.** Toute évolution du schéma de données doit gérer gracieusement les anciennes données (migration automatique dans `normalize()`, jamais de perte silencieuse). Toujours confirmer explicitement la rétrocompatibilité avant de livrer.
2. **Édits chirurgicaux plutôt que réécritures complètes.** Un fichier de 10 000+ lignes ne se laisse pas réécrire en un coup sans risque. Localiser précisément (numéros de ligne, `grep`), lire le contexte exact avant de modifier, valider après chaque changement.
3. **Toujours valider la syntaxe après modification** (voir §2.4) — aucune exception, même pour un changement d'une ligne.
4. **Vérifier l'étanchéité We Green / We Cover.** Après toute modification censée ne toucher qu'une des deux apps, comparer par diff la portion "de l'autre app" avec une version antérieure connue pour confirmer qu'elle n'a pas bougé.
5. **Se méfier du code mort.** Un composant qui existe dans le fichier n'est pas forcément utilisé — vérifier son usage réel avant de le prendre comme référence ou de l'étendre.
6. **Erreurs silencieuses = ennemi n°1.** Plusieurs bugs de production (WC_TABLE, photos perdues) venaient de `try/catch` qui avalaient l'erreur sans aucune remontée visible. Préférer un statut visible (ex: `saveStatus`) à un échec totalement silencieux.

---

## 6. Historique des évolutions majeures (chronologique, résumé)

- **AS-Built We Green** — dossier client généré à partir du rapport, mode édition WGE (`?asbuilt=1`) vs vue client (`?client=1`), sélection de photos incluses/exclues, informations techniques important-export depuis le Briefing, PDF avec pages complètes via PDF.js.
- **We Cover — refonte complète** — simplification du modèle de données (passages datés + % manuel → sections plates avec migration automatique), ajout d'une vue "Avancement" façon We Green (par jour / par intervenant), fix du bug de sauvegarde, clôture de journée + notification WhatsApp au PM, type de chantier en mode "ajout + suppression à double confirmation" (jamais de désélection accidentelle).
- **We Green** — photos HD pour les usages marketing (vue générale, vue drone), import du nombre de strings depuis le Briefing.

---

## 7. Pour reprendre le travail avec un nouvel outil ou une nouvelle session

1. Récupérer la dernière version du fichier HTML déployé (ne pas repartir d'une version locale potentiellement périmée).
2. Mettre en place le script de validation syntaxique (§2.4) avant toute modification.
3. Relire ce document en entier — en particulier §4 (pièges connus) avant de "redécouvrir" un bug déjà résolu.
4. Pour toute nouvelle fonctionnalité côté Supabase, réutiliser les tables existantes (§3.1) plutôt que d'en créer une nouvelle — plus robuste, pas de risque d'oubli de création manuelle.
5. Toujours vérifier l'isolation We Green / We Cover avant de livrer (§5, point 4).
