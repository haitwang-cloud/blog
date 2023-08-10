> 本文是 [Using The Helm Tpl Function To Refer Values In Values Files](https://austindewey.com/2021/02/22/using-the-helm-tpl-function-to-refer-values-in-values-files/)的中文翻译版本，内容有删减

我正在为客户编写Helm Charts，以帮助他们更轻松地将应用程序部署到Kubernetes。通常，他们会问我是否可以在他们的values.yaml文件中引用值，以避免重复的字符串。例如，不是这样做：

```yaml
image: myregistry.io/dev/myImage:1.0
env:
  - name: ENVIRONMENT
    value: dev
```

他们期望用下面的方式来优化他们的values文件：

```yaml
environment: dev
image: myregistry.io/{{ .Values.environment }}/myImage:1.0
env:
  - name: ENVIRONMENT
    value: “{{ .Values.environment }}”

```

其实我们可以像上面所示的那样在values文件中引用值，具体是您可以通过在Helm chart中使用**tpl**函数来实现这一点。

## Tpl 函数

根据 [Helm 文档](https://helm.sh/docs/howto/charts_tips_and_tricks/), “**tpl** 函数允许开发人员在模板中使用字符串作为模板。” 假设Deployment资源模板包含以下片段：

```yaml
    spec:
      containers:
        - name: main
          image: {{ .Values.image }}
          env:
            {{- toYaml .Values.env | nindent 12 }}

```

这个代码片段**不会**允许用户在他们的values文件中引用其他值。如果他们尝试这样做，"helm install"或"helm template"的结果会看起来像这样：

```yaml
    spec:
      containers:
        - name: main
          image: myregistry.io/{{ .Values.environment }}/myImage:1.0
          env:
            - name: ENVIRONMENT
              value: {{ .Values.environment }}

```
如你缩减，helm引擎渲染了字面值`{{.Values.environment}}`而不是该值的实际设置。

幸运的是，使用**tpl**函数很容易解决这个问题。下面是使用tpl函数重写的部署片段：

```yaml
    spec:
      containers:
        - name: main
          image: {{ tpl .Values.image . }}
          env:
            {{- tpl (toYaml .Values.env) . | nindent 12 }}

```

现在，当你运行“helm install”时，Helm模板引擎将能够用values文件中的设置替换`{{.Values.environment}}`：

```yaml
    spec:
      containers:
        - name: main
          image: myregistry.io/dev/myImage:1.0
          env:
            - name: ENVIRONMENT
              value: 'dev'

```

`tpl`函数的语法与`_include_`函数类似，我在[my post about Named Templates](https://austindewey.com/2020/08/09/how-to-reduce-helm-chart-boilerplate-with-named-templates/)中进行过介绍。`tpl`函数第一个参数是您想要作为模板呈现的字符串，第二个参数是作用域，通常会是`.` （点号），除非您在循环中执行tpl，这种情况下您会想要使用`$`。

在values.yaml中，字符串应该以以`{{`开头，**并且要将值用引号`()`括起来**。否则Helm模板引擎会抛出错误。

## Tpl安全性顾虑
---------------------------

Keep in mind that tpl is used to render any user input as a template, and can be used for use cases other than simply referencing other Helm values. Consider your user provides the following values file:

请记住，`tpl`函数用于将输入渲染为模板，同时也可以用于除了简单引用其他Helm值之外的场景。考虑一下，您的用户提供了以下的values文件：

```yaml
environment: dev
image: myregistry.io/{{ .Values.environment }}/myImage:1.0
env:
  - name: PODS
    value: '{{ lookup "v1" "Pod" "" "" }}'

```

这是恶意输入吗？也许是，也许不是。无论如何，如果用户有权限执行此操作，这个输入会显示集群中的所有Pod。请始终在集群中使用适当的RBAC来防止用户读取或写入不应被允许的资源。
