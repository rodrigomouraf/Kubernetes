# Kubernetes

## Motivação

Kubernetes assim como o Docker Swarm, pode ser usado como orquestrador de containers Docker, mas não para por aí, ele também contêm diversos recursos para implementar desde persistência de dados, pods que seria um encapsulamento de um container, deploy, entre outros. Resumidamente no mundo Docker criamos, produzimos e gerenciamos containers já no mundo do Kubernets fazemos a mesma coisa só que para pods.

Em cenários que temos uma aplicação consumindo todo o poder computacional de uma máquina, precisamos de alguém que possa gerenciar essa aplicação, podendo escalar e gerenciar uma ou múltiplas máquinas trabalhando em conjunto, que nós chamaremos de cluster.

O Kubernetes é capaz de gerenciar e "levantar" tanto os containers como as aplicações de acordo com a necessidade, e distribuir os containers conforme necessidade computacional para a máquina mais adequada.

   

## Cluster

Dentro de um Cluster temos máquinas que podem receber as seguintes definições:

### Master

- Gerenciar o cluster
- Manter e atualizar o estado desejado
- Receber e executar novos comandos

Control Plane

| api  | c-m  | sched | etcd |
| ---- | ---- | ----- | ---- |



### Node

- Executar as aplicações

| Kubelet | k-proxy |
| ------- | ------- |

A comunicação entre nossa máquina e o kubernetes é feita através de uma API, essa ferramenta se chama kubectl.

## Pod

Um pod, pode conter um ou mais containers dentro de si, a partir da comunicação da nossa máquina com o kubectl para API, nós não vamos pedir a criação de um container mas sim de um pod.

Sempre que criamos um pod ele ganha um endereço de IP. Isso ganha algumas implicações, vamos abordar algumas:

Os containers dentro do pod são acessados a partir do IP do pod mais o mapeamento da sua porta, ou seja, dentro de um pod não deve haver containers utilizando a mesma porta.

 Outro ponto é que os pods são efêmeros, caso os containers dentro desse pod falhe o Kubernets tem total autonomia de criar um novo pod para substituir o antigo, mas não necessariamente com o mesmo IP.

Resumidamente, os containers dentro de um pod compartilham os mesmos namespaces de rede e processo, também podem compartilhar volumes. Isso se torna interessante pois eles compartilham o mesmo localhost, assim podendo se comunicar de uma forma muito mais fácil.

### Criar

```powershell
kubectl run <nome-pod> --image=<image>
```

ex:

```powershell
kubectl run nginx-pod --image=nginx:latest
```

### Verificar

```powershell
kubectl get pods
```

Caso queiramos acompanhar em tempo real o status de um pod podemos utilizar o comando:

```powershell
kubectl get pods --watch
```

Uma forma mais descritiva de analisar nossos pods

```bash
kubectl get pods -o wide
```

### Descrever

```powershell
kubectl describe pod <nome_pod>
```

ex:

```powershell
kubectl describe pod nginx-pod
```

### Editar

```powershell
kubectl edit pod <nome_pod>
```

ex:

```powershell
kubectl edit pod nginx-pod
```

### Deletar

```bash
kubectl delete pod <nome_pod>
```

### Executar em modo iterativo

```bash
kubectl exec -it <nome-pod> -- bash 
```



### Criando pods de maneira declarativa

