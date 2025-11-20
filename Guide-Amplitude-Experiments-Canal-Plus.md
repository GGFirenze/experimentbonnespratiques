# Guide de Bonnes Pratiques : Amplitude Experiments
## Résolution des Problèmes de Cohérence des Populations

**Destiné à :** Équipe Canal+ - Test A/B Page de Contact Service Client  
**Date :** Novembre 2025

---

## 🔍 Résumé du Problème

Vous avez observé des incohérences entre trois sources de données :
- **Amp** (export Amplitude) : 94% présents dans Nav
- **Exp** (colonne Experiments) : 94% présents dans Nav  
- **Cohérence Amp/Exp** : Seulement 86%
- **Incohérence** : 3,5%
- **Manquants** : 0,5% dans Nav mais absents de Amp et Exp

Ces écarts révèlent **4 problèmes majeurs** dans votre configuration d'expérience qui compromettent la fiabilité des résultats.

---

## 🔴 Problème #1 : Bucketing par `amplitude_id` (CRITIQUE)

### Ce Que Vous Faites Actuellement

```json
"bucketingKey": "amplitude_id"
```

### Pourquoi C'est Problématique

L'`amplitude_id` est un identifiant **par appareil/navigateur**, pas par utilisateur :

```
Utilisateur "marie@canal.fr" :
├─ Ordinateur portable : amplitude_id = abc123 → Affecté au CONTRÔLE
├─ Téléphone mobile    : amplitude_id = xyz789 → Affecté au TRAITEMENT
├─ Efface ses cookies  : amplitude_id = def456 → NOUVEAU variant
└─ Résultat : Même personne voit plusieurs variants !
```

**Conséquences directes :**
- ✗ **Contamination** : Les utilisateurs voient les deux variants
- ✗ **Dilution de l'effet** : L'impact du traitement est moyenné
- ✗ **Incohérence avec votre entrepôt de données** : Votre DWH utilise probablement `user_id` (ID abonné), créant un désalignement structurel
- ✗ **86% de cohérence** : Explique directement pourquoi Amp et Exp ne correspondent pas

### ✅ Solution

```json
"bucketingKey": "user_id"
```

**Utilisez l'ID utilisateur unique** (ID abonné Canal+) qui reste constant :
- ✅ Multi-appareils (même variant partout)
- ✅ Persistant après suppression des cookies
- ✅ Aligné avec votre entrepôt de données
- ✅ Cohérence à 100% entre les systèmes

---

## 🔴 Problème #2 : Logique d'Exposition Trop Complexe

### Ce Que Vous Faites Actuellement

```json
"exposureEvents": [{
  "event_type": "view page",
  "filters": [
    {"subprop_key": "gp:app zone", "subprop_type": "user"},
    {"subprop_key": "gp:app users", "subprop_type": "user"},
    {"subprop_key": "gp:subscriber status", "subprop_type": "user"},
    {"subprop_key": "platform test", "subprop_type": "derivedV2"},
    {"subprop_key": "gp:preferences analytics", "subprop_type": "user"},
    {"subprop_key": "page title", "subprop_type": "event"}
  ]
}]
```

**6 filtres** dont 5 propriétés utilisateur !

### Pourquoi C'est Problématique

**Problème de timing :**

```
10:00:00.000 - L'utilisateur visite la page
10:00:00.100 - Événement "view page" envoyé
              ├─ "page title" est présent ✓
              ├─ "gp:app zone" pas encore défini ✗
              ├─ "gp:subscriber status" sera défini à 10:00:00.500 ✗
10:00:01.000 - Amplitude évalue l'exposition → CERTAINS FILTRES ÉCHOUENT

Pendant ce temps...
Votre DWH évalue les propriétés APRÈS qu'elles soient toutes définies → MISMATCH
```

**Conséquences :**
- ✗ **94% de chevauchement** : 6% échouent à cause de problèmes de timing
- ✗ **0,5% manquants** : Visitent les pages mais propriétés non définies à temps
- ✗ **Incohérence** : Les deux systèmes évaluent à des moments différents

### ✅ Solution

**Séparez le ciblage de l'exposition :**

