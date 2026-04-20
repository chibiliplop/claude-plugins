---
name: generate-architecture-doc
description: Génère une documentation d'architecture technique (ARCHITECTURE.md) complète, standardisée, visuelle et factuelle en utilisant le LSP et des subagents.
---

# Skill: Generate Architecture Doc

## But
Produire un document `ARCHITECTURE.md` agissant comme une Single Source of Truth (SSOT) pour les humains et les IA, basé sur un template strict avec diagrammes Mermaid.

## Règle d'or absolue : Zéro Omission
Chaque composant, fichier ou preuve mentionné par ce système DOIT utiliser son **chemin relatif complet** depuis la racine du projet. 
Les raccourcis (`...`), impressions, ou noms de fichiers isolés (ex: `auth.ts` au lieu de `backend/src/api/auth.ts`) sont **STRICTEMENT INTERDITS**.

## Workflow Stratégique (MapReduce)
1. **Initialisation (Orchestrateur) :**
   - Analyser la racine du projet pour détecter les langages et la stack globale.
   - Identifier les domaines/modules majeurs (ex: `apps/*`, `packages/*`, `services/*`).

2. **Phase d'Exploration (Subagents `repo-explorer`) :**
   - Lancer une instance de `repo-explorer` par module majeur identifié.
   - **Priorité LSP :** Utiliser les outils LSP pour cartographier les dépendances de manière économique (sans lire tout le code).
   - Extraire les configurations d'infrastructure, de bases de données et de sécurité.

3. **Phase de Rédaction (Subagent `architecture-writer`) :**
   - Fournir tous les rapports des `repo-explorer` au rédacteur.
   - Le rédacteur compile les données dans le template imposé.
   - Le rédacteur génère le diagramme global de dépendances en syntaxe Mermaid.

## Résultat Attendu
Un fichier `ARCHITECTURE.md` à la racine, prêt pour l'onboarding, où chaque chemin cité peut être lu directement via une commande `cat [chemin_cite]`.