Até o momento, o jeito que trabalhamos não foi a melhor maneira para se configurar nossas aplicações, a forma recomendada a se trabalhar principalmente no ambiente de produção é criando arquivos, pode ser tanto json como yaml, com as configurações de nossos pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: primeiro-pod-declarativo
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
```

Para rodarmos nosso arquivo:

```bash
kubectl apply -f .\<nome_arquivo>
```

#### Deletar pod declarativo

```bash
kubectl delete -f .\<nome_arquivo>
```

## SVC

Acima conversamos que quando um pod é deletado e recriado não existe nenhuma garantia que o mesmo usará a mesma porta anterior, então como garantir que outros pods continuem se comunicando com as outras aplicações? o kubernetes nos trás um excelente recurso chamado service (SVC), o svc é uma abstração que expõe as aplicações executadas em um ou mais pods, ou seja, utilizamos os IPs dos serviços, mas além disso, temos a possibilidade de utilizar DNS para nos comunicar entre um ou mais pods.

### Verificar

```powershell
kubectl get svc
```

Caso queiramos acompanhar em tempo real o status de um serviço podemos utilizar o comando:

```powershell
kubectl get svc --watch
```

### ClusterIP

Faz a comunicação de diversos pods dentro de um mesmo cluster usando o IP ou DNS do SVC. A comunicação é interna, ou seja, não conseguimos acessar o pod de fora do cluster.

Arquivo exemplo:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <nome_service>
spec:
  type: ClusterIP
  selector:
    app: <label_pod>
  ports:
    - port: <porta_servico>
      targetPort: <porta_pod>
```

### NodePort

Tipo de serviço que permite a comunicação com o mundo externo. Também funciona como ClusterIP.

```bash
kubectl get nodes -o wide
```

Arquivo exemplo:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <name_node>
spec:
  type: NodePort
  ports:
    - port: 80
    nodePort: 30000
  selector:
    app: <label_pod>
```

### Internal-IP

Aqui temos uma diferença entre o SO Linux e Windows, para o Windows ele faz o bind automaticamente do docker-desktop para o localhost.

Já no Linux basta utilizar o internalIP dado pelo comando get nodes.

### LoadBalancer

Basicamente é um ClusterIP que permite a comunicação entre uma máquina do mundo externo com o provedor.

Utilizam automaticamente os balanceadores de carga de cloud providers, por serem um Load Balancer também são um NodePort e ClusterIP ao mesmo tempo. 

 ## Config Map

Responsável por armazenar configurações que precisamos utilizar dentro de nossos pods. Podemos guardar dentro deles nossas informações para não acoplarmos os nossos recursos, permite também a reutilização das configs para outros pods.  

### Criar

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-configmap
data:
  MYSQL_ROOT_PASSWORD: q1w2e3r4
  MYSQL_DATABASE: empresa
  MYSQL_PASSWORD: q1w2e3r4
```

### Listar

```bash
kubectl get configmap
```

### Descrever

```bash
kubectl describe configmap db-configmap
```

## Replica Set

Caso um pod falhe, esse será o fim da vida do pod, como criar outro para assumir o lugar de maneira automática? Como vimos até o momento um pod é uma estrutura que encapsula um ou mais containers e um replica set pode encapsular ou gerenciar um ou mais pods, ou seja, é através do replica set que podemos reestartar os pods.

### Criar

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: <name_replicaset>
spec:
  template: 
    metadata:
      name: <name_pod>
      labels:
        app: <label_pod>
    spec:
      containers:
        - name: <name_container>
          image: <image_container>
          ports:
            - containerPort: <port_container>
          envFrom:
            - configMapRef:
                name: <name_configmap>
  replicas: <quantidade_desejada_replicas>
  selector:
    matchLabels:
      app: <label_pod>
```



### Listar

```bash
kubectl get rs
kubectl get rs -o wide
kubectl get rs --watch
```

### Detalhar ReplicaSet

```bash
kubectl get replicasets
```

## Deployment

É uma camada acima de um replica set, quando definimos um deployment estamos automaticamente definindo um replica set. Permitem o controle de versionamento das nossas imagens e pods.

### Arquivo de configuração

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <nome-deployment>
spec:
  replicas: 3
  template:
    metadata:
      name: <nome_pod>
      labels:
        app: <label_pod>
    spec:
      containers:
        - name: <name_container>
          image: <image_docker>
          #lembrando que o parâmetro port não é necessária, 
          #mas é muito bom usar para documentação.
          ports:
            - containerPort: <port_container>
  selector:
    matchLabels:
      app: <label_pod>
```

### Criar

```
kubectl apply .\<arquivo_deployment>
```

### Listar

```bash
kubectl get deployments
kubectl get deploy
```

