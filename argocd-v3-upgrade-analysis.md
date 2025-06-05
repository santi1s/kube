# ArgoCD v3 Upgrade Analysis

## Pre-upgrade Checklist:

1. **Backup Current Configuration**
   ```bash
   kubectl get configmap -n argocd -o yaml > argocd-configmaps-backup.yaml
   kubectl get secret -n argocd -o yaml > argocd-secrets-backup.yaml
   kubectl get application -A -o yaml > argocd-applications-backup.yaml
   ```

2. **Review RBAC Policies**
   - Check if using fine-grained RBAC inheritance
   - Update policies if needed

3. **Check Custom Resources**
   - Verify custom resource configurations still valid
   - Test ignoreDifferences configurations

4. **Redis Configuration**
   - Ensure Redis is properly configured
   - Check Redis persistence settings

## Upgrade Process:

1. **Apply v3.0.5 Manifests**
   ```bash
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.0.5/manifests/install.yaml
   ```

2. **Update ConfigMaps**
   - Disable RBAC log enforcement if not needed:
     ```yaml
     server.rbac.log.enforce.enable: "false"
     ```
   - Enable fine-grained RBAC if needed:
     ```yaml
     server.rbac.fine.grained.inheritance: "true"
     ```

3. **Verify Rollouts Extension Compatibility**
   - May need to update to latest version
   - Test in non-production first

## Risks and Mitigations:

| Risk | Impact | Mitigation |
|------|--------|------------|
| RBAC changes break permissions | High | Test RBAC policies in staging |
| Log volume increase | Medium | Configure log retention/filtering |
| Redis dependency issues | Medium | Ensure Redis HA configuration |
| Extension incompatibility | Low | Update extensions post-upgrade |

## Rollback Plan:

1. Save current v2.14.13 manifests
2. Document current configurations
3. Test rollback procedure in staging
4. Keep backup of all applications

## Post-upgrade Validation:

- [ ] All applications syncing correctly
- [ ] RBAC policies working as expected
- [ ] Rollouts extension functional
- [ ] Performance metrics normal
- [ ] No unexpected log volume