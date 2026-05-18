# Kubernetes — от архитектуры до продакшна

Слайды для митапа/вебинара по Kubernetes для senior/mid+ SRE и DevOps инженеров.

## Просмотр слайдов

**GitHub Pages:** https://setloger.github.io/k8s/

## Содержание

1. Intro — что такое Kubernetes, история, концепции
2. Архитектура кластера — control plane, data plane
3. Жизненный цикл пода — фазы, probes, CrashLoopBackOff
4. Сеть в Kubernetes — CNI, Calico BGP, Gateway API
5. Scheduler и affinity — фильтрация, scoring, taints
6. Хранилище — PV/PVC/StorageClass, StatefulSet
7. RBAC и безопасность — ServiceAccount, Secrets, Admission
8. etcd deep dive — Raft, кворум, бэкап
9. Обновление кластера — Ansible, drain/uncordon, CIS
10. Kubernetes 2025–2026: курс на AI

## Локальный просмотр

Требуется [Marp for VS Code](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode) или Marp CLI:

```bash
npm install -g @marp-team/marp-cli
marp slides.md --preview
```

## Структура

```
k8s/
├── slides.md           # Исходник слайдов (Marp Markdown)
├── README.md
└── .github/
    └── workflows/
        └── deploy.yml  # GitHub Actions: Marp → HTML → Pages
```
