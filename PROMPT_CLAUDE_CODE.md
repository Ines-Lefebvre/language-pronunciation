# PROMPT — Claude Code / Language Pronunciation Trainer

Tu prends en charge le projet **LANGUAGE Pronunciation Trainer**, une app d'apprentissage de la prononciation déployée sur Vercel + n8n.

## Lis d'abord

Lis intégralement `CONTEXT.md` avant toute action. Il contient :
- L'architecture complète
- Les IDs des workflows n8n
- Les credentials utilisés
- L'état actuel et les bugs connus
- Ce qui reste à faire

## Accès

- **Repo GitHub :** `https://github.com/Ines-Lefebvre/language-pronunciation`
- **Vercel :** auto-deploy sur push `main` (compte Ines-Lefebvre)
- **n8n :** accessible via MCP `Mon n8n` (déjà connecté)
- **Fichiers locaux :** `C:\Users\LapuyadeFamily\Documents\language-pronunciation\`

## Règles de travail

1. **Avant de modifier un workflow n8n**, lis-le en full avec `n8n_get_workflow` pour avoir l'état réel
2. **Après chaque modification n8n**, vérifie dans les executions que ça passe
3. **Pour les modifications frontend**, modifie `index.html` localement puis guide le commit/push PowerShell (commandes séparées, pas de `&&`)
4. **Ne touche pas** aux autres workflows n8n du compte (AccidentDoc, ATEXYA, Brain Export, etc.)

## Première tâche

Vérifier que le fix `.weba → .webm` fonctionne en testant le flux complet :
1. Appeler `GET /webhook/start-session?lang=en` → vérifier réponse avec `audio_base64`
2. Regarder la dernière exécution de `KV0vzTr7wHzQ19I6` — est-ce qu'elle passe Whisper ?
3. Si encore en erreur → lire le message d'erreur exact et corriger

## Stack technique en place

- Frontend : HTML vanilla (un seul fichier)
- TTS : OpenAI `tts-1`, voix `nova`, format `mp3`
- STT : OpenAI `whisper-1`
- Évaluation : Mistral `mistral-small-latest`, JSON mode
- Audio browser : MediaRecorder API, webm/opus
