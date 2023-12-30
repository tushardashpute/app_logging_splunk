# Capture container logs in Kubernetes with Splunk Connect

Splunk allows the collection and analyzes of high volumes of machine-generated data (e.g. application logs). 
Once the data becomes indexes in Splunk, one can build reporting dashboard and alerts based of specific search. 
For instance, one can build a dashboard for application crashes, or failures to handle incoming request and track this over time. 
Splunk provides many integrations that makes it very easy to collect logs from a varied of sources. 
In this article, we will examine how to collect logs from cloud native applications running on Kubernetes.

Setting up Splunk
====

**1. Splunk Operator**

We can easily setup Splunk on Kubernetes using the official operator - [link](https://github.com/splunk/splunk-operator/).

First, create a Kubernetes namespace to host the pods of the Splunk operator as well as Splunk itself.

      $ kubectl create namespace monit

Second, install Splunk official operator in the newly created namespace as follows

      $ kubectl apply -f https://github.com/splunk/splunk-operator/releases/download/1.0.2/splunk-operator-install.yaml -n monit
      customresourcedefinition.apiextensions.k8s.io/clustermasters.enterprise.splunk.com created
      customresourcedefinition.apiextensions.k8s.io/indexerclusters.enterprise.splunk.com created
      customresourcedefinition.apiextensions.k8s.io/licensemasters.enterprise.splunk.com created
      customresourcedefinition.apiextensions.k8s.io/searchheadclusters.enterprise.splunk.com created
      customresourcedefinition.apiextensions.k8s.io/standalones.enterprise.splunk.com created
      serviceaccount/splunk-operator created
      role.rbac.authorization.k8s.io/splunk:operator:namespace-manager created
      rolebinding.rbac.authorization.k8s.io/splunk:operator:namespace-manager created
      deployment.apps/splunk-operator created

After few seconds, the operator will become ready to use, you can check the Pod status with

      $ kubectl get pods -n monit
      NAME                              READY   STATUS    RESTARTS   AGE
      splunk-operator-75b749554-vcpp8   1/1     Running   0          10s

**2. Splunk Standalone**

Now, we can deploy Splunk using this operator

      $ cat <<EOF | kubectl apply -n monit -f -
      apiVersion: enterprise.splunk.com/v2
      kind: Standalone
      metadata:
        name: s1
        finalizers:
        - enterprise.splunk.com/delete-pvc
      EOF
      standalone.enterprise.splunk.com/s1 created

After few moments, Splunk Pods will become available and ready to be used. We can check their status as follows:

      $ kubectl get pods -n monit              
      NAME                                  READY   STATUS    RESTARTS   AGE
      splunk-default-monitoring-console-0   1/1     Running   0          3m19s
      splunk-operator-75b749554-vcpp8       1/1     Running   0          6m38s
      splunk-s1-standalone-0                1/1     Running   0          5m56s

**3. Splunk credentials**

To get the credentials to access Splunk Web UI with kubectl we can print the secret created as part of the deployment of Splunk as follows:

      $ kubectl get secret splunk-monit-secret -n monit -o jsonpath='{.data}'
      {"hec_token":"Mzc3RjJCRTMtNTE4Ni05MzAyLTY1NTUtRjBBMTk1MDIxQTVE","idxc_secret":"bTI4cnRqSVhpaVI0cGdpUGRFR3lKSjV2","pass4SymmKey":"RjNrYURia3Zyck11ZTlkMEFvYk5Pd
      29F","password":"bzMwZnhSRkJYdTlLRjlRUnU5enFPM3ht","shc_secret":"OXpOaTY5MmdrVkt4Nm5xQ3FzR0RlMkQ4"} 

**Note**: the secret name splunk-monit-secret is a composed name of the --secret. If splunk is deployed in the default namespace, the secret name will be splunk-default-secret

To get one specific secret, for instance the Splunk Web UI password of the admin user we can do

      $ kubectl get secret splunk-monit-secret -n monit -o jsonpath='{.data.password}' |base64 -d
      o30fxRFBXu9KF9QRu9zqO3xm

Then we can access the Web UI by setting up port-forwarding to Splunk as follows

      $ kubectl port-forward splunk-s1-standalone-0 8000 -n monit
      Forwarding from 127.0.0.1:8000 -> 8000
      Forwarding from [::1]:8000 -> 8000

**4. Splunk Connect**

To be able to send logs to our Splunk deployment we need to get credentials. In our case, we specifically need [HEC (HTTP Event Collector) token](https://docs.splunk.com/Documentation/Splunk/8.2.4/Data/UsetheHTTPEventCollector).

To get the HEC token using kubectl

      $ kubectl get secret splunk-monit-secret -n monit -o jsonpath='{.data.hec_token}' |base64 -d
      377F2BE3-5186-9302-6555-F0A195021A5D

**Configuration file**

We need to pupulate a custom version of [values.yaml](https://github.com/splunk/splunk-connect-for-kubernetes/blob/develop/helm-chart/splunk-connect-for-kubernetes/values.yaml) with information specific to our Splunk instance like hostname and HEC token.

We need to create some environment variables to use when filling in the values.yaml file.

1. get Get splunk server address, use DNS name <service>.<namespace> or just <service>

        $ hostname="splunk-s1-standalone-service"

2. get the Splunk HEC token into a variable

       $ token=`kubectl get secret splunk-monit-secret -n monit -o jsonpath='{.data.hec_token}' |base64 -d`

3. get the Splunk admin password into a variable

        $ password=`kubectl get secret splunk-monit-secret -n monit -o jsonpath='{.data.password}' |base64 -d`

4. choose the index name to be used by Splunk to host the logs

        $ index="main"

5. pick a filename where the values will created.

        $ file=$(mktemp /tmp/splunk-connect-values.XXXXXX)

6. Create the values file and configure each section.

For instance, the bare minimum file would look like this where we disable sending to Splunk the kubernetes objects and metrics but only allow logging messages to be sent.

        $ cat >"${file}" << EOF
        global:
          splunk:
            hec:
              host: ${hostname}
              port: 8088
              token: ${token}
              protocol: https
              indexName: ${index}
              insecureSSL: true
        
        splunk-kubernetes-logging:
          enabled: true
          containers:
            logFormat: '%Y-%m-%dT%H:%M:%S.%N%:z'
            logFormatType: cri
          logs:
            applogs:
              from:
                pod: '*'
              multiline:
                firstline: /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}[-+]\d{4}/
                separator: ""
        
        splunk-kubernetes-objects:
          enabled: false
        splunk-kubernetes-metrics:
          enabled: false
        EOF

**Installation with Helm**

To be able ot install Splunk Connect with Helm, we to indicate to Helm where to find the charts. For this, add the Splunk Connect github repository to the list of local Help repositories

        $ helm repo add splunk https://splunk.github.io/splunk-connect-for-kubernetes/
        $ helm repo update

Now we can install Splunk Connect on the monitoring namespace using the custom values file we created in the previous section.

        $ helm install splunkconnect -n monit -f "${file}" splunk/splunk-connect-for-kubernetes
        NAME: splunkconnect
        LAST DEPLOYED: Fri Dec 29 16:22:53 2023
        NAMESPACE: monit
        STATUS: deployed
        REVISION: 1
        TEST SUITE: None
        NOTES:
        ███████╗██████╗ ██╗     ██╗   ██╗███╗   ██╗██╗  ██╗██╗
        ██╔════╝██╔══██╗██║     ██║   ██║████╗  ██║██║ ██╔╝╚██╗
        ███████╗██████╔╝██║     ██║   ██║██╔██╗ ██║█████╔╝  ╚██╗
        ╚════██║██╔═══╝ ██║     ██║   ██║██║╚██╗██║██╔═██╗  ██╔╝
        ███████║██║     ███████╗╚██████╔╝██║ ╚████║██║  ██╗██╔╝
        ╚══════╝╚═╝     ╚══════╝ ╚═════╝ ╚═╝  ╚═══╝╚═╝  ╚═╝╚═╝
        
        Listen to your data.
        
        Splunk Connect for Kubernetes is spinning up in your cluster.
        After a few minutes, you should see data being indexed in your Splunk.
        
        If you get stuck, we're here to help.
        Look for answers here: http://docs.splunk.com
        
        Warning: Disabling TLS will send the data unencrypted and will be vulnerable to MiTM attacks

After successfully deploying Splunk Connect an index called main will be created, we can check this in the Splunk UI (at http://localhost:8000 with login admin:${password})

![image](https://github.com/tushardashpute/app_logging_splunk/assets/74225291/0027ebf2-cd5f-4d64-b7ea-1481e5d591d2)

Deploy Sample App in test1 namespace:

      $ kubectl create ns test1
      $ kubectl create deployment  python-flask --image=tushardashpute/python_app:v1 -n test1 --port=5000
            deployment.apps/python-flask created
      $ kubectl port-forward deploy/python-flask -n test1 5000:5000
            Forwarding from 127.0.0.1:5000 -> 5000
            Forwarding from [::1]:5000 -> 5000

Now access the app using : localhost:5000

![image](https://github.com/tushardashpute/app_logging_splunk/assets/74225291/8d2c9ab7-b7fe-4e48-b3fa-67728d0ab3c2)

This app will log below messages:

          app.logger.info('This is an INFO message')
          app.logger.debug('This is a DEBUG message')
          app.logger.warning('This is a WARNING message')
          app.logger.error('This is an ERROR message')
          app.logger.critical('This is a CRITICAL message')
          return 'Hello, World!'

We can also check logs from our Pods are forwarded properly to splunk

![image](https://github.com/tushardashpute/app_logging_splunk/assets/74225291/73639bfb-5326-4a9c-8515-7b1cda39446f)













