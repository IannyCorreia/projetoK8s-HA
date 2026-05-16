# k8s-HA

# Cluster MySQL de Alta Disponibilidade com Kubernetes e Amazon EKS

## Visão Geral

Este projeto apresenta a implementação de um ambiente MySQL em alta disponibilidade sobre Kubernetes, utilizando Amazon EKS, StatefulSet, Headless Service, PersistentVolumeClaims e volumes persistentes provisionados via Amazon EBS.

O objetivo principal foi replicar uma arquitetura de banco de dados stateful em Kubernetes, validando conceitos fundamentais de infraestrutura em nuvem, persistência de dados, comunicação interna entre réplicas, replicação MySQL e tolerância a falhas em um ambiente gerenciado pela AWS.

A implementação também serve como base para estudos futuros envolvendo segurança em ambientes Kubernetes, especialmente com ferramentas de análise como Checkov e Kubescape.

---

## Objetivos do Projeto

Os principais objetivos deste projeto foram:

- Criar um cluster Kubernetes gerenciado com Amazon EKS;
- Implantar um banco de dados MySQL em arquitetura de alta disponibilidade;
- Utilizar StatefulSet para garantir identidade estável aos Pods;
- Configurar persistência de dados com PersistentVolumeClaims e Amazon EBS;
- Validar a comunicação interna entre réplicas por meio de Headless Service e DNS interno;
- Configurar e testar replicação entre instâncias MySQL;
- Avaliar o comportamento do ambiente diante da recriação de Pods;
- Documentar os resultados obtidos como estudo prático de infraestrutura em nuvem.

---

## Tecnologias Utilizadas

- Amazon EKS
- Kubernetes
- Amazon EC2
- Amazon EBS
- EBS CSI Driver
- AWS VPC CNI
- CoreDNS
- MySQL
- kubectl
- eksctl
- YAML
- StatefulSet
- PersistentVolumeClaim
- ConfigMap
- Secret

---

## Arquitetura da Solução

A solução foi construída sobre um cluster Amazon EKS com três nós de trabalho, responsáveis por executar as réplicas do MySQL. Cada réplica foi implantada como um Pod gerenciado por um StatefulSet, garantindo identidade persistente, armazenamento individual e nomes de rede previsíveis.

A comunicação entre os Pods ocorre por meio de um Headless Service, permitindo que cada instância MySQL seja acessada individualmente por DNS interno. A persistência dos dados é realizada por volumes Amazon EBS, provisionados dinamicamente a partir de uma StorageClass customizada.

![Visão geral do cluster Amazon EKS](docs/evidencias/eks-cluster-overview.png)

> Adicione aqui a imagem da página 1 do relatório, que mostra o painel do Amazon EKS com o cluster `mysql-ha-cluster` ativo.

---

## Componentes da Arquitetura

| Componente | Função |
|---|---|
| Amazon EKS | Serviço gerenciado da AWS utilizado para executar o cluster Kubernetes. |
| EC2 Worker Nodes | Instâncias responsáveis por executar os Pods do cluster. |
| StatefulSet | Controlador utilizado para manter identidade estável dos Pods MySQL. |
| Headless Service | Serviço responsável pela descoberta individual das réplicas MySQL. |
| PersistentVolumeClaim | Solicitação de armazenamento persistente para cada réplica. |
| Amazon EBS | Armazenamento persistente utilizado pelos Pods MySQL. |
| EBS CSI Driver | Driver responsável por integrar volumes EBS ao Kubernetes. |
| CoreDNS | Serviço de DNS interno do Kubernetes. |
| ConfigMap | Recurso utilizado para armazenar configurações do MySQL. |
| Secret | Recurso utilizado para armazenar credenciais sensíveis. |

---

## Estrutura do Repositório

```txt
k8s-HA/
│
├── README.md
├── .gitignore
│
├── mysql-configmap.yaml
├── mysql-service.yaml
├── mysql-statefulset.yaml
├── storage.yaml
├── mysql-secret.example.yaml
│
└── docs/
    └── evidencias/
        ├── eks-cluster-overview.png
        ├── system-pods.png
        ├── statefulset-mysql.png
        ├── worker-nodes.png
        ├── endpoints-headless-service.png
        ├── pvc-bound.png
        ├── dns-test.png
        ├── replication-status.png
        └── data-replication-validation.png

```


## Arquivos Kubernetes

| Arquivo | Descrição |
| :--- | :--- |
| `storage.yaml` | Define a `StorageClass` utilizada para provisionamento dos volumes persistentes. |
| `mysql-configmap.yaml` | Define configurações customizadas do MySQL. |
| `mysql-service.yaml` | Define o serviço de rede utilizado pelo MySQL, incluindo o *Headless Service*. |
| `mysql-statefulset.yaml` | Define o `StatefulSet` responsável por executar as réplicas MySQL. |

---

## Cluster Amazon EKS

O ambiente foi implantado em um cluster **Amazon EKS**, utilizando um plano de controle gerenciado pela AWS. Essa abordagem elimina a necessidade de administrar manualmente componentes críticos do Kubernetes, como o *API Server* e o *etcd*.