```json
{
  "targetSegments": [{
    "conditions": [
      {"user_property": "gp:app zone", "is": "FRANCE"},
      {"user_property": "gp:app users", "is": "Customer Care"},
      {"user_property": "gp:subscriber status", "is": "Subscriber"},
      {"user_property": "platform test", "is": "Web"},
      {"user_property": "gp:preferences analytics", "is": "Optin"}
    ]
  }],
  
  "exposureEvents": [{
    "event_type": "view page",
    "filters": [
      {
        "subprop_key": "page title",
        "subprop_type": "event",
        "subprop_value": [
          "Customer Care - Contact - Manage my subscription - With commitment",
          "Customer Care - Contact - Solve my technical problems - With commitment",
          "Customer Care - Contact - Cancel my subscription - With commitment"
        ]
      }
    ]
  }]
}
```

**Avantages :**
- ✅ **Target Segments** : Évalués sur les propriétés utilisateur les plus récentes
- ✅ **Exposure Events** : Ne vérifient que les propriétés d'événement (toujours disponibles)
- ✅ Séparation claire : "Qui qualifie" vs "Quand sont-ils exposés"
- ✅ Cohérence parfaite entre les systèmes

---

## 🔴 Problème #3 : Pas d'Utilisation du SDK Experiment

### Ce Que Vous Faites Actuellement

Exposition basée sur des filtres complexes appliqués rétroactivement.

### Pourquoi C'est Problématique

Sans le SDK Experiment d'Amplitude :
- ✗ Pas de garantie de cohérence d'évaluation
- ✗ Timing différent entre Amplitude et votre système
- ✗ Logique dupliquée = risque de divergence

### ✅ Solution : Utilisez le SDK Experiment

**Côté client (JavaScript) :**

```javascript
// 1. Initialiser le SDK Experiment
import { Experiment } from '@amplitude/experiment-js-client';

const experiment = Experiment.initializeWithAmplitudeAnalytics(
  'VOTRE_DEPLOYMENT_KEY',
  {
    // SDK Analytics déjà initialisé avec user_id
  }
);

// 2. Récupérer l'affectation de variant
const user = {
  user_id: 'user123',  // ID abonné Canal+
  user_properties: {
    'gp:app zone': 'FRANCE',
    'gp:subscriber status': 'Subscriber',
    // ... autres propriétés
  }
};

const variants = await experiment.fetch(user);
const variant = variants['ab_mon_55-customer-care-contact-page-oae'];

// 3. Amplitude track automatiquement [Experiment] Exposure
// Les deux systèmes utilisent maintenant la MÊME logique au MÊME moment

// 4. Appliquer le variant
if (variant.value === 'treatment') {
  // Afficher la version traitement
} else {
  // Afficher la version contrôle
}
```

**Configuration Amplitude simplifiée :**

```json
{
  "exposureEvents": [{
    "event_type": "[Experiment] Exposure",
    "filters": []  // Aucun filtre nécessaire !
  }]
}
```

**Avantages :**
- ✅ Exposition automatique et cohérente
- ✅ Un seul système d'affectation (Amplitude)
- ✅ Pas de désalignement temporel
- ✅ Cohérence garantie à 100%

---

## 🔴 Problème #4 : T-Test au Lieu de Sequential Testing

### Ce Que Vous Faites Actuellement

```json
"statisticalMethod": "tTest"
```

### Pourquoi C'est Problématique

Le **t-test** nécessite une taille d'échantillon fixe déterminée à l'avance :
- ✗ Chaque fois que vous consultez les résultats avant la fin = "peeking"
- ✗ Chaque peek augmente le taux de faux positifs (loin au-dessus de 5%)
- ✗ Avec plusieurs peeks, l'alpha réel peut atteindre 15-30%+
- ✗ Risque de déclarer un gagnant alors qu'il n'y en a pas

### ✅ Solution

```json
"statisticalMethod": "sequential"
```

**Le test séquentiel permet :**
- ✅ Consultation des résultats à tout moment
- ✅ Contrôle du taux de faux positifs
- ✅ Arrêt anticipé si un variant gagne clairement
- ✅ Analyse continue sans compromettre la validité statistique

