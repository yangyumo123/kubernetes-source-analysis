typeGroupVersionKinds注册的信息
=================================================================
## 简介
所有注册到typeGroupVersionKinds中的信息如下（格式：key=包路径.type名称， value={group,version,kind}。我们用key=value来表示这些信息）：

    k8s.io/kubernetes/pkg/apis/rbac.RoleList={rbac.authorization.k8s.io, __internal, RoleList}
    k8s.io/kubernetes/pkg/api.Node={"", __internal, Node}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={apps, v1beta1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={authorization.k8s.io, v1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={authorization.k8s.io, v1beta1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/batch.CronJobList={batch, __internal, ScheduledJobList}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.NetworkPolicyList={extensions, v1beta1, NetworkPolicyList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={rbac.authorization.k8s.io, v1beta1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={batch, v1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/batch.CronJob={batch, __internal, ScheduledJob}
    k8s.io/kubernetes/pkg/apis/settings.PodPresetList={settings.k8s.io, __internal, PodPresetList}
    k8s.io/kubernetes/pkg/api/v1.EndpointsList={"", v1, EndpointsList}
    k8s.io/kubernetes/pkg/api.EventList={"", __internal, EventList}
    k8s.io/kubernetes/pkg/api.Pod={"", __internal, Pod}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={federation, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/api/v1.List={"", v1, List}
    k8s.io/kubernetes/pkg/api/v1.ServiceAccountList={"", v1, ServiceAccountList}
    k8s.io/kubernetes/pkg/apis/authorization/v1beta1.SelfSubjectAccessReview={authorization.k8s.io, v1beta1, SelfSubjectAccessReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={authorization.k8s.io, v1beta1, WatchEvent}
    k8s.io/kubernetes/pkg/api.LimitRangeList={"", __internal, LimitRangeList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={authentication.k8s.io, v1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ThirdPartyResourceList={extensions, v1beta1, ThirdPartyResourceList}
    k8s.io/kubernetes/pkg/apis/extensions.Ingress={extensions, __internal, Ingress}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.Role={rbac.authorization.k8s.io, v1alpha1, Role}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={rbac.authorization.k8s.io, v1alpha1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/rbac.ClusterRoleBindingList={rbac.authorization.k8s.io, __internal, ClusterRoleBindingList}
    k8s.io/kubernetes/pkg/api/v1.Event={"", v1, Event}
    k8s.io/kubernetes/pkg/apis/authentication/v1beta1.TokenReview={authentication.k8s.io, v1beta1, TokenReview}
    k8s.io/kubernetes/pkg/apis/batch/v1.JobList={batch, v1, JobList}
    k8s.io/kubernetes/pkg/apis/extensions.NetworkPolicyList={extensions, __internal, NetworkPolicyList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={imagepolicy.k8s.io, v1alhpa1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/imagepolicy.ImageReview={imagepolicy.k8s.io, __internal, ImageReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={networking.k8s.io, v1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/batch/v2alpha1.CronJob={batch, v2alpha1, ScheduledJob}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ReplicaSetList={extensions, v1beta1, ReplicaSetList}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.NetworkPolicy={extensions, v1beta1, NetworkPolicy}
    k8s.io/kubernetes/pkg/apis/extensions.DeploymentRollback={extensions, __internal, DeploymentRollback}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={networking.k8s.io, v1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={rbac.authorization.k8s.io, v1beta1, ListOptions}
    k8s.io/kubernetes/pkg/api/v1.ResourceQuotaList={"", v1, ResourceQuotaList}
    k8s.io/kubernetes/pkg/api.PodProxyOptions={"", __internal, PodProxyOptions}
    k8s.io/kubernetes/pkg/api.ComponentStatus={"", __internal, ComponentStatus}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={admissionregistration.k8s.io, v1alpha1, GetOptions}
    k8s.io/kubernetes/pkg/apis/authorization/v1.LocalSubjectAccessReview={authorization.k8s.io, v1, LocalSubjectAccessReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={settings.k8s.io, v1alpha1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={networking.k8s.io, v1, GetOptions}
    k8s.io/kubernetes/pkg/api.EndpointsList={"", __internal, EndpointsLIst}
    k8s.io/kubernetes/pkg/api.RangeAllocation={"", __internal, RangeAllocation}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.DeploymentRollback={apps, v1beta1, DeploymentRoolback}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={apps, v1beta, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={authentication.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={autoscaling, v1, ListOptions}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.Scale={extensions, v1beta1, Scale}
    k8s.io/kubernetes/pkg/api/v1.PersistentVolumeList={"", v1, PersistentVolumeList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={admissionregistration.k8s.io, v1alpha1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={batch, v1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.IngressList={extensions, v1beta1, IngressList}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.ClusterRole={rbac.authorization.k8s.io, v1beta1, ClusterRole}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={storage.k8s.io, v1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={extensions, v1beta1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={authentication.k8s.io, v1beta1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={autoscaling, v1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={batch, v2alpha1, GetOptions}
    k8s.io/kubernetes/pkg/apis/batch.CronJob={batch, __internal, CronJob}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.DeploymentRollback={extensions, v1beta1, DeploymentRoolback}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ThirdPartyResourceDataList={extensions, v1beta1, ThridPartyResourceDataList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={extensions, v1beta1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions.ThirdPartyResourceList={extensions, __internal, ThirdPartyResourceList}
    k8s.io/kubernetes/pkg/apis/componentconfig.KubeSchedulerConfiguration={componentconfig, __internal, KubeSchedulerConfiguration}
    k8s.io/kubernetes/pkg/api/v1.ComponentStatusList={"", v1,ComponentStatusList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.APIResourceList={"", v1, ComponentStatusList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={apps, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/api/v1.ReplicationControllerList={"", v1, ReplicationControllerList}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.DeploymentList={extensions, v1beta1, DeploymentList}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.PodSecurityPolicyList={extensions, v1beta1, PodSecurityPolicyList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={policy, v1beta1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={policy, v1beta1, ExportOptions}
    k8s.io/kubernetes/pkg/api/v1.EventList={"", v1, EventList}
    k8s.io/kubernetes/pkg/api/v1.SecretList={"", v1, SecretList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.APIVersions={"", v1, APIVersions}
    k8s.io/kubernetes/pkg/api.ConfigMap={"", __internal, ConfigMap}
    k8s.io/kubernetes/pkg/apis/extensions.DeploymentList={apps, __internal, DeploymentList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={authentication.k8s.io, v1beta1, GetOptions}
    k8s.io/kubernetes/pkg/apis/policy/v1beta1.Eviction={policy, v1beta1, Eviction}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={settings.k8s.io, v1alpha1, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={rbac.authorization.k8s.io, v1beta1, GetOptions}
    k8s.io/kubernetes/pkg/api/v1.ReplicationController={"", v1, ReplicationController}
    k8s.io/kubernetes/pkg/api.Service={"", __internal, Service}
    k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1.ExternalAdmissionHookConfigurationList={admissionregistration.k8s.io, v1alpha1, ExternalAdmissionHookConfigurationList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={authorization.k8s.io, v1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={autoscaling, v2alpha1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/batch/v2alpha1.CronJobList={batch, v2alpha1, ScheduledJobList}
    k8s.io/kubernetes/pkg/apis/certificates/v1beta1.CertificateSigningRequestList={certificates.k8s.io, v1beta1, CertificateSigningRequestList}
    k8s.io/kubernetes/pkg/apis/settings/v1alpha1.PodPresetList={settings.k8s.io, v1alpha1, PodPresetList}
    k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1.ImageReview={imagepolicy.k8s.io, v1alpha1, ImageReview}
    k8s.io/kubernetes/pkg/api/v1.Endpoints={"", v1, Endpoints}
    k8s.io/kubernetes/pkg/api/v1.Node={"", v1, Node}
    k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1.InitializerConfigurationList={admissionregistration.k8s.io, v1alpha1, InitializerConfigurationList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={authorization.k8s.io, v1beta1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={extensions, v1beta1, ListOptions}
    k8s.io/kubernetes/pkg/apis/rbac.RoleBindingList={rbac.authorization.k8s.io, __internal, RoleBindingList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={storage.k8s.io, v1beta1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1.KubeletConfiguration={componentconfig, v1alpha1, KubeletConfiguration}
    k8s.io/kubernetes/pkg/api.PodStatusResult={"", __internal, PodStatusResult}
    k8s.io/kubernetes/pkg/api.PersistentVolumeList={"", __internal, PersistentVolumeList}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.Scale={apps, v1beta1, Scale}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={networking.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/policy/v1beta1.PodDisruptionBudgetList={policy, v1beta1, PodDisruptionBudgetList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={rbac.authorization.k8s.io, v1alpha1, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={storage.k8s.io, v1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={storage.k8s.io, v1, WatchEvent}
    k8s.io/kubernetes/pkg/api.ServiceAccount={"", __internal, ServiceAccount}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={admissionregistration.k8s.io, v1alpha1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={authentication.k8s.io, v1, ListOptions}
    k8s.io/kubernetes/pkg/apis/extensions.ThirdPartyResourceDataList={extensions, __internal, ThirdPartyResourceDataList}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.RoleBindingList={rbac.authorization.k8s.io, v1beta1, RoleBindingList}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.ClusterRoleList={rbac.authorization.k8s.io, v1alpha1, ClusterRoleList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={settings.k8s.io, v1alpha1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={imagepolicy.k8s.io, v1alpha1, GetOptions}
    k8s.io/kubernetes/federation/apis/federation/v1beta1.ClusterList={federation, v1beta1, ClusterList}
    k8s.io/kubernetes/pkg/api/v1.Secret={"", v1, Secret}
    k8s.io/kubernetes/pkg/api.Namespace={"", __internal, Namespace}
    k8s.io/kubernetes/pkg/api.PodExecOptions={"", __internal, PodExecOptions}
    k8s.io/kubernetes/pkg/apis/extensions.Scale={extensions, __internal, Scale}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.ClusterRole={rbac.authorization.k8s.io, v1alpha1, ClusterRole}
    k8s.io/kubernetes/pkg/api/v1.LimitRangeList={"", v1, LimitRangeList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={imagepolicy.k8s.io, v1alpha1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={imagepolicy.k8s.io, v1alpha1, ExportOptions}
    k8s.io/kubernetes/federation/apis/federation.ClusterList={federation, __internal, ClusterList}
    k8s.io/kubernetes/pkg/api/v1.PodLogOptions={"", v1, PodLogOptions}
    k8s.io/kubernetes/pkg/apis/authorization.LocalSubjectAccessReview={authorization.k8s.io, __internal, LocalSubjectAccessReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={extensions, v1beta1, GetOptions}
    k8s.io/kubernetes/pkg/apis/settings/v1alpha1.PodPreset={settings.k8s.io, v1alpha1, PodPreset}
    k8s.io/kubernetes/pkg/api/v1.PodTemplate={"", v1, PodTemplate}
    k8s.io/kubernetes/pkg/api.ConfigMapList={"", __internal, ConfigMapList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={authentication.k8s.io, v1beta1, ListOptions}
    k8s.io/kubernetes/pkg/apis/extensions.IngressList={extensions, __internal, IngressList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={storage.k8s.io, v1, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={imagepolicy.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={autoscaling, v2alpha1, GetOptions}
    k8s.io/kubernetes/pkg/api/v1.PodProxyOptions={"", v1, PodProxyOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={"", v1, GetOptions}
    k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1.InitializerConfiguration={admissionregistration.k8s.io, v1alpha1, InitializerConfiguration}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={admissionregistration.k8s.io, v1alpha1, ListOptions}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.Deployment={apps, v1beta1, Deployment}
    k8s.io/kubernetes/pkg/apis/extensions.Deployment={apps, __internal, Deployment}
    k8s.io/kubernetes/pkg/apis/autoscaling/v1.HorizontalPodAutoscalerList={autoscaling, v1, HorizontalPodAutoscalerList}
    k8s.io/kubernetes/pkg/apis/autoscaling.Scale={autoscaling, __internal, Scale}
    k8s.io/kubernetes/pkg/apis/batch/v2alpha1.CronJob={batch, v2alpha1, CronJob}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={certificates.k8s.io, v1beta1, ListOptions}
    k8s.io/kubernetes/pkg/apis/autoscaling/v1.Scale={autoscaling, v1, Scale}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={autoscaling, v2alpha1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={batch, v2alpha1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={certificates.k8s.io, v1beta1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/extensions.ReplicationControllerDummy={extensions, __internal, ReplicationControllerDummy}
    k8s.io/kubernetes/pkg/apis/batch.CronJobList={batch, __internal, CronJobList}
    k8s.io/kubernetes/pkg/apis/networking/v1.NetworkPolicyList={networking.k8s.io, v1, NetworkPolicyList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={policy, __internal, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={authorization.k8s.io, v1beta1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/batch.JobList={batch, __internal, JobList}
    k8s.io/kubernetes/pkg/apis/policy.Eviction={policy, __internal, Eviction}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={rbac.authorization.k8s.io, v1beta1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={rbac.authorization.k8s.io, v1alpha1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/extensions.DaemonSet={extensions, __internal, DaemonSet}
    k8s.io/kubernetes/pkg/api/v1.PersistentVolumeClaimList={"", v1, PersistentVolumeClaimList}
    k8s.io/kubernetes/pkg/api.NamespaceList={"", __internal, NamespaceList}
    k8s.io/kubernetes/pkg/api.PodLogOptions={"",  __internal, PodLogOptions}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.StatefulSet={apps, v1beta1, StatefulSet}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={authentication.k8s.io, v1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={authorization.k8s.io, v1, ListOptions}
    k8s.io/kubernetes/pkg/apis/certificates.CertificateSigningRequestList={certificates.k8s.io, __internal, CertificateSigningRequestList}
    k8s.io/kubernetes/pkg/apis/networking.NetworkPolicyList={networking.k8s.io, __internal, NetworkPolicyList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={federation, v1beta1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={"", v1, DeleteOptions}
    k8s.io/kubernetes/pkg/api.SerializedReference={"", __internal, SerializedReference}
    k8s.io/kubernetes/pkg/apis/authorization/v1.SelfSubjectAccessReview={authorization.k8s.io, v1, SelfSubjectAccessReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={autoscaling, v1, GetOptions}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.ClusterRoleBindingList={rbac.authorization.k8s.io, v1beta1, ClusterRoleBindingList}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.ClusterRoleBindingList={rbac.authorization.k8s.io, v1alpha1, ClusterRoleBindingList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={storage.k8s.io, v1beta1, WatchEvent}
    k8s.io/kubernetes/pkg/api/v1.NodeList={"", v1, NodeList}
    k8s.io/kubernetes/pkg/api.ServiceList={"", __internal, ServiceList}
    k8s.io/kubernetes/pkg/apis/autoscaling/v2alpha1.HorizontalPodAutoscalerList={autoscaling, v2alpha1, HorizontalPodAutoscalerList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={autoscaling, v2alpha1, ListOptions}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.Ingress={extensions, v1beta1, Ingress}
    k8s.io/kubernetes/pkg/apis/rbac.ClusterRole={rbac.authorization.k8s.io, __internal, ClusterRole}
    k8s.io/kubernetes/pkg/apis/batch.JobTemplate={batch, __internal, JobTemplate}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.Status={"", v1, Status}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={"", v1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={"", v1, ListOptions}
    k8s.io/kubernetes/pkg/api.ResourceQuotaList={"", __internal, ResourceQuotaList}
    k8s.io/kubernetes/pkg/api.PersistentVolumeClaim={"", __internal, PersistentVolumeClaim}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={authentication.k8s.io, v1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/authorization.SubjectAccessReview={authorization.k8s.io, __internal, SubjectAccessReview}
    k8s.io/kubernetes/pkg/apis/policy.PodDisruptionBudget={policy, __internal, PodDisruptionBudget}
    k8s.io/kubernetes/pkg/api.List={"", __internal, List}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.RoleList={rbac.authorization.k8s.io, v1beta1, RoleList}
    k8s.io/kubernetes/pkg/api/v1.LimitRange={"", v1, LimitRange}
    k8s.io/kubernetes/pkg/api/v1.ConfigMap={"", v1, ConfigMap}
    k8s.io/kubernetes/pkg/apis/admission/v1alpha1.AdmissionReview={admission.k8s.io, v1alpha1, AdmissionReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={batch, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/api/v1.Namespace={"", v1, Namespace}
    k8s.io/kubernetes/pkg/api/v1.PodPortForwardOptions={"", v1, PodPortForwardOptions}
    k8s.io/kubernetes/pkg/api.LimitRange={"", __internal, LimitRange}
    k8s.io/kubernetes/pkg/api.ResourceQuota={"", __internal, ResourceQuota}
    k8s.io/kubernetes/pkg/api.ServiceAccountList={"", __internal, ServiceAccountList}
    k8s.io/kubernetes/pkg/api.PersistentVolume={"", __internal, PersistentVolume}
    k8s.io/kubernetes/pkg/api.PodPortForwardOptions={"", __internal, PodPortForwardOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={certificates.k8s.io, v1beta1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.RoleBinding={rbac.authorization.k8s.io, v1beta1, RoleBinding}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={imagepolicy.k8s.io, v1alpha1, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={admission.k8s.io, v1alpha1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={admission.k8s.io, v1alpha1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions.DeploymentRollback={apps, __internal, DeploymentRollback}
    k8s.io/kubernetes/pkg/apis/extensions.Scale={apps, __internal, Scale}
    k8s.io/kubernetes/pkg/apis/authorization.SelfSubjectAccessReview={authorization.k8s.io, __internal, SelfSubjectAccessReview}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={batch, v1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={policy, v1beta1, ListOptions}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.ClusterRoleBinding={rbac.authorization.k8s.io, v1alpha1, ClusterRoleBinding}
    k8s.io/kubernetes/pkg/apis/rbac.Role={rbac.authorization.k8s.io, __internal, Role}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={apps, v1beta1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={autoscaling, v2alpha1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.PodSecurityPolicy={extensions, v1beta1, PodSecurityPolicy}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={extensions, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/policy/v1beta1.PodDisruptionBudget={policy, v1beta1, PodDisruptionBudget}
    k8s.io/kubernetes/pkg/api.Binding={"", __internal, Binding}
    k8s.io/kubernetes/pkg/api.Event={"", __internal, Event}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={authentication.k8s.io, v1beta1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/autoscaling/v1.HorizontalPodAutoscaler={autoscaling, v1, HorizontalPodAutoscaler}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={certificates.k8s.io, v1beta1, GetOptions}
    k8s.io/kubernetes/pkg/apis/extensions.ThirdPartyResource={extensions, __internal, ThirdPartyResource}
    k8s.io/kubernetes/pkg/apis/storage/v1beta1.StorageClass={storage.k8s.io, v1beta1, StorageClass}
    k8s.io/kubernetes/pkg/apis/networking/v1.NetworkPolicy={networking.k8s.io, v1, NetworkPolicy}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={settings.k8s.io, v1alpha1, GetOptions}
    k8s.io/kubernetes/pkg/api/v1.Binding={"", v1, Binding}
    k8s.io/kubernetes/pkg/apis/admissionregistration.InitializerConfigurationList={admissionregistration.k8s.io, __internal, InitializerConfigurationList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={batch, v1, ListOptions}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.ClusterRoleBinding={rbac.authorization.k8s.io, v1beta1, ClusterRoleBinding}
    k8s.io/kubernetes/pkg/apis/componentconfig.KubeProxyConfiguration={componentconfig, __internal, KubeProxyConfiguration}
    k8s.io/kubernetes/pkg/api/v1.PersistentVolume={"", v1, PersistentVolume}
    k8s.io/kubernetes/pkg/api/v1.PodAttachOptions={"", v1, PodAttachOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={admissionregistration.k8s.io, v1alpha1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={certificates.k8s.io, v1beta1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.DaemonSet={extensions, v1beta1, DaemonSet}
    k8s.io/kubernetes/pkg/apis/rbac.ClusterRoleBinding={rbac.authorization.k8s.io, __internal, ClusterRoleBinding}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={storage.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={admission.k8s.io, v1alpha1, ListOptions}
    k8s.io/kubernetes/federation/apis/federation.Cluster={federation, __internal, Cluster}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={authorization.k8s.io, v1beta1, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOption={autoscaling, v1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ReplicationControllerDummy={extensions, v1beta1, ReplicationControllerDummy}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={federation, v1beta1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={authentication.k8s.io, v1beta1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/authentication.TokenReview={authentication.k8s.io, __internal, TokenReview}
    k8s.io/kubernetes/pkg/apis/settings.PodPreset={settings.k8s.io,__internal,PodPreset}
    k8s.io/kubernetes/pkg/apis/storage.StorageClass={storage.k8s.io,__internal,StorageClass}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={admission.k8s.io, v1alpha1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.RoleBinding={rbac.authorization.k8s.io, v1alpha1,RoleBinding}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.RoleBindingList={rbac.authorization.k8s.io, v1alpha1, RoleBindingList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.APIGroupList={"", v1, APIGroupList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={authorization.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions.NetworkPolicy={extensions, __internal, NetworkPolicy}
    k8s.io/kubernetes/pkg/api/v1.Service={"", v1, Service}
    k8s.io/kubernetes/pkg/api.ReplicationController={"", __internal, ReplicationController}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={batch, v2alpha1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/batch.Job={batch, __internal, Job}
    k8s.io/kubernetes/pkg/apis/extensions.Deployment={extensions, __internal, Deployment}
    k8s.io/kubernetes/pkg/apis/extensions.ReplicaSet={extensions,__internal, ReplicaSet}
    k8s.io/kubernetes/pkg/api.ReplicationControllerList={"", __internal, ReplicationControllerList}
    k8s.io/kubernetes/pkg/apis/admissionregistration.ExternalAdmissionHookConfigurationList={admissionregistration.k8s.io, __internal, ExternalAdmissionHookConfigurationList}
    k8s.io/kubernetes/pkg/apis/certificates/v1beta1.CertificateSigningRequest={certificates.k8s.io, v1beta1, CertificateSigningRequest}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ReplicaSet={extensions, v1beta1, ReplicaSet}
    k8s.io/kubernetes/pkg/apis/extensions.ThirdPartyResourceData={extensions, __internal, ThirdPartyResourceData}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={rbac.authorization.k8s.io, v1alpha1, GetOptions}
    k8s.io/kubernetes/pkg/apis/storage.StorageClassList={storage.k8s.io,__internal, StorageClassList}
    k8s.io/kubernetes/pkg/api/v1.NodeProxyOptions={"", v1, NodeProxyOptions}
    k8s.io/kubernetes/pkg/api/v1.ResourceQuota={"", v1, ResourceQuota}
    k8s.io/kubernetes/pkg/api/v1.RangeAllocation={"", v1, RangeAllocation}
    k8s.io/kubernetes/pkg/api.PodAttachOptions={"", __internal, PodAttachOptions}
    k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1.ExternalAdmissionHookConfiguration={admissionregistration.k8s.io, v1alpha1, ExternalAdmissionHookConfiguration}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.ControllerRevision={apps, v1beta1, ControllerRevision}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={batch,v2alpha1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={settings.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/api/v1.PodTemplateList={"", v1, PodTemplateList}
    k8s.io/kubernetes/pkg/api/v1.ComponentStatus={"", v1, ComponentStatus}
    k8s.io/kubernetes/pkg/api.NodeList={"", __internal, NodeList}
    k8s.io/kubernetes/pkg/api.NodeProxyOptions={"", __internal, NodeProxyOptions}
    k8s.io/kubernetes/pkg/api.PersistentVolumeClaimList={"", __internal, PersistentVolumeClaimList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={autoscaling, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/rbac/v1alpha1.RoleList={rbac.authorization.k8s.io, v1alpha1, RoleList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={storage.k8s.io, v1beta1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={admission.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1.KubeProxyConfiguration={componentconfig, v1alpha1, KubeProxyConfiguration}
    k8s.io/kubernetes/pkg/api/v1.Pod={"", v1, Pod}
    k8s.io/kubernetes/pkg/api/v1.PersistentVolumeClaim={"", v1, PersistentVolumeClaim}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={"", __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/apps.StatefulSetList={apps, __internal, StatefulSetList}
    k8s.io/kubernetes/pkg/apis/extensions.DaemonSetList={extensions, __internal, DaemonSetList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={policy, v1beta1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={storage.k8s.io, v1beta1, ListOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={federation, v1beta1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={admission.k8s.io, v1alpha1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.APIGroup={"", v1, APIGroup}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={apps, v1beta1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={authorization.k8s.io, v1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/autoscaling/v2alpha1.HorizontalPodAutoscaler={autoscaling, v2alpha1, HorizontalPodAutoscaler}
    k8s.io/kubernetes/pkg/apis/autoscaling.HorizontalPodAutoscalerList={autoscaling, __internal, HorizontalPodAutoscalerList}
    k8s.io/kubernetes/pkg/apis/batch/v1.Job={batch, v1, Job}
    k8s.io/kubernetes/pkg/apis/rbac.RoleBinding={rbac.authorization.k8s.io, __internal, RoleBinding}
    k8s.io/kubernetes/pkg/apis/apps.StatefulSet={apps, __internal, StatefulSet}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ThirdPartyResource={extensions, v1beta1, ThirdPartyResource}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.DaemonSetList={extensions, v1beta1, DaemonSetList}
    k8s.io/kubernetes/pkg/apis/policy.PodDisruptionBudgetList={policy, __internal, PodDisruptionBudgetList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={storage.k8s.io, v1beta1, ExportOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={storage.k8s.io, v1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.StatefulSetList={apps, v1beta1, StatefulSetList}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.ControllerRevisionList={apps, v1beta1, ControllerRevisionList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={apps, v1beta1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={authorization.k8s.io, v1, WatchEvent}
    k8s.io/kubernetes/pkg/apis/extensions.ReplicaSetList={extensions, __internal, ReplicaSetList}
    k8s.io/kubernetes/pkg/apis/networking.NetworkPolicy={networking.k8s.io, __internal, NetworkPolicy}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={policy, v1beta1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/admissionregistration.ExternalAdmissionHookConfiguration={admissionregistration.k8s.io,__internal, ExternalAdmissionHookConfiguration}
    k8s.io/kubernetes/pkg/apis/batch/v2alpha1.CronJobList={batch, v2alpha1, CronJobList}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.ThirdPartyResourceData={extensions, v1beta1, ThirdPartyResourceData}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={networking.k8s.io, v1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={settings.k8s.io, v1alpha1, ExportOptions}
    k8s.io/kubernetes/pkg/api/v1.ServiceProxyOptions={"", v1, ServiceProxyOptions}
    k8s.io/kubernetes/pkg/api.PodList={"", __internal, PodList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={rbac.authorization.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={rbac.authorization.k8s.io, v1beta1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1.KubeSchedulerConfiguration={componentconfig, v1alpha1, KubeSchedulerConfiguration}
    k8s.io/kubernetes/pkg/apis/componentconfig.KubeletConfiguration={componentconfig, __internal, KubeletConfiguration}
    k8s.io/kubernetes/federation/apis/federation/v1beta1.Cluster={federation, v1beta1, Cluster}
    k8s.io/kubernetes/pkg/api/v1.PodStatusResult={"", v1, PodStatusResult}
    k8s.io/kubernetes/pkg/api/v1.ServiceAccount={"", v1, ServiceAccount}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={batch, v2alpha1, ListOptions}
    k8s.io/kubernetes/pkg/apis/storage/v1.StorageClass={storage.k8s.io, v1, StorageClass}
    k8s.io/kubernetes/pkg/api.Endpoints={"", __internal, Endpoints}
    k8s.io/kubernetes/pkg/apis/authentication/v1.TokenReview={authentication.k8s.io, v1, TokenReview}
    k8s.io/kubernetes/pkg/apis/authorization/v1beta1.LocalSubjectAccessReview={authorization.k8s.io, v1beta1, LocalSubjectAccessReview}
    k8s.io/kubernetes/pkg/apis/extensions.PodSecurityPolicyList={extensions, __internal, PodSecurityPolicyList}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.ClusterRoleList={rbac.authorization.k8s.io, v1beta1, ClusterRoleList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={rbac.authorization.k8s.io, v1alpha1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/admission.AdmissionRevie={admission.k8s.io, __internal, AdmissionReview}
    k8s.io/kubernetes/pkg/api/v1.PodList={"", v1, PodList}
    k8s.io/kubernetes/pkg/api/v1.PodExecOptions={"", v1, PodExecOptions}
    k8s.io/kubernetes/pkg/api.ServiceProxyOptions={"", __internal, ServiceProxyOptions}
    k8s.io/kubernetes/pkg/api.Secret={"", __internal, Secret}
    k8s.io/kubernetes/pkg/apis/apps.ControllerRevisionList={apps, __internal, ControllerRevisionList}
    k8s.io/kubernetes/pkg/apis/authorization/v1beta1.SubjectAccessReview={authorization.k8s.io, v1beta1, SubjectAccessReview}
    k8s.io/kubernetes/pkg/apis/batch/v2alpha1.JobTemplate={batch, v2alpha1, JobTemplate}
    k8s.io/kubernetes/pkg/apis/authorization/v1.SubjectAccessReview={authorization.k8s.io, v1, SubjectAccessReview}
    k8s.io/kubernetes/pkg/apis/extensions.PodSecurityPolicy={extensions, __internal, PodSecurityPolicy}
    k8s.io/kubernetes/pkg/apis/storage/v1.StorageClassList={storage.k8s.io, v1, StorageClassList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.GetOptions={federation, v1beta1, GetOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={federation, v1beta1, ListOptions}
    k8s.io/kubernetes/pkg/api/v1.SerializedReference={"", v1, SerializedReference}
    k8s.io/kubernetes/pkg/api.PodTemplate={"", __internal, PodTemplate}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.WatchEvent={autoscaling, v1, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={certificates.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={extensions, v1beta1, DeleteOptions}
    k8s.io/kubernetes/pkg/apis/extensions.DeploymentList={extensions, __internal, DeploymentList}
    k8s.io/kubernetes/pkg/apis/rbac/v1beta1.Role={rbac.authorization.k8s.io, v1beta1, Role}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.InternalEvent={admissionregistration.k8s.io, __internal, WatchEvent}
    k8s.io/kubernetes/pkg/apis/apps/v1beta1.DeploymentList={apps, v1beta1, DeploymentList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={batch, v1, ExportOptions}
    k8s.io/kubernetes/pkg/apis/storage/v1beta1.StorageClassList={storage.k8s.io, v1beta1, StorageClassList}
    k8s.io/kubernetes/pkg/api/v1.ConfigMapList={"", v1, ConfigMapList}
    k8s.io/kubernetes/pkg/api.PodTemplateList={"", __internal, PodTemplateList}
    k8s.io/kubernetes/pkg/api.ComponentStatusList={"", __internal, ComponentStatusList}
    k8s.io/kubernetes/pkg/apis/admissionregistration.InitializerConfiguration={admissionregistration.k8s.io, __internal, InitializerConfiguration}
    k8s.io/kubernetes/pkg/apis/autoscaling.HorizontalPodAutoscaler={autoscaling, __internal, HorizontalPodAutoscaler}
    k8s.io/kubernetes/pkg/apis/certificates.CertificateSigningRequest={certificates.k8s.io, __internal, CertificateSigningRequest}
    k8s.io/kubernetes/pkg/apis/extensions/v1beta1.Deployment={extensions, v1beta1, Deployment}
    k8s.io/kubernetes/pkg/api/v1.NamespaceList={"", v1, NamespaceList}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ExportOptions={"", v1, ExportOptions}
    k8s.io/kubernetes/pkg/api.SecretList={"", __internal, SecretList}
    k8s.io/kubernetes/pkg/apis/rbac.ClusterRoleList={rbac.authorization.k8s.io, __internal, ClusterRoleList}
    k8s.io/kubernetes/pkg/api/v1.ServiceList={"", v1, ServiceList}
    k8s.io/kubernetes/pkg/apis/apps.ControllerRevision={apps, __internal, ControllerRevision}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.DeleteOptions={authentication.k8s.io, v1, DeleteOptions}
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1.ListOptions={networking.k8s.io, v1, ListOptions}

