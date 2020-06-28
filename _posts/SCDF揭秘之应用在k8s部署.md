# SCDF揭秘之应用在k8s部署

应用部署的实现在类```KubernetesAppDeployer```中实现，构造器代码如下：

```java
public class KubernetesAppDeployer extends AbstractKubernetesDeployer implements AppDeployer {

   private static final String SERVER_PORT_KEY = "server.port";

   protected final Log logger = LogFactory.getLog(getClass().getName());

   @Autowired
   public KubernetesAppDeployer(KubernetesDeployerProperties properties, KubernetesClient client) {
      this(properties, client, new DefaultContainerFactory(properties));
   }

   @Autowired
   public KubernetesAppDeployer(KubernetesDeployerProperties properties, KubernetesClient client,
      ContainerFactory containerFactory) {
      this.properties = properties;
      this.client = client;
      this.containerFactory = containerFactory;
   }
```

其中部署类的deploy() 函数将实现应用在k8s上的部署工作，并返回部署成功的appid

```java
@Override
public String deploy(AppDeploymentRequest request) {
   String appId = createDeploymentId(request);
   logger.debug(String.format("Deploying app: %s", appId));

   try {
      AppStatus status = status(appId);

      if (!status.getState().equals(DeploymentState.unknown)) {
         throw new IllegalStateException(String.format("App '%s' is already deployed", appId));
      }

      int externalPort = configureExternalPort(request);
      String indexedProperty = request.getDeploymentProperties().get(INDEXED_PROPERTY_KEY);
      boolean indexed = (indexedProperty != null) ? Boolean.valueOf(indexedProperty) : false;
      logPossibleDownloadResourceMessage(request.getResource());
      Map<String, String> idMap = createIdMap(appId, request);

      if (indexed) {
         logger.debug(String.format("Creating Service: %s on %d with", appId, externalPort));
         createService(appId, request, idMap, externalPort);

         logger.debug(String.format("Creating StatefulSet: %s", appId));
         createStatefulSet(appId, request, idMap, externalPort);
      }
      else {
         logger.debug(String.format("Creating Service: %s on %d", appId, externalPort));
         createService(appId, request, idMap, externalPort);

         logger.debug(String.format("Creating Deployment: %s", appId));
         createDeployment(appId, request, idMap, externalPort);
      }

      return appId;
   }
   catch (RuntimeException e) {
      logger.error(e.getMessage(), e);
      throw e;
   }
}
```

