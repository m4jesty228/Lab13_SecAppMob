# Lab 13 — Bypass de la Détection de Root Android : Objection

**Auteur :** DOSSAH Yao Landry  
**Filière :** Génie CyberDefense et Systèmes de Télécommunications Embarquées (GCDSTE)  
**Établissement :** ENSA Marrakech

---

## Contexte pédagogique

Ce laboratoire porte sur le contournement des mécanismes de détection de root Android via **Objection**, une CLI Python construite sur Frida qui automatise les hooks courants sans écrire de scripts manuellement. L'objectif est de neutraliser les vérifications Java (Build.TAGS, File.exists, RootBeer, Runtime.exec) et, si nécessaire, les checks natifs (syscalls C/C++), puis de valider que l'application ne détecte plus l'environnement rooté.

> **Cadre éthique :** ces techniques sont appliquées uniquement sur une application de test autorisée dans le cadre du cours.

---

## Environnement

| Composant | Détail |
|-----------|--------|
| OS hôte | Windows / Linux / macOS |
| Python | 3.8+ |
| Objection | Dernière version (via pipx ou pip) |
| Frida | Version alignée avec frida-server |
| ADB | Android Platform Tools |
| Appareil | Android 8.0+ (débogage USB activé) |
| Application cible | `com.example.rootcheck` |

---

## Différence Objection vs Frida pur

```
Frida pur                          Objection
─────────────────────────────      ─────────────────────────────
Écriture manuelle de scripts JS    Commandes CLI prêtes à l'emploi
Contrôle total, flexible           Interface interactive (REPL)
Apprentissage plus long            Idéal pour l'exploration rapide
frida -U -f pkg -l script.js       objection -g pkg explore
```

Objection est une surcouche Frida — il utilise frida-server en arrière-plan. Les deux doivent avoir des versions alignées.

---

## Étape 1 — Installation d'Objection

```bash
# Méthode recommandée (isolation)
pip install --user pipx
pipx ensurepath
pipx install objection

# Ou via pip classique
pip install --upgrade objection

# Vérification
objection --version
```

> **Windows :** si `objection` n'est pas reconnu, ajouter le dossier `Scripts` de Python au PATH (`%USERPROFILE%\AppData\Roaming\Python\Python3xx\Scripts`).

---

## Étape 2 — Déploiement de frida-server

```bash
# Identifier l'architecture
adb shell getprop ro.product.cpu.abi

# Pousser et démarrer frida-server
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0 &"

# Vérification
frida-ps -Uai
```

**Résultat observé :** `frida-ps -Uai` liste toutes les applications installées sur l'appareil. `com.example.rootcheck` est visible dans la liste avec son PID.

---

## Étape 3 — Bypass avec Objection

### Stratégie 1 — Spawn (recommandé)

Injecte les hooks avant le démarrage du code applicatif — indispensable quand les vérifications se font dans `Application.onCreate()` :

```bash
objection -g com.example.rootcheck explore --startup-command "android root disable"
```

### Stratégie 2 — Attach

Attacher à une application déjà lancée — utile si le spawn provoque un crash :

```bash
# Lancer l'app manuellement sur le téléphone, puis :
objection -g com.example.rootcheck explore
# Dans la console Objection :
android root disable
```

### Commandes d'exploration utiles dans la console Objection

```
android hooking search classes root
android hooking search methods isRoot
android hooking watch class com.example.RootCheck
android hooking set return_value com.example.RootCheck isRooted false
help android root
```

### Enchaîner plusieurs actions au démarrage

```bash
objection -g com.example.rootcheck explore \
  --startup-command "android root disable" \
  --startup-command "android hooking search classes root"
```

---

## Étape 4 — Checks natifs (C/C++)

Objection cible principalement la couche Java. Si l'application implémente des vérifications natives (`open`, `access`, `stat` sur `/system/xbin/su`, lecture de `/proc/mounts`), trois options complémentaires :

**Option A — Hook Java passerelle** (souvent suffisant) :
```
android hooking set return_value com.example.RootCheck isRooted false
```

**Option B — Script Frida natif en parallèle** :
```bash
frida -U -n "NomDuProcessus" -l bypass_native.js
```

**Option C — Découverte des appels natifs avec frida-trace** :
```bash
frida-trace -U -i open -i access -i stat -i openat com.example.rootcheck
```

---

## Résultats observés

**Avant Objection :** l'application affiche "Root detected" et bloque l'accès aux fonctionnalités.

**Après `android root disable` :**

```
[objection] android root disable
(agent) Attempting to disable root detection...
(agent) Hook on Build.TAGS installed
(agent) Hook on File.exists installed - blocking suspicious paths
(agent) Hook on Runtime.exec installed - blocking su calls
(agent) RootBeer hooks installed
[objection] Root detection disabled
```

L'application affiche "Not rooted" et les fonctionnalités précédemment bloquées deviennent accessibles.

| Vérification | Sans Objection | Avec Objection |
|-------------|---------------|----------------|
| `Build.TAGS` | `test-keys` → root détecté | `release-keys` → OK |
| `File.exists(/system/xbin/su)` | `true` → root détecté | `false` → OK |
| `RootBeer.isRooted()` | `true` → root détecté | `false` → OK |
| `Runtime.exec("su")` | succès → root détecté | bloqué → OK |
| Accès application | bloqué | débloqué |

---

## Dépannage

| Problème | Cause probable | Solution |
|----------|---------------|----------|
| `objection: command not found` | Scripts Python hors PATH | Ajouter le dossier Scripts de Python au PATH |
| `unable to connect to remote frida-server` | frida-server non lancé ou version non alignée | Vérifier `adb shell ps \| grep frida` et aligner les versions |
| Root encore détecté après bypass | Checks natifs non couverts ou hooks trop tardifs | Passer en spawn, ajouter hooks natifs via Frida |
| App détecte Frida/Objection | Anti-instrumentation implémentée | Préférer attach au lieu de spawn, réduire le nombre de hooks |

---

## Points clés retenus

- Objection automatise les hooks Frida les plus courants via une interface interactive — il remplace l'écriture manuelle de scripts pour les cas standard.
- `--spawn` est préférable à `--attach` quand les checks sont effectués au démarrage (`Application.onCreate`, blocs statiques) — mais certaines applications crashent sous spawn et nécessitent l'attach.
- Objection couvre la couche Java ; les checks natifs (syscalls C/C++) nécessitent des hooks Frida complémentaires via `bypass_native.js` ou `frida-trace`.
- Une application qui détecte la présence de frida-server lui-même (`anti-instrumentation`) résiste à Objection comme à Frida pur — ce cas relève de techniques avancées hors scope de ce lab.
- `android sslpinning disable` est la commande Objection équivalente pour le bypass de certificate pinning — même logique, même stratégie spawn/attach.

---

*Lab réalisé dans le cadre du cours Sécurité des Applications Mobiles — ENSA Marrakech, Filière GCDSTE*
