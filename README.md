k8s-manifests/ (branch: infra)
│
├── namespaces/
│   └── airbnb.yaml                          # airbnb namespace
│
├── databases/
│   ├── postgres/
│   │   ├── configmap.yaml
│   │   ├── secret.yaml (template)
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── mongodb/
│   │   ├── secret.yaml (template)
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── redis/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── elasticsearch/
│       ├── statefulset.yaml
│       └── service.yaml
│
├── apps/
│   ├── service-auth/
│   │   ├── configmap.yaml
│   │   ├── sealed-secret.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── service-booking/    (same structure)
│   ├── service-payment/    (same structure)
│   ├── service-review/     (same structure)
│   ├── service-listing/    (same structure)
│   ├── service-chat/       (same structure)
│   ├── frontend/           (same structure)
│   └── data-seeder/        (same structure)
│
├── ingress/
│   └── ingress.yaml                         # Nginx Ingress → routes all /api/v1/* + /ws + /
│
├── argocd-apps/
│   └── airbnb-app.yaml                      # ArgoCD watches this repo → auto-deploys to K3s
│
└── system/
    ├── sealed-secrets/                      # installed via Helm (Ansible)
    ├── argocd/                              # installed via Helm (Ansible)
    ├── nginx-ingress/                       # installed via Helm (Ansible)
    └── falco/                               # installed via Helm (Ansible)