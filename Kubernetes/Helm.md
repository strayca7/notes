[Helm 官方文档](https://helm.sh/zh/docs/)

# 命令行

## create

该命令创建chart目录和chart用到的公共文件目录

```bash
 helm create [NAME] [flags]
```

比如'helm create foo'会创建一个目录结构看起来像这样：

```bash
foo/
├── .helmignore   # Contains patterns to ignore when packaging Helm charts.
├── Chart.yaml    # Information about your chart
├── values.yaml   # The default values for your templates
├── charts/       # Charts that this chart depends on
└── templates/    # The template files
    └── tests/    # The test files
```





## install

安装一个 Helm Chart。

```bash
$ helm install [release-NAME] [CHART] [flags]
```

Flags:

- -f，--values string：指定 values 的 yaml 文件，可以使用多个 -f 参数，最右侧指定的文件优先级最高。

	比如，如果两个文件myvalues.yaml和override.yaml 都包含名为'Test'的可以，override.yaml中的值优先：

	```bash
	$ helm install -f myvalues.yaml -f override.yaml  myredis ./redis
	```

- --set [key]=[values]：在命令行传递配置

- --set-string [key]=[string]：强制使用字符串

可以用 `helm install --debug --dry-run goodly-guppy ./my-chart`，将只会渲染模板内容到控制台（用于测试），而不会部署到 Kubernetes 中。使用`--dry-run`会让你变得更容易测试，但不能保证Kubernetes会接受你生成的模板。 最好不要仅仅因为`--dry-run`可以正常运行就觉得chart可以安装。



## template

在本地渲染 chart 模板，不发送给 Kubernetes，默认直接打在命令行。

```bash
helm template [release-name] [chart] [flags]
```

Flags：

- -f，--values string：指定 values 的 yaml 文件
- --output-dir string：指定输出文件路径



# Chart

Helm 模板参考 Go 模板的

## 流控制

### If/Else

```yaml
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

如果是以下值时，管道会被设置为 *false*：

- 布尔false
- 数字0
- 空字符串
- `nil` (空或null)
- 空集合(`map`, `slice`, `tuple`, `dict`, `array`)

在所有其他条件下，条件都为true。

示例 `values.yaml`：

```yaml
favourite:
  drink: coffee
  food: pizza
```

配置文件：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}mug: "true"{{ end }}
```

运行结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: "true"
```



### 控制空格

`{{ }}` 在模板引擎运行后*移除了* `{{` 和 `}}` 里面的内容，但是留下的空白完全保持原样，所以会保留原有的空白（空格 + 换行）。

首先，模板声明的大括号语法可以通过特殊的字符修改，并通知模板引擎取消空白。`{{- ` (包括添加的横杠和空格)表示向左删除空白， 而` -}}` 表示右边的空格应该被去掉。 *一定注意空格就是换行*

> [!WARNING]
>
> 要确保`-`和其他命令之间有一个空格。 `{{- 3 }}` 表示“删除左边空格并打印3”，而`{{-3 }}`表示“打印-3”。

这样就可以去掉新加的空白行：

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
  mug: "true"
  {{- end }}
```



### with

`with` 用于为特定对象设定当前作用域(`.`)比如，我们已经在使用 `.Values.favorite`。 修改配置映射中的 `.` 的作用域指向 `.Values.favorite`：

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
```

使用 `with` 时需要用 `{{ end }}` 指定作用域范围，**但是在限定的作用域内，无法使用 `.` 访问父作用域的对象**。

如果在 `with` 作用域中需要使用父作用域的的对象有两种方法：

1. 提前结束 `with` 的作用域范围：

	```yaml
	{{- with .Values.favorite }}
	drink: {{ .drink | default "tea" | quote }}
	food: {{ .food | upper | quote }}
	{{- end }}
	release: {{ .Release.Name }}
	```

2. 用 `$` 从父作用域中访问 `Release.Name` 对象。当模板开始执行后 `$` 会被映射到根作用域，且执行过程中不会更改：

	```yaml
	{{- with .Values.favorite }}
	drink: {{ .drink | default "tea" | quote }}
	food: {{ .food | upper | quote }}
	release: {{ $.Release.Name }}
	{{- end }}
	```



### range

开始之前先添加在 `values.yaml` 文件添加一个披萨的配料列表：

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

在 `values.yaml` 中定义了一个 pizzaToppings 切片，修改模板把这个列表打印到配置映射中：

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
    # title 首字母大写
    # quote 引号引起
    - {{ . | title | quote }}
    {{- end }}    
```

`range` 会遍历列表中的值，并送入管道：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"   
```

`toppings: |-` 行是声明的多行字符串且每行去除结尾换行，而不是一个列表。

也可通过 Helm 模板的 `tuple` 实现快速创建列表让后迭代：

```yaml
sizes: |-
  {{- range tuple "small" "medium" "large" }}
  - {{ . }}
  {{- end }}    
```

上述模板会生成以下内容：

```yaml
sizes: |-
  - small
  - medium
  - large    
```

除了列表和元组，`range`可被用于迭代有键值对的集合（像`map`或`dict`）。



## 变量

变量，在模板中，很少被使用。但是我们可以使用变量简化代码，并更好地使用 `with` 和 `range`。

在上述 `with` 中，可以看到下面的代码会失效：

```yaml
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}
food: {{ .food | upper | quote }}
release: {{ .Release.Name }}
{{- end }}
```

Helm 模板中，变量是对另一个对象的命名引用。使用 `$name` 声明一个变量并用 Go 中的赋值 `:=` 为符号将变量 `$name` 赋值。

变量的值可以在 `with` 块中使用。

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

也可以在 `range` 中使用变量，语法规则符合 Go `for loop` 规范，包括slice、map 等遍历。

遍历 slice：

```yaml
  toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
      {{ $index }}: {{ $topping }}
    {{- end }}    
```

```yaml
  toppings: |-
      0: mushrooms
      1: cheese
      2: peppers
      3: onions   
```

遍历 map：

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

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eager-rabbit-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```



变量一般不是"全局的"。作用域是其声明所在的块。上面我们在模板的顶层赋值了 `$relname`。变量的作用域会是整个模板。 但在最后一个例子中 `$key` 和 `$val` 作用域会在 `{{ range... }}{{ end }}` 块内。