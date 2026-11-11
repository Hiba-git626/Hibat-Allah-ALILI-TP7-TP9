# TP 9 — API météo

Application Vue 3 qui recherche une ville, affiche la météo actuelle et regroupe les prévisions OpenWeatherMap par jour.

## Démarrage

1. Créez un compte OpenWeatherMap et récupérez une clé API personnelle.
2. Copiez `.env.example` vers `.env` à la racine du projet.
3. Remplacez `votre_cle_openweathermap` par votre clé dans `.env`.
4. Lancez :

```bash
npm install
npm run dev
```

Vite charge la variable `VITE_OWM_API_KEY` depuis `.env`. Redémarrez le serveur après avoir créé ou modifié ce fichier. Le fichier `.env` est ignoré par Git : ne publiez jamais votre vraie clé.

## Fonctionnalités

- Recherche différée de 400 ms, avec annulation de la recherche précédente.
- Suggestions de géocodage et messages distincts pour clé invalide, quota dépassé et ville absente.
- Sélection d’une ville, historique des cinq dernières recherches enregistré dans `localStorage`.
- Météo actuelle et prévisions lancées en parallèle.
- Bascule Celsius/Fahrenheit qui recharge les valeurs via le paramètre `units` de l’API.
- Prévisions regroupées par jour avec températures minimale et maximale.
- Bonus : géolocalisation du navigateur.

L’API OpenWeatherMap peut prendre un peu de temps à activer une clé nouvellement créée.
