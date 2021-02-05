## Vault

O [Vault](https://www.vaultproject.io/) é uma ferramenta desenvolvida pela [HashiCorp](https://www.hashicorp.com/) com objetivo de auxiliar na gestão de secrets e proteção de dados sensíveis.    
Utilizaremos o [Vault Helm Chart](https://github.com/hashicorp/vault-helm) para instalação e configuração do Vault em um cluster Kubernetes, conforme a recomendação oficial.   

### Pré Requisitos
- [Helm instalado](https://helm.sh/docs/intro/install/)
- [Cluster Kubernetes configurado](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)
- [Credenciais AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)

### Procedimentos de Instalação
#### Configurações
Por padrão o Vault necessita ser aberto (unseal) manualmente todas as vezes que for iniciado, o que pode ocorrer com muita frequência quando executado em um cluster Kubernetes. Desta forma, vamos configurá-lo para executar o unseal automaticamente utilizando o [AWS KMS](https://aws.amazon.com/en/kms/).   
Primeiramente, faça uma cópia do arquivo modelo values/values.yaml e renomeie de acordo com o ambiente desejado, para QA por exemplo foi criado o arquivo values-QA.yaml. Agora, no arquivo que você acabou de criar, atualize as informações de acesso ao KMS (KMS_REGION e KMS_KEY_ID)   

Agora crie uma Secret com as credenciais para o Vault acessar o KMS   
```
$ kubectl create secret generic eks-creds \
    --from-literal=AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID?}" \
    --from-literal=AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY?}"
```

#### Instalação
A instalação do Vault será feita no modo standalone, que possibilita executarmos uma única instância do Vault com armazenamento das secrets em uma estrutura de diretórios no filesystem local da instância. Caso necessite instalar o Vault em alta disponibilidade veja as instruções [aqui](https://www.vaultproject.io/docs/platform/k8s/helm/run#ha-mode).   

O primeiro passo é configurar o repositório Vault Helm Chart    
```
$ helm repo add hashicorp https://helm.releases.hashicorp.com
```
Agora vamos instalar o Vault utilizando o arquivo de configuração que acabamos de criar:
```
$ helm install vault hashicorp/vault -f ./values/values-<ENVIRONMENT>.yaml --create-namespace --namespace vault
```
Se tudo ocorrer bem, os componente do Vault serão criados no namespace vault, verifique os pods em execução:
```
$ kubectl get pods --namespace vault
```
Assim que o Pod do Vault estiver com status running, vamos inicializar o Vault
```
$ kubectl exec -ti vault-0 -- vault operator init
```
O comando acima exibirá o seu *Initial Root Token* (algo como: s.LqIjL7WuWiy69mOiEizfmats), ele será necessário para acessar a interface gráfica do Vault.    
Por fim, é possível verificar o status de execução para confirmar se o unseal foi executado com sucesso:
```
$ kubectl exec -ti vault-0 -- vault status
```
Para validar o acesso a User Interface de maneira local, execute o comando abaixo e então acesse o endereço http://127.0.0.1:8200 em seu navegador
```
$ kubectl port-forwarding vault-0 8200
```
