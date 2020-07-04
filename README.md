# helmchart
This repository is for Learning purpose , content has been taken from https://helm.sh/

Some points to be notes

URL for Template function : https://masterminds.github.io/sprig/

Values.yaml used for below examples
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

Template Functions and Pipelines
===================================
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

```yaml
Behavior								                                Lookup function
--------								                                ---------------
kubectl get pod mypod -n mynamespace	                  lookup "v1" "Pod" "mynamespace" "mypod"
kubectl get pods -n mynamespace			                    lookup "v1" "Pod" "mynamespace" ""
kubectl get pods --all-namespaces		                    lookup "v1" "Pod" "" ""
kubectl get namespace mynamespace		                    lookup "v1" "Namespace" "" "mynamespace"
kubectl get namespaces					                        lookup "v1" "Namespace" "" ""
```
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













