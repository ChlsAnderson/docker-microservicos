Docker: Utilização Prática no cenário de Microserviços

Criando um container mysql com docker;
	http://hub.docker.com/_mysql
	-Conectando no banco, no exemplo utilizando o fedora usado o sgdb sequeler
Testando e estressando serviços;
	https://loader.io/tests
		Deve informar o host e gerar um token
			Criar e salvar o arquivo de token .txt com mesmo nome do conteúdo.
				no site criar teste e especificar os parametros como quantidade requisições e tempo.
				Isso vai servir para analisar se o container tá suportando ou se será necessário escalonar mais container.
Para testar os serviços web temos que rodar um container web
	Exemplo de criação usando docker:
		docker run --name web-server -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7
		E aqui foi criado o index.php para chamar aplicação através do servidor web e registrar dados no mysql.
Iniciar cluster Swarm;
	Antes se tiverem container rodando devemos matar ou parar, com o seguinte comando.
		docker rm --force web-server
	Consultar execute o seguinte comando.
		docker ps
	Iniciar o docker swarm
		docker swarm init
	Em outra instancia colocar o comando disponibilizado no primeiro nó onde foi iniciado o swarm, onde informará o ip e porta, depois de rodar o comando, ele irá informar que o nó foi juntado com o swarm do primeiro nó como worker
	Checando os nós do cluster (divisão de carga).
		docker node ls
	Iniciando serviço de vários container em cluster (exemplo acima foi mostrado como iniciar o swarm usando 3 máquinas), e dentro dessas 3 máquinas é possível iniciar mais de 3 containers por exemplo.
	No exemplo abaixo, não é usado o run porque não iremos rodar apenas um container normal.
		docker service create --name web-server --replicas 10 -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7
	Validando onde foi aplicado os containers dentro do cluster com 3 maquinas.
		docker service ps web-server
	Obs.: O problema que dessa forma, ele não replica o conteúdo automáticamente, que são os arquivos index.php e o loader do exemplo.
Replicando o conteúdo do volume.
	Copiar a pasta do servidor lider e replicar ou colar ou criar nos demais servidor worker, exemplo "/var/lib/docker/volumes/app/_data"
	Instalar o serviço de replicação de volume com o nfs-server com o comando abaixo:
		apt-get install nfs-server
	Para o servidores worker, é necessário instalar o cliente com o comando abaixo:
		apt-get install nfs-common
	No servidor configurar arquivo de replicaçao de conteúdo, exemplo abaixo o local do arquivo de configuração.
		/etc/exports
		colocar o caminho da pasta onde está o conteúdo do aplicativo e salvar, exemplo:
			/var/lib/docker/volumes/app/_data *(rw,sync,subtree_check)
		executar o comando abaixo:
			exportfs -ar
	Listando pastas compartilhadas com comando abaixo.
		showmount -e
	Com a informação acima, deve ser montado nas outras máquinas, após confirmar o comando ele demora para copiar.
		mount -o v3 172.31.0.127:/var/lib/docker/volumes/app/_data /var/lib/docker/volumes/app/_data
Criando proxy com nginx para replicar as requisições para todos os servidores automaticamente.
	Entrar na pasta raiz, cd /
	Criar pasta do proxy, mkdir proxy
	Acessar a pasta, cd proxy
	Criar arquivo do nginx, nano nginx.conf
		Conteúdo do arquivo:
			http {
			   
				upstream all {
					server 172.31.0.37:80;
					server 172.31.0.151:80;
					server 172.31.0.149:80;
				}

				server {
					 listen 4500;
					 location / {
						  proxy_pass http://all/;
					 }
				}

			}
			events { }
	Criar um docker file para ser utilizado nas configurações da imagem do nginx do docker quando for criar ele ser copiado para dentro do serviço.
		Conteúdo do docker file, nano dockerfile:
			FROM nginx
			COPY nginx.conf /etc/nginx/nginx.conf
	Criar mais um conteiner com o docker, com isso ele vai utilizar o arquivo do dockerfile e o nginx.conf.
		docker build -t proxy-app .
Conferir todas as imagens do docker rodando.
	docker image ls
	Rodar o proxy do docker.
		docker container run --name my-proxy-app -dti -p 4500:4500 proxy-app
	Validando o container docker
		docker container ls
Estressando o docker com o cluster, utilizando a porta do proxy 4500.
	Criar novo teste no site loader.io e inserindo o host https://host:4500
		Com isso é criado um novo arquivo de autenticação com token
			Criar o novo arquivo no caminho da aplicação /var/lib/docker/volumes/app/_data
				nano colarCódigoDoTokenComoNomeDoArquivo.txt
			Depois autenticar ou testar, e após atenticação criar novo teste.
	Alem de validar com os graficos do site, pode ser feito select no banco para saber se tá gravando correta.
	
Isso tudo pode ser automatizado, por exemplo, utilizando a infraestrutura como código, pode ser programado onde se o processador estiver á 70% criar os container e distribuição de carga.

	
			

	


