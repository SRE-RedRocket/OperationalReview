# lab-linkerd-k3d

**Resumo**: Criar um cluster utilizando o k3d, fazer a instalação básica do Linkerd e subir algumas aplicações python que simulam o fluxo de comunicação entre microsserviços.

**Sistema Operacional**: Ubuntu 20.04.2 LTS

**Ferramentas e versões**:

Ferramenta          | Versão        | Descrição
------------------- | ------------- | -------------
Linkerd             | Client version: *stable-2.10.2* | É um service mesh open-source e projeto membro da Cloud Native Computing Foundation
k3d                 | *v4.4.7* | É um produto Open Source utilizado para automatizar a implantação, o dimensionamento e o gerenciamento de aplicativos em contêiner
Docker              | 20.10.7 |

**Descrição:**

O objetivo da PoC é testar o Linkerd, uma opção de Service Mesh concorrente do Istio. Uma "malha de serviços" é um termo relativo à infraestrutura de comunicação existente entre vários serviços que compõem uma determinada aplicação/solução. Antes de iniciar o teste precisamos de um ambiente Kubernetes atualizado e o binário do Istio(o istioctl). Existem diversas ferramentas que permitem executar o kubernetes localmente como: *k3s, minikube, kind, microK8s e k3d*. Nesse exemplo optei por utilizar o k3d por ser bem mais leve e fácil de configurar.

Instalando a última versão do k3d:

```sh
curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
k3d --version
```

Para criar um cluster com o k3d:

```sh
k3d cluster create lab-linkerd-k3d --servers 1 --agents 3 --port 9080:80@loadbalancer --port 9443:443@loadbalancer --api-port 6443 --k3s-server-arg '--no-deploy=traefik'
```

> Nesse caso 'lab-linkerd-k3d' pode ser substituido por qualquer nome

Para testar o cluster criado:

```sh
kubectl config current-context
kubectl get nodes
kubectl get pods -A
```

Instale o CLI do Linkerd

```sh
curl -sL https://run.linkerd.io/install | sh
istioctl version # valide se foi instalado e esta no PATH
```

O próxmo passo é validar se seu cluster é capaz de hospedar o plano de controle do Linkerd.

```sh
linkerd check --pre
```

Instalação e validação do Control Plane

```sh
linkerd install | kubectl apply -f -
linkerd check
```

Instalando o viz e algumas extensões:

```sh
linkerd viz install | kubectl apply -f - # on-cluster metrics stack
linkerd jaeger install | kubectl apply -f - # Jaeger collector and UI
```

Dashboard:

```sh
linkerd viz dashboard &
```

Instalação do aplicativo de demonstração, injeção do Linkerd e depois checagem pra ver está funcionando corretamente:

```sh
curl -sL https://run.linkerd.io/emojivoto.yml \
  | kubectl apply -f -

kubectl -n emojivoto port-forward svc/web-svc 8080:80


kubectl get -n emojivoto deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```

Após rodar o comando curl que instala a aplicação ele ficará no dashboard na coluna Meshed 0/1. Após rodar o último comando do passo anterior a aplicação irá restartar e ao invés de subir um pod, irá subir dois(sidecar) e o status no dashboard vai atualizar automaticamente para Meshed 1/1. O curl abaixo pode ser usado para fazer várias requisições na api que foi feito o port-forward.

```sh
while true; do curl -s -o /dev/null -w "%{http_code}" 'http://localhost:8080/api/vote?choice=:flushed:'; done
```

O objetivo agorá é fazer o deploy de um conjunto de aplicações escritas em python para simular algumas requisições a um API. Basicamente temos uma API Flask que responde vários paths com status code diferentes. Além disso ela foi configurada para expor métricas pro Prometheus no path /metrics.

```sh
cat main-tower-example.yaml | linkerd inject - | kubectl apply -f -
cat tower-one-example.yaml | linkerd inject - | kubectl apply -f -
```

Remover o cluster

```sh
k3d cluster rm lab-linkerd-k3d
```
