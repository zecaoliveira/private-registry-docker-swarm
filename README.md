# Private registry Docker Swarm

Olá a todos.

Tenho um ambiente de estudos DevOps usando a solução de orquestração Docker Swarm.

Neste caso meu propósito é evoluir na carreira profissional adquirindo conhecimento sobre a cultura DevOps e as soluções de automação peculiares a ela.

Para alcançar este objetivo estou simulando um ambiente real de uma empresa que usa destas soluções.

### Por que usar uma solução de registro de imagens local? 🔗

> Fonte: https://docker-docs.uclv.cu/registry/

Você deve usar o "Registry" se quiser:

- Controlar rigorosamente onde suas imagens Docker estão sendo armazenadas.
- Ter total propriedade sobre seu pipeline de distribuição de imagens.
- Integrar o armazenamento e a distribuição de imagens ao seu fluxo de trabalho de desenvolvimento interno (CI/CD).

### Etapas a serem cumpridas!

1. Criar um certificado de uma Autoridade certificadora (CA) auto-assinado.
2. Um cluster Docker Swarm com 1 servidor Controller e 2 Nodes.
3. Um solução Portainer CE implementada para uma análise mais rápida dos logs dos serviços / containers.
4. Uma solução de serviço DNS local, como PiHole ou UnboundDNS.
5. Implementar o container / serviço do Registry Private.


### 1 - Criar um certificado autoassinado:

Eu usei a documentação do Christian Lempa e recomendo uma atenção particular a parte da instalação do certificado no host do Docker, conforme está descrito na documentação oficial Docker: após a inclusão do certificado "ca.pem" é necessário reiniciar o Docker.

Fontes:

