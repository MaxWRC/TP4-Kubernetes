# TP4 - Kubernetes : Ressources, Autoscaling & Observabilité

## Description
Rendu du TP4 : gestion des ressources, résilience, stack Prometheus/Grafana et réponse aux questions.

---

## Screens :
### Validation des QoS et métriques initiales

<img width="1070" height="999" alt="image" src="https://github.com/user-attachments/assets/5ccf5e25-75f0-4d79-a483-1ed25dd43f08" />

### Autoscaling HPA

<img width="945" height="173" alt="image" src="https://github.com/user-attachments/assets/30e1c8f8-8c4f-4e00-8323-b77b132f7020" />

### Dashboard Grafana

<img width="945" height="493" alt="image" src="https://github.com/user-attachments/assets/1e834809-8fb2-4392-8a7e-1f7d1f69c92c" />


### Alerte Prometheus

<img width="945" height="336" alt="image" src="https://github.com/user-attachments/assets/04b952e1-3a18-4144-9ef6-6595a128b40a" />



---

## Réponses aux questions

### Partie A : Gestion des ressources

**Question 1 : Pour chaque application, justifiez les valeurs de requests/limits choisies et la classe de QoS obtenue.**

Pour dimensionner les applications, j'ai défini des demandes et des limites adaptées à leurs profils d'exécution réels observés lors des phases de charge nominale. Concernant le service MySQL, j'ai alloué une demande de 300m CPU et 256Mi de mémoire vive, tout en fixant des plafonds maximaux à 1000m CPU et 1Gi de mémoire. Cette configuration garantit un socle matériel suffisant pour faire tourner le moteur InnoDB tout en lui laissant une marge d'extension substantielle pour absorber des requêtes complexes sans risquer une défaillance brutale. Pour WordPress, j'ai retenu une demande de 100m CPU et 128Mi de mémoire, encadrée par une limite fixée à 300m CPU et 256Mi. Le serveur web PHP-FPM nécessite peu de ressources à l'arrêt mais a besoin de petits pics de calcul lors des rendus de pages dynamiques. Étant donné que les valeurs de requêtes sont strictement inférieures aux limites maximales sur les deux conteneurs, Kubernetes assigne automatiquement à ces pods la classe de qualité de service Burstable. Ce niveau de service représente le choix idéal en production car il garantit les allocations planifiées tout en autorisant une surallocation contrôlée sur le serveur hôte.

**Question 2 : Expliquez la différence entre LimitRange et ResourceQuota et pourquoi les deux sont complémentaires.**

Le LimitRange et le ResourceQuota répondent à deux impératifs de gouvernance distincts mais totalement complémentaires au sein d'un cluster mutualisé. Le LimitRange applique des contraintes unilatérales à l'échelle micro, c'est-à-dire directement au niveau individuel du conteneur ou du pod. Son rôle consiste à injecter des valeurs par défaut lorsqu'un développeur oublie de déclarer ses besoins, tout en interdisant le déploiement d'objets aux dimensions disproportionnées. À l'inverse, le ResourceQuota agit à l'échelle macro sur la globalité d'un namespace. Il additionne l'ensemble des réservations en temps réel pour bloquer toute création supplémentaire dès que le cumul de processeur, de mémoire vive ou de nombre d'objets atteint la limite autorisée. Utilisés conjointement, ils empêchent qu'un pod unique monopolise un nœud entier et évitent qu'une prolifération de micro-pods ne vienne asphyxier l'espace alloué au projet.

**Question 3 : Que se passe-t-il pour un pod Best Effort en cas de pression mémoire sur un nœud ?**

Lorsqu'un nœud worker subit une saturation sévère de sa mémoire vive physique et franchit le seuil critique d'éviction, le démon kubelet déclenche immédiatement une procédure de protection du système. Le planificateur classe les pods selon leur niveau d'importance et calcule un score de sacrifice interne. Les pods rattachés à la catégorie BestEffort, qui ne précisent absolument aucune demande ni limite de ressources dans leurs spécifications, sont arrêtés et expulsés en priorité absolue. Le système détruit ces charges de travail sans préavis afin de libérer instantanément la mémoire vive nécessaire au bon fonctionnement des composants internes du cluster et des conteneurs critiques bénéficiant de garanties de ressources.

---

### Partie B : Autoscaling et résilience

**Question 1 : Expliquez comment le HPA décide de scaler (quelle métrique, quel seuil, quel comportement de stabilisation).**

L'Horizontal Pod Autoscaler évalue la topologie applicative en interrogeant à intervalles réguliers le serveur de métriques du cluster. Le mécanisme calcule la moyenne de la consommation CPU actuelle constatée sur l'ensemble des réplicas en activité, puis la confronte à la cible relative définie dans le manifeste, que j'ai établie à 50% de la demande unitaire. Dès que l'activité globale dépasse ce palier, l'algorithme applique une équation de proportionnalité pour déterminer le nombre exact de pods nécessaires et ajuste l'échelle de manière dynamique. Afin de prévenir tout comportement d'oscillation brutale qui verrait des conteneurs se créer et se supprimer en boucle au gré des micro-variations de trafic, le contrôleur applique une fenêtre de stabilisation temporelle de quelques minutes avant d'exécuter une réduction du nombre de réplicas vers l'état de repos.

