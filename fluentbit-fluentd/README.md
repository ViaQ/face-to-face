# Various testing options using combination of fluentbit to fluentD

## Setup

1. spin up OCP cluster with CLO and EO
1. Update clusterlogging/instance to be unmanaged
1. Delete fluentd ds
    ```
    oc delete ds fluentd
    ```

## Configurations

### Combined
This scenario test using multi-container daemonset deployed to each node with collection being performed by fluentbit and processing by fluentd.  

1. Create fluentd deployment:
    ```
    oc create -f fluentd-ds.yaml
    ```
1. create fluentbit cm:
    ```
    oc create configmap fluentbit --from-file=fluentbit.conf
    ```
1. create fluentbit ds:
    ```
    oc extract configmap fluentd --to=.
    cp run.sh fluent-conf
    oc create configmap fluentd --from-file fluent-conf --dry-run | oc replace -f -
    ```

## Ref
* https://github.com/jcantrill/fluent-bit