### Histórico do Deployment

```bash
kubectl rollout history deployment nginx-deployment
```

### Criando uma revisão com change-cause

```bash
kubectl apply .\<arquivo> --record #a flag --record está para ser removida
```

Podemos alterar o change-cause

```bash
kubectl annotate deployment <nome_deployment> kubernetes.io/change-cause="mensagem com o motivo da alteração do arquivo"
```

### Retornando para outra versão

```bash
kubectl rollout undo deployment <nome_deployment> --to-revision=<revision_desejada>
```

## Persistindo dados

### Volumes

Se quisermos compartilhar arquivos entre containers dentro do Docker utilizamos volumes, no Kubernetes fazemos a mesma coisa, a única diferença é que os volumes dentro do kubernetes possuem ciclos de vida independentes dos containers, sendo dependentes dos pods.

Existem vários tipos de volumes no Kubernetes, aqui vamos usar o hostPath para exemplificarmos, mas alguns outros volumes interessantes a ser estudados são o awsElasticBlockStore, azureDisk e azureFile.

#### Arquivo de configuração

Vamos criar um pod simples, apenas para demonstração de como criar efetivamente um volume, funcionaria da mesma forma para um replicate set e um deployment.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <nome_pod>
spec:
  containers:
    - name: <nome_container_1>
      image: <imagem_container_1>
      volumeMounts:
        - mountPath: /<volume-dentro-do-container>
          name: <nome_diretorio_volume>
    - name: <nome_container_2>
      image: <imagem_container_2>
      volumeMounts:
        - mountPath: /volume-dentro-do-container
          name: <nome_diretorio_volume>
  volumes:
    - name: <nome_diretorio_volume>
      hostPath:
        path: <caminho_diretorio_volume>
        type: Directory
```

#### Entrando dentro de um dos containers

Podemos entrar dentro de um dos containers para analisar a arvore e os volumes

```bash
kubectl exec -it <nome_pod> --container <nome_container> -- bash
```

Se deletarmos esse pod, o volume criado dentro dele também será deletado, mas como nós criamos uma persistência em disco utilizando os volumes, quando recriarmos o pod ele utilizara o diretório e terá acessos aos arquivos anteriormente criados.

#### Config Linux:

Para o Linux precisamos criar nosso diretório dentro também de nossa máquina virtual.

```bash
minikube ssh
sudo mkdir <nome_diretorio_volume>
```

Uma forma mais simples também é alterar dentro de volumes de type: Directory para DirectoryOrCreate.

Ex:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: um-pod-qualquer
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - mountPath: /pasta-de-arquivos
          name: volume-pod
  volumes:
    - name: volume-pod
      hostPath:
        path: /C/Users/Daniel/Desktop/uma-pasta-no-host
        type: Directory
```

Resultado:

Caso a pasta `/C/Users/Daniel/Desktop/uma-pasta-no-host` exista no host, um volume chamado `volume-pod` será criado e montado na pasta `pasta-de-arquivos` dentro do container do Pod.

### Persistent Volumes

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Resource independente do pod, responsável por abstrair como o cloud provider armazena os dados.

Disco de armazenamento num Cloud Provider.

Primeiramente precisamos criar um disco de dados no nosso cloud provider, ele funcionará como nosso hostPath.

Nas configurações deste disco precisamos passar algumas informações como quando Gi queremos utilizar, se apenas um pod pode acessar por vez o disco ou se múltiplos pods podem ler, etc.

#### Arquivo de configuração

```yaml
apiVersion: v1
kind: PersistVolume
metadata:
    name: pv-1
spec:
    capacity:
        storage: 10Gi
    accessModes:
        - ReadWriteOnce
    gcePersistentDisk:
        pdName: pv-disk
    storageClassName: standard
```

#### Listar

```bash
kubectl get pv
```

Obs: Reclaim Policy = Retain, esse parâmetro nos informa que o volume deve existir independente do que aconteça, ou seja, por mais que não tenhamos nenhum claim o consumindo ele ainda vai continuar existindo.

### Persistent Volumes Claim

