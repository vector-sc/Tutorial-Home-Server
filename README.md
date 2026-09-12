
Aprenda como criar um servidor simples em sua própria casa utilizando um computador velho, uma Raspberry PI ou, neste caso, uma TV Box formatada.


# Escopo, Importância e Requisitos
O objetivo deste tutorial é ensinar como criar um servidor local em casa. Um servidor local pode ser utilizado para inúmeras coisas:
- Atuação como host de dispositivos IoT. (ex: Home Assistant) 
- Servidor de mídia (ex: Jellyfin)
- Host de servidores para jogos
- Armazenamento compartilhado por rede
- Tudo isso junto (por meio de proxmox)

Devido à essa variedade e considerando as limitações físicas de uma TV Box, focaremos em um servidor com armazenamento compartilhado por rede, porém não se limite à isso. Após compreender os mecanismos básicos, pesquise e experimente!

Além disso, se você é um estudante de Ciência da Computação, o estudo de instalações como esta abre portas para aplicações bem diferentes da computação, fora do desktop comum, ampliando as oportunidades que você pode encontrar futuramente.

Para este tutorial, serão utilizadas duas (2) TV Boxes formatadas, dois (2) cabos de rede, um (1) roteador e os periféricos necessários para a operação das TV Boxes (monitores, teclados, mouses). Em termos de software, utilizaremos o Armbian como sistema operacional, em conjunto com os pacotes nfs-kernel-server, nfs-common e autofs.

A formatação de TV Boxes não será discutida neste tutorial por ser algo que varia bastante e depende muito do modelo, porém é bem fácil pesquisar sobre seu modelo específico para encontrar informações à respeito do processo (se possível).

## Informações gerais do Hardware e Software
As TV Boxes utilizadas nesse tutorial são do modelo "RPC PLUS" e apresentam cerca de 8 GB de armazenamento, 1 GB de RAM, arquitetura ARMv7l, processador rk3228A e GPU Mali-400MP. Em resumo, é um dispositivo extremamente fraco. Se o seu computador ter propriedades semelhantes ou melhores, você já sabe que é possível ter um servidor funcional.

O sistema operacional Armbian foi escolhido por ser mais compatível e otimizado neste modelo de TV Box, porém a maioria dos sistemas Linux tem o potencial de atuar como servidores locais. Os passos desse tutorial, porém, só são geralmente aplicáveis para sistemas derivados do Debian (como Ubuntu).

O roteador é essencial para estabelecer a rede do servidor. Este sistema foi testado apenas com conexões por meio de cabos de rede, porém é sim possível estabelecer conexões sem fio.

A criação da rede de armazenamento será possível por meio do Network File System (NFS), essencial para o compartilhamento de arquivos por rede. Por meio do auto file system (autofs), conseguiremos configurar o sistema para que o armazenamento compartilhado seja montado no seu computador de forma automática quando uma conexão com o servidor estiver sendo detectada.

# SETUP
Neste tutorial, uma TV Box atuará como "Cliente", ou seja, qualquer computador configurado para mandar arquivos para o servidor, enquanto a outra será o servidor em sí, armazenando os arquivos e enviando-os para quem pedir.

Primeiramente é preciso instalar os pacotes essenciais. No servidor, execute esses comandos no terminal:

```
sudo apt install nfs-kernel-server
```
Os clientes, por outro lado, utilizará esses pacotes:

```
sudo apt install nfs-common
sudo apt install autofs
```
Não é preciso instalar o pacote nfs-common, pois ele já vem incluído na instalação padrão do Armbian.

### Configuração da Rede
Para criar a rede local, será necessário definir endereços de IP estáticos. Isto é feito localmente em cada dispositivo por meio do netplan. Para isso, é preciso primeiro identificar a rede do roteador por meio do comando `ìp a`. O computador precisa estar conectado à rede e, ao utilizar o comando, várias informações aparecerão na tela.
```
  ..
  ..
end0: <###################> #################### UP ################
   ##############################################
    #### xxx.xxx.x.30/24 #### xxx.xxx.x.255 #######################
    ..
    ..
```
O exemplo acima mostra algo parecido com o que buscamos. Neste caso específico, end0 é o nome da conexão por cabo de rede, mas não é a única possibilidade (ex:eno0, eno1, enp0s25, eth0). O primeiro endereço de IP encontrado é o ip do dispositivo atual, enquanto o segundo endereço é o "gateway", utilizado para ir até o roteador. Utilizaremos essas informações na configuração do netplan.

