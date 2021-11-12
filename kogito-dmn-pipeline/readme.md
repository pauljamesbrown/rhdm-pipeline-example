```
tkn pipeline start build-and-deploy-rule-application \
    --use-param-defaults \
    --param APP_NAME=self-healing-simple \
    --param SOURCE_GIT_CONTEXT_DIR=self-healing-simple-rule \
    --workspace name=app-source,claimName=workspace-pvc \
    --workspace name=maven-settings,emptyDir= \
    --workspace name=images-url,emptyDir= \
    --showlog \
    -n ${PREFIX}-pipeline
```
