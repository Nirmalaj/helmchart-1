*********CONCEPTS & COMMANDS***************
====================
'helm search': Finding Charts
================================
1. helm search hub searches the Helm Hub, which comprises helm charts from dozens of different repositories.

```bash
		$ helm search hub wordpress					--> Search by hub
		URL                                                 CHART VERSION APP VERSION DESCRIPTION
		https://hub.helm.sh/charts/bitnami/wordpress        7.6.7         5.2.4       Web publishing platform for building blogs and ...
		https://hub.helm.sh/charts/presslabs/wordpress-...  v0.6.3        v0.6.3      Presslabs WordPress Operator Helm Chart
		https://hub.helm.sh/charts/presslabs/wordpress-...  v0.7.1        v0.7.1      A Helm chart for deploying a WordPress site on ...
```
2. helm search repo searches the repositories that you have added to your local helm client (with helm repo add). This search is done over local data, and no public network connection is needed.
```bash
		$ helm repo add brigade https://brigadecore.github.io/charts
		$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
		$ helm repo add bitnami https://charts.bitnami.com/bitnami
		"brigade" has been added to your repositories
		$ helm search repo brigade
		NAME                          CHART VERSION APP VERSION DESCRIPTION
		brigade/brigade               1.3.2         v1.2.1      Brigade provides event-driven scripting of Kube...
		brigade/brigade-github-app    0.4.1         v0.2.1      The Brigade GitHub App, an advanced gateway for...
		brigade/brigade-github-oauth  0.2.0         v0.20.0     The legacy OAuth GitHub Gateway for Brigade
		brigade/brigade-k8s-gateway   0.1.0                     A Helm chart for Kubernetes
		brigade/brigade-project       1.0.0         v1.0.0      Create a Brigade project
		brigade/kashti                0.4.0         v0.4.0      A Helm chart for Kubernetes
```
'helm install': Installing a Package
====================================
To install a new package, use the helm install command. At its simplest, it takes two arguments: A release name that you pick, and the name of the chart you want to install.

```
$ helm install happy-panda stable/mariadb
WARNING: This chart is deprecated
NAME: happy-panda
LAST DEPLOYED: Fri May  8 17:46:49 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
This Helm chart is deprecated
```

Now the mariadb chart is installed. Note that installing a chart creates a new release object. The release above is named happy-panda. 
(If you want Helm to generate a name for you, leave off the release name and use --generate-name.)

Helm does not wait until all of the resources are running before it exits. Many charts require Docker images that are over 600M in size, and may take a long time to install into the cluster.
To keep track of a release's state, or to re-read configuration information, you can use helm status:

```
		$ helm status happy-panda                
		NAME: happy-panda
		LAST DEPLOYED: Fri May  8 17:46:49 2020
		NAMESPACE: default
		STATUS: deployed
		REVISION: 1
		NOTES:
		This Helm chart is deprecated
```

Customizing the Chart Before Installing
==========================================
To see what options are configurable on a chart, use helm show values:

```
$ helm show values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/helm.sh/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3
```

You can then override any of these settings in a YAML formatted file, and then pass that file during installation.
```
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ helm install -f config.yaml stable/mariadb --generate-name
```

There are two ways to pass configuration data during install:

--values (or -f): Specify a YAML file with overrides. This can be specified multiple times and the rightmost file will take precedence
--set: Specify overrides on the command line.

More Installation Methods
============================
The helm install command can install from several sources:

```
--> A chart repository (as we've seen above)
--> A local chart archive (helm install foo foo-0.1.1.tgz)
--> An unpacked chart directory (helm install foo path/to/foo)
--> A full URL (helm install foo https://example.com/charts/foo-1.2.3.tgz)
```

'helm upgrade' and 'helm rollback': Upgrading a Release, and Recovering on Failure
============================================================

When a new version of a chart is released, or when you want to change the configuration of your release, you can use the helm upgrade command.

