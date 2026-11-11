# Brightspace Agenda — intégration Home Assistant

Affiche les prochains événements de [Brightspace Agenda](https://github.com/MrTh0m/Brightspace_agenda) dans un tableau de bord Home Assistant.

## Prérequis

- Home Assistant 2024.1+
- [HACS](https://hacs.xyz/) installé
- Brightspace Agenda déployé avec le **partage activé** (icône Partager dans l'app > Copier le lien)

## Installation

### 1 — Ajouter le dépôt dans HACS

HACS → Menu (⋮) → **Dépôts personnalisés** → coller `https://github.com/MrTh0m/Brightspace_agenda_HACS` → catégorie **Intégration** → Ajouter.

Puis : HACS → Intégrations → **Brightspace Agenda** → Télécharger → Redémarrer HA.

La card Lovelace est copiée automatiquement dans `www/` lors de l'installation.

### 2 — Configurer l'intégration

Paramètres → Appareils & Services → **+ Ajouter une intégration** → rechercher *Brightspace Agenda*.

| Champ | Exemple |
|---|---|
| URL de partage | `https://votre-serveur/index.html?share=abc123...` |
| Nom affiché | `Brightspace Agenda` |

L'URL est celle générée par l'application (icône Partager > Copier le lien). L'intégration en extrait automatiquement l'adresse du serveur et le token.

### 3 — Ajouter la card au tableau de bord

Éditer un dashboard → **+ Ajouter une carte** → chercher *Brightspace* dans les cartes personnalisées.

Ou en YAML manuel :

```yaml
type: custom:brightspace-agenda-card
entity: sensor.brightspace_agenda_evenements_a_venir
app_url: https://votre-serveur/    # optionnel — lien "Ouvrir l'agenda"
stats: [devoirs, individuels, collectifs, urgents]   # optionnel — tuiles affichées
```

Tuiles disponibles (`stats`) :

| Clé | Tuile | Contenu |
|---|---|---|
| `devoirs` | Devoirs | Devoirs non rendus dans la fenêtre (= Individuels + Collectifs) |
| `individuels` | Individuels | Devoirs individuels non rendus |
| `collectifs` | Collectifs | Devoirs collectifs non rendus |
| `urgents` | Urgents ≤7j | Devoirs non rendus à échéance dans 7 jours ou moins |
| `a_venir` | Événements | Tous les événements de la fenêtre (devoirs, sessions, ateliers) |

Les types individuel / collectif suivent ceux de l'application, y compris les types corrigés à la main (app v43+).

## Options

Paramètres → Appareils & Services → Brightspace Agenda → **Configurer** :

| Option | Défaut | Description |
|---|---|---|
| `limit` | 5 | Nombre d'événements affichés (1–20) |
| `days` | 14 | Horizon en jours (1–60) |
| `scan_interval` | 600 | Intervalle de polling en secondes |

## Sensor exposé

`sensor.brightspace_agenda_evenements_a_venir`

| Attribut | Type | Description |
|---|---|---|
| `state` | int | Nombre d'événements à venir (tous types) |
| `a_venir` | int | Identique à `state` |
| `devoirs` | int | Devoirs non rendus (présent si `api.php` est à jour) |
| `individuels` | int | Devoirs individuels non rendus |
| `collectifs` | int | Devoirs collectifs non rendus |
| `urgents` | int | Devoirs non rendus à échéance ≤ 7 jours |
| `events` | list | Liste des événements |
| `generated_at` | ISO 8601 | Horodatage du dernier refresh |

Structure d'un événement :

```json
{
  "uid":        "abc123",
  "summary":    "Rendu stratégie digitale",
  "type":       "deadline",
  "devoir_type": "individuel",
  "start_iso":  "2026-06-25T23:59:00+02:00",
  "end_iso":    "2026-06-25T23:59:00+02:00",
  "days_until": 2,
  "subject":    "Marketing",
  "rendu":      false
}
```

Types : `deadline`, `session`, `workshop`, `event`.

`devoir_type` vaut `individuel` ou `collectif` pour un devoir, `null` sinon. `days_until` est en jours calendaires (0 = aujourd'hui, 1 = demain).

## Automatisation exemple

Notification si une deadline tombe dans moins de 2 jours :

```yaml
alias: Alerte deadline Brightspace
trigger:
  - platform: template
    value_template: >
      {% set events = state_attr('sensor.brightspace_agenda_evenements_a_venir', 'events') %}
      {% if events %}
        {{ events | selectattr('type', 'eq', 'deadline')
                  | selectattr('days_until', 'le', 1)
                  | list | count > 0 }}
      {% endif %}
action:
  - service: notify.mobile_app
    data:
      title: "Deadline imminente — Brightspace"
      message: >
        {% set ev = state_attr('sensor.brightspace_agenda_evenements_a_venir', 'events')
                    | selectattr('type', 'eq', 'deadline')
                    | selectattr('days_until', 'le', 1)
                    | first %}
        {{ ev.summary }} — {{ ev.start_iso[:10] }}
```

## Notes de version

### 1.3.0 · 11 novembre 2026

Nécessite un `api.php` de Brightspace Agenda du 11/11/2026 ou plus récent pour les nouveautés ; reste compatible avec un `api.php` plus ancien.

- **Correctif** : le bouton Actualiser de la card ne faisait rien (l'écouteur était branché sur un bouton aussitôt recréé par le rendu)
- **Correctif** : les badges `J-n`, « Aujourd'hui » et « Demain » sont calculés en jours calendaires ; un événement du lendemain matin pouvait s'afficher « Aujourd'hui » la veille au soir
- **Correctif** : les tuiles Individuels / Collectifs reflètent les vrais types de devoirs (correction côté `api.php`), types corrigés à la main inclus
- Nouvelle tuile **Devoirs** (devoirs non rendus), par défaut à la place de « À venir » ; l'ancienne clé `a_venir` reste utilisable et s'affiche désormais **Événements**, ce qu'elle compte réellement
- Pastille **Individuel / Collectif** sur chaque devoir de la liste
- Nouvel attribut `devoirs` sur le sensor
- Textes du calendrier et de la configuration échappés avant affichage ; `app_url` limité aux liens http(s)
- La card n'est plus reconstruite à chaque changement d'état de Home Assistant, seulement quand le sensor change
- « Il y a 600 min » devient « Il y a 10 h »
