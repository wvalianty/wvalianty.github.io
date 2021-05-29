![k8s-crd-group-version](http://wyong.cn/images/blog/k8s/crd/crd_object_group_version.png)

![k8s-crd-controller-workprocess](http://wyong.cn/images/blog/k8s/crd/controller_work_process.png)

Kuberentes的API对象由三部分组成，通常可以归结为： /apis/group/version/resource，例如

    apiVersion: Group/Version
    kind: Resource

APIServer在接收一个请求之后，会按照 /apis/group/version/resource的路径找到对应的Resource类型定义，根据这个类型定义和用户提交的yaml里的字段创建出一个对应的Resource对象

CRD机制：
（1）声明一个CRD，让k8s能够认识和处理所有声明了API是"/apis/group/version/resource"的YAML文件了。包括：组（Group）、版本（Version）、资源类型（Resource）等。
（2）编写go代码，让k8s能够识别yaml对象的描述。包括：Spec、Status等
（3）使用k8s代码生成工具自动生成clientset、informer和lister
（4） 编写一个自定义控制器，来对所关心对象进行相关操作
