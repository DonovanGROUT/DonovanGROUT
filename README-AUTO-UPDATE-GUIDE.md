# Guide de mise à jour automatique du README GitHub

## 📚 Introduction

Ce document explique le fonctionnement du système de mise à jour automatique du README GitHub, en particulier les fonctionnalités avancées :

- Récupération intelligente des liens de déploiement depuis les README des projets
- Affichage conditionnel des stars/forks (uniquement si > 0)
- Harmonisation du style des projets

## 🔄 Fonctionnement général

Le workflow GitHub Actions exécute les étapes suivantes toutes les 3 heures :

1. Récupère le dernier repository public créé
2. Extrait ses informations (nom, description, langage, etc.)
3. Recherche intelligemment un lien de déploiement dans son README
4. Formate les statistiques de manière conditionnelle
5. Met à jour les sections du README principal avec ces informations
6. Commit et pousse les changements

## 🔍 Recherche intelligente des liens de déploiement

### Méthode optimisée

La fonction `get_deployment_link()` utilise une approche par priorité :

1. Elle cherche **d'abord** des balises de commentaire HTML spécifiques
2. Si aucune n'est trouvée, elle recherche des liens externes selon certains critères

```bash
get_deployment_link() {
  local username="$1"
  local repo_name="$2"
  
  >&2 echo "🔍 Recherche du lien de déploiement pour $repo_name..."
  
  # Cas spécial pour le profil GitHub
  if [ "$repo_name" = "DonovanGROUT" ]; then
    >&2 echo "   🌟 Cas spécial détecté: Profil GitHub"
    echo "https://github.com/$username"
    return 0
  fi
  
  # Essayer différents noms de fichiers README
  for readme_file in "README.md" "readme.md" "README.MD" "readme.MD"; do
    readme_url="https://raw.githubusercontent.com/$username/$repo_name/main/$readme_file"
    
    # Télécharger le README
    readme_content=$(curl -s "$readme_url")
    
    if [[ -n "$readme_content" ]]; then
      # PRIORITÉ 1 : Chercher les balises de commentaire dédiées
      for pattern in "DEPLOY-LINK-START" "DEPLOYMENT-URL-START" "DEMO-LINK-START" "LIVE-DEMO-START" "PROJECT-URL-START"; do
        if deploy_link=$(echo "$readme_content" | grep -oP "<!-- $pattern -->(.*?)<!-- .*?-END -->" | sed 's/<!-- [^>]* -->//g' | grep -oE 'https?://[^[:space:]")\`]+' | head -1); then
          if [[ -n "$deploy_link" ]]; then
            >&2 echo "   🎯 Lien trouvé via balise $pattern: $deploy_link"
            echo "$deploy_link"
            return 0
          fi
        fi
      done
      
      # PRIORITÉ 2 : Chercher les liens en contexte (après des mots-clés spécifiques)
      for keyword in "démo" "demo" "live" "deployment" "déploiement" "deployed" "déployé" "site" "app" "application"; do
        if deploy_link=$(echo "$readme_content" | grep -i -B1 -A2 "$keyword" | grep -oE 'https?://[^[:space:]")\`]+' | grep -v 'github.com/[^/]+/[^/]+$' | head -1); then
          if [[ -n "$deploy_link" ]]; then
            >&2 echo "   🔎 Lien trouvé via mot-clé '$keyword': $deploy_link"
            echo "$deploy_link"
            return 0
          fi
        fi
      done
      
      # PRIORITÉ 3 : Accepter les liens GitHub spécifiques (ex: pages GitHub)
      if deploy_link=$(echo "$readme_content" | grep -oE 'https?://[^[:space:]")\`]+\.github\.io/[^[:space:]")\`]+'); then
        >&2 echo "   🔎 Lien GitHub Pages trouvé: $deploy_link"
        echo "$deploy_link"
        return 0
      fi
      
      # PRIORITÉ 4 : Liens de déploiement populaires (Netlify, Vercel, Heroku, etc.)
      if deploy_link=$(echo "$readme_content" | grep -oE 'https?://[^[:space:]")\`]+\.(netlify\.app|vercel\.app|herokuapp\.com|railway\.app|render\.com)'); then
        >&2 echo "   🚀 Lien de plateforme de déploiement trouvé: $deploy_link"
        echo "$deploy_link"
        return 0
      fi
    fi
  done
  
  echo ""
}
```

