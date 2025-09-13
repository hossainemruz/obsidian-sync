#### Add update hook to watching resource
```go
// SetupWithManager sets up the controller with the Manager.
func (r *QdrantClusterReconciler) SetupWithManager(mgr ctrl.Manager, config *models.OperatorConfig) error {
    endpointsHandler, err := events.EnqueueRequestForEndpoint(mgr.GetClient())
    if err != nil {
       return err
    }
    b := ctrl.NewControllerManagedBy(mgr).
       For(&qdrantov1.QdrantCluster{}, builder.WithPredicates(predicate.Funcs{
          // Add an update hook so that we can delete and recreate the info metrics when labels have changed
          UpdateFunc: func(e event.TypedUpdateEvent[client.Object]) bool {
             metrics.UpdateQdrantClusterInfo(e.ObjectOld.(*qdrantov1.QdrantCluster), e.ObjectNew.(*qdrantov1.QdrantCluster))
             return true
          },
       })).                                   // Watch QdrantCluster objects
       Owns(&corev1.Pod{}).                   // Watch Pods and reconcile parent QdrantCluster objects
       Owns(&corev1.PersistentVolumeClaim{}). // Watch PersistentVolumeClaims and reconcile parent QdrantCluster objects
       Owns(&corev1.Service{}).               // Watch Services and reconcile parent QdrantCluster objects
       Owns(&corev1.ConfigMap{}).             // Watch ConfigMap and reconcile parent QdrantCluster objects
       Owns(&networkingv1.Ingress{}).         // Watch Ingresses and reconcile parent QdrantCluster objects
       Owns(&networkingv1.NetworkPolicy{}).   // Watch NetworkPolicies and reconcile parent QdrantCluster objects
       Owns(&policyv1.PodDisruptionBudget{}). // Watch PodDisruptionBudgets and reconcile parent QdrantCluster objects
       Watches(&corev1.Endpoints{}, endpointsHandler).
       WithOptions(controller.Options{MaxConcurrentReconciles: config.Features.ClusterManagement.GetMaxConcurrentReconciles()})
    if config.Features.ClusterManagement.Ingress.GetProvider() == models.QdrantCloudTraefik {
       b = b.
          Owns(&traefikv1alpha1.Middleware{}).                      // Watch Middlewares and reconcile parent QdrantCluster objects
          Watches(&traefikv1alpha1.IngressRoute{}, handler.Funcs{}) // Watch IngressRoutes, so they are in the cache, however do not reconcile QdrantCluster objects [Note there is no owner ref, because they are in the `kube-system` namespace]
    }
    return b.Complete(errors.WithSilentRequeueOnConflict(r))
}
```