An upgrade takes an existing release and upgrades it according to the information you provide. Because Kubernetes charts can be large and complex, Helm tries to perform the least invasive upgrade. It will only update things that have changed since the last release.

```
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/helm.sh/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded. Happy Helming!
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```
In the above case, the happy-panda release is upgraded with the same chart, but with a new YAML file:

```
mariadbUser: user1
```

We can use helm get values to see whether that new setting took effect.

```
$ helm get values happy-panda
mariadbUser: user1
```

The helm get command is a useful tool for looking at a release in the cluster. And as we can see above, it shows that our new values from panda.yaml were deployed to the cluster.
Now, if something does not go as planned during a release, it is easy to roll back to a previous release using helm rollback [RELEASE] [REVISION].


```
$ helm rollback happy-panda 1
```

The above rolls back our happy-panda to its very first release version. A release version is an incremental revision. Every time an install, upgrade, or rollback happens, the revision number is incremented by 1. The first revision number is always 1. And we can use helm history [RELEASE] to see revision numbers for a certain release.

Helpful Options for Install/Upgrade/Rollback
=============================================
There are several other helpful options you can specify for customizing the behavior of Helm during an install/upgrade/rollback. Please note that this is not a full list of cli flags. To see a description of all flags, just run helm <command> --help.

--timeout: A value in seconds to wait for Kubernetes commands to complete This defaults to 5m0s
--wait: Waits until all Pods are in a ready state, PVCs are bound, Deployments have minimum (Desired minus maxUnavailable) Pods in ready state and Services have an IP address (and Ingress if a LoadBalancer) before marking the release as successful. It will wait for as long as the --timeout value. If timeout is reached, the release will be marked as FAILED. Note: In scenarios where Deployment has replicas set to 1 and maxUnavailable is not set to 0 as part of rolling update strategy, --wait will return as ready as it has satisfied the minimum Pod in ready condition.
--no-hooks: This skips running hooks for the command
--recreate-pods (only available for upgrade and rollback): This flag will cause all pods to be recreated (with the exception of pods belonging to deployments). (DEPRECATED in Helm 3)

Creating Your Own Charts
=========================
you can get started quickly by using the helm create command:
```
$ helm create deis-workflow
Creating deis-workflow
```
Now there is a chart in ./deis-workflow. You can edit it and create your own templates.

As you edit your chart, you can validate that it is well-formed by running helm lint.

When it's time to package the chart up for distribution, you can run the helm package command:
```
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

And that chart can now easily be installed by helm install:

```
$ helm install deis-workflow ./deis-workflow-0.1.0.tgz
...
```

*********************************************CODING SECTION*****************************************************
=================================
# helmchart
This repository is for Learning purpose , content has been taken from https://helm.sh/

Prerequisite: Values.yaml used for all below examples
===================
```yaml
favorite:
   drink: coffee
   food: pizza
technology:
   container: docker
   container-cluser: kubernetes
namespace: helm
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