---

## 📊 Problème Bonus : Exports Inutiles

### Ce Que Vous Pensez Devoir Faire

> "Nous devons exporter d'Amplitude les populations A et B, ce qui nécessite une validation juridique (RGPD)."

### Pourquoi C'est une Fausse Contrainte

**Vous n'avez PAS besoin d'exporter !**

Au lieu d'exporter des données **HORS** d'Amplitude, envoyez des données **VERS** Amplitude.

### ✅ Solution : Envoyez Tous les Événements à Amplitude

**Pour les appels au service client :**

```javascript
// Backend - Système de centre d'appels
if (user.hasAnalyticsConsent) {  // Vérifier le flag de consentement
  amplitude.track('Customer Service Call Received', {
    call_duration: 180,
    issue_resolved: true,
    channel: 'phone',
    issue_type: 'subscription'
  }, {
    user_id: user.id  // Même user_id utilisé pour le bucketing
  });
}
```

**Pour les interactions chatbot :**

```javascript
// Backend - Système de chatbot
if (user.hasAnalyticsConsent) {
  amplitude.track('Chatbot Session Completed', {
    messages_sent: 5,
    issue_resolved: false,
    session_duration: 120
  }, {
    user_id: user.id
  });
}
```

**Pour les achats :**

```javascript
// Backend - Système de facturation
if (user.hasAnalyticsConsent) {
  amplitude.track('Purchase Completed', {
    amount: 49.99,
    subscription_tier: 'premium',
    payment_method: 'card'
  }, {
    user_id: user.id
  });
}
```

**Ensuite, utilisez ces événements comme métriques d'expérience directement dans Amplitude.**

### Avantages

- ✅ **Pas d'exports** → Pas de validation juridique nécessaire
- ✅ **Analyse en temps réel** dans Amplitude
- ✅ **Cohérence parfaite** des populations (100%)
- ✅ **Toutes les données au même endroit**
- ✅ **Métriques automatiques** : taux d'appels, durée, résolution, etc.

### Gestion du Consentement

Le consentement est déjà géré :

```javascript
// Quand l'utilisateur donne son consentement sur le web
amplitude.setUserProperties({
  'gp:preferences analytics': 'Optin'
});

// Stocker aussi dans votre base de données utilisateur
database.updateUser(userId, { analytics_consent: true });

// Tous vos systèmes backend vérifient ce flag
if (user.analytics_consent) {
  // Envoyer des événements à Amplitude
}
```

**Le consentement s'applique à TOUTES les données de l'utilisateur sur TOUS les canaux.**

---

## ✅ Configuration Recommandée Complète

```json
{
  "key": "ab_mon_55-customer-care-contact-page-oae",
  "bucketingKey": "user_id",
  "rolloutWeights": {
    "control": 1,
    "treatment": 1
  },
  
  "targetSegments": [{
    "conditions": [
      {"user_property": "gp:app zone", "is": "FRANCE"},
      {"user_property": "gp:app users", "is": "Customer Care"},
      {"user_property": "gp:subscriber status", "is": "Subscriber"},
      {"user_property": "platform test", "is": "Web"},
      {"user_property": "gp:preferences analytics", "is": "Optin"}
    ]
  }],
  
  "analysisParams": {
    "statisticalMethod": "sequential",
    "exposureAttribution": "FIRST",
    "exposureEvents": [{
      "event_type": "[Experiment] Exposure",
      "filters": []
    }]
  },
  
  "metrics": [
    {
      "event_type": "Customer Service Call Received",
      "analysisParams": {
        "testDirection": "smaller",
        "metricGoalType": "primary"
      }
    },
    {
      "event_type": "Purchase Completed",
      "analysisParams": {
        "testDirection": "greater",
        "metricGoalType": "secondary"
      }
    }
  ]
}
```

---

## 🎯 Plan d'Action

### Pour l'Expérience Actuelle

**Recommandation : Ne pas faire confiance aux résultats actuels.**

Les 4 problèmes combinés compromettent la validité :
- ❌ Mauvaise clé de bucketing (contamination des variants)
- ❌ Incohérences de population entre systèmes  
- ❌ Logique d'exposition incohérente
- ❌ Analyse statistique potentiellement invalide

