Kubernetes 실습가이드

```
az login [엔터]
로그인화면이 팝업되면 azure 계정을 클릭하여 로그인시킨다.
az aks get-credentials --resource-group (리소스그룹이름 ) --name (aks 클러스터이름)
az acr login --name (azure container registry 서비스이름)
az aks update -n (클러스터이름) -g (리소스그룹이름) --attach-acr (acr 이름)

```

** gklabacr01.azurecr.io 는 강사가 생성한 acr 서비스 입니다. 

​    반드시 각자가 생성하신 azure container registry 서비스 이름으로 변경하시고 작업하셔야 합니다. 

1. kubectl 명령어 연습하기

   ```
   kubectl [엔터]
   source <(kubectl completion bash)
   echo "source <(kubectl completion bash)" >> ~/.bashrc
   
   alias k=kubectl
   complete -F __start_kubectl k
   
   kubectl config view
   cat $HOME/.kube/config
   
   kubectl get componentstatus
   kubectl cluster-info
   kubectl api-resources
   kubectl api-resources --namespaced=[ture | false]
   kubectl explain pods
   
   kubectl get nodes
   kubectl get namespaces
   ```

2. 벡엔드 / 프로트엔드 어플리케이션을 pod 로 배포하자

   ```
   docker tag gowebapp:v1 (acr이름).azurecr.io/gowebapp:v1
   
   docker tag gowebapp-mysql:v1 (acr이름).azurecr.io/gowebapp-mysql:v1
   
   docker push (acr이름).azurecr.io/gowebapp:v1
   docker push (acr이름).azurecr.io/gowebapp-mysql:v1
   ```

   ```
   http://portal.azure.com 에 로그인하여 이미지가 정상적으로 push 되어있는지 확인합니다.
   ```

   ![image-20200520004129807](C:\Users\young\AppData\Roaming\Typora\typora-user-images\image-20200520004129807.png)

   

   ```
   vim gowebapp-mysql.yaml
   :set paste ==> 들여쓰기가 깨지지 않고 copy/paste 가 된다.
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: gowebapp-mysql
     labels:
       app: gowebapp-mysql
       tier: backend
   spec:
     containers:
     - name: gowebapp-mysql
       env:
       - name: MYSQL_ROOT_PASSWORD
         value: mypassword
       image: gklabacr01.azurecr.io/gowebapp-mysql:v1
       ports:
       - containerPort: 3306
   ```

   ```
   kubectl apply -f gowebapp-mysql.yaml
   kubectl get pods
   kubectl expose pod gowebapp-mysql --type=ClusterIP
   ```

   

   ```
   vim gowebapp.yaml
   
   apiVersion: v1
   kind: Pod
   metadata:
     name: gowebapp
     labels:
       app: gowebapp
       tier: frontend
   spec:
     containers:
     - name: gowebapp
       env:
       - name: MYSQL_ROOT_PASSWORD
         value: mypassword
       image: gklabacr01.azurecr.io/gowebapp:v1
       ports:
       - containerPort: 80
         protocol: TCP
      
   ```

   ```
   kubectl apply -f gowebapp.yaml
   kubectl get pods
   kubectl expose pod gowebapp --type=LoadBalancer 
   kubectl get service,endpoints
   ```

3. 브라우저를 띄우고

   http://External-IP  로 접속해보기

4. gowebapp 파드와 gowebapp-mysql 파드에 대한 정보 확인해보기

   ```
   kubectl describe pod gowebapp
   kubectl describe pod gowebapp-mysql
   
   kubectl get pods -o wide 
   kubectl get pods --show-labels
   kubectl get pods -l app
   kubectl get pods --selector tier=frontend
   kubectl get pods gowebapp -o yaml 
   kubectl logs gowebapp
   
   ```

5. gowebapp, gowebapp-mysql 파드 모두 삭제하기

   ```
   kubectl delete pods gowebapp,gowebapp-mysql 
   kubectl delete service gowebapp,gowebapp-mysql
   ```

