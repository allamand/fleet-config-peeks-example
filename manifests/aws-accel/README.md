# Ressources AWS d'accélération du démarrage à froid

Ces CR ACK servent les pistes 1 et 2 de la note
`pistes-acceleration-gpu-burst.md` : rapatrier **en région** l'image vLLM et
les poids du modèle, qui sortent aujourd'hui sur Internet par le NAT.

## Le problème, mesuré le 06/11

| Étage | Mesuré | Débit effectif |
|---|---|---|
| Pull image vLLM 11,5 Go | **152 s** | **0,60 Gbps** |
| Poids Mistral-7B depuis HuggingFace | **250 s** | **0,46 Gbps** |

Une `g6.2xlarge` est documentée à **5,0 Gbps de base / 10,0 en burst**. Le
débit observé est donc à **9–12 %** du plancher de l'instance : le goulot
n'est ni le GPU, ni le disque, c'est le chemin réseau.

## Ce que contient ce répertoire

| Fichier | Ressource | Rôle |
|---|---|---|
| `00-ecr-repository.yaml` | `Repository` ECR `peeks/vllm-openai` | l'image en région, tag immuable |
| `10-vpce-ecr-api.yaml` | `VPCEndpoint` interface | authentification et manifeste ECR |
| `11-vpce-ecr-dkr.yaml` | `VPCEndpoint` interface | protocole de registre |
| `12-vpce-s3-gateway.yaml` | `VPCEndpoint` gateway | **les couches d'image ET les poids** |

**Aucun endpoint VPC n'existait dans eu-west-3 avant ce commit** — requête à
l'échelle du compte, zéro résultat.

Les trois endpoints ne valent que pris ensemble. Sans le gateway S3,
l'authentification devient privée mais les 11,5 Go de couches continuent de
sortir par le NAT : le gain serait nul et l'erreur invisible.

## Aucun rôle IAM n'est créé

Vérifié par simulation IAM sur `paris-inference-demo-cluster-node-role`, le
rôle de la NodeClass GPU, qui porte déjà `AmazonEC2ContainerRegistryPullOnly` :

```
ecr:GetAuthorizationToken        allowed
ecr:BatchGetImage                allowed
ecr:GetDownloadUrlForLayer       allowed
ecr:BatchCheckLayerAvailability  implicitDeny   ← normal, c'est du PUSH
```

Aucune modification de groupe de sécurité non plus : le groupe
`eks-cluster-sg-…` porte déjà une règle entrante `IpProtocol: -1` depuis
lui-même, et les ENI des nœuds Auto Mode le portent.

## ⚠ Ce répertoire exige une Application Argo CD posée À LA MAIN

**Même exception que `manifests/vllm`, et pour la même raison.** Sur la
révision appmod épinglée `68fccbcc`, il n'existe aucune voie GitOps
permettant à notre overlay de livrer des CR arbitraires :

- les RGD kro vivent dans appmod, gelé jusqu'après le 17/11 ;
- une entrée de registre portant un `path` voit son `repoURL` **forcé** sur
  appmod — `platform-charts/appset-chart/templates/application-set.yaml`,
  lignes 157-160, avec `$repoURLGit` fixé une seule fois ligne 7 et sans
  surcharge possible par entrée ;
- le chart `ack-pod-identity` d'appmod ne sait exprimer qu'un ensemble
  **fermé** d'identités et n'expose pas `inlinePolicies`.

L'Application est donc le **seul** objet non géré par git de ce montage. Elle
suit la branche, avec `prune: false` et sans finalizer, pour qu'une
suppression accidentelle n'emporte jamais les ressources AWS.

> **À VÉRIFIER LE 16/11 — ajouté à la checklist J-1 :** l'Application
> `aws-accel-paris` existe, est `Synced`/`Healthy`, et **aucun** des
> ApplicationSets du hub ne la revendique (contrôle : `ownerReferences` vide).

## Politique de suppression

Les quatre CR portent `services.k8s.aws/deletion-policy: retain`. Supprimer un
CR **ne supprime pas** la ressource AWS. C'est délibéré : un dépôt ECR vidé
par erreur rendrait tout nœud de débordement incapable de démarrer, et un
endpoint supprimé ferait repartir silencieusement le trafic par le NAT.
