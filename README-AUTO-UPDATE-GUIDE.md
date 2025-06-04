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
  
  echo "🔍 Recherche du lien de déploiement pour $repo_name..."
  
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
            echo "   🎯 Lien trouvé via balise $pattern: $deploy_link"
            echo "$deploy_link"
            return 0
          fi
        fi
      done
      
      # PRIORITÉ 2 : Chercher les liens en contexte (après des mots-clés spécifiques)
      for keyword in "démo" "demo" "live" "deployment" "déploiement" "deployed" "déployé" "site" "app" "application"; do
        if deploy_link=$(echo "$readme_content" | grep -i -B1 -A2 "$keyword" | grep -oE 'https?://[^[:space:]")\`]+' | grep -v 'github.com/[^/]+/[^/]+$' | head -1); then
          if [[ -n "$deploy_link" ]]; then
            echo "   🔎 Lien trouvé via mot-clé '$keyword': $deploy_link"
            echo "$deploy_link"
            return 0
          fi
        fi
      done
      
      # PRIORITÉ 3 : Accepter les liens GitHub spécifiques (ex: pages GitHub)
      if deploy_link=$(echo "$readme_content" | grep -oE 'https?://[^[:space:]")\`]+\.github\.io/[^[:space:]")\`]+'); then
        echo "   🔎 Lien GitHub Pages trouvé: $deploy_link"
        echo "$deploy_link"
        return 0
      fi
      
      # PRIORITÉ 4 : Premier lien externe non GitHub repository
      if deploy_link=$(echo "$readme_content" | grep -oE 'https?://[^[:space:]")\`]+' | grep -v 'github.com/[^/]+/[^/]+$' | head -1); then
        if [[ -n "$deploy_link" ]]; then
          echo "   📎 Premier lien externe trouvé: $deploy_link"
          echo "$deploy_link"
          return 0
        fi
      fi
      
      # PRIORITÉ 5 : Accepter n'importe quel lien si rien d'autre n'est trouvé
      if deploy_link=$(echo "$readme_content" | grep -oE 'https?://[^[:space:]")\`]+' | head -1); then
        echo "   ⚠️ Aucun lien spécifique trouvé, utilisation du premier lien: $deploy_link"
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

Si votre lien de déploiement est lui-même un lien GitHub (comme dans le cas de votre profil), utilisez une des méthodes ci-dessus pour le marquer explicitement.

## ⭐ Affichage conditionnel des stars/forks

Les stars et forks ne s'affichent que s'ils sont supérieurs à 0 :

```python
def format_stats_line(lang, stars, forks, is_french=True):
    # Conversion des variables pour vérification
    try:
        stars_int = int(stars)
        forks_int = int(forks)
    except (ValueError, TypeError):
        stars_int = 0
        forks_int = 0
    
    # Préfixe de la ligne selon la langue
    prefix = "**Langage principal:**" if is_french else "**Main language:**"
    
    # Construction de la ligne de statistiques
    stats_line = f"{prefix} {lang}"
    
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