### Pour la Prochaine Expérience

**Liste de contrôle :**

1. ✅ **Changer `bucketingKey` à `"user_id"`**
   - Utilisez l'ID abonné Canal+ unique
   - Garantit un variant par utilisateur, tous appareils confondus

2. ✅ **Déplacer les filtres de propriétés utilisateur vers `targetSegments`**
   - Ne gardez que les propriétés d'événement dans `exposureEvents`
   - Évite les problèmes de timing

3. ✅ **Implémenter le SDK Experiment d'Amplitude**
   - Utiliser `experiment.fetch()` côté client
   - Exposition automatique via `[Experiment] Exposure`
   - Garantit la cohérence

4. ✅ **Passer à `"statisticalMethod": "sequential"`**
   - Permet la consultation continue des résultats
   - Validité statistique maintenue

5. ✅ **Envoyer les événements backend à Amplitude**
   - Appels service client
   - Sessions chatbot
   - Achats et abonnements
   - Utiliser comme métriques d'expérience

6. ✅ **Vérifier le consentement dans tous les systèmes**
   - Même flag `analytics_consent` partout
   - Envoyer uniquement pour les utilisateurs consentants

---

## 📋 Exemple d'Implémentation Complète

### 1. Configuration Frontend

```javascript
// app.js - Initialisation avec consentement
import * as amplitude from '@amplitude/analytics-browser';
import { Experiment } from '@amplitude/experiment-js-client';

// Attendre le consentement utilisateur
if (userConsent === 'Optin') {
  
  // Initialiser Analytics
  amplitude.init('VOTRE_API_KEY', {
    userId: user.subscriberId,  // ID abonné Canal+
    defaultTracking: {
      pageViews: true,
      sessions: true
    }
  });
  
  // Définir les propriétés utilisateur
  amplitude.setUserId(user.subscriberId);
  amplitude.setUserProperties({
    'gp:app zone': 'FRANCE',
    'gp:app users': 'Customer Care',
    'gp:subscriber status': 'Subscriber',
    'gp:preferences analytics': 'Optin'
  });
  
  // Initialiser Experiment SDK
  const experiment = Experiment.initializeWithAmplitudeAnalytics(
    'VOTRE_DEPLOYMENT_KEY'
  );
  
  // Récupérer l'affectation de variant
  const variants = await experiment.fetch({
    user_id: user.subscriberId,
    user_properties: {
      'gp:app zone': 'FRANCE',
      'gp:subscriber status': 'Subscriber'
      // ... autres propriétés
    }
  });
  
  const contactPageVariant = variants['ab_mon_55-customer-care-contact-page-oae'];
  
  // Appliquer le variant
  if (contactPageVariant.value === 'treatment') {
    // Afficher la version traitement de la page de contact
    showTreatmentContactPage();
  } else {
    // Afficher la version contrôle
    showControlContactPage();
  }
  
  // L'événement [Experiment] Exposure est automatiquement tracké
}
```

### 2. Configuration Backend

```javascript
// call-center-service.js
const Amplitude = require('@amplitude/node');

const amplitude = Amplitude.init('VOTRE_API_KEY');

async function handleCustomerCall(userId, callData) {
  // Récupérer l'utilisateur depuis la base de données
  const user = await database.getUser(userId);
  
  // Vérifier le consentement
  if (user.analytics_consent) {
    amplitude.track({
      event_type: 'Customer Service Call Received',
      user_id: user.subscriber_id,  // Même ID que le frontend
      event_properties: {
        call_duration: callData.duration,
        issue_resolved: callData.resolved,
        issue_type: callData.issueType,
        channel: 'phone',
        agent_id: callData.agentId
      }
    });
  }
  
  // Continuer le traitement de l'appel...
}
```

```javascript
// chatbot-service.js
async function endChatbotSession(userId, sessionData) {
  const user = await database.getUser(userId);
  
  if (user.analytics_consent) {
    amplitude.track({
      event_type: 'Chatbot Session Completed',
      user_id: user.subscriber_id,
      event_properties: {
        messages_sent: sessionData.messageCount,
        issue_resolved: sessionData.resolved,
        session_duration: sessionData.duration,
        topics: sessionData.topics
      }
    });
  }
}
```

