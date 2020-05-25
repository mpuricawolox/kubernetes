## Configuring your local environment to work with kubectl

.....


## Setting an application running on AWS EC2

1.  Since we’re using [ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html), we should create a repository in the [AWS console](https://console.aws.amazon.com/ecr/repositories?region=us-east-1) and add a policy that allows access to the corresponding user or role or even from another AWS account. Here's a sample policy in JSON format.

```
{
	"Version": "2008-10-17",
	"Statement": [{
		"Sid": "new statement",
		"Effect": "Allow",
		"Principal": {
			"AWS": [
				"arn:aws:iam::737897690563:root",
				"arn:aws:iam::426645307445:role/ecosis-global-eks-rol",
				"arn:aws:iam::917222561357:root"
			]
		},
		"Action": [
			"ecr:BatchCheckLayerAvailability",
			"ecr:BatchGetImage",
			"ecr:CompleteLayerUpload",
			"ecr:DescribeImages",
			"ecr:DescribeRepositories",
			"ecr:GetDownloadUrlForLayer",
			"ecr:GetLifecyclePolicy",
			"ecr:GetRepositoryPolicy",
			"ecr:InitiateLayerUpload",
			"ecr:ListImages",
			"ecr:PutImage",
			"ecr:UploadLayerPart"
		]
	}]
}
```

In the example we're given all the basic repository operations permissions to an `eks` role in the same account and to the root role for two other accounts. These way they can, for example, set images in the repository.

2.  To create objects in the cluster you can use the command `kubectl apply -f manifest.yaml` which is an idempotent method or  `kubectl create -f manifest.yaml` that will create the object defined in the manifest if it doesn't already exist.

3.  Create a deployment object with a default image using kubectl. After this, the application can be deployed using the `kubectl set image` command from a console or from the pipeline system you're using (e.g. bitbucket pipelines). Here's an example yaml file for a deployment:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: arauco-map-web
  name: arauco-map-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: arauco-map-web
  template:
    metadata:
      labels:
        app: arauco-map-web
    spec:
      containers:
      - image: 426645307445.dkr.ecr.us-east-1.amazonaws.com/arauco-map:b7ae14f2bbc87600f56340a3d5600af3d34be842
        imagePullPolicy: Always
        name: arauco-map-web
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: registry
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

Run the command:

 ```
    kubectl create -f deployment.yaml
 ```

So now everytime a `set image` is executed this controller is going to pull the image from the repository and try to create a container and initialize it.

4.  Create a NodePort service that points to the application. (Use the deployment label as a reference). Here's an example yaml:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: arauco-map-web-svc
  name: arauco-map-web-svc
spec:
  type: NodePort
  selector:
    app: arauco-map-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Notice that in the selector section, in the app attribute we're using the same name as in the labels app parameter of the deployment file, this way we link the service with the corresponding deployment. Now you can run the command:
 ```
    kubectl apply -f service.yaml
 ```

5.  Create a security group with the propper inbound and outbound rules. Here's an example:

![sg inbound rules example](https://raw.githubusercontent.com/mpuricawolox/kubernetes/master/sg-inbound-rules-example.png)

![sg outbound rules example](https://raw.githubusercontent.com/mpuricawolox/kubernetes/master/sg-outbound-rules-example.png)

6.  Create a kubectl ingress which is going to configure an ALB (load balancer) and make the needed settings (inbound rules and target groups) so the traffic to the load balancer is redirected to the service. You should use a security group with the propper rules. Here's an example yaml:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig":
      { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:917222561357:certificate/758a9a18-a090-4f18-946e-548822bca9e3
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/security-groups: sg-0df1578c2b434c963
    kubernetes.io/ingress.class: alb
  generation: 1
  labels:
    app: arauco-map-web
  name: arauco-map-web
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: ssl-redirect
          servicePort: use-annotation
        path: /*
      - backend:
          serviceName: arauco-map-web-svc
          servicePort: 80
        path: /*
```

Now run the command:

  ```
    kubectl apply -f ingress.yaml
 ```


After that you’re going to have an ELB url. So you will probably need a DNS so your users will find a nice and readable URL.

## Road map
- Add local environment configurations for kubectl working with AWS profiles
- Add details about what represents the parameters in the yaml files
