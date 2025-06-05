# Guia Prático de CI/CD com Jenkins, Docker e Kubernetes no WSL

Este repositório contém um exemplo prático de como configurar um pipeline de Integração Contínua (CI) e Entrega Contínua (CD) utilizando Jenkins, Docker e Kubernetes rodando no **Windows Subsystem for Linux (WSL)**. A solução utiliza o **Docker Desktop** como ambiente de containerização e cluster Kubernetes local, e o **Docker Hub** para gerenciamento de imagens Docker. A automação dos builds é feita via GitHub Webhooks, seguindo conceitos de um fluxo de CI/CD.

## Sumário

1.  [Visão Geral](#1-visão-geral)
2.  [Pré-requisitos](#2-pré-requisitos)
3.  [Instalação do Ambiente](#3-instalação-do-ambiente)
    * [Configuração Inicial do WSL](#configuração-inicial-do-wsl)
    * [Docker Desktop](#docker-desktop)
    * [Java (JDK 17)](#java-jdk-17)
    * [Jenkins](#jenkins)
    * [Kubectl](#kubectl)
    * [ngrok](#ngrok) (Para webhooks em ambiente local)
4.  [Configuração do Jenkins Pipeline](#4-configuração-do-jenkins-pipeline)
    * [Criação do Job](#criação-do-job)
    * [Configuração do SCM](#configuração-do-scm)
    * [Configuração de Credenciais Docker Hub](#configuração-de-credenciais-docker-hub)
    * [Configuração de Credenciais Kubernetes (Kubeconfig)](#configuração-de-credenciais-kubernetes-kubeconfig)
    * [Conteúdo do Jenkinsfile](#conteúdo-do-jenkinsfile)
    * [Configuração do Gatilho de Automação (Webhook com ngrok)](#configuração-do-gatilho-de-automação-webhook-com-ngrok)
5.  [Estrutura do Projeto](#5-estrutura-do-projeto)
6.  [Execução e Verificação](#6-execução-e-verificação)
    * [Testando a Automação](#testando-a-automação)
    * [Verificando a Aplicação no Kubernetes](#verificando-a-aplicação-no-kubernetes)

---

## 1. Visão Geral

Este projeto demonstra um pipeline de CI/CD que automatiza as seguintes etapas:
* **Checkout:** O Jenkins obtém o código-fonte do repositório Git.
* **Build Docker Image:** Constrói uma imagem Docker da aplicação.
* **Push Docker Image:** Envia a imagem construída para o **Docker Hub**.
* **Deploy no Kubernetes:** Implanta a aplicação (Deployment e Service) em um cluster Kubernetes local gerenciado pelo **Docker Desktop**.
Tudo isso é disparado automaticamente por eventos de `git push` via GitHub Webhooks.

## 2. Pré-requisitos

* Sistema Operacional: Windows 10 (versão 1903 ou superior) ou Windows 11.
* **WSL 2 instalado e configurado.**
* **Docker Desktop instalado e com Kubernetes habilitado.**
* Virtualização (Intel VT-x ou AMD-V) habilitada na BIOS/UEFI.
* Conexão com a internet.
* Conta no [GitHub](https://github.com/) e no [Docker Hub](https://hub.docker.com/).

## 3. Instalação do Ambiente

Todos os comandos a seguir devem ser executados no terminal da sua distribuição Linux no WSL (ex: Ubuntu), a menos que especificado.

### Configuração Inicial do WSL

Certifique-se de que o WSL 2 está instalado e é a versão padrão.
Abra o PowerShell como administrador e execute:

```powershell
wsl --install                 # Para nova instalação
wsl --update                  # Para atualizar
wsl --set-default-version 2   # Definir WSL 2 como padrão
wsl --install -d Ubuntu       # Instalar Ubuntu (se não tiver)
Docker Desktop
O Docker Desktop é fundamental para fornecer o ambiente Docker e o cluster Kubernetes local.

Baixar e Instalar o Docker Desktop:

Acesse https://www.docker.com/products/docker-desktop/ e baixe o instalador para Windows.
Execute o instalador, certificando-se de que a opção "Enable WSL 2 Windows Subsystem for Linux features" esteja marcada.
Conclua a instalação e reinicie o computador.
Configurar Docker Desktop para Usar WSL 2:

Abra o Docker Desktop.
Vá para Settings (Configurações) (ícone de engrenagem).
Em WSL Integration, certifique-se de que a integração com sua distro WSL (ex: Ubuntu) está habilitada.
Em Resources > WSL Integration, ative o Docker Engine e Kubernetes para sua distro.
Habilitar Kubernetes no Docker Desktop:

Nas Settings (Configurações) do Docker Desktop, vá para Kubernetes.
Marque "Enable Kubernetes".
Clique em Apply & Restart. Isso pode levar alguns minutos para baixar e configurar o cluster Kubernetes.
Java (JDK 17)
Para instalar o Java JDK 17, execute os seguintes comandos no seu terminal WSL:

Bash

sudo apt update
sudo apt install openjdk-17-jdk -y
Configurar JAVA_HOME (Recomendado):
Descubra o caminho de instalação do JDK:

Bash

sudo update-alternatives --config java
Anote o caminho (ex: /usr/lib/jvm/java-17-openjdk-amd64).

Edite o arquivo ~/.bashrc:

Bash

nano ~/.bashrc
Adicione as seguintes linhas no final do arquivo (substitua o caminho):

Bash

# Configuração JAVA_HOME
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64" # Substitua pelo seu caminho real
export PATH=$JAVA_HOME/bin:$PATH
Salve e feche. Recarregue o .bashrc:

Bash

source ~/.bashrc
Verifique:

Bash

echo $JAVA_HOME
java -version
Jenkins
Para instalar o Jenkins, siga os passos abaixo no seu terminal WSL:

Adicione a chave do Jenkins ao seu sistema:
Bash

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  [https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key](https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key)
Adicione o repositório do Jenkins à lista de fontes do apt:
Bash

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] [https://pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable) binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
Atualize a lista de pacotes e instale o Jenkins:
Bash

sudo apt-get update
sudo apt-get install -y jenkins
Iniciar e Acessar Jenkins:

Inicie o serviço Jenkins:
Bash

sudo systemctl start jenkins
Verifique o status:
Bash

sudo systemctl status jenkins
Acesse o Jenkins no seu navegador: http://localhost:8080
Chave de Segurança Inicial:
Para obter a senha inicial de administrador do Jenkins:

Bash

sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Siga as instruções na interface web para concluir a instalação (instale os plugins sugeridos).

Configurar Permissões Docker para Jenkins:
Adicione o usuário jenkins ao grupo docker para que ele possa interagir com o daemon Docker sem sudo:

Bash

sudo usermod -aG docker jenkins
Reinicie os serviços para aplicar as permissões:

Bash

sudo systemctl restart docker
sudo systemctl restart jenkins
Você também pode precisar reiniciar o Docker Desktop no Windows para garantir que o socket seja resetado.

Kubectl
Para instalar o kubectl (ferramenta de linha de comando do Kubernetes), siga os passos abaixo no seu terminal WSL:

Atualize a lista de pacotes e instale os pré-requisitos:
Bash

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
Adicione a chave GPG do Kubernetes:
Bash

curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
Adicione o repositório do Kubernetes:
Bash

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.30/deb/](https://pkgs.k8s.io/core:/stable:/v1.30/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
Atualize a lista de pacotes e instale o kubectl:
Bash

sudo apt-get update
sudo apt-get install -y kubectl
Verifique a instalação:

Bash

kubectl version --client
Observação: O Kubernetes do Docker Desktop já configura o kubectl para usar seu cluster automaticamente.

ngrok
O ngrok cria um túnel seguro do seu localhost para a internet, permitindo que serviços externos (como o GitHub Webhook) acessem seu Jenkins local.

Instalar ngrok:
Bash

sudo snap install ngrok
Conectar ngrok à sua conta:
Crie uma conta gratuita no ngrok.com.
Faça login e vá para https://dashboard.ngrok.com/get-started/your-authtoken.
Copie o comando ngrok config add-authtoken <SEU_AUTHTOKEN_AQUI>.
Cole e execute este comando no seu terminal WSL. <!-- end list -->
Bash

ngrok config add-authtoken <SEU_AUTHTOKEN_AQUI>
4. Configuração do Jenkins Pipeline
Após instalar todos os componentes, configure o job de pipeline no Jenkins.

Criação do Job
Acesse seu Jenkins (http://localhost:8080).
Clique em "Novo Item" (New Item) no menu lateral.
Dê um nome ao seu job (ex: guia-jenkins1).
Selecione "Pipeline" e clique em "OK".
Configuração do SCM
Na tela de configuração do job, role até a seção "Pipeline".
Definição (Definition): Selecione "Pipeline script from SCM".
Na seção "SCM":
SCM: Selecione "Git".
URL do Repositório (Repository URL): https://github.com/stampini81/guia-pratico-jenkins
Credenciais (Credentials): Deixe como - none - (para repositórios públicos).
Ramos para Construir (Branches to build) > Especificador de Ramificação: */main (ou o nome do seu branch principal, se diferente).
Comportamentos Adicionais (Additional Behaviours): Clique em "Adicionar" e selecione "Limpar antes do checkout (Clean before checkout)".
Caminho do Script (Script Path): Deixe vazio.
Configuração de Credenciais Docker Hub
Para o Jenkins poder enviar imagens para o Docker Hub, você precisa configurar um Personal Access Token (PAT).

Crie um PAT no Docker Hub:

Vá para https://hub.docker.com/settings/security.
Faça login, se necessário.
Clique em "New Access Token".
Dê um nome ao token (ex: jenkins-ci-token).
Permissões: Marque "Read & Write" (Leitura e Escrita) para repositórios.
Clique em "Generate" e COPIE o token gerado IMEDIATAMENTE.
Configure a credencial no Jenkins:

Vá para "Gerenciar Jenkins" > "Gerenciar Credenciais".
Clique em "Jenkins" (no store global).
Clique em "+ Adicionar Credenciais".
Tipo: "Nome de usuário com senha".
Escopo: "Global".
Nome de usuário: Seu nome de usuário do Docker Hub (ex: leandro282).
Senha: Cole o PAT (Personal Access Token) gerado no Docker Hub.
ID: Digite dockerhub (esta ID será usada no Jenkinsfile).
Clique em "Criar".
Configuração de Credenciais Kubernetes (Kubeconfig)
Para o Jenkins interagir com seu cluster Kubernetes (o do Docker Desktop), ele precisa das credenciais do kubeconfig.

Obtenha seu arquivo kubeconfig (geralmente em ~/.kube/config no seu ambiente WSL onde o kubectl está configurado para o Docker Desktop).

Configure a credencial no Jenkins:

Vá para "Gerenciar Jenkins" > "Gerenciar Credenciais".
Clique em "Jenkins" (no store global).
Clique em "+ Adicionar Credenciais".
Tipo: "Arquivo Secreto" (Secret file).
Escopo: "Global".
ID: Digite kubeconfig (esta ID será usada no Jenkinsfile).
Arquivo: Clique em "Escolher arquivo" e faça upload do seu arquivo ~/.kube/config (você pode copiá-lo do seu WSL para o Windows se for mais fácil para fazer o upload, ou usar um comando como cat ~/.kube/config > /mnt/c/temp/kubeconfig.txt no WSL e depois upload C:\temp\kubeconfig.txt no Jenkins).
Clique em "Criar".
Conteúdo do Jenkinsfile
Este é o Jenkinsfile que deve estar na raiz do seu repositório Git, com a lógica de build, push e deploy.

Groovy

// Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    // Constrói a imagem Docker. A tag usa o BUILD_ID do Jenkins.
                    // Substitua 'leandro282' pelo seu username EXATO do Docker Hub.
                    docker.build("leandro282/guia-jenkins1:<span class="math-inline">\{env\.BUILD\_ID\}", "\./src"\)
\}
\}
\}
stage\('Push Docker Image'\) \{
steps \{
script \{
// Autentica no Docker Hub usando a credencial 'dockerhub' configurada no Jenkins\.
// Empurra a imagem com a tag 'latest' e com a tag BUILD\_ID\.
docker\.withRegistry\('\[https\://registry\.hub\.docker\.com\]\(https\://registry\.hub\.docker\.com\)', 'dockerhub'\) \{
docker\.image\("leandro282/guia\-jenkins1\:</span>{env.BUILD_ID}").push('latest')
                        docker.image("leandro282/guia-jenkins1:<span class="math-inline">\{env\.BUILD\_ID\}"\)\.push\("</span>{env.BUILD_ID}")
                    }
                }
            }
        }

        stage('Deploy no Kubernetes') {
            environment {
                // Define uma variável de ambiente para a BUILD_ID para usar no shell script
                tag_version = "<span class="math-inline">\{env\.BUILD\_ID\}"
\}
steps \{
// Usa a credencial 'kubeconfig' configurada no Jenkins para acessar o cluster Kubernetes\.
withKubeConfig\(\[credentialsId\: 'kubeconfig'\]\) \{
// Substitui o placeholder \{\{tag\}\} no deployment\.yaml pela tag da versão atual \(BUILD\_ID\)\.
// O 'sed \-i' edita o arquivo no local dentro do workspace do Jenkins\.
sh "sed \-i 's\|\{\{tag\}\}\|</span>{tag_version}|g' ./k8s/deployment.yaml"

                    // Aplica o arquivo YAML modificado ao cluster Kubernetes.
                    // Isso fará um rolling update dos seus Pods para a nova imagem.
                    sh 'kubectl apply -f k8s/deployment.yaml'

                    // Opcional: Aguarda o rollout do Deployment para confirmar que foi bem-sucedido.
                    sh 'kubectl rollout status deployment/conversao-temperatura'
                }
            }
        }
    }
}
Configuração do Gatilho de Automação (Webhook com ngrok)
Para que o Jenkins dispare o pipeline automaticamente a cada git push:

Inicie o túnel ngrok:

Abra um novo terminal no seu WSL.
Execute: ngrok http 8080
Copie a URL HTTPS pública que o ngrok exibir (ex: https://algum-subdominio-aleatorio.ngrok-free.app). Esta URL muda a cada nova sessão do ngrok. Mantenha este terminal aberto.
Configure o Webhook no GitHub:

Vá para seu repositório no GitHub (https://github.com/stampini81/guia-pratico-jenkins).
Clique em "Settings" > "Webhooks".
Adicione (ou edite) um novo webhook.
Payload URL: Cole a URL do ngrok que você copiou, ADICIONANDO /github-webhook/ no final.
Exemplo: https://92e8-170-81-189-240.ngrok-free.app/github-webhook/
Content type: application/json.
Which events...: "Just the push event." (ou "Apenas o evento push.").
Clique em "Add webhook" (ou "Update webhook").
Habilite o Gatilho no Jenkins Job:

Vá para a Configuração do seu job no Jenkins.
Na seção "Gatilhos de Compilação (Build Triggers)", marque a opção:
"GitHub hook trigger for GITScm polling".
"Salvar".
5. Estrutura do Projeto
A estrutura do seu repositório deve ser a seguinte:

guia-pratico-jenkins/
├── Jenkinsfile
├── Dockerfile
├── k8s/
│   └── deployment.yaml
├── src/
│   ├── server.js
│   ├── convert.js
│   ├── package.json
│   ├── package-lock.json
│   └── config/
│       └── system-life.js
├── README.md
└── .gitignore
6. Execução e Verificação
Testando a Automação
Certifique-se de que o ngrok está rodando e o webhook no GitHub está configurado com a URL correta do ngrok (verifique as "Recent Deliveries" no GitHub Webhooks para 200 OK).
Faça uma pequena alteração em qualquer arquivo no seu projeto (ex: server.js).
Faça um git commit e git push para o seu repositório GitHub.
Vá para a página do seu job no Jenkins. Você deverá ver um novo build sendo disparado automaticamente em poucos segundos.
Verificando a Aplicação no Kubernetes
Após o pipeline do Jenkins ser concluído com SUCESSO, verifique a implantação:

Verificar recursos do Kubernetes:

Bash

kubectl get deployments conversao-temperatura
kubectl get pods -l app=conversao-temperatura
kubectl get services conversao-temperatura
Confirme que o Deployment está READY, os Pods estão Running e o Service é NodePort com a porta 30000.

Acessar a aplicação:
Abra seu navegador web e acesse:
http://localhost:30000

Você também pode acessar os endpoints da API:
http://localhost:30000/health
http://localhost:30000/ready
http://localhost:30000/celsius/25/fahrenheit
<!-- end list -->