```javascript
// billing-service.js
async function processPurchase(userId, purchaseData) {
  const user = await database.getUser(userId);
  
  if (user.analytics_consent) {
    amplitude.track({
      event_type: 'Purchase Completed',
      user_id: user.subscriber_id,
      event_properties: {
        amount: purchaseData.amount,
        subscription_tier: purchaseData.tier,
        payment_method: purchaseData.paymentMethod,
        is_upgrade: purchaseData.isUpgrade
      }
    });
  }
  
  // Continuer le traitement du paiement...
}
```

---

## 🎓 Bonnes Pratiques Générales

### 1. Bucketing

- ✅ **Toujours utiliser `user_id`** pour les utilisateurs connectés
- ✅ Utiliser `amplitude_id` uniquement pour les expériences pré-connexion
- ✅ S'assurer que l'ID est le même sur tous les canaux

### 2. Exposition

- ✅ **Utiliser le SDK Experiment** pour l'affectation automatique
- ✅ Laisser Amplitude tracker `[Experiment] Exposure` automatiquement
- ✅ Ne PAS dupliquer la logique d'affectation

### 3. Ciblage

- ✅ **Propriétés utilisateur** → `targetSegments`
- ✅ **Propriétés d'événement** → `exposureEvents.filters`
- ✅ Éviter les propriétés dérivées dans les filtres d'exposition

### 4. Méthode Statistique

- ✅ **Utiliser `sequential`** pour consultation continue
- ✅ Définir un seuil de puissance (80-90%)
- ✅ Définir un effet détectable minimum (MDE)

### 5. Données Multi-Canaux

- ✅ **Envoyer VERS Amplitude** (pas exporter DEPUIS)
- ✅ Vérifier le consentement dans chaque système
- ✅ Utiliser le même `user_id` partout
- ✅ Enrichir les événements avec du contexte

### 6. RGPD/Consentement

- ✅ Stocker le consentement dans la base de données utilisateur
- ✅ Vérifier avant d'envoyer chaque événement
- ✅ Ne PAS initialiser les SDKs avant le consentement
- ✅ Le consentement s'applique à tous les canaux

---

## 📞 Support

Pour toute question sur cette implémentation :

- 📧 **Documentation Amplitude** : [docs.amplitude.com](https://docs.amplitude.com)
- 📖 **Experiment SDK Guide** : [docs.amplitude.com/experiment](https://docs.amplitude.com/experiment)
- 💬 **Amplitude Community** : [community.amplitude.com](https://community.amplitude.com)

---

## 📝 Résumé Exécutif

### Problèmes Identifiés

| Problème | Impact | Priorité |
|----------|--------|----------|
| `bucketingKey: amplitude_id` | Contamination des variants, 86% cohérence | 🔴 CRITIQUE |
| Filtres de propriétés utilisateur dans exposition | 94% overlap, désalignement temporel | 🔴 HAUTE |
| Pas de SDK Experiment | Incohérences d'évaluation | 🔴 HAUTE |
| T-test au lieu de sequential | Invalidité statistique si "peeking" | 🟡 MOYENNE |
| Exports de données | Complexité inutile, délais RGPD | 🟡 MOYENNE |

### Solutions

1. ✅ Changer à `bucketingKey: "user_id"`
2. ✅ Déplacer propriétés utilisateur vers `targetSegments`
3. ✅ Implémenter Experiment SDK
4. ✅ Passer à `statisticalMethod: "sequential"`
5. ✅ Envoyer événements backend à Amplitude (pas d'exports)

### Résultat Attendu

- ✅ **Cohérence à 100%** entre tous les systèmes
- ✅ **Variants corrects** pour chaque utilisateur
- ✅ **Données fiables** pour la prise de décision
- ✅ **Pas de problèmes RGPD** liés aux exports
- ✅ **Analyse en temps réel** de toutes les métriques

---

**Document préparé pour Canal+ - Novembre 2025**  
**Amplitude Solutions Engineering**

