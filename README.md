# homelab-docs
Documentation, Architecture, ADRs, Runbooks and Guides

## Architecture Decision Records

Key decisions documented in `adrs/`:

1. [kubeadm over managed K8s](./adrs/adr-001-kubeadm.md)

## Docs Structure

```
homelab-docs/
├── README.md             ← Executive 
├── adrs/                 # Architecture Decision Records
└── guides/               # Bootstrap guides
```

## Key Design Principles

1. **Everything as Code** 
2. **Defense in Depth** 
3. **Zero Trust Access**
4. **Resilience by Design**
5. **Observability First** 
6. **Incremental Build**