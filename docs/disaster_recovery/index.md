# Disaster Recovery

This section develops a practical recovery discipline from objectives and failure models through backup architecture, platform-specific recovery, testing, and disaster-recovery operations.

Each part has its own index. Lesson pages currently preserve the lesson title and source link; they are intentionally small starting points for adding detailed notes, examples, and runbooks.

## Course map

- [01. Recovery objectives, vocabulary and failure models](01-objectives-and-failure-models/index.md)
- [02. RPO, RTO and recovery sequencing](02-rpo-rto-and-sequencing/index.md)
- [03. Backup architecture: copies, chains, retention and capacity](03-backup-architecture/index.md)
- [04. Consistency, integrity and proof of restorability](04-consistency-integrity-and-proof/index.md)
- [05. Linux file-level backup and restore](05-linux-file-level-backup-and-restore/index.md)
- [06. Snapshots: LVM, Btrfs and ZFS](06-snapshots/index.md)
- [07. Block images, bare-metal recovery and reconstruction](07-block-images-and-reconstruction/index.md)
- [08. Backup repositories: restic, Borg and repository failure](08-backup-repositories/index.md)
- [09. Encryption, keys and key recovery](09-encryption-keys-and-key-recovery/index.md)
- [10. Object storage, versioning and retention](10-object-storage-and-retention/index.md)
- [11. Immutability, air gap and ransomware resilience](11-immutability-air-gap-and-ransomware/index.md)
- [12. Virtual machine and hypervisor recovery](12-virtual-machine-and-hypervisor-recovery/index.md)
- [13. Container and Kubernetes recovery](13-container-and-kubernetes-recovery/index.md)
- [14. Database backup and point-in-time recovery](14-database-backup-and-point-in-time-recovery/index.md)
- [15. Infrastructure reconstruction: IaC, config, network and identity](15-infrastructure-reconstruction/index.md)
- [16. Monitoring, restore testing and recovery assurance](16-monitoring-and-restore-testing/index.md)
- [17. Disaster recovery: failover, failback and the recovery estate](17-disaster-recovery-failover-and-failback/index.md)
- [18. Backup platform DR, media, cost and compliance](18-backup-platform-media-cost-and-compliance/index.md)
- [19. Production capstone](19-capstone/index.md)

## Suggested progression

Start with objectives and failure models, then establish RPO/RTO targets before choosing technologies. Work through architecture and integrity, study the platform-specific recovery paths that match your estate, and finish with testing, failover, governance, and the capstone.

