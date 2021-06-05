# Template-Charts

Template charts are one way to reduce redundancy and update hassle while working with many charts of similar structure. (And lets face it, most kubernetes deployments look very alike when you take a look from a bit afar)

## How they work

Helm comes with the subchart functionality, templating and integrated dependency management. If you put all of it together you get template charts. You create a chart containing only files to define templates without actually creating any resources themselves. (a common pattern for these files is to start them with a underscore(_))

A parent chart can now define a dependency to your template chart and invoke that template by creating a file containing this:

```
{{ template "templates.deployment" . }}
```

In this case 'deployment' can be replaced by all names you might have given to your template.
All the configuration of the added resource is now done via the values.yaml file.

## Why is this great?

### Profiting from learning curve without much maintenance
For starters, it can save a lot of work recreating ever so similar manifest files for kubernetes that can development start out simple, but develop more and more advanced features, that you want to profit from. 
What if you need a specific type of annotation or labels on all your resources but you keep forgetting some? A template chart can be a great way to counter this. Just add it to the template and all parent charts that depend on your chart have the update in a matter of "helm dep up".


### Setting standards

If your company wants to enforce some ground rules on naming patterns, security or just simple style, template charts can help by showing an example on how to do it.

Example charts from template-charts are easily created and adjusted for your personal need. Just make sure you document your possible configurations well.

### Resources on demand

Since your template chart only creates the templates, no resources are created. However new resources are added to your chart with one of code. How much easier can kubernetes become?
Need Ingress? Create file "template/ingress.yaml", use the template 'templates.ingress', add some values if needed and here we go.

## Why is it horrible?

### Update hell

Well all your charts now have a common dependency! Any Software Architect will tell you, that this is where things will go wrong. A bad version of your template chart might cause an upgrade to fail or a security leak because of an unprotected port or secret.

With all things code, this can be avoided by proper testing and sane version management. It is however something to be aware of with every change that you are making.

### Api written in stone

Once your template chart is written and in use by more than 5 running applications, changing the api is basically impossible. Imagine this:

You have decided that there should not always be a fixed port on the main container. Sometimes you need multiple ports, sometimes none at all or you want to use tls termination in your app instead of relying on a sidecar or service mesh. What do you do? Now your code looks like this:

```
ports:
    - name: http
        containerPort: {{ .Values.port }}
```

But you want to make it more generic like this:
```
ports:
    {{- with .Values.extraPorts }}
        {{ toYaml . | nindent 8 }}
    {{- end }}
```

A week later your colleagues (or customers) come to you and ask for a refactor of the name to
```
ports:
    {{- with .Values.ports }}
        {{ toYaml . | nindent 8 }}
    {{- end }}
```

Every application that updates to a new version of your chart will have to change the syntax in the values.yaml. In order not to break them during automated updates, you will have to do major version upgrades. 2 in only a quite small change.

And these changes will happen a lot during the beginning of your template chart. So be sure to extract as much valuable information from many different charts and keep verything possible variable.

### The pretty yaml trap aka no values.yaml defaults
Look at this piece of template:

```
image: "{{ .Values.image.repository }}"
```
The values.yaml for it contains this:
```
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""
```

Great, we have a hierarchy of values and sane defaults.

Let's put this in our template-charts/values.yaml and render our parent-chart with an empty values.yaml.

```
Error: template: parent-chart/charts/template-chart/templates/_deployment-template.yaml:37:28: executing "templates.deployment" at <.V
alues.image.repository>: nil pointer evaluating interface {}.repository

Use --debug flag to render out invalid YAML
```

You might have guessed it from the title, but template-charts can not define defaults for parent charts in their values.yaml. Defaults will have to be added into the template directly. With this comes a bit of a problem when working with deeper hierarchies. Every object in the hierarchy tree will have to be checked for "nil" prior to being used. And how do you react towards it then? Error out and tell the parent-chart user to add the values? What if things have been working with the existing configuration for a while now and your new feature that no one is using yet, is breaking every deployment?

Here we go again, update hell...

Again, smart design of your templates and values.yaml file structure can protect you here. Flat hiearchies, preferably root values with longer names and _helper.tpl files that check for nil can protect you from breaking something while also keeping your code clean.

### How is it helping if all my config is in the values.yaml now?

Another symptom of this, is that more and more complex configuration wanders of into the values.yaml.

Let's have a look into this:
```
{{- with .Values.extraContainers }}
    {{ toYaml . | nindent 8 }}
{{- end }}
```

Considering everything we have read so far, we might think. Great, this keeps me flexible in my charts. 