O cluster foi configurado com **três nós de trabalho**, fornecendo uma base para distribuição das réplicas MySQL em diferentes instâncias EC2.

![Nós do cluster e instâncias EC2](docs/evidencias/worker-nodes.png)

> 📸 *Configuração dos nós do cluster e instâncias EC2 utilizadas.*

### Nós de Trabalho

Foram utilizadas instâncias EC2 do tipo `t3.medium`, cada uma com 2 vCPUs e 4 GiB de memória. O cluster foi configurado com três nós, permitindo distribuir as réplicas do MySQL entre diferentes máquinas virtuais.

Essa configuração contribui para a alta disponibilidade, pois, caso um nó apresente falha, o Kubernetes pode reagendar os Pods em outros nós disponíveis.

### Componentes de Sistema do Kubernetes

Durante a criação do cluster, foram utilizados componentes essenciais do ecossistema Kubernetes e AWS:

* **`aws-node`**: plugin de rede da AWS, responsável pela integração com a VPC;
* **`ebs-csi-node` e `ebs-csi-controller`**: responsáveis pela integração entre Kubernetes e Amazon EBS;
* **`coredns`**: responsável pela resolução de nomes internos no cluster;
* **`kube-proxy`**: responsável por regras de rede e encaminhamento interno.

![Pods de Sistema](docs/evidencias/system-pods.png)

> 📸 *Pods de sistema como aws-node, ebs-csi-node, ebs-csi-controller e coredns em execução.*

---

## StatefulSet do MySQL

O MySQL foi implantado utilizando um `StatefulSet` com três réplicas:

* `mysql-0`
* `mysql-1`
* `mysql-2`

O `StatefulSet` foi escolhido porque bancos de dados são aplicações *stateful*, ou seja, dependem de estado, identidade persistente e armazenamento associado.

Diferente de um `Deployment` (mais adequado para aplicações *stateless*), o `StatefulSet` mantém nomes previsíveis e volumes persistentes associados a cada Pod. Isso garante que, mesmo após uma falha ou recriação, um Pod como o `mysql-1` retorne com a mesma identidade e tente reutilizar seu volume persistente correspondente.

![StatefulSet do MySQL](docs/evidencias/statefulset-mysql.png)

> 📸 *Visualização do StatefulSet do MySQL e suas respectivas réplicas.*

---

## Persistência de Dados

A persistência foi configurada com `PersistentVolumeClaims` (PVC) vinculados a volumes **Amazon EBS**. Cada réplica do MySQL possui seu próprio volume persistente, evitando perda de dados em caso de reinicialização ou recriação dos Pods.

A `StorageClass` customizada `mysql-storage` foi utilizada para provisionar os volumes necessários no ambiente AWS. Os volumes foram criados com modo de acesso `ReadWriteOnce`, garantindo que cada volume seja montado por apenas um nó por vez, evitando riscos de corrupção por acesso simultâneo indevido.

![Status dos Volumes Persistentes](docs/evidencias/pvc-bound.png)

> 📸 *Exibição dos volumes mysql-data-mysql-0, mysql-data-mysql-1 e mysql-data-mysql-2 com status Bound.*

---

## Rede Interna e Service Discovery

A comunicação entre as réplicas MySQL foi realizada por meio de um **Headless Service**. Esse tipo de serviço permite que cada Pod seja resolvido individualmente por DNS interno.

Exemplos de nomes utilizados:

* `mysql-0.mysql`
* `mysql-1.mysql`
* `mysql-2.mysql`

Esse comportamento é essencial em ambientes de banco de dados replicado, pois cada instância precisa identificar corretamente o nó primário e os nós secundários.

![Endpoints do Headless Service](docs/evidencias/endpoints-headless-service.png)
<img width="1190" height="416" alt="image" src="https://github.com/user-attachments/assets/a4921f97-43ca-46d4-b9b9-91738b2a75dd" />


> 📸 *Endpoints e EndpointSlices associados ao serviço mysql.*

### Teste de DNS Interno

A resolução de nomes foi validada com comandos executados dentro dos Pods, confirmando que as réplicas conseguiam resolver os nomes umas das outras.

```bash
kubectl exec -it mysql-0 -- getent hosts mysql-1.mysql
kubectl exec -it mysql-0 -- getent hosts mysql-2.mysql
```
![dns](docs/evidencias/dns-test.png)

### Instancias EC2
- **Tipo de Instância:** Foram utilizadas instâncias do tipo **t3.medium**. Cada nó possui 2 vCPUs e 4 GiB de memória, configuração ideal para suportar tanto o orquestrador quanto os processos do MySQL.
- **Quantidade de Nós:** O cluster foi configurado com um grupo de nós gerenciados contendo **3 instâncias**, garantindo que as três réplicas do banco de dados (mysql-0, mysql-1 e mysql-2) residam em máquinas físicas/virtuais distintas.
![EC2](docs/evidencias/Instancias-EC2.png)