Serve como passaporte para o pod acessar o pv.

Consumidor de um pv.

Para que consigamos ligar um pvc a um pv, diferentemente dos pods criados até o momento, não fazemos através das labels e sim precisamos passar o mesmo modo de acesso e mesma capacidade.

#### Arquivo de configuração

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-1
spec:
    accessModes:
        - ReadWriteOnde
    resources:
        requests:
            storage: 10Gi
    storageClassName: standard
```

#### Criando um exemplo de pod utilizando pvc

Aqui o pod acesso o pv através do pvc.

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: pod-pv
spec:
    containers:
        - name: nginx-container
            image: nginx-latest
            volumeMounts:
                - mountPath: /volume-dentro-do-container
                name: primeiro-pv
    volumes:
        - name: primeiro-pv
            hostPath:
                persistentVolumeClaim:
                    claimName: pvc-1
```



### Storage Classes

https://kubernetes.io/docs/concepts/storage/storage-classes/

Gerencia os discos e volumes dinamicamente.

Cria um pv dinamicamente assim que um pvc é vinculado em um storage class.

#### Arquivo de configuração

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: slow
provisioner: kubernetes.io/gce-pd
parameters:
    type: pd-standard
    fstype: ext4
    replication-type: none
```

#### Listar

```bash
kubectl get sc
```

Basta criamos um pvc com as configurações do nosso sc e podemos utilizarmos o pv tranquilamente pois todas as configurações serão criadas dinamicamente.

exemplo:

criando um pvc utilizando as configs de sc.

````yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-2
spec:
    accessModes:
        - ReadWriteOnde
    resources:
        requests:
            storage: 10Gi
    storageClassName: slow
````

utilizando pv através do pod

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: pod-sc
spec:
    containers:
        - name: nginx-container
            image: nginx-latest
            volumeMounts:
                - mountPath: /volume-dentro-do-container
                name: primeiro-pv
    volumes:
        - name: primeiro-pv
            hostPath:
                persistentVolumeClaim:
                    claimName: pvc-2
```

 

### StatefulSets

Quando criamos um StatefulSet devemos criar um pvc para acessar um pv. Funciona igual a um deployment mas é voltado para aplicações que devem manter o seu status.

Quando criamos um pvc sem atribuirmos um pv o nosso cluster utiliza um storage class padrão.

Vamos criar abaixo um exemplo utilizando o diretório Praticando do projeto.

#### Criar

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sistema-noticias-statefulset
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sistema-noticias
      name: sistema-noticias
    spec:
      containers:
        - name: sistema-noticias-container
          image: aluracursos/sistema-noticias:1
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: sistema-configmap
          volumeMounts:
            - name: imagens
              mountPath: /var/www/html/uploads
            - name: sessao
              mountPath: /tmp
      volumes:
        - name: imagens
          persistentVolumeClaim:
            claimName: imagens-pvc
        - name: sessao
          persistentVolumeClaim:
            claimName: sessao-pvc
  selector:
    matchLabels:
      app: sistema-noticias
  serviceName: svc-sistema-noticias
```

#### Criando os persistent volumes claims

Imagens do portal noticias:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: imagens-pvc
spec:
  accessModels:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Sessão do portal noticia:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: imagens-pvc
spec:
  accessModels:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Probes

Quando temos uma aplicação rodando como podemos garantir que ela está rodando do jeito que esperamos que ela rode. 

A principal utilização do Probes é tornar visível ao Kubernetes que uma aplicação não está se comportando da maneira esperada.

O kubelet usa o liveness probe como critério para saber quando reiniciar um pod.

### LivenessProbe

Indicará falha caso o código de retorno seja menor que 200 ou maior/igual a 400.

parte de definição do arquivo:

```yaml
livenessProbe:
  httpGet:
    path: / #caminho da url a ser testada, caso seja basta identificarmos o /
    port: 80 #porta do container em que a aplicação roda
  periodSeconds: 10 #intervalo em segundos em que será realizado os testes
  failureThreshold: 3 #quantidade de falhas que devem ocorrer para o restart da aplicação
  initialDelaySeconds: 20 #tempo que a aplicação deverá esperar antes de começar a efetuar os testes
