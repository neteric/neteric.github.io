


k8s.Client
operator.Parameters
recorder       record.EventRecorder
licenseChecker license.Checker

esObservers *observer.Manager

dynamicWatches watches.DynamicWatches

// expectations help dealing with inconsistencies in our client cache,
// by marking resources updates as expected, and skipping some operations if the cache is not up-to-date.
expectations *expectations.ClustersExpectation

// iteration is the number of times this controller has run its Reconcile method
iteration uint64


1. 入队了Generic事件，用于检测es实际状态
2. 入队了不是本资源的NamespacedName，用于将一个资源的变化转换成一个或者多个其他资源入队
3. 动态watch，允许在调谐期间加入watch的资源


watch的资源：
// Watch for changes to Elasticsearch
// Watch StatefulSets
// Watch pods belonging to ES clusters
// Watch services
// Watch owned and soft-owned secrets
// Trigger a reconciliation when observers report a cluster health change


operator重启，