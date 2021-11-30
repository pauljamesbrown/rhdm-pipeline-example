# This requires the following operators to be installed.

* Red Hat Openshift Pipelines
* Red Hat Openshift ServerlessArgoCD
* Kustomise
* Openshift Pipelines
* RHDM Knative Service

# Develop a Quarkus Based DMN Rule

# Test a Quarkus Based DMN Rule

# Build and Deploy a Quarkus DMN Rule

There are a number of options for building a Quarkus DMN rule. 

- JVM Mode
- Native Mode

## Option 1 - JVM Mode Using the Docker Image Extension

For this option you need to have a docker daemon running otherwise, you will see "Cannot connect to the Docker daemon" errors appearing in the logs.

Mention JIB

1. Add the container-image-docker extension to your application pom.xml file.
   ```
   ./mvnw quarkus:add-extension -Dextensions="container-image-docker"

  Running this command will add this dependency to your project POM.xml

   <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-container-image-docker</artifactId>
    </dependency>
   ```

   The extension quarkus-container-image-docker is using the Docker binary and the generated Dockerfiles under src/main/docker in order to perform Docker builds.

2. Configure the following Quarkus properties in your application.properties file (src/main/resource/application.properties)
   - quarkus.openshift.build-strategy=docker
   - quarkus.openshift.expose=true


3. To build a container image for your project, quarkus.container-image.build=true needs to be set 

Option 1 - Add it to the command line 
  - ./mvnw clean package -Dquarkus.container-image.build=true
  - This command will create a docker image and install into you local docker repository, so if you type "docker images" you should see a new entry 
  - Use docker run --rm -p 8080:8080 \<image-name>:\<image-tag> to validate the image starts

  - To test use a curl command once the docker image is up and running
```
curl --location --request POST 'http://localhost:8080/simple-dmn-rule' \
--header 'Content-Type: application/json' \
--data-raw '{ 
  "infraData": {
                "alertName": 25,
                "alertLevel": "High"

  }
}
```

Option 2 - Add it to the application properties file as per above.

 - quarkus.container-image.build=true
 
## Option 2 - JVM Mode Using the Quarkus Openshift Extension

For this option you need to be logged into an Openshift instance and have a namespace created.

1. Add the openshift extension to your application pom.xml file.
   ```
   ./mvnw quarkus:add-extension -Dextensions="openshift"

   Running this command will add this dependency to you POM.xml

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-openshift</artifactId>
    </dependency>

   ```
  The OpenShift extension is a wrapper extension that brings together the kubernetes and container-image-s2i extensions with sensible defaults so that it’s easier for the user to get started with Quarkus on OpenShift.

2. Configure the following Quarkus properties in your application.properties file (src/main/resource/application.properties)
   - ??? quarkus.openshift.build-strategy=docker
   - quarkus.container-image.build=true +
   
   
3. Login into your Openshift cluster and select the relevant namespace using either the following CLI commands of the Openshift console: 
    - oc new-project \<your-project-name>\
    or
    - oc project \<your-project-name>\
  
Tip: if you want to get the Openshift console using the CLI command prompt you can use:
    - oc whoami --show-console
      
4. Run the following command to build you project 
    - ./mvnw clean package

5. Once the build completes successfully check the image has been created in your Openshift cluster
    ```
    oc get is
    ```
6. Now create a new-application using the image
    ```
    oc new-app --name=simple-dmn <project>/<image-name>:<image-tag>
    
    Example: 

    oc new-app --name=simple-dmn default-route-openshift-image-registry.apps.sandbox.x8i5.p1.openshiftapps.com/dev-namespace/simple-dmn-rule:1.0.0-SNAPSHOT 
    ```
6. Expose the route to be able to invoke the  create a new-application using the image
```
 oc get svc
  oc expose svc/greeting
  oc get routes
  curl http://<route>/greeting
```

## Option 3 - Native Mode Using the Docker Image Extension

— Need a docker Daemon installed—

1. Add the following extension to the project:
```
./mvnw quarkus:add-extension -Dextensions="container-image-docker"
```
This essential adds this dependency to the project POM
```
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-container-image-docker</artifactId>
</dependency>
```

2. To build a container image for your project, quarkus.container-image.build=true needs to be set using any of the ways that Quarkus supports. Easiest option is to add it to application.properties file
```
quarkus.container-image.build=true
```

3. Run this command to create the native build image. 
```
./mvnw clean package -Pnative -Dquarkus.native.container-build=true
```
Warning - if you don’t use the “-Dquarkus.native.container-build=true” flag you will probably see the following error when you try to start the docker container. “standard_init_linux.go:211: exec user process caused "exec format error”

4. Now login to Openshift cluster

5. Create a new Openshift project

6. Create a build config based on the src/main/docker/Dockerfile.native file:
```
cat src/main/docker/Dockerfile.native | oc new-build --name simple-dmn-docker-native --strategy=docker --dockerfile -
```
This will create a new build config based on the Dockerfile. To verify this run this follow command:
```
oc get bc -n <your namespace>
```
7. Start the build which will upload the binary input
```
oc start-build simple-dmn-docker-native --from-dir .
```
8. Create a new-app from the build 
```
oc new-app <build-name>
```
9. Now, expose the service as a route so you can access from outside of the cluster
```
oc get svc
oc expose svc <build-name>
```
10. Now, test your service

