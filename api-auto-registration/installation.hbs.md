# Install API Auto Registration

This topic describes how to install API Auto Registration from the Tanzu Application Platform package repository.

>**Note:** The "iterate", "run", and "full" profiles include API Auto Registration by default. 
> If your cluster is one of these profiles, skip the install and proceed to the [Usage section](usage.md).
> For more information about profiles, see [About TAP profiles](../about-package-profiles.md#profiles-and-packages).

## <a id='prereqs'></a>TAP Prerequisites

Before installing API Auto Registration, complete all prerequisites to install Tanzu Application Platform. 
For more information, see [TAP Prerequisites](../prerequisites.md).

## <a id='install'></a>Install

To install the API Auto Registration package:

1. List version information for the package by running:

    ```console
    tanzu package available list apis.apps.tanzu.vmware.com --namespace tap-install
    ```

    For example:

    ```console
    $ tanzu package available list apis.apps.tanzu.vmware.com --namespace tap-install
    - Retrieving package versions for apis.apps.tanzu.vmware.com...
      NAME                                     VERSION  RELEASED-AT
      apis.apps.tanzu.vmware.com  0.0.6        2022-08-23 19:00:00 -0500 -05
      apis.apps.tanzu.vmware.com  0.1.0        2022-08-30 19:00:00 -0500 -05
    ```

1. (Optional) Gather values schema:

    Display values schema of the package:

    ```console
    tanzu package available get apis.apps.tanzu.vmware.com/VERSION-NUMBER --values-schema --namespace tap-install
    ```

    Where `VERSION-NUMBER` is the version of the package listed in earlier.

    For example:

    ```console
    tanzu package available get apis.apps.tanzu.vmware.com/0.1.0 --values-schema --namespace tap-install

    Retrieving package details for apis.apps.tanzu.vmware.com/0.1.0... 
    KEY                        DEFAULT                                       TYPE     DESCRIPTION
    ca_cert_data                                                             string   Optional: PEM-encoded certificate data for the controller to trust TLS connections with a custom CA
    cluster_name               dev                                           string   Name of the cluster that will be used for setting the API entity lifecycle in TAP GUI. The value should be unique for each run cluster.
    tap_gui_url                http://server.tap-gui.svc.cluster.local:7000  string   FQDN URL for TAP GUI
    replicas                   1                                             integer  Number of controller replicas to deploy
    resources.limits.cpu       500m                                          string   CPU limit of the controller
    resources.limits.memory    500Mi                                         string   Memory limit of the controller
    resources.requests.cpu     20m                                           string   CPU request of the controller
    resources.requests.memory  100Mi                                         string   Memory request of the controller
    logging_profile            production                                    string   Logging profile for controller. If set to development, use console logging with full stack traces, else use JSON logging
    ```

2. Locate the Tanzu Application Platform GUI URL.

    When running on a full profile Tanzu Application Platform cluster, the default value of Tanzu Application Platform GUI URL is sufficient. You might want to edit this to match the externally available FQDN of Tanzu Application Platform GUI to display the entity URL in the externally accessible APIDescriptor status.

    When installed in a run cluster or without a profile where Tanzu Application Platform GUI is not installed in the same cluster, you must set the `tap_gui_url` parameters correctly for successful entity registration with Tanzu Application Platform GUI.

    You can locate the `tap_gui_url` by going to the view cluster with the Tanzu Application Platform GUI you want to register the entity with:

    ```console
    kubectl get secret tap-values -n tap-install -o jsonpath="{.data['tap-values\.yaml']}" | base64 -d | yq '.tap_gui.app_config.app.baseUrl'
    ``` 

1. (Optional but recommended) Create `api-auto-registration-values.yaml`

    If you'd like to overwrite the default values when installing the package, create a `api-auto-registration-values.yaml` file as follows:


    ```yaml
    tap_gui_url: https://tap-gui.view-cluster.com
    cluster_name: staging-us-east
    ca_cert_data:  |
        -----BEGIN CERTIFICATE-----
        MIICpTCCAYUCBgkqhkiG9w0BBQ0wMzAbBgkqhkiG9w0BBQwwDgQIYg9x6gkCAggA
        ...
        9TlA7A4FFpQqbhAuAVH6KQ8WMZIrVxJSQ03c9lKVkI62wQ==
        -----END CERTIFICATE-----
    ```

4. Install the package using the Tanzu CLI:
    
    ```console
    tanzu package install api-auto-registration 
    --package-name apis.apps.tanzu.vmware.com
    --namespace tap-install
    --version $VERSION
    --values-file api-auto-registration-values.yaml
    ``` 

5. Verify the package installation by running:

    ```console
    tanzu package installed get api-auto-registration -n tap-install
    ```

    Verify that `STATUS` is `Reconcile succeeded`:

    ```console
    kubectl get pods -n api-auto-registration
    ```

6. Verify that applying an APIDescriptor resource to your cluster results in the `STATUS` showing `Ready`:

    ```console
    kubectl apply -f - <<EOF
    apiVersion: apis.apps.tanzu.vmware.com/v1alpha1
    kind: APIDescriptor
    metadata:
      name: sample-api-descriptor-with-absolute-url
    spec:
      type: openapi
      description: A sample APIDescriptor to validate package installation successful
      system: test-installation
      owner: test-installation
      location:
        path: "/api/v3/openapi.json"
        baseURL:
          url: https://petstore3.swagger.io
    EOF
    ```

    Verify that the APIDescriptor status shows `Ready`:

    ```console
    kubectl get apidescriptors
    NAME                                       STATUS
    sample-api-descriptor-with-absolute-url    Ready

    kubectl get apidescriptors -owide
    NAME                                       STATUS    TAP GUI ENTITY URL     API SPEC URL
    sample-api-descriptor-with-absolute-url    Ready     <url-to-the-entity>    <url-to-the-api-spec>
    ```

    If the status does not show `Ready`, you can inspect the reason with the detailed message shown by running:

    ```console
    kubectl get apidescriptor sample-api-descriptor-with-absolute-url -o jsonpath='{.status.conditions[?(@.type=="Ready")].message}'
    ```

    Verify that the entity is created successfully in your Tanzu Application Platform GUI: 
    `<TAP_GUI_URL>/catalog/default/api/sample-api-descriptor-with-absolute-url`
