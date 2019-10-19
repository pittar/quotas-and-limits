# Quotas and LimitRanges Demo

## Setup

```
#  As an admin:
oc new-project quota-test
oc apply -f project-resource-quota.yaml -n quota-test
oc apply -f project-limit-ranges.yaml -n quota-test

# As either admin or a user that can create deployments.
oc apply -f test-deployment.yaml -n quota-test
```

## Key Components

In the file `project-resource-quota.yaml`, no project-level `limit` is defined for **CPU**.  This is intentional.  We don't want to set a quota limit on CPU here in order to let individual pods burst.  Limiting CPU to a reasonable burst level is taken care of in the next step.  This will only apply to **NotTerminating** resources (your apps) and not build or deploy pods.  This means **Terminating** pods (deployment pods, build pods, etc...) should not count towards the project quota.

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1000m"
    requests.memory: "2Gi"
    limits.memory: "2Gi"
  scopes:
  - NotTerminating 
```

In the file `project-limit-ranges.yaml` we will setup *max,min, and default* values for CPU and Memory for pods and containers in the project.  There are defaults set so that other kinds of pods (deployement, build, etc...) will get default values.  The most important lines are the last two:

```
      maxLimitRequestRatio:
        cpu: 4
```

This states that a continer can only burt to 4x it's CPU **reqeust**.  This gives you a more dynamic limit.  There are other actual limits set in this file as well, just to safe guard other resources that may not be so dynamic (such as memory).

In the file `test-deployment.yaml` the important part is the `resources` stanza:

```
          resources:
            requests:
              cpu: 100m 
              memory: 128Mi 
            limits:
              cpu: 300m 
              memory: 128Mi 
```

Here, the `request` is for **100m* of CPU, and the limit is for *300m* of CPU.  This is allowed because it is in the 4x range we defined at the project level.  This container would be rejected if the value was greater than 500m (100m + 400m).

## Results

Although this is a simplified example, it demonstrates that you do not need a **hard quota for CPU limit** at the project level.  This can be managed with `limit ranges` and setting an appropriate `maxLimitRequestRatio`.

This will let you allow certain pods to burts to your configured max ration (e.g. 4x) while still protecting overall cluster resources, since the total **requests** is being enfoced, and anything above **requests** is not guranteed and can be reclaimed when required.

## Further Reading

* [Quotas](https://docs.openshift.com/container-platform/3.11/admin_guide/quota.html)
* [Limit Ranges](https://docs.openshift.com/container-platform/3.11/admin_guide/limits.html)