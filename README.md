
AVI背景知识：

1，avi network：一家印度的技术型创业公司，被vmware收购，做的东西是各种software定义的东东，例如loadbalnacer等
2，AVI cloud，感觉是在云基础设施上用avi技术逻辑出来的一个东东，里面虚拟出virtual service等概念。avi controller通过avi cloud来管理SE；avi cloud来处理IPAM和DNS等功能
3，avi controller：avi cloud里的控制器,也负载处理IPAM，sdn等功能
4，Vantage: avi网络数据面处理逻辑，具体到AKO里，概念上等同于下面的Service Engine:
       Vantage will process the client connetcion or request againt a list of settings,policy and profiles, then send valid client traffic to a backend server that is listed  as member of the virtual server's pool
        Create a cloud in Avi vantage platform
5，virtual service: avi网络中，对外暴露的一个访问入口，可以理解为vip:port, virtual service 对应的真正的后端维护在一个逻辑的 pool对象里 
       A virtual service can be thought of as an ip address that vantage is listenging on
6，Service Engine: AKO架构里avi cloud里数据转发平面，根据路由规则等转发流量到真正的后端


AKO（avi k8s/openshift operator）
1，就是k8s里面的一个operator，以pod方式部署在k8s里
2，监听k8s里ingress/lb类型service等对象，然后通过CRD将这些对象转换为AVI Controller API
3，根据监听到的ingress/lb类型service的变化，调AVI Controller 的api来完成对 Service Engine的配置

部署架构如下：


从上面的描述可以看出，AKO架构是构建在avi cloud之上，从商业角度看可以理解为是avi network的生态。
1.The Avi infrastructure cloud will be used to place the virutal services that are create for k8s/openshift application to....
2.A single Avi controller cluster supports multiple cocurrent vcenter cloud(or other support avisuport)（avi cluster其实就是一块请求的入口区，提供了virtual service等可编程的控制对象？，avi controller需要在avi cloud之上，才能创建和配置virtual service）
以lb类型svc为例：


apiVersion: v1
kind: Service
metadata:
  name: avisvc-lb
  namespace: red
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    name: eighty
  selector:
    app: avi-server

lb中有个loadbalancerIP，以及这里的port信息，这个应该就是virtual service对外暴露服务的ip和端口；
AKO 感知到创建lb类型svc后，在AVI cloud里创建一个 virtual service, 并给这个virtual server保留一个作为访问入口的IP。
给这个virtual service维护的后端pool，就是k8s里相应loadbalancer类型svc对应的后端POD，
至于客户端请求到VIP后，怎么寻址到pod IP里，本文最后部分会介绍。


若AKO感知到k8s里创建ingress对象，也会调AVI controller的api来创建一个virtual service对象并配置它，不过此时，这个virtual service对象并不是这个ingress独享的，
后续创建的ingress对象也能共享。
（打个简单的比喻就明白了：www.yitu.io/a,  www.yitu.io/b,  www.yitu.io/c  这三个ingress共享www.yitu.io这一个virtual service）。
这些ingress对应的后端组成了这个virtual service 的pool group， 不过每个pool通过一个label来区分，label就是这个ingress yaml里hosts字段的FQDN，当客户端一个请求到达这个virtual server时，会解析其url，根据最长前缀匹配原则，确定它对应的那个pool，这样这个请求就对应到相应的ingress对应的后端pod了。


=========================================================================================
AKO里将k8s里的ingress，lb类型service，nodeport类型的service（可能还有其他的如ip rule等等）如何在avi cloud里暴露出去的大概逻辑就是上面这些了，接下来还有一块逻辑要弄清楚：当client端的请求到达avi cloud后（访问的是virutal server）, 作为数据转发面的Service Engine是如何将请求路由到k8s集群的pod里的：
A, POD在集群外部不可路由的场景(Canal, calico,Antrea,Flannel...)
此种场景下，每个node上一般都被分配了一个pod的cidr，对具体的node节点上的pod来说，node节点相当于一个路由器/网关，此时pod和SE之间的联通取决于SE和node之间是部署在同一网络。
如果SE和node部署在同一网络，可以开启AKO里的静态路由功能，AKO将感知到每个节点上cidr信息下发给avi controller，这样SE那边只需要将目的地是pod的流量下一跳到相应的node节点即可。

如果SE和node不是部署在同一网络的场景时，SE无法将访问pod的请求直接丢到node上(因为此时中间必然会经过一跳，中间这跳的节点不知道要将pod的流量打倒node上就丢包了)，很显然此种情况，只有nodeport类型的svc还能继续玩得转咯。。。

B，POD在集群外可被路由的场景(NSX-T CNI， AWS CNI， Azure CNI等)
pod的子网在集群外部能被直接路由，此种情况就不需要做什么配置啦。。。AKO里的静态路由功能也没必要开着了

===========================================================================================

从上面的描述可以看出，AKO主要是在l7层面（应该能提供服务暴露，负载均衡，流控，检查等功能），既然是在l7，跟这个系统相关还有一个域名解析的组件。
既然是自己做的域名解析，那么可以控制访问ingress的流量哪些打到avi cloud1, 哪些打到avi cloud2 ,更何况一个avi cloud本身就能对接多个k8s集群(SE Group每个集群独有即可) ==> 自然很容易做到vmware 这个ppt里说的跨集群了