Cette approche hiérarchisée permet une détection beaucoup plus intelligente et fiable des liens.

### Bonnes pratiques pour structurer vos README

#### 1️⃣ Méthode recommandée : Utiliser des balises de commentaire HTML

C'est la méthode la plus fiable pour garantir la détection du bon lien :

```markdown
<!-- DEPLOY-LINK-START -->
[Voir la démo en ligne](https://votre-projet-demo.com)
<!-- DEPLOY-LINK-END -->
```

Balises reconnues :

- `DEPLOY-LINK-START` / `DEPLOY-LINK-END`
- `DEPLOYMENT-URL-START` / `DEPLOYMENT-URL-END`
- `DEMO-LINK-START` / `DEMO-LINK-END`
- `LIVE-DEMO-START` / `LIVE-DEMO-END`
- `PROJECT-URL-START` / `PROJECT-URL-END`

#### 2️⃣ Méthode alternative : Utiliser un contexte clair

Placez votre lien près de mots-clés comme "démo", "deployment", "live", etc. :

```markdown
## 🚀 Déploiement

Découvrez l'application en ligne : [Démo live](https://votre-projet-demo.com)
```

#### 3️⃣ Pour GitHub Pages ou cas spéciaux

Les liens GitHub Pages (*.github.io/*) sont automatiquement reconnus comme liens de déploiement.

Un cas spécial a été implémenté pour le profil GitHub : lorsque le repository est "DonovanGROUT", le lien de déploiement est automatiquement défini sur l'URL du profil GitHub.

#### 4️⃣ Pour les projets non encore déployés

Si votre projet n'est pas encore déployé, utilisez des balises de commentaire vides pour éviter les faux positifs :

```markdown
<!-- DEPLOY-LINK-START -->
<!-- Pas encore déployé -->
<!-- DEPLOY-LINK-END -->
```

Ou indiquez explicitement l'état :

```markdown
<!-- DEPLOY-LINK-START -->
<!-- En cours de développement - Déploiement prévu -->
<!-- DEPLOY-LINK-END -->
```

Cette approche garantit qu'aucun lien ne sera pris par erreur comme lien de déploiement.

## 🔧 Détection des technologies utilisées

La fonction `get_tech_stack()` détecte les technologies utilisées dans le projet :

```bash
get_tech_stack() {
  local username="$1"
  local repo_name="$2"
  
  >&2 echo "🧰 Recherche des technologies utilisées pour $repo_name..."
  
  # Cas spécial pour le profil GitHub
  if [ "$repo_name" = "DonovanGROUT" ]; then
    >&2 echo "   🌟 Cas spécial détecté: Profil GitHub"
    echo "Markdown, YAML, GitHub Actions"
    return 0
  fi
  
  # Récupérer les langages via l'API
  languages_json=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
    "https://api.github.com/repos/$username/$repo_name/languages")
  
  # Vérifier si des langages ont été trouvés
  if [ -n "$languages_json" ] && [ "$languages_json" != "{}" ] && [ "$languages_json" != "null" ]; then
    # Lister les langages (top 3 seulement)
    formatted_langs=$(echo "$languages_json" | jq -r 'keys | [.[0:3][] | select(.)] | join(", ")')
    
    if [ -n "$formatted_langs" ]; then
        >&2 echo "   💻 Technologies détectées via API: $formatted_langs"
        echo "$formatted_langs"
        return 0
    fi
  fi
  
  >&2 echo "   ⚠️ Aucune technologie détectée via l'API, recherche dans README..."
  
  # Essayer différents noms de fichiers README
  for readme_file in "README.md" "readme.md" "README.MD" "readme.MD"; do
    readme_url="https://raw.githubusercontent.com/$username/$repo_name/main/$readme_file"
    
    # Télécharger le README
    readme_content=$(curl -s "$readme_url")
    
    if [[ -n "$readme_content" ]]; then
      # Chercher des patterns de technologies dans le README
      tech_pattern="[Tt]ech[s]?:.*|[Tt]echnologies:.*|[Ss]tack:.*|[Dd]éveloppé avec:.*|[Dd]eveloped with:.*|[Bb]uilt with:.*"
      techs=$(echo "$readme_content" | grep -oP "$tech_pattern" | head -1 | sed -E 's/[Tt]ech[s]?:|[Tt]echnologies:|[Ss]tack:|[Dd]éveloppé avec:|[Dd]eveloped with:|[Bb]uilt with://g' | xargs)
      
      if [[ -n "$techs" ]]; then
        >&2 echo "   📦 Technologies trouvées dans README: $techs"
        echo "$techs"
        return 0
      fi
    fi
  done
  
  echo "Non spécifié"
}
```

Cette fonction utilise l'API GitHub pour récupérer les technologies utilisées dans le projet. Si l'API ne retourne pas de données, elle cherche dans le README du projet.

## ⭐ Affichage conditionnel des stars/forks

Les stars et forks ne s'affichent que s'ils sont supérieurs à 0 :

```python
def format_stats_line(techs, stars, forks, is_french=True):
    # Conversion des variables pour vérification
    try:
        stars_int = int(stars)
        forks_int = int(forks)
    except (ValueError, TypeError):
        stars_int = 0
        forks_int = 0
    
    # Préfixe de la ligne selon la langue
    prefix = "**Techs:**" if is_french else "**Techs:**"
    
    # Construction de la ligne de statistiques
    stats_line = f"{prefix} {techs}"
    
    # Ajout des stars uniquement si > 0
    if stars_int > 0:
        stats_line += f" | **⭐ Stars:** {stars}"
        
    # Ajout des forks uniquement si > 0
    if forks_int > 0:
        stats_line += f" | **🔀 Forks:** {forks}"
        
    return stats_line
```

## 🔄 Comment sont appliquées les modifications au README

1. Le script lit le README actuel
2. Pour chaque section, il recherche des marqueurs HTML spécifiques (comme `<!-- AUTO-UPDATE: LATEST-PROJECT-FR-START -->`)
3. Il remplace le contenu entre ces marqueurs par le nouveau contenu généré
4. Le README est sauvegardé avec les modifications

### Marqueurs HTML utilisés

```markdown
<!-- AUTO-UPDATE: LATEST-PROJECT-FR-START -->
... Contenu généré automatiquement ...
<!-- AUTO-UPDATE: LATEST-PROJECT-FR-END -->

<!-- AUTO-UPDATE: LATEST-PROJECT-EN-START -->
... Contenu généré automatiquement ...
<!-- AUTO-UPDATE: LATEST-PROJECT-EN-END -->

<!-- AUTO-UPDATE: TIMESTAMP-FR-START -->
... Horodatage généré automatiquement ...
<!-- AUTO-UPDATE: TIMESTAMP-FR-END -->

<!-- AUTO-UPDATE: TIMESTAMP-EN-START -->
... Horodatage généré automatiquement ...
<!-- AUTO-UPDATE: TIMESTAMP-EN-END -->
```

## 📝 Résumé : Comment organiser vos projets pour une détection optimale

1. **Dans votre README principal** :
   - Conservez les marqueurs HTML en place
   - Ne modifiez pas manuellement les sections auto-générées

2. **Dans les README de vos projets** :
   - Option 1 (recommandée) : Utilisez des balises de commentaire HTML pour marquer explicitement le lien de déploiement
   - Option 2 : Placez le lien près de mots-clés comme "démo", "live", "déploiement"
   - Option 3 : Pour les GitHub Pages, aucun marquage spécial n'est nécessaire

3. **Format de lien recommandé** :

   ```markdown
   <!-- DEPLOY-LINK-START -->
   ➡️ [Voir le projet déployé](https://votre-demo.com)
   <!-- DEPLOY-LINK-END -->
   ```

## ⚠️ Dépannage

Si le lien de déploiement n'est pas détecté correctement :

1. **Utilisez des balises de commentaire HTML** pour marquer explicitement le lien
2. **Vérifiez les logs du workflow** pour voir quelles priorités ont été utilisées pour la détection
3. **Pour les liens GitHub** (comme votre profil) : utilisez impérativement les balises de commentaire pour éviter la confusion avec les liens de repository
4. **Testez localement** la fonction de détection en la modifiant au besoin

## 🚀 Personnalisation avancée

Pour personnaliser davantage la logique de détection :

1. Modifiez l'ordre des priorités dans la fonction `get_deployment_link()`
2. Ajoutez des mots-clés supplémentaires pour la détection contextuelle
3. Créez des règles spécifiques pour vos cas d'utilisation particuliers
