# LANGUAGE — Pronunciation Trainer
## Fichier de recontextualisation — pour Claude Code

---

## Statut actuel

L'application est **déployée sur Vercel** et **fonctionnelle**.
Bugs corrigés (session du 31 mars 2026) :
- **Format audio** `.weba → .webm` : corrigé et validé end-to-end (Brave Android + Chrome)
- **audio_base64 vide** : les noeuds Build Response (WF1) et Build Correct/Incorrect Response (WF2) utilisent désormais `getBinaryDataBuffer()` au lieu de `.data` (qui retournait `"filesystem-v2"`)
- **Phonétique IPA** : le prompt Mistral demande maintenant une phonétique lisible par un francophone (ex: "ouoh-ter" pour water) au lieu de symboles IPA

---

## Architecture générale

```
Vercel (frontend HTML vanilla)
    └── appelle n8n webhooks directement depuis le browser
            ├── GET /start-session  → charge 270 mots shufflés + TTS premier mot
            └── POST /check         → Whisper → Mistral évalue → TTS mot suivant/même
```

**Stack :**
- Frontend : HTML vanilla, zéro framework, zéro build step
- Backend : n8n (VPS Hostinger) + OpenAI (Whisper STT + TTS nova) + Mistral (évaluation)
- Repo : `https://github.com/Ines-Lefebvre/language-pronunciation`
- Vercel : déployé sur le compte Ines-Lefebvre, auto-deploy sur push `main`

---

## Fichiers du projet

```
language-pronunciation/
├── index.html      — frontend complet (tout en un fichier)
└── vercel.json     — { "version": 2 } — suffisant pour static deploy
```

---

## N8N — Workflows concernés

**Serveur n8n :** `https://n8n.srv833062.hstgr.cloud`

### WF1 — Start Session
- **ID :** `Sm0W5srTDcL7L628`
- **Webhook :** `GET /webhook/start-session?lang=en|es`
- **Rôle :** Charge liste 270 mots (hardcodée dans un Code node), shuffle, appelle OpenAI TTS sur le mot[0], retourne JSON avec `words[]`, `audio_base64`, `session_id`
- **Statut :** ✅ Fonctionne — 484 mots EN / 483 mots ES (listes étendues le 31/03/2026)

**Réponse type :**
```json
{
  "session_id": "sess_1234_abc",
  "langue": "en",
  "words": ["water", "fire", ...],
  "total": 270,
  "current_index": 0,
  "current_word": "water",
  "audio_base64": "<mp3 en base64>"
}
```

### WF2 — Check Pronunciation
- **ID :** `KV0vzTr7wHzQ19I6`
- **Webhook :** `POST /webhook/check` (multipart/form-data)
- **Rôle :** Reçoit audio + métadonnées → Whisper transcrit → Mistral évalue → TTS mot suivant ou même mot → retourne JSON
- **Statut :** ✅ Fonctionne — fix audio .weba→.webm validé, extraction base64 corrigée, phonétique FR naturelle

**Corps de la requête (FormData) :**
```
audio       : blob audio (webm/opus)
session_id  : string
card_index  : number
mot         : string (mot attendu)
next_word   : string (mot suivant dans la liste)
langue      : "en" | "es"
```

**Réponse si correct :**
```json
{
  "resultat": "correct",
  "phonetique": "...",
  "transcription": "...",
  "next_word": "fire",
  "next_index": 1,
  "audio_base64": "<mp3 mot suivant>"
}
```

**Réponse si incorrect :**
```json
{
  "resultat": "incorrect",
  "phonetique": "ouoh-ter",
  "transcription": "...",
  "mot_attendu": "water",
  "audio_base64": "<mp3 même mot rejoué>"
}
```

### WF3 — Hint (non utilisé pour l'instant)
- **ID :** `jbFAwsKZKtwkUmFd`
- Laissé en place, pas connecté au frontend

---

## Credentials n8n utilisés

| Credential | ID | Usage |
|---|---|---|
| OpenAi account | `ReY01TMBOwF2KV4Q` | Whisper STT + TTS |
| Mistral OCR | `ky1L3tKefy3aLQey` | Mistral evaluation |

---

## Frontend — logique clé (index.html)

