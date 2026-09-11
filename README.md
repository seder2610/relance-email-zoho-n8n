# Relances Email auto — Notion → Zoho

Chaque matin (Lun-Ven, 8h Paris), le workflow scanne ta base Notion **"📋 Prospection — Suivi vagues"**, trouve les fiches dont une relance email est due aujourd'hui, et les envoie une par une à des heures aléatoires entre 9h et 15h via ton compte Zoho Mail — sans que tu aies à y toucher.

---

## 1. Comment ça marche

```
8h (Lun-Ven)
   │
   ▼
Notion : requête sur la base — Canal = "Email" ET "Relance due" = true
         ET Statut ∈ {Message envoyé, Relance 1 envoyée, Relance 2 envoyée}
   │
   ▼
Code : pour chaque fiche → détermine quelle relance envoyer (1, 2 ou 3),
       extrait Objet + Corps du texte stocké dans la colonne correspondante,
       tire une heure aléatoire entre 9h et 15h, trie par ordre d'envoi
   │
   ▼
Boucle (1 fiche à la fois) :
   ⏳ Attendre l'heure tirée → 📧 Envoyer via Zoho (SMTP) → ✅ Notion : Statut mis à jour
   │
   └─ répète jusqu'à la dernière fiche du jour
```

**Ce que la base Notion fait déjà pour toi :** la colonne formule **"Relance due"** calcule si une fiche doit être relancée aujourd'hui (à partir de "Date dernière action" + "Prochaine relance"). Le workflow s'appuie dessus — il n'y a pas de logique de date à gérer côté n8n.

**Important — la base mélange 2 canaux.** Les colonnes "Relance 1/2/3" servent aussi bien aux relances LinkedIn (Canal = "LinkedIn") qu'aux relances email (Canal = "Email"). Le filtre du workflow ne prend que les fiches **Canal = "Email"** — les relances LinkedIn dues le même jour ne sont pas touchées par ce workflow (elles ont besoin de leur propre automatisation ou d'un envoi manuel).

---

## 2. Avant de commencer

- [ ] Compte **Zoho Mail** actif
- [ ] Intégration **Notion** déjà connectée à n8n (credential `Notion API` — probablement déjà créée si tu as d'autres workflows Notion dans ce repo)
- [ ] `n8n` self-hosted et accessible

### Générer un mot de passe d'application Zoho (5 min, pas de code)

Le workflow envoie les emails en SMTP — la méthode la plus simple, pas besoin d'app OAuth Zoho :

1. Connecte-toi sur [mail.zoho.com](https://mail.zoho.com) → **Paramètres** → **Sécurité** → **Mots de passe d'application**.
2. Génère un nouveau mot de passe d'application (nom libre, ex. "n8n relances").
3. Note-le — il ne sera plus jamais affiché après.

### Créer le credential SMTP dans n8n

1. n8n → **Credentials** → **New** → **SMTP**.
2. Renseigne :
   - **Host** : `smtp.zoho.com` (ou `smtp.zoho.eu` si ton compte Zoho est hébergé en Europe)
   - **Port** : `465`
   - **SSL/TLS** : activé
   - **User** : ton adresse Zoho complète (ex. `toi@tondomaine.com`)
   - **Password** : le mot de passe d'application généré ci-dessus
3. Sauvegarde, note l'ID du credential.

---

## 3. Installer le workflow

1. Dans n8n, **importe** `relance-email-zoho-v1.json` (menu `...` → Import from File).
2. Remplace les placeholders (Ctrl+F dans l'éditeur, ou édite le JSON avant import) :

   | Placeholder | Où le trouver |
   |---|---|
   | `REMPLACER_PAR_ID_CREDENTIAL_NOTION` | ID du credential Notion API (2 nodes concernés) |
   | `REMPLACER_PAR_ID_CREDENTIAL_ZOHO_SMTP` | ID du credential SMTP créé à l'étape 2 |
   | `REMPLACER_PAR_TON_EMAIL_ZOHO` | Ton adresse Zoho complète (champ "From" du node d'envoi) |

3. Reconnecte les credentials sur chaque node concerné si l'import ne les a pas rattachés automatiquement.

L'ID de la base Notion (`49c11bda-c9b4-47f7-82a2-3fbaac5a159a`) est déjà dans le workflow — pas besoin de le toucher.

---

## 4. Point à vérifier avant d'activer (je n'ai pas d'instance n8n connectée pour tester en live)

Le filtre Notion sur la colonne formule **"Relance due"** suppose qu'elle retourne un résultat de type **texte** (`"true"`/`"false"`), déduit de la configuration de ta vue "Relances dues" existante. Si le premier test renvoie 0 résultat alors que tu sais qu'une relance Email est due :

1. Exécute manuellement le node **"🔍 Notion — Relances Email dues"** dans n8n.
2. Si ça renvoie une erreur Notion du type *"Formula property doesn't support this filter"*, remplace dans le `body` du node :
   ```
   "formula":{"string":{"equals":"true"}}
   ```
   par :
   ```
   "formula":{"checkbox":{"equals":true}}
   ```

---

## 5. Premier test (sans rien casser)

1. Laisse le workflow **inactif** (`active: false` par défaut).
2. Dans n8n, clique **"Execute Workflow"** en manuel un matin où tu sais qu'au moins une fiche Email est due (regarde la vue **"Relances dues"** dans Notion, filtre sur Canal = Email).
3. Vérifie : le node Code sort bien une fiche avec Objet/Corps corrects, une heure `sendAt` entre 9h et 15h.
4. Laisse l'exécution tourner (ou stoppe après le node Code si tu ne veux pas attendre/envoyer pour de vrai) — le Wait peut durer jusqu'à 6h, c'est normal, n8n gère ça nativement.
5. Une fois confiant : active le workflow.

---

## 6. Dépannage

| Symptôme | Cause probable | Fix |
|---|---|---|
| 0 résultat alors qu'une relance Email est due | Filtre formule "Relance due" au mauvais type | Voir section 4 |
| Erreur SMTP 535 (auth failed) | Mot de passe normal utilisé au lieu du mot de passe d'application | Régénère un mot de passe d'application Zoho |
| Email envoyé mais Notion pas mis à jour | `pageId` introuvable après le Wait | Vérifie que le node "✅ Notion — statut mis à jour" référence bien `$('🧹 Préparer + planifier...')` et non `$json` |
| "Template utilisé" pas mis à jour sur la Relance 3 | Normal — l'option "Relance 3" n'existe pas dans ce champ select Notion (seules "Relance 1"/"Relance 2" existent). Le Statut, lui, est toujours mis à jour correctement. | Si tu veux ce suivi, ajoute l'option "Relance 3" dans la colonne Notion |
| Relances envoyées le week-end alors que tu ne veux que Lun-Ven | — | Déjà en place (`0 8 * * 1-5`). Pour tous les jours, remplace par `0 8 * * *` |