## Now Turn the Simple Rule into a Serverless Component.
   
Note: We need to use the image tag from the previous step.

1. Use the same namespace as used in the previous deployment steps.

2. Deploy your rule services container image as a serverless service 
    1. Use the previously created image from your development project: 
    2. Set the labels for: 
        - app.openshift.io/runtime=quarkus 
        - app.kubernetes.io/part-of=simple-dmn-rule-rule 
   
    3. Deploy the service as a no-cluster-local service which means that the service is not specified as private and is publicly available outside of the OpenShift cluster. (--no-cluster-local will make the service publicly available this is the - default) +
    
    ```
    kn service create simple-dmn-rule --image <project>/<image-name>:<image-tag> --label app.openshift.io/runtime=quarkus --label app.kubernetes.io/part-of=simple-dmn-rule --port 8080 --no-cluster-local --request cpu=100m, memory=256mi -n <your namespace>
    ```
3. Retrieve the generated URL
  ```
 kn route list -n <your namespace>
  ```

4. Check that everything is working ok


## Argo CD, Kustomise & Tekton.

1. Create YAML manifest for all of the application components 
        a. Get all of the resources required
            ```
            oc get all --selector build=simple-dmn-docker-native
            ```
### Overview
* Prereqs - Ensure the simple dmn application is deployed and working.
* To use kustimise we need to create YAML Manifests from deployed applications
* When the need to build the Re-create the application from YAML Manifests to test

#### Steps :
  * Make the following directories

    * simple-rule-yaml
    * `oc get -o yaml` will return a lot of information that is not necessary to re-create an application - things like server-side apply fields, current status or timestamps. You can use the `kubectl-neat` plugin (`oc neat`) to remove almost all of the extra information and save you a lot of time cleaning up the YAML files.
    * Use oc get deployment simple-rule -o yaml -n `<namespace> | oc neat > deployment.yaml `

      * Even with the `kubetlc-neat` plugin there are a few fields that we need to remove. Open the file in an editor (`vi` or `nano`) and remove the following fields (if they exist):

        * **metadata.annotations.deployment.kubernetes.io/revision**
        * **metadata.annotations.image.openshift.io/triggers**
        * **metadata.annotations.openshift.io/generated-by**
        * **metadata.namespace**
        * **spec.template.metadata.annotations**
        * **spec.template.metadata.creationTimestamp**
    * Use oc get service simple-rule -o yaml -n `<namespace> | oc neat > service.yaml `

      * Remove the following fields from the YAML file:
        * **metadata.annotations**
        * **metadata.namespace**
        * **spec.clusterIP**
        * **spec.clusterIPs**
    * Use oc get route simple-rule -o yaml -n `<namespace> | oc neat > service.yaml `
    * Use oc get configmap simple-rule -o yaml -n `<namespace> | oc neat > configmap.yaml`
    * Use oc get is simple-rule -o yaml -n `<namespace> | oc neat > imagestream.yaml`
    * Use oc get buildconfig simple-rule -o yaml -n `<namespace> | oc neat > buildconfig.yaml`
    * Use oc get `service.serving.knative.dev simple-rule -o yaml -n <namespace>` | oc neat > knative-service.yaml
* ~~Protect credentials by converting secrets to sealed secrets~~

#### The Kustomize Part
Create the Kustomize files:
* Create the folllowing directory structure
- Copy the yaml files across
- Edit the deployment.yaml
        - Remove the entire .spec.template.metatdata section
        - Change the spec.template.spec.image to <PATCH_ME>
- Create a kustomization.yaml file in the base directory 
    - kustomize create --autodetect --labels app:kogito-dmn-simple
- Test your base kustomization
    - Customise build . 
- Create the development directory
- Create a kutomization.yaml file <include example>
- Create a development-patches.yaml file
- Populate the development-patches.yaml file https://kustomize.io/tutorial
- Test your base kustomization
    - Customise build . 

- Create the production directory
- Create a knative-serivce-patches.yaml file
- Populate the knative-serivce-patches.yaml.yaml file https://kustomize.io/tutorial
- Create a kutomization.yaml file <include example> for the 
- Test your base kustomization
    - Customise build . 

#### Tekton Pipeline Part:

**Invoking the pipe line**

tkn pipeline start build-and-deploy-quarkus-application \
    --use-param-defaults \
    --param DEPLOY_SERVERLESS=false \
    --param APP_NAME=coffee-shop \
    --param SOURCE_GIT_CONTEXT_DIR=coffee-shop \
    --param KUSTOMIZE_GIT_CONTEXT_DIR=coffee-shop-kustomize/coffee-shop \
    --param KUSTOMIZE_GIT_FILE_NAME=overlays/production/deployment-patches.yaml \
    --workspace name=app-source,claimName=workspace-pvc \
    --workspace name=maven-settings,emptyDir= \
    --workspace name=images-url,emptyDir= \
    --pod-template=$HOME/coffee-shop-final-lab/coffee-shop-pipeline/pod-template.yaml \
    --showlog \
    -n ${PREFIX}-pipeline


Red Hat Openshift GitOps

* AppProject
  * appproject-coffee-shop.yaml
* Application
  * application-coffee-shop.yaml
  *
* ArgoCD
  * argocd-argocd-yaml


