# Runbook — ris-plongee.com

Site vitrine du club (Jekyll + GitHub Pages). Ce document sert de référence
d'exploitation : comment le site est publié, comment le dépanner, et de quoi il
dépend.

## Architecture en une ligne

Site statique **Jekyll** → publié par **GitHub Pages** (build classique depuis
la branche `main`) → servi via le CDN Fastly de GitHub sur le domaine
**www.ris-plongee.com** (le domaine nu `ris-plongee.com` redirige vers `www`).

## Cycle de publication

1. Un commit arrive sur `main` (idéalement via une Pull Request).
2. La CI (`.github/workflows/ci.yml`) construit le site et vérifie les liens
   internes. **Si elle échoue, ne pas merger.**
3. Une fois sur `main`, GitHub Pages reconstruit et met en ligne en ~1–2 min.
4. `cache-control: max-age=600` → une page peut rester en cache 10 min côté CDN.

> Il n'y a **pas** d'environnement de préproduction. La prévisualisation se fait
> en local (voir plus bas). Toujours ouvrir une PR pour laisser la CI valider.

## Prévisualiser en local

```bash
bundle install
bundle exec jekyll serve   # http://localhost:4000
```
Ruby attendu : voir `.ruby-version`. En cas de souci de dépendances :
`bundle update` (le `Gemfile.lock` n'est pas utilisé par GitHub Pages, qui
utilise sa propre version épinglée de la gem `github-pages`).

## Surveillance

- **CI** — échoue sur tout build cassé / lien interne mort (bloquant sur PR).
- **Liens** (`.github/workflows/links.yml`) — audit hebdomadaire ; ouvre une
  issue `broken-links` si un lien externe est réellement mort.
- **Disponibilité** — *à mettre en place* : un moniteur externe (UptimeRobot /
  Better Stack, offre gratuite) sur `https://www.ris-plongee.com` avec :
  - une vérification par mot-clé (« Baptême ») pour détecter une page blanche ;
  - une alerte d'expiration du certificat TLS ;
  - idéalement un second check sur le lien HelloAsso (tunnel d'inscription).

## Playbook incident

**Le site est down / renvoie une erreur**
1. Vérifier l'état de GitHub Pages : https://www.githubstatus.com/
2. Onglet **Actions** du dépôt : le dernier déploiement Pages a-t-il échoué ?
3. Reproduire en local (`jekyll serve`). Si ça casse en local → bug de contenu.
4. Correctif rapide : `git revert <commit fautif>` puis push sur `main`.

**Une page s'affiche mal / lien cassé**
1. Lancer la CI localement : `bundle exec jekyll build && \
   gem install html-proofer && htmlproofer ./_site --disable-external`
2. Corriger, ouvrir une PR, laisser la CI valider.

**Le domaine ne répond plus**
1. Vérifier l'expiration du domaine et les enregistrements DNS chez le
   registraire (voir « Dépendances externes »).
2. Vérifier que le fichier `CNAME` du dépôt contient bien `www.ris-plongee.com`.

## Reprise après sinistre (DR)

Le code est intégralement versionné dans Git → aucune perte de contenu possible
tant que le dépôt GitHub existe. Les vrais points de défaillance uniques sont les
**comptes externes**, pas le code :

| Dépendance | Rôle | Point de vigilance |
|---|---|---|
| Dépôt GitHub `RisPlongee/ris-plongee` | code + hébergement Pages | garder ≥ 2 admins |
| Domaine `ris-plongee.com` | nom de domaine | **renouvellement** (auto-renew ?) |
| DNS | pointage vers GitHub Pages | enregistrements A/CNAME |
| Boîte `risplongee@gmail.com` | contact + agenda + réseaux | récupération / 2FA |
| HelloAsso (`asrp-ris-plongee`) | inscriptions baptême | tunnel critique |
| Google Maps / Agenda, YouTube, Instagram, Facebook, WhatsApp | embeds | comptes club |

> **À compléter par un responsable** : qui détient chaque compte, où est géré le
> renouvellement du domaine, et où sont stockés les accès (gestionnaire de mots
> de passe partagé recommandé).

## Actions à réaliser côté propriétaire (droits admin requis)

Ces réglages ne peuvent pas être faits par PR — un admin du dépôt doit les
activer dans les *Settings* :

- [ ] **Forcer HTTPS** : Settings → Pages → cocher « Enforce HTTPS »
      (actuellement désactivé).
- [ ] **Protéger `main`** : Settings → Branches → exiger le passage de la CI et
      une PR avant merge.
- [ ] **Moniteur de disponibilité** externe (cf. section Surveillance).