Em /etc/netplan, crie um arquivo chamado "99_config.yaml" e preencha-o da forma abaixo (mas não escreva os comentários):
```
network:
  version: 2
  renderer: networkd
  ethernets:
    eno0: # nome da conexão por cabo.
      addresses:
        - xxx.xxx.x.y/24 # ip estático. X igual ao endereço obtido pelo ip a, mas o y pode ser qualquer valor entre 0 e 255.
      routes:
        - to: default
          via: xxx.xxx.x.x # ip da rede, o gateway.
      nameservers:
        search: [mydomain, otherdomain]
        addresses: [10.10.10.1, 1.1.1.1]
```
Para aplicar a configuração, digite no terminal ``sudo netplan apply``. Caso tudo esteje correto, não haverá nenhum aviso de erro grave, porem, se necessário, a configuração pode ser revertida renomeando o arquivo para 99_config e aplicando novamente. Faça isto para todos os computadores, providenciando um ip estático para cada um dos clientes e o servidor.

Para confirmar que a rede está funcional, execute o comando ping ``ping xxx.xxx.x.y``, fornecendo o endereço estático de outro computador na rede, como o servidor. Se tudo estiver correto, o terminal avisará que o computador recebeu bytes do computador escolhido e informará quanto tempo demorou para receber uma resposta. Uma falha neste teste pode indicar que o firewall está impedindo a comunicação ou há algum problema na configuração. No caso do firewall, use o comando ``sudo ufw allow from xxx.xxx.x.x/24`` (onde o endereço fornecido é o gateway da rede) nos dispositivos envolvidos.

Por fim, é ideal editar o arquivo em /etc/hosts. Neste arquivo, coloque o endereço estático do servidor e um nome para ele. Isto permite associar a palavra homeServer com o endereço do servidor (ex: ping homeServer). Faça isso para todos os computadores.
```
xxx.xxx.x.y homeServer
```
### Compartilhamento de arquivos
Para começar, é preciso criar uma pasta onde os arquivos compartilhados serão armazenados. No servidor, crie uma pasta no diretório /mnt, como por exemplo ``/mnt/pastaCompartilhada`` e depois faça as seguintes alterações de permissão por meio do terminal:
```
sudo chown nobody:nogroup /mnt/pastaCompartilhada
sudo chmod 777 /mnt/pastaCompartilhada
```
Isto faz com que todos tenham permissão de acessar e modificar os conteúdos desta pasta.

Agora, vá até /etc/exports e adicione o caminho para a pasta à ser compartilhada:
```
/mnt/pastaCompartilhada *(rw,sync,no_subtree_check)
```
Reinicie o serviço nfs-kernel-server para aplicar a alteração com o comando ``sudo service nfs-kernel-server restart``. Digite ``showmount -e homeServer`` e se o comando retornar o caminho para a pasta, a configuração está correta.

Com isso, já é possível testar o compartilhamento. No computador cliente, digite o seguinte comando:
```
sudo mount homeServer:/mnt/pastaCompartilhada /mnt/pastaCompartilhada
```
O primeiro argumento é o caminho para a pasta compartilhada no servidor, enquanto o segundo é o local onde a pasta será colocada no dispositivo atual. Caso tudo esteje funcionando, a pasta irá aparecer no computador cliente. Ao colocar um arquivo na pasta, este arquivo será transmitido por rede até o servidor, onde será armazenado. 

### Montagem automática
O único problema com a configuração atual é que para acessar o diretório compartilhado, o cliente deve montar manualmente a pasta toda vez que ligar o computador, sem considerar também que caso o servidor seja desligado, a conexão irá travar, causando vários problemas. Isto pode ser resolvido com o pacote autofs.

No arquivo /etc/auto.master, é preciso adicionar um caminho para um arquivo de configuração para o nosso uso.
```
/- /etc/auto.fsweb
```
A extensão do arquivo pode ser qualquer coisa, como .servidor, .fsweb, etc. Neste caso, utilizarei auto.fsweb. Crie o arquivo mencionado no caminho indicado e preencha-o com o seguinte:
```
/mnt/pastaCompartilhada -vers=4,rw,soft,bg,intr,retry=0,retrans=1,timeo=1 homeServer:/mnt/pastaCompartilhada
```
O primeiro argumento é o local onde a pasta será montada no dispositivo atual, seguido por argumentos que configuram o sistema como número de tentativas de montagem, tempo até timeout, etc. Por fim, o caminho onde a pasta está localizada no servidor. Após isso, basta reiniciar o computador cliente e, se tudo estiver configurado corretamente, a pasta será montada de forma automática e removida caso a conexão não responda por muito tempo.

# Conclusão
Apesar deste servidor demonstrado servir para algo bem básico, é importante entender que ter o próprio servidor lhe permite ter mais controle sobre muitas coisas, já que possibilita trazer serviços exclusivamente armazenados na nuvem para sua própria casa por meio de alternativas open-source. Sobre isso, o vídeo abaixo destaca um dos vários problemas causados pela dependência de serviços na nuvem.

https://www.youtube.com/watch?v=PFbxapX4bRE