- Install the CA Cert as a trusted root CA - Christian Lempa: [https://github.com/ChristianLempa/cheat-sheets/blob/main/misc/ssl-certs.md](https://github.com/ChristianLempa/cheat-sheets/blob/main/misc/ssl-certs.md#install-the-ca-cert-as-a-trusted-root-ca)
- Testing an Insecure registry - Docker: https://docker-docs.uclv.cu/registry/insecure/#use-self-signed-certificates

### 2 - Configurar um cluster Docker Swarm

Eu estou usando o Proxmox com imagem Ubuntu e automação usando Terraform para construir o meu cluster: https://github.com/zecaoliveira/proxmox-terraform e usei a documentação oficial:

Fontes:

- Getting started with Swarm mode: https://docs.docker.com/engine/swarm/swarm-tutorial/
- Create a swarm: https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/

### 3 - Deploy Portainer CE

Fonte:

- Install Portainer CE with Docker Swarm on Linux: https://docs.portainer.io/start/install-ce/server/swarm/linux

### 4 - Deploy DNS Server in Docker Swarm

Segue a minha documentação no GitHub: https://github.com/zecaoliveira/dns-in-docker-swarm

Após a implantação do DNS crie um apontamento para o controller do swarm, exemplo:

- O servidor de DNS atende as requisições da rede no IP: 172.31.0.10
- O controller está configurado para usar o endereço IP: 172.31.0.100.
- Os nodes estão usando os endereços IP's 172.31.0.101 e 102 respectivamente.
- Criar um apontamento DNS do tipo A no PiHole para o controller:
  - Tipo de ponteiro: A
  - Nome: myregistry.mydomain.net
  - IP: 172.31.0.100

Com a configuração acima qualquer máquina usando o servidor de DNS 172.31.0.10 consegue resolver o nome myregistry.mydomain.net para o IP 172.31.0.100.

### 5 - Use self-signed certificates for private registry in Docker Swarm

Como a própria documentação menciona este modelo é o mais seguro se comparado ao de usar HTTP no lugar do HTTPS inserindo no arquivo 'daemon.json' (/etc/docker/daemon.json) o apontamento do nome do host:
```
{
  "insecure-registries" : ["myregistrydomain.com:5000"]
}
```
> ### Nota: este arquivo não existe e ele deve ser criado conforme explicado aqui: https://docker-docs.uclv.cu/registry/insecure/#deploy-a-plain-http-registry

5.1. Criar as pastas de certificados no diretório home do host do Docker Controller:

```
$ mkdir =p certs
```

5.2 - Copiar os certificados "myregistry.domain.com.crt" e "myregistry.domain.com.key" para o diretório ~/certs:

SCP: comando de transferência segura em sistemas UNIX.
/home/$USER/certs: diretório onde estão armazenados os certificados que foram criados.
sysadmx: conta configurada para a administração do host do Docker (Ubuntu Server por exemplo) sem permissão root.
172.31.0.100: IP do servidor host do Docker.
~/certs/: diretório dentro do servidor 172.31.0.100 para onde será enviado os arquivos dos certificados, é o mesmo que "/home/sysadmx/certs".

```
scp /home/$USER/certs sysadmx@172.31.0.100:~/certs/
```

5.3 - Mover o certificado ca.pem para o diretório /usr/local/share/ca-certificates/

```
$ sudo mv ~/certs/ca.pem /usr/local/share/ca-certificates/ca.crt
$ sudo update-ca-certificates
$ sudo reboot
```

5.4 - Após o processo de reboot crie um container com usando os comandos abaixo:
```
docker service create --name my_registry --publish=5000:5000 \
--constraint=node.role==manager \
--mount=type=bind,src=/home/ubuntu/certs,dst=/certs \
-e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/jager.net.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/jager.net.key \
registry:2
```

5.5 - Seguindo a boa prática de automatizar tudo faça um teste usando um Script Shell simulando um dowload de uma imagem, o upload dela no seu container de registro privado de imagens usando TAG e depois faça um download dela a partir do seu reporistório local:

5.5.1 - Script:

- Nota:
  - Após iniciar a criação do script usando o programa "VIM" use o comando de teclado "ESC+I" para começar a editar o arquivo no terminal Linux!
  - Após a inserção do "ESC+I" é possível copiar e colar usando o comando de teclado "CTRL+SHIFT+C" e "CTRL+SHIFT+V" no terminal Linux.
  - Terminou de editar salve o arquivo usando o comando de teclado "ESC+wq+!+ENTER".

```
$ vim test_myregistry.sh

docker pull alpine
docker tag alpine srvnode01.jager.net:5000/my-alpine-2024
docker push srvnode01.jager.net:5000/my-alpine-2024
docker pull srvnode01.jager.net:5000/my-alpine-2024
```
5.5.2 - Após salvar o arquivo dê a permissão para que ele seja executado:

```
$ chmod +x test_myregistry.sh
```
5.5.3 - Execute o teste usando o comando:

```
$ ./test_myregistry.sh
```
Resultado do comando acima:
```
ubuntu@dockercontroller01[~/certs]$ ~/./test_myregistry.sh 
Using default tag: latest
latest: Pulling from library/alpine
Digest: sha256:c5b1261d6d3e43071626931fc004f70149baeba2c8ec672bd4f27761f8e1ad6b
Status: Image is up to date for alpine:latest
docker.io/library/alpine:latest
Using default tag: latest

# Nesta etapa o arquivo foi enviado com sucesso para o container registry:

The push refers to repository [srvnode01.jager.net:5000/my-alpine-2024]
d4fc045c9e3a: Pushed
#
latest: digest: sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0 size: 528
Using default tag: latest

# Aqui estamos fazendo a solicitação (pull) ao registry local e mostra que foi atendida:

latest: Pulling from my-alpine-2024
Digest: sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
Status: Image is up to date for srvnode01.jager.net:5000/my-alpine-2024:latest
srvnode01.jager.net:5000/my-alpine-2024:latest
```
Pelo Portainer dá para acompanhar os logs das solicitações efetuadas pelo script:

![image](https://github.com/zecaoliveira/private-registry-docker-swarm/assets/42525959/ca70c3d3-159f-4c09-9d23-11143d6183fd)

Com isso nós temos todo um ambiente de estudo para praticar um ambiente de desenvolvimento completo usando CI/CD.

# Observações importantes:

Este projeto serve como base para implementações customizadas em ambientes de testes. 

O uso em abientes de produção requer a adição de outros elementos para garantir a alta disponibilidade e segurança e que não eram o escopo deste laboratório.

Adapte-o de acordo com suas necessidades específicas. 

Mantenha-se atualizado sobre as melhores práticas de segurança em Docker e redes containerizadas. 

Desenvolva suas habilidades em DevOps e conquiste as melhores oportunidades no mercado!

Junte-se à comunidade e vamos construir soluções inovadoras e eficientes juntos!

Sucesso na sua jornada pessoal e profissional!

# Créditos e Referências:

Links:

- Using private registry in Docker Swarm: https://codeblog.dotsandbrackets.com/private-registry-swarm/#comment-292784
- Self-Signed Certificates: https://github.com/ChristianLempa/cheat-sheets/blob/main/misc/ssl-certs.md#self-signed-certificates
- Test an insecure registry: https://docker-docs.uclv.cu/registry/insecure/#deploy-a-plain-http-registry
- Install Portainer CE with Docker Swarm on Linux: https://docs.portainer.io/start/install-ce/server/swarm/linux

