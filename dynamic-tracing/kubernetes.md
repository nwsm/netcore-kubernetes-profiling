# dynamic-tracing-kubernetes

This guide will take you through setting up a pod to be dynamically traced in Kubernetes using a sidecar.  See [dynamic-tracing.yaml](./dynamic-tracing.yaml) for the end product.  It executes a fairly intimidating configmap at startup to generate a native image.  See [the overview](./overview.md) for a description of what it does.

#### Environment Variables
Set the following environment variables for your main process.

```
env:
- name: COMPlus_PerfMapEnabled
  value: "1"
```

`COMPlus_PerfMapEnabled` creates a perf map in `/tmp` that perf can read to symbolicate stack traces. 

#### Liveness Probe
If you have a liveness probe you will want to increase the `initialDelaySeconds`.  The startup script can take awhile to run.

```
livenessProbe:
  initialDelaySeconds: 600
```

#### shareProcessNamespace
Setting `shareProcessNamespace` to true allows the sidecar to easily access the process you want to debug.

#### Use runNative on startup
Adjust the command to run [runNative.sh](./runNative.sh) instead of the actual app.  [runNative.sh](./runNative.sh) will generate a native image of the app dll and then run the native image.  See [the overview](./overview.md) for a description of what it does.

```
command: ["/run-native/runNative.sh"]
args: ["/app/app.dll"]
```

```
- name: run-native-volume
  mountPath: /run-native
...
- name: run-native-volume
  configMap:
    defaultMode: 0740
    name: run-native
```

#### Mount shared folders
Both `/tmp` and `/app-profile` need to be mounted as an emptydir and shared between your sidecar and your main container to allow for dynamic tracing.

```
- name: tmp
  emptyDir: {}
- name: app
  emptyDir: {}
...
- name: app
  mountPath: /app-profile
- name: tmp
  mountPath: /tmp
```

Additionally mount `/sys` on the host for bcc.

```
- name: sys
  mountPath: /sys
...
- name: sys
  hostPath:
    path: /sys
    type: Directory
```

#### Set privileged
The sidecar executes kernel functions that requires elevated privileges.

```
securityContext:
  privileged: true
```

#### Mount host headers

The `setup.sh` script will attempt to pull linux headers when it is run, but in a lot of cases this will fail to find the correct linux headers in the repos setup in the sidecar container.  The following lines appear commented out in the `dynamic-tracing.yaml` and have been confirmed to work in GKE with an Ubuntu node.  If you're having trouble with the setup.sh script pulling Linux headers try this approach.

```
- mountPath: /usr/src
  name: modules 
  readOnly: true
- mountPath: /lib/modules
  name: headers
  readOnly: true
...
- hostPath:
    path: /usr/src
    type: Directory
  name: modules
- hostPath:
    path: /lib/modules
    type: Directory
  name: headers
```

## Next Steps
See [probes](./probes.md) for more information about the kinds of probes you can place.  This document has examples of using both perf and bcc to place dynamic probes on [this app](https://github.com/joe-elliott/sample-netcore-app)