6. gowebapp, gowebapp-mysql 어플리케이션을 deployment 를 통해서 배포해보자

   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata: 
     name: gowebapp-mysql
     labels:
       app: gowebapp-mysql
       tier: backend
   spec:
     replicas: 1
     strategy: 
       type: Recreate
     selector:
       matchLabels:
         app: gowebapp-mysql
         tier: backend
     template:
       metadata:
         labels:
           app: gowebapp-mysql
           tier: backend
       spec:
         containers:
         - name: gowebapp-mysql
           env:
           - name: MYSQL_ROOT_PASSWORD
             value: mypassword
           image: gklabacr01.azurecr.io/gowebapp-mysql:v1
           ports:
           - containerPort: 3306
   ```

   ```
   kubectl apply -f gowebapp-mysql-deployment.yaml
   kubectl get pods
   kubectl get deployment
   kubectl get rs
   
   kubectl expose deployement gowebapp-mysql --type=ClusterIP
   
   kubectl get service,endpoints
   ```

   

   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata: 
     name: gowebapp
     labels:
       app: gowebapp
       tier: frontend
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: gowebapp
         tier: frontend
     template:
       metadata:
         labels:
           app: gowebapp
           tier: frontend
       spec:
         containers:
         - name: gowebapp
           env:
           - name: MYSQL_ROOT_PASSWORD
             value: mypassword
           image: gklabacr01.azurecr.io/gowebapp:v1       
           ports:
           - containerPort: 80
   ```

   ```
   kubectl apply -f gowebapp-deployment.yaml
   kubectl get pods
   kubectl get deployment
   kubectl get rs
   
   kubectl expose deployement gowebapp --type=LoadBalancer
   
   kubectl get service,endpoints
   ```

7. gowebapp:v2 이미지 태그 생성해서 docker hub 에 업로드하기

   (인증오류가 발생하면 az acr login --name (acr 서비스이름) 로 다시 로그인 하기 )

   ```
   docker tag gowebapp:v1 gklabacr01.azurecr.io/gowebapp:v2 
   docker push gklabacr01.azurecr.io/gowebapp:v2
   docker image ls
   ```

   

8. gowebapp deployment 변경하기

   ```
   kubectl scale deployment gowebapp --replicas=2
   kubectl get pods 
   kubectl describe deployment gowebapp
   kubectl get rs
   
   kubectl set image deployment gowebapp gowebapp=gklabacr01.azurecr.io/gowebapp:v2
   kubectl rollout history deployment gowebapp
   kubectl rollout undo deployment gowebapp --to-revision=1 
   or
   kubectl rollout undo deployment gowebapp 
   ===================================================================================
   kubectl edit deployemnt gowebapp
   
   vim gowebapp-deployemnt.yaml 
   변경할 내용 편집
   
   kubectl replace -f gowebapp-deployment.yaml 
   or 
   kubectl apply -f gowebapp-deployment.yaml
   ```

   

9. gowebapp-mysql 파드에게 azure 가 제공하는  persistent volume  할당하기

   ```
   kubectl delete deployment gowebapp-mysql
   kubectl delete service  gowebapp-mysql
   ```

   

   ```
   kubectl get sc 
   ```

   ```
   vim pvc.yaml
   
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: azure-managed-disk
   spec:
     accessModes:
     - ReadWriteOnce
     storageClassName: managed-premium
     resources:
       requests:
         storage: 5Gi
   
   ```

   ```
   kubectl get pvc
   
   ```

   ```
   vim gowebapp-mysql-pv.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata: 
     name: gowebapp-mysql
     labels:
       app: gowebapp-mysql
       tier: backend
   spec:
     replicas: 1
     strategy: 
       type: Recreate
     selector:
       matchLabels:
         app: gowebapp-mysql
         tier: backend
     template:
       metadata:
         labels:
           app: gowebapp-mysql
           tier: backend
       spec:
         containers:
         - name: gowebapp-mysql
           env:
           - name: MYSQL_ROOT_PASSWORD
             value: mypassword
           image: gklabacr01.azurecr.io/gowebapp-mysql:v1
           ports:
           - containerPort: 3306
           volumeMounts:
           - mountPath: /var/lib/mysql
             name: mysql
         volumes:
         -  name: mysql
            persistentVolumeClaim:
              claimName: azure-managed-disk
   
   
   
   ```

   ```
   kubectl apply -f gowebapp-mysql-pv.yaml
   kubectl expose deployment gowebapp-mysql 
   
   kubectl get pods
   kubectl get deployment
   kubectl describe deployment gowebapp-mysql
   kubectl get service,endpoints
   ```

   브라우저를 띄우고 http://Node_IP:NodePort 로 접속해서 계정생성하고 로그인해서  note 작성해보기

   ```
   kubectl delete pod gowebapp-mysql-xxxxx 
   kubectl get pods
   ```

   기존 pod 삭제가 되었지만 deployment의 rs controller 에 의해서 새로운 Pod 가  생성되지만 위에서 생성한 계정으로 여전히 로그인이 가능하다.

10. configmap 을 이용하여 환경변수값을 중앙에서 안전하게 관리하기

   ```
   kubectl delete deployment gowebapp
   ```

   

   ```
   kubectl create configmap mysqlpwd --from-literal=password=mypassword
   kubectl describe configmap mysqlpwd
   ```

   ```
   vim gowebapp-deployment-config.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata: 
     name: gowebapp
     labels:
       app: gowebapp
       tier: frontend
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: gowebapp
         tier: frontend
     template:
       metadata:
         labels:
           app: gowebapp
           tier: frontend
       spec:
         containers:
         - name: gowebapp
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               configMapKeyRef:
                   name: mysqlpwd
                   key: password
           image: gklabacr01.azurecr.io/gowebapp:v1       
           ports:
           - containerPort: 80
   
   
   ```

   ```
   kubectl get svc,ep
   로 확인해보고 gowebapp 의 external IP 로 브라우저를 통해서 접속해보자
   
   http://External_IP   
   ```

   

11. 간단한 job 연습해보기

    ```
    vim job.yaml
    
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: pi
    spec:
      template:
        metadata:
          name: pi
        spec:
          containers:
          - name: pi
            image: perl
            command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          restartPolicy: Never
    
    kubectl apply -f job.yaml
    kubectl get pods
    kubectl logs (pod-name)
    
    kubectl delete job pi 
    
    ```

    

12. 간단한 cronjob 실행해보기

    ```
    vim cronjob.yaml
    
    apiVersion: batch/v1beta1
    kind: CronJob
    metadata:
      name: hello
    spec:
      schedule: "*/1 * * * *"
      jobTemplate:
        spec:
          template:
            spec:
              containers:
              - name: hello
                image: busybox
                args:
                - /bin/sh
                - -c
                - date; echo Hello from the Kubernetes cluster
              restartPolicy: OnFailure
    
    
    watch kubectl get pods
    kubectl logs (pod이름)
    ```