***Concepts starting below***
===============
Built-in Objects
=================
The built-in values always begin with a capital letter. This is in keeping with Go's naming convention.
```bash
--> Release
--> Values
--> Chart
--> Files
--> Capabilities
--> Template
```
## Release
```bash
Release: This object describes the release itself. It has several objects inside of it:
Release.Name: The release name
Release.Namespace: The namespace to be released into (if the manifest doesn’t override)
Release.IsUpgrade: This is set to true if the current operation is an upgrade or rollback.
Release.IsInstall: This is set to true if the current operation is an install.
Release.Revision: The revision number for this release. On install, this is 1, and it is incremented with each upgrade and rollback.
Release.Service: The service that is rendering the present template. On Helm, this is always Helm.
```
## Values
```bash
Values: Values passed into the template from the values.yaml file and from user-supplied files. By default, Values is empty.
```
## Chart
```bash
Chart: The contents of the Chart.yaml file. Any data in Chart.yaml will be accessible here. For example {{ .Chart.Name }}-{{ .Chart.Version }} will print out the mychart-0.1.0.
The available fields are listed in the Charts Guide (https://helm.sh/docs/topics/charts/#the-chartyaml-file)
```
## Files
```bash
Files: This provides access to all non-special files in a chart. While you cannot use it to access templates, you can use it to access other files in the chart. See the section Accessing Files for more.
Files.Get is a function for getting a file by name (.Files.Get config.ini)
Files.GetBytes is a function for getting the contents of a file as an array of bytes instead of as a string. This is useful for things like images.
Files.Glob is a function that returns a list of files whose names match the given shell glob pattern.
Files.Lines is a function that reads a file line-by-line. This is useful for iterating over each line in a file.
Files.AsSecrets is a function that returns the file bodies as Base 64 encoded strings.
Files.AsConfig is a function that returns file bodies as a YAML map.
```
## Capabilities
```bash
Capabilities: This provides information about what capabilities the Kubernetes cluster supports.
Capabilities.APIVersions is a set of versions.
Capabilities.APIVersions.Has $version indicates whether a version (e.g., batch/v1) or resource (e.g., apps/v1/Deployment) is available on the cluster.
Capabilities.KubeVersion and Capabilities.KubeVersion.Version is the Kubernetes version.
Capabilities.KubeVersion.Major is the Kubernetes major version.
Capabilities.KubeVersion.Minor is the Kubernetes minor version.
```
## Template
```bash
Template: Contains information about the current template that is being executed
Name: A namespaced file path to the current template (e.g. mychart/templates/mytemplate.yaml)
BasePath: The namespaced path to the templates directory of the current chart (e.g. mychart/templates).
```


Template Functions and Pipelines
===================================

> Important URL for Template function :
> https://masterminds.github.io/sprig/

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```
After debug through commmad "helm install [release-name] [direcctory-name] --debug --dry-run" below output

```bash
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```
## Using the default function
One function frequently used in templates is the default function: default DEFAULT_VALUE GIVEN_VALUE. 
This function allows you to specify a default value inside of the template, in case the value is omitted. 
Let's use it to modify the drink example above:

```bash
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

In an actual chart, all static default values should live in the values.yaml, and should not be repeated using the default command (otherwise they would be redundant). 
However, the default command is perfect for computed values, which can not be declared inside values.yaml. For example:

```bash
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```
## Using the lookup function

The lookup function can be used to look up resources in a running cluster. 
The synopsis of the lookup function is 
```bash
lookup apiVersion, kind, namespace, name -> resource or resource list.
```
Both name and namespace are optional and can be passed as an empty string ("").

The following combination of parameters are possible:

|Behavior|  Lookup function|
|--|--|
| `kubectl get pod mypod -n mynamespace` | `lookup "v1" "Pod" "mynamespace" "mypod"` |
| `kubectl get pods -n mynamespace` | `lookup "v1" "Pod" "mynamespace" ""` | 
| `kubectl get pods --all-namespaces` | `lookup "v1" "Pod" "" ""` |
|`kubectl get namespace mynamespace`  | `lookup "v1" "Namespace" "" "mynamespace"` |
| `kubectl get namespaces` | `lookup "v1" "Namespace" "" ""` |


When lookup returns an object, it will return a dictionary. This dictionary can be further navigated to extract specific values.
The following example will return the annotations present for the mynamespace object:
```bash
(lookup "v1" "Namespace" "" "mynamespace").metadata.annotations
```
When lookup returns a list of objects, it is possible to access the object list via the items field:
```bash
{{ range $index, $service := (lookup "v1" "Service" "mynamespace" "").items }}
		{{/* do something with each service */}}
{{ end }}
```
When no object is found, an empty value is returned. This can be used to check for the existence of an object.

Keep in mind that Helm is not supposed to contact the Kubernetes API Server during a helm template or a helm install|update|delete|rollback --dry-run, 
so the lookup function will return nil in such a case.

# Flow Control