```

Podemos colocar essas alterações tanto no nosso arquivo sistema-noticias-statefulset como no portal-noticias-deployment

sistema-noticias-statefulset:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sistema-noticias-statefulset
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sistema-noticias
      name: sistema-noticias
    spec:
      containers:
        - name: sistema-noticias-container
          image: aluracursos/sistema-noticias:1
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: sistema-configmap
          volumeMounts:
            - name: imagens
              mountPath: /var/www/html/uploads
            - name: sessao
              mountPath: /tmp
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 3
            initialDelaySeconds: 20
      volumes:
        - name: imagens
          persistentVolumeClaim:
            claimName: imagens-pvc
        - name: sessao
          persistentVolumeClaim:
            claimName: sessao-pvc
  selector:
    matchLabels:
      app: sistema-noticias
  serviceName: svc-sistema-noticias
```

portal-noticias-deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portal-noticias-deployment
spec:
  replicas: 3
  template:
    metadata:
      name: portal-noticias
      labels:
        app: portal-noticias
    spec:
      containers:
        - name: portal-noticias-container
          image: aluracursos/portal-noticias:1
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: portal-noticias-configmap
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 3
            initialDelaySeconds: 20
  selector:
    matchLabels:
      app: portal-noticias
```

### Readiness Probes

O container dentro do Pod pode ainda não estar 100% para receber requisições, através do Readiness Probes podemos garantir que ele faça o balanceamento de carga entre todos os Pods que estejam habilitados para receber requisições. 

```yaml
readinessProbe:
  httpGet:
    path: / #caminho da url a ser testada, caso seja basta identificarmos o /
    port: 80 #porta do container em que a aplicação roda
  periodSeconds: 10 #intervalo em segundos em que será realizado os testes
  failureThreshold: 5 #após passar a quantidade de vezes configuradas a aplicação vai passar a receber requisições da mesma forma e passará a ignorar o readinessProbe
  initialDelaySeconds: 3 #tempo que a aplicação deverá esperar antes de começar a efetuar os testes
```

### Startup Probe

Há um terceiro tipo de probe voltado para aplicações legadas, o Startup Probe. Algumas aplicações legadas exigem tempo adicional para inicializar na primeira vez. Nem sempre Liveness ou Readiness Probes vão conseguir resolver de maneira simples os problemas de inicialização de aplicações legadas.

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes

## Horizontal Pod Autoscaler

Por mais que tenhamos as aplicações com deployment, statefulset, etc. Se tivermos muitas requisições sendo realizadas para um Pod e ele não estiver conseguindo suprir a demanda computacional requerida, ainda teremos problemas ocorrendo. 

Através de métricas o hpa consegue escalar nossa aplicação criando mais pods ou diminuindo a quantidade de pods existentes.

Obs: O horizontalPodAutoscaler pode não conseguir identificar o consumo de recursos dos Pods, pelo motivo de não definição de um servidor de métricas. Dentro do nosso projeto em praticando temos o arquivo components.yaml que contém as configurações que devem ser aplicadas.

### Definição do arquivo

Precisamos passar algumas métricas para nosso Pod para que ele saiba o quanto será requisitado por aplicação de cpu e memória para que ele faça o autoscale.

### Arquivos de definição

No arquivo de definição do nosso Pod devemos passar os recursos a serem utilizados, exe:

```yaml
resources:
  requests:
    cpu: 10m
```

No arquivo do hpa devemos criar

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: portal-noticias-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1 #api utilizada no pod
    kind: Deployment #definição do pod
    name: portal-noticias-deployment #nome do pod
  minReplicas: 1 #minimo de replicas do Pod
  maxReplicas: 10 #número máximo de réplicas que deverão ser criadas
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50 #no exemplo acima definimos cpu 10mc então quando chegar a 5mc será criado um novo pod
```

### Listar

```bash
kubectl get hpa
```



