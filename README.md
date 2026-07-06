# Homelab Roadmap — Mac Mini Server

Ordre de déploiement pensé pour que chaque étape soit indépendamment fonctionnelle
avant de passer à la suivante. Ne pas paralléliser — cocher, tester, puis avancer.

---

## Phase 0 — Socle machine

- [ ] Installer Docker Desktop (ou Colima/Podman si tu préfères éviter Docker Desktop) sur le Mac Mini
- [ ] (Optionnel mais recommandé) Mettre en place `nix-darwin` pour déclarer la config de base de la machine
- [ ] Créer un dossier de travail central, ex. `~/homelab/`, avec un sous-dossier par service
- [ ] Créer un réseau Docker dédié partagé entre tous les services : `docker network create homelab`
- [ ] Installer Tailscale sur le Mac Mini (accès distant sans exposer de ports publics)

**Critère de fin de phase :** tu peux SSH/accéder au Mac Mini depuis ton laptop via Tailscale.

---

## Phase 1 — Gitea (ton point de départ)

- [ ] `docker-compose.yml` pour Gitea + sa base (SQLite pour commencer, ou Postgres si tu veux practicer)
- [ ] Premier lancement, création du compte admin
- [ ] Configurer un accès SSH pour le git push (clé dédiée)
- [ ] Créer le repo du projet "dashboard" (celui qu'on va coder ensuite)
- [ ] Push un premier commit vide/README pour valider le flow

**Critère de fin de phase :** `git clone` / `git push` fonctionnent depuis ton laptop NixOS vers Gitea sur le Mac Mini.

---

## Phase 2 — Reverse proxy (à faire tôt, sinon tu le regretteras)

- [ ] Déployer Caddy (ou Traefik) en conteneur
- [ ] Router `gitea.local` (ou ton domaine/tailscale hostname) vers le conteneur Gitea
- [ ] Vérifier le HTTPS auto (Caddy le fait tout seul si tu as un vrai domaine ; sinon certif local)

**Critère de fin de phase :** tu accèdes à Gitea via une URL propre, plus via `IP:port`.

---

## Phase 3 — Monitoring infra (générique, une fois pour toutes)

- [ ] Déployer `node_exporter` (métriques host : CPU, RAM, disk)
- [ ] Déployer `cAdvisor` (métriques par conteneur)
- [ ] Déployer Prometheus, avec scrape config pointant sur `node_exporter` + `cAdvisor`
- [ ] Déployer Grafana, ajouter Prometheus comme datasource
- [ ] Importer un dashboard Grafana communautaire pour node_exporter (ID `1860` sur grafana.com) et un pour cAdvisor (ID `193` ou équivalent récent)
- [ ] Router `grafana.local` via Caddy (comme pour Gitea)

**Critère de fin de phase :** tu as un dashboard Grafana qui montre l'état réel du Mac Mini et de tous les conteneurs en cours.

---

## Phase 4 — Le projet dashboard (front + back)

- [ ] Squelette API Go (`cmd/api/main.go`, routes de base, healthcheck `/healthz`)
- [ ] Modèle + CRUD `projects` en SQLite
- [ ] Frontend minimal (htmx ou SPA légère) qui consomme l'API
- [ ] Dockerfile multi-stage pour l'API
- [ ] `docker-compose.yml` ajouté au stack global, routé via Caddy (`dashboard.local`)
- [ ] Endpoint `/metrics` avec `prometheus/client_golang` (au moins un counter + un histogram)
- [ ] Ajouter `dashboard-api` comme nouveau job de scrape dans `prometheus.yml`
- [ ] Dashboard Grafana dédié pour les métriques custom de l'app

**Critère de fin de phase :** ton propre projet apparaît dans Grafana avec ses propres métriques métier.

---

## Phase 5 — CI/CD (fermer la boucle DevOps)

- [ ] Activer Gitea Actions (ou installer Woodpecker CI)
- [ ] Pipeline : build de l'image Docker du dashboard à chaque push
- [ ] Pipeline : déploiement automatique sur le Mac Mini (via SSH + `docker compose up -d`, ou webhook)
- [ ] (Bonus) Ajouter un test Go basique qui doit passer avant le déploiement

**Critère de fin de phase :** un `git push` déclenche build + déploiement automatique, sans intervention manuelle.

---

## Phase 6 — Confort / extensions (à piocher librement, pas d'ordre imposé)

- [ ] Uptime Kuma (monitoring externe de tes services + jolie UI)
- [ ] Alertmanager + notif Telegram/ntfy quand un service tombe ou qu'une ressource est critique
- [ ] Loki + Promtail pour centraliser les logs (complète Prometheus, qui ne fait que des métriques)
- [ ] Pi-hole / AdGuard Home
- [ ] Dashboard d'accueil (Homepage/Homarr) qui liste tous tes services

---

## Règle du jeu

Une phase = une chose qui marche visiblement, pas un empilement de configs.
Si une phase prend plus de 2-3 sessions, c'est qu'elle est trop grosse : la découper encore.