13. Ingress Controller 설치하기

    ```
    1. 간단하게 서비스를 하나더 추가한다.
        kubectl create deployment nginx --image=nginx
        kubectl expose deployment nginx --type=NodePort --port=80
    2. kubectl get svc, ep 
       로 기존의 gowebapp 서비스와 새로생성한 nginx 서비스를 확인한다.
       
    3. ingress controller 를 위한 rbac 구성작업을 한다.
       
       vim ingress-rbac.yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: traefik-ingress-controller
    rules:
      - apiGroups:
          - ""
        resources:
          - services
          - endpoints
          - secrets
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - extensions
        resources:
          - ingresses
        verbs:
          - get
          - list
          - watch
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: traefik-ingress-controller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: traefik-ingress-controller
    subjects:
    - kind: ServiceAccount
      name: traefik-ingress-controller
      namespace: kube-system
    
    =====================================================================
    
    kubectl apply -f ingress-rbac.yaml
    
    4. ingress controller 를 생성하고 LoadBalancer 타입으로 서비스를 생성한다.
    
    vim ingress-controller.yaml
    
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: traefik-ingress-controller
      namespace: kube-system
    ---
    kind: DaemonSet
    apiVersion: apps/v1
    metadata:
      name: traefik-ingress-controller
      namespace: kube-system
      labels:
        k8s-app: traefik-ingress-lb
    spec:
      selector:
        matchLabels:
          name: traefik-ingress-lb
      template:
        metadata:
          labels:
            k8s-app: traefik-ingress-lb
            name: traefik-ingress-lb
        spec:
          serviceAccountName: traefik-ingress-controller
          terminationGracePeriodSeconds: 60
          hostNetwork: True
          containers:
          - image: traefik:1.7.13
            name: traefik-ingress-lb
            ports:
            - name: http
              containerPort: 80
              hostPort: 80
            - name: admin
              containerPort: 8080
              hostPort: 8080
            args:
            - --api
            - --kubernetes
            - --logLevel=INFO
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: traefik-ingress-service
      namespace: kube-system
    spec:
      selector:
        k8s-app: traefik-ingress-lb
      type: LoadBalancer 
      ports:
        - protocol: TCP
          port: 80
          name: web
        - protocol: TCP
          port: 8080
          name: admin
    
    ======================================================
     kubectl apply -f ingress-controller.yaml
     
     kubectl get svc,ep --namespace=kube-system
     
     
     5. ingress rule 생성하기
     예를 들어  http://gowebapp.example.com 으로 접속하면 gowebapp 서비스로 보내고 
     http://nginx.example.com 으로 접속하면 nginx 서비스로 트래픽을 보내고자 한다면,
     
     vim ingress-rule.yaml
     
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: ingress-test
      annotations:
        kubernetes.io/ingress.class: traefik
    spec:
      rules:
      - host: gowebapp.example.com
        http:
          paths:
          - backend:
              serviceName: gowebapp
              servicePort: 80
      - host: nginx.example.com
        http:
          paths:
          - backend:
              serviceName: nginx  
              servicePort: 80
     ====================================================================
     
     kubectl apply -f ingress-rule.yaml
     
     kubectl describe ingress ingress-test 로 rule 를 확인해본다.
     
     ======================================================================
     
     
    ```

    

    6. kubectl get svc, ep --namespace=kube-system 을 통해서 ingress-controller 의 External_IP 를 확인한다

    ![image-20200521184829780](C:\Users\young\AppData\Roaming\Typora\typora-user-images\image-20200521184829780.png)

​       7. 접속을 확인한다.

​          데스크탑의  시작메뉴 -> 오른쪽마우스 -> windows powershell (관리자) 클릭 

​           cd\

​			cd windows\system32\drivers\etc 

​            notepad hosts  를 실행하여 맨 하단에 다음처럼 입력하세요.

​            ingress-controller-exteranal-ip               gowebapp.example.com  nginx.example.com



​           저장하고 나서.

​            브라우저를 켜고

​           http://gowebapp.example.com

​           http://nginx.example.com 을 확인해보세요.



14. Kubernetes Dashboard 을 이용한 모니터링

    ```
    1.kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
    
    2. az aks browse --resource-group myResourceGroup --name myAKSCluster
       클릭하면 browser 가 열리고 http://127.0.0.1:8001 로 접속을 한다.
    만약 브라우저가 자동으로 열리지 않으면 직접 브라우저를 열어서 
    http://127.0.0.1:8001 로 접속해본다.
    
      * 만약 브라우저로 접속이 안되면 ctrl+c  누르고 
      다른 포트로 다시 시도해보자
      az aks browse --resource-group myResourceGroup --name myAKSCluster --listen-port 8002 
    ```

    

​     

​     