```
startSession(lang)
  → GET /start-session?lang=en
  → reçoit words[], audio_base64
  → joue audio (TTS premier mot)
  → affiche le mot à l'écran

handleRecordClick()
  → state: idle → recording → sending
  → MediaRecorder (webm/opus)
  → POST /check avec FormData

handleResult(data)
  → correct : session.correct++, joue audio suivant, avance index après 1.6s
  → incorrect : affiche phonétique, rejoue même mot, reste sur la carte

endSession()
  → score X/270 + pourcentage
```

**Point critique navigateur :**
Brave et Chrome envoient l'audio avec mimeType `audio/webm;codecs=opus` mais extension `.weba`.
Le nœud `Load Session & Card` dans WF2 force `fileName: 'audio.webm'` + `mimeType: 'audio/webm'` avant Whisper.
Si un autre navigateur se comporte différemment, c'est là qu'il faut regarder.

**Point critique extraction base64 (n8n filesystem-v2) :**
Les nœuds Code qui lisent un binaire (audio TTS) doivent utiliser :
```js
const buf = await this.helpers.getBinaryDataBuffer(0, key);
audioBase64 = buf.toString('base64');
```
et **PAS** `binary[key].data` qui retourne la string `"filesystem-v2"` au lieu du contenu réel.

---

## Flux pédagogique (méthode Assimil)

```
1. Mot affiché à l'écran
2. TTS joue le mot automatiquement (voix nova OpenAI)
3. Utilisateur appuie sur le micro → parle → relâche
4. Whisper transcrit → Mistral compare phonétiquement
5a. CORRECT → son ✓, TTS joue le mot SUIVANT, avance
5b. INCORRECT → son ✗ + phonétique affichée, TTS REJOUE le même mot, reste sur la carte
6. Boucle jusqu'aux 270 mots → écran de score
```

---

## Mots — source de données

Liste hardcodée directement dans le Code node `Init Session` de WF1.
**~500 mots** : 484 EN / 483 ES (listes étendues le 31/03/2026 — base 299/297 + complément ~200 dédupliqué via `Set`).
Catégories : corps, nourriture, maison, nature, animaux, ville, temps, émotions, verbes, adjectifs.
Sélection par `?lang=en` ou `?lang=es`.
Shufflée à chaque nouvelle session.

---

## Ce qui reste à faire / améliorer

### Priorité haute
- [x] **Tester end-to-end** le fix `.weba → .webm` sur Brave et Chrome — ✅ validé 31/03/2026
- [x] **Fix audio_base64 vide** — ✅ corrigé avec `getBinaryDataBuffer()` le 31/03/2026
- [x] **Phonétique en français naturel** — ✅ prompt Mistral corrigé le 31/03/2026
- [ ] **Vérifier l'autoplay** : certains browsers bloquent l'audio sans interaction préalable — le bouton replay est là comme fallback
- [ ] **CORS** : si erreurs CORS en prod Vercel, vérifier que les headers `Access-Control-Allow-Origin: *` passent bien depuis n8n

### Priorité normale
- [ ] Ajouter traduction FR sous le mot (les mots EN/ES ont une traduction en mémoire — à ajouter dans le Word Bank)
- [ ] Compter les tentatives par mot (actuellement pas de compteur, boucle infinie sur un mot raté)
- [ ] Sauvegarder le score dans un Google Sheet ou Supabase pour tracking progression

### Backlog
- [ ] Ajouter le russe (besoin d'un autre TTS — Voxtral TTS ne couvre pas RU)
- [ ] Mode "révision" des mots ratés
- [ ] Intégration dans le Projet Mentat (collision avec vault Obsidian)

---

## Commandes utiles

**Tester WF1 directement :**
```
https://n8n.srv833062.hstgr.cloud/webhook/start-session?lang=en
```

**Push et redéploiement Vercel :**
```powershell
cd C:\Users\LapuyadeFamily\Documents\language-pronunciation
git add .
git commit -m "..."
git push
```
Vercel redéploie automatiquement sur push `main`.

**Voir les logs n8n :**
Aller dans n8n → Executions → filtrer sur `KV0vzTr7wHzQ19I6` (WF2) pour voir les erreurs Whisper/Mistral.

---

## Contexte projet plus large

Cette app fait partie du **Projet Mentat** (Franck Lapuyade / Metafactory).
Objectif : courbe d'apprentissage linguistique par répétition espacée phonétique.
Phase actuelle : outil dégradé/MVP pour valider le flux avant intégration dans l'app Mentat complète.
Infrastructure principale : n8n sur VPS Hostinger + Obsidian vault + Claude API.
