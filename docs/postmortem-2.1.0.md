# Postmortem : la version 2.1.0 arrêtée par l'analyse automatique

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 15/10/2026, entre 09:14 et 09:26 UTC |
| Version en cause | TaskFlow 2.1.0 (`ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0`) |
| PR à l'origine | PR #26 `deploy: taskflow 2.1.0` (changement du numéro d'image dans `apps/taskflow/rollout.yaml`) |
| Durée d'exposition | Environ une minute, de 09:16 (le canary reçoit du trafic) à 09:17 (abort automatique) |
| Part du trafic touché | 25 % au plus : un seul pod sur quatre était en 2.1.0 |
| Détecté par | Le test de charge k6 lancé par l'`AnalysisRun` du Rollout, pas par une personne ni par une probe |
| Résolu par | Abort automatique du Rollout (retour immédiat des 4 pods en 2.0.0), puis `git revert` de la PR #26 par la PR #27 pour remettre Git d'accord |

## Résumé

La 2.1.0 a été déployée en canary par une PR normale, relue et approuvée. Dès le palier de 25 %, l'analyse automatique a lancé un test de charge sur le seul pod en 2.1.0. Le test a mesuré **33 % de requêtes en erreur** (seuil : 2 %) et un **p95 de 311 ms** (seuil : 250 ms). L'analyse a échoué, et le Rollout a annulé la mise à jour tout seul. Les utilisateurs n'ont été exposés qu'environ une minute, et seulement via un pod sur quatre. Git est ensuite resté sur 2.1.0 pendant environ neuf minutes, jusqu'au revert.

## Chronologie

Heures UTC, relevées dans l'historique GitHub, l'historique d'Argo CD et les événements Kubernetes.

| Heure | Événement |
| --- | --- |
| 09:14 | Merge de la PR #26 : l'image passe de 2.0.0 à 2.1.0 |
| 09:16 | Argo CD déploie la révision 6 ; un pod en 2.1.0 est créé (25 %) ; le Rollout se met en pause 20 s (`RolloutPaused`) |
| 09:16 | Le Rollout reprend et lance l'`AnalysisRun` `taskflow-df976ccb5-6-2` : un Job k6 vise `taskflow-canary` |
| 09:17 | k6 termine : les deux seuils sont franchis. `MetricFailed`, `AnalysisRunFailed`, `RolloutAborted` ; le pod 2.1.0 est supprimé, la 2.0.0 reprend 4 pods |
| 09:26 | Merge de la PR #27 (revert de la #26) ; Argo CD synchronise quelques secondes plus tard ; le Rollout redevient `Healthy` en 2.0.0 |
| 09:34 | Livraison de la 2.2.0 : l'analyse réussit, promotion automatique jusqu'à 100 % |

## Composant défaillant et cause racine

**Quel composant a échoué ?** L'application TaskFlow dans sa version 2.1.0. Preuve, extrait des logs du Job k6 (`kubectl -n taskflow logs job/<nom-du-job>`) :

```
http_req_duration  ✗ 'p(95)<250'  p(95)=311.52ms
http_req_failed    ✗ 'rate<0.02'  rate=33.00%
```

Le message de l'analyse (`kubectl -n taskflow describe analysisrun taskflow-df976ccb5-6-2`) confirme : `Metric "test-de-charge-k6" assessed Failed due to failed (1) > failureLimit (0)`. La 2.0.0 répondait en 4 à 13 ms sans erreur, la 2.2.0 en 6,6 ms sans erreur : le défaut est propre à la 2.1.0.

**D'où vient l'erreur ?** Le code de l'application (`/app/app/main.py` dans l'image) contient un intercepteur `inject_faults`, prévu pour simuler une version défaillante. Pour toute route autre que `/health`, il attend `LATENCY_MS` millisecondes, puis répond une erreur 500 avec la probabilité `FAILURE_RATE`. Ces réglages sont fixés dans chaque image (`docker image inspect`) :

| Version | `FAILURE_RATE` | `LATENCY_MS` |
| --- | --- | --- |
| 2.0.0 | 0 | 0 |
| **2.1.0** | **0,3** | **300** |
| 2.2.0 | 0 | 0 |

Les mesures correspondent : 30 % d'erreurs réglées pour 33 % observées (99 sur 300), et 300 ms de retard pour une moyenne de 305 ms et un p95 de 311 ms.

**Pourquoi les probes Kubernetes ne l'ont-elles pas vu ?** Les sondes (`readinessProbe` et `livenessProbe`) interrogent uniquement `/health`, et le code exclut justement cette route du retard et des erreurs (`if request.url.path != "/health"`). Le pod 2.1.0 répondait donc toujours vite et correctement aux sondes, et Kubernetes le jugeait sain, alors que toutes les autres routes étaient lentes et plantaient une fois sur trois. Une sonde dit « le programme tourne », pas « le programme rend le service attendu ».

**Cause racine.** Dans cet exercice, le défaut est volontaire : l'image 2.1.0 a été construite avec des réglages de panne (`FAILURE_RATE=0.3`, `LATENCY_MS=300`). Ce n'est pas un vrai bug de code que nous aurions introduit. Dans un cas réel, la même chaîne de causes s'appliquerait à une vraie régression. Ce qui l'a laissée atteindre le cluster : la PR ne changeait que le numéro d'image, donc la relecture ne pouvait pas voir ce qui se trouve dans l'image, et aucun test sous charge n'existait avant le déploiement, hormis la sonde `/health`, qui est aveugle à ce défaut.

## Ce qui a bien fonctionné

- L'analyse automatique a détecté le problème et **annulé le déploiement sans intervention humaine**, environ une minute après l'arrivée du trafic.
- Seul un pod sur quatre a été exposé, grâce au canary.
- Le test visait `taskflow-canary`, donc uniquement la nouvelle version : le défaut n'a pas été dilué dans le trafic de l'ancienne version.
- Les preuves (AnalysisRun, logs k6, événements) ont été relevées tout de suite, avant que Kubernetes ne les oublie (environ une heure).
- Le retour dans Git s'est fait par une PR relue (`git revert`), donc tracé.

## Ce qui est à améliorer

- Git est resté sur 2.1.0 pendant environ neuf minutes après l'abort, le temps de faire le revert. Le cluster et Git n'étaient pas d'accord.
- Le test ne dure que 30 secondes avec 5 utilisateurs virtuels : un défaut qui n'apparaît qu'à plus forte charge ou plus tard passerait.
- L'analyse n'a lieu qu'au palier de 25 % : rien ne revérifie la qualité aux paliers 50 % et 75 %.
- Les sondes ne regardent que `/health`, ce qui les rend aveugles à ce type de défaut.

## Actions correctives

Propositions à ajuster par l'équipe : les échéances sont indicatives.

| Action | Responsable | Échéance |
| --- | --- | --- |
| Remplacer la sonde `/health` par une vérification qui touche une vraie route de l'application (par exemple `/tasks`) | Hamza | 22/10/2026 |
| Ajouter un test de charge sur l'image dans la CI de l'application, avant de publier une version | Ezzat | 22/10/2026 |
| Ajouter l'analyse k6 aux paliers 50 % et 75 %, et allonger le test pour couvrir plus de trafic | Hamza | 29/10/2026 |
| Écrire la procédure : après un abort automatique, faire le revert dans Git dans la foulée, pour que Git ne reste pas en retard | Ezzat | 22/10/2026 |