Control structures (called "actions" in template parlance) provide you, the template author, with the ability to control the flow of a template's generation. Helm's template language provides the following control structures:

```bash
--> if/else for creating conditional blocks
--> with to specify a scope
--> range, which provides a "for each"-style loop
```

## if/else example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: {{ .Release.Name }}-configmap
data:
 myvalue: "Hello World"
 drink: {{ .Values.favorite.drink | default "tea" | quote }}
 food: {{ .Values.favorite.food | upper | quote }}
 {{- if eq .Values.favorite.drink "coffee" }}
 mug: true
 {{- end }}
```

## Modifying scope using "with"

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  namespace: {{ .Values.namespace }}
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end}}
  {{- if eq .Values.favorite.drink "coffee" }}
  mug: true
  {{- end }}
```
## Looping with the "range" action

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: {{ .Release.Name }}-configmap
data:
 myvalue: "Hello World"
 {{- with .Values.favorite }}
 drink: {{ .drink | default "tea" | quote }}
 food: {{ .food | upper | quote }}
 {{- end }}
 toppings: |-
 {{- range .Values.pizzaToppings }}
 - {{ . | title | quote }}
 {{- end }}
```
Variables
==============
In Helm templates, a variable is a named reference to another object. It follows the form $name. 
Variables are assigned with a special assignment operator: :=. We can rewrite the above to use a variable for Release.Name.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: {{ .Release.Name }}-configmap
data:
 myvalue: "Hello World"
 {{- $relname := .Release.Name -}}
 {{- with .Values.favorite }}
 drink: {{ .drink | default "tea" | quote }}
 food: {{ .food | upper | quote }}
 release: {{ $relname }}
 {{- end }}
```
Variables are particularly useful in range loops. They can be used on list-like objects to capture both the index and the value:

```yaml
toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
      {{ $index }}: {{ $topping }}
    {{- end }}
```
For data structures that have both a key and a value, we can use range to get both. For example, 
we can loop through .Values.favorite like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```
## Global variable
Variables are normally not "global". They are scoped to the block in which they are declared. 
Earlier, we assigned $relname in the top level of the template. That variable will be in scope for the entire template. 
But in our last example, $key and $val will only be in scope inside of the {{ range... }}{{ end }} block.

However, there is one variable that is always global - $ - this variable will always point to the root context. 
This can be very useful when you are looping in a range and you need to know the chart's release name.

```yaml
{{- range .Values.tlsSecrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .name }}
  labels:
    # Many helm templates would use `.` below, but that will not work,
    # however `$` will work here
    app.kubernetes.io/name: {{ template "fullname" $ }}
    # I cannot reference .Chart.Name, but I can do $.Chart.Name
    helm.sh/chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
    app.kubernetes.io/instance: "{{ $.Release.Name }}"
    # Value from appVersion in Chart.yaml
    app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
    app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
type: kubernetes.io/tls
data:
  tls.crt: {{ .certificate }}
  tls.key: {{ .key }}
---
{{- end }}
```
Named Templates
============================
```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

Conventionally, Helm charts put these templates inside of a partials file, usually _helpers.tpl. Let's move this function there:

```bash
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```
By convention, define functions should have a simple documentation block ({{/* ... */}}) describing what they do.

Even though this definition is in _helpers.tpl, it can still be accessed in configmap.yaml:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

## Setting the scope of a template

In the template we defined above, we did not use any objects. We just used functions. Let's modify our defined template to include the chart name and chart version:

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```
If we render this, the result will not be what we expect:

```bash
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moldy-jaguar-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart:
    version:
```
No scope was passed in, so within the template we cannot access anything in .. This is easy enough to fix, though. We simply pass a scope to the template:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
```
## The include function

For better handling output format / indentation include function prefered over template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
```

*********Important Points to remember*********
=============================
*The install order of Kubernetes types is given by the enumeration InstallOrder in kind_sorter.go (see the Helm source file https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go).







