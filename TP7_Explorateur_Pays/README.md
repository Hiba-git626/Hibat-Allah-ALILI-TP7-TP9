# TP 7 — Explorateur d’API REST

Application Vue 3 avec une liste de pays chargée depuis REST Countries, recherche insensible à la casse et fiche détaillée par pays.

## Démarrage

```bash
npm install
npm run dev
```

La liste complète et les fiches sont récupérées directement depuis l’API publique REST Countries. Aucune clé n’est nécessaire.

## Fonctionnalités

- Chargement asynchrone avec états de chargement, erreur et nouvelle tentative.
- Filtrage des pays par nom, sans tenir compte de la casse.
- Route dédiée `/pays/:code` et chargement indépendant de la fiche.
- Gestion de la réponse détail sous forme de tableau ou d’objet.
- Informations : drapeau, capitale, population, superficie, langues, monnaie, frontières et lien cartographique.
