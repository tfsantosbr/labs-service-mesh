# poc-kubernetes-service-mesh

Criação de uma prova de conceito para realizar canary e a/b deploy

## Criação das Imagens

Modifique a versão na aplicação antes de gerar as imagens versionadas:

```json
// src/WebApi/appsettings.json

{
  "AppVersion": "v2"
}
```

Compile e envie as imagens versionadas ao dockerhub:

```bash
export IMAGE_TAG=v2
docker build -t tfsantosbr/webapi:$IMAGE_TAG ./src/WebApi
docker push tfsantosbr/webapi:$IMAGE_TAG
```

## Deploy no Kubernetes

Configure as imagens dos containers em ambos os Deployments.
Especifique os labels em `spec.template.metadata.labels`, pois o Istio usa estes labels
para fazer o match com as Destionation Rules.

```yml
# k8s/Deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapi-v1
  template:
    ...
    spec:
      template:
        metadata:
          labels:
            app: webapi
            version: v1 #especificar versao
      ...
      containers:
        - name: webapi
          image: tfsantosbr/webapi:v1 # trocar a imagem versionada aqui
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapi-v2
  template:
    ...
    spec:
      template:
        metadata:
          labels:
            app: webapi
            version: v2 #especificar versao
      ...
      containers:
        - name: webapi
          image: tfsantosbr/webapi:v2 # trocar a imagem versionada aqui
---
```

Faça o deploy dos manifestos `.yml` para o Kubernetes:

```bash
# aplicando os manifestos
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/

# checar os manifestos criados
kubectl get all -n service-mesh
kubectl get gateways.networking.istio.io -n service-mesh
```

## Test on K8S (sem gateway)

Entre em um dos pods (de preferencia o pod do nginx):

```bash
kubectl exec -it nginx -c nginx -n service-mesh -- bash
```

Execute o comando `curl` e veja a resposta de acordo com a configuração do `virtual-service`:

```bash
while true; do curl http://webapi/version; echo; sleep 0.5; done;
```

## Tests on K8S (com gateway)

```bash
while true; do curl http://localhost/version; echo; sleep 0.5; done;
```

## Configurando Dominios Locais

Antes de executar os testes de versões por URL e para os testes do consistent hash
é necessário realizar configurações locais de dominio.

```bash
# C:\Windows\System32\drivers\etc\hosts
127.0.0.1 blue.tfsantos.com green.tfsantos.com hash.tfsantos.com
```

## URL Versions Tests

```bash
curl http://blue.tfsantos.com/version # deve cair sempre na v1
curl http://green.tfsantos.com/version # deve cair sempre na v2
```

## Consistent Hash Tests

```bash
# testando sem passar o header x-user-id
# retornará versões aleatórias
curl http://hash.tfsantos.com/version

# testando com o header x-user-id=1
# retornará a mesma versão para o mesmo valor do header
curl --header "x-user-id: 1" http://hash.tfsantos.com/version
curl --header "x-user-id: 2" http://hash.tfsantos.com/version
```

## AB Test

```bash
kubectl apply -f k8s/abtest.yml
curl --header "email: user@cocacola.com" http://abtest.tfsantos.com/version
curl --header "email: user@email.com" http://abtest.tfsantos.com/version
```

## Limpar todos os recursos

Para limpar todos os recursos instalados e remover o namespace, execute o comando abaixo:

```bash
kubectl delete namespace service-mesh
```