**Question 2 : Quelles sont les limites de l'autoscaling basé uniquement sur le CPU ? Dans quel cas privilégier une métrique custom ou KEDA ?**

Le dimensionnement horizontal fondé exclusivement sur l'activité du processeur présente des limites structurelles importantes dans des environnements modernes. Le CPU demeure un indicateur tardif qui ne reflète pas toujours la charge réelle subie par le système, notamment face à des processus bloqués en attente d'entrées-sorties sur le réseau ou le stockage. Dans le cas de systèmes pilotés par les événements ou gérant de lourdes files de messages asynchrones, un composant peut afficher une charge processeur minime tout en accumulant des millions de messages non traités. Il devient alors indispensable de recourir à des métriques personnalisées via Prometheus ou d'adopter KEDA pour adapter dynamiquement le nombre de pods au volume réel des requêtes HTTP par seconde ou à la profondeur des messages en attente.

**Question 3 : Pourquoi combiner HPA et PDB est important en production ?**

L'association de l'Horizontal Pod Autoscaler et d'un PodDisruptionBudget constitue la pierre angulaire du maintien de la disponibilité en environnement de production. Alors que le rôle du HPA consiste à faire varier la capacité de traitement en fonction des variations imprévisibles de la charge utilisateur, le PodDisruptionBudget impose un plancher strict de disponibilité lors des interventions d'administration programmées. Lors d'opérations d'ingénierie telles que l'évacuation d'un nœud par une commande drain pour une mise à jour du noyau Linux, l'ordonnanceur a l'interdiction de détruire un conteneur si cela enfreint le quota minimal de pods sains. Le PDB oblige l'infrastructure à instancier et vérifier la disponibilité d'une nouvelle instance sur un autre serveur avant d'autoriser l'interruption de la précédente.

---

### Partie C : Observabilité : Prometheus et Grafana

**Question 1 : Expliquez le choix du type de ressource (Deployment/StatefulSet) pour Prometheus et pour Grafana.**

L'architecture de ma pile d'observabilité sépare distinctement les frontaux des moteurs de stockage. Grafana est déployé sous la forme d'un Deployment conventionnel car il fonctionne comme un tableau de bord sans état intrinsèque sensible. Ses définitions de panels, ses identifiants et ses configurations de sources de données peuvent être injectés dynamiquement via des ConfigMaps et des scripts de provisioning sans interruption de service. À l'opposé, Prometheus est instancié au travers d'un StatefulSet car il héberge une base de données temporelle exigeante. Ce type de ressource fournit des identifiants stables sur le réseau, un ordre séquentiel strict de déploiement et surtout un attachement persistant et ordonné aux volumes de stockage distribués Longhorn, évitant ainsi toute corruption d'index lors des redémarrages.

**Question 2 : Quelle politique de rétention avez-vous configurée pour les métriques Prometheus, et pourquoi (comparez avec la rétention des logs du TP3-partie-2) ?**

J'ai appliqué une durée de rétention de quinze jours pour le stockage des métriques au sein de Prometheus. Ce paramétrage permet d'analyser les tendances de consommation, d'anticiper la saturation des quotas et de diagnostiquer les anomalies de cycle de vie récentes sans saturer l'espace disque du cluster. Cette durée d'archivage est délibérément plus courte que celle dédiée aux journaux d'événements gérés par la pile Elasticsearch. Les métriques servent principalement à la détection d'alertes instantanées et au pilotage opérationnel en temps réel, alors que les logs applicatifs et d'accès contiennent des traces juridiques, d'authentification et de conformité réglementaire qui doivent demeurer consultables sur plusieurs mois.

**Question 3 : Décrivez un scénario d'incident et comment vos dashboards + alertes + logs vous auraient permis de le diagnostiquer.**

Dans le cas d'une dérive logicielle introduisant une fuite de mémoire sur un composant applicatif, le croisement des briques de surveillance assure une résolution rapide de l'incident. Lorsque le conteneur dépasse son plafond autorisé, le noyau l'arrête brutalement, entraînant des redémarrages successifs. La règle Prometheus dédiée détecte ce comportement anormal et fait basculer l'alerte PodCrashLooping à l'état actif, notifiant immédiatement l'administrateur. En consultant le tableau de bord Grafana, je repère immédiatement sur la courbe de suivi de mémoire une ascension continue et rectiligne jusqu'au seuil critique du quota. Une fois le nom du pod et l'horodatage exact du plantage isolés sur le graphique, je bascule sur Kibana pour filtrer les journaux de ce conteneur précis juste avant l'arrêt, mettant en évidence l'exception fatale à l'origine du sinistre.

### Environ 700 mots.
