# Primeiro acesso e preparação do ambiente
O core 5G para este projeto deverá ser instalado dentro de uma VM Ubuntu server 22.04 com 8gb de memória RAM e 8 processadores. Esta máquina será alocada bare in metal no ambiente de virtualização que também é um Ubuntu XXX. Sendo assim, o primeiro passo é configurar o acesso persistente ao servidor de virtualização e em seguida instalar as dependencias necessárias para adminsitração das VMs via virtmanager.
## Acesso SSH
Pra isso na máquina cliente (o seu host pessoal) é necessário gerar uma chave criptográfica para autenticação automática ssh. A ferramenta utilizada é o gerador que vem instalado por padrão nas dristribuições linux-based ssh-keygen. Para ser gerado onde ele será utilizado o comando abaixo deve ser executado na pasta **/home/user/.ssh**  
```
ssh-keygen 
``` 
Isso vai gerar dois arquivos, no qual o conteúdo do arquivo de extenção **.pub**, deve ser copiado para dentro do arquivo authorized_keys, presente dentro da pasta .ssh do usuário de acesso ao servidor (**/home/server/.ssh**). Existem diversas formas de realizar esta ação, sendo a mais simples através do utilitário **ssh-copy-id**.
```
ssh-copy-id resitech@10.7.101.101
```
## Instalação e gerenciamento com VirtManager
As máquinas virtuais criadas neste servidor poderão ser acessadas através da interface gráfica do VirtManager instalado na máquina cliente. Por isso as seguintes dependencias devem ser instaladas no servidor:
```
apt install -y qemu-system libvirt-daemon-system libvirt-clients bridge-utils virtinst
```
Quando finalizar esta intalação [pde ser necessário iniciar o serviço com o utilitário systemctl e verificar seu status.
```
systemctl status libvirtd
systemctl enable libvirtd
```
No VirtManager instalado na máquina cliente é possivel obter o acesso ao ambiente de virtualização do servidor clicando em **File**, **Add Connection**, marcar a caixa **Connect to remote host over SSH** adicionar o **Username** que no caso é resitech, e o **Hostname** que é 10.7.101.101.
## Virtualizando a interface de rede e configurando vlans
Antes de prosseguir para a criação do host, vamos virtualizar a interface de rede do servidor. Com este nível de segragação, também será possível definir VLAN e atribuir interfaces à vm por meio de passthrough.
O primeiro passo é identificar qual placa rede tem a capacidade de virtualizar interfaces. O primeiro comando abaixo lista os barramentos que contém o termo ethernet. O segundo verifica as informação de cada interface. Para ser considerada habilitada, em suas informações deve conter a "capabilitie" **Single Root I/O Virtualization (SR-IOV)**.
```
lspci | grep -i "ethernet"
sudo lscpi -vs <identificador PCI>
```
Depois de localizar uma interface que pode ser virtualizada, podemos verificar quantas sr-IOV podem ser criadas e criá-las por meio de edição de arquivos de configuração. É preciso cruzar os dados para descobrir o nome da interface. Isso pode ser feito identificando o mesmo MAC address apresentado no comando anterior com as informações apresentadas do comando **ip a**. O primeiro comando abaixo vai listar quantas interfaces a placa de rede suporta. O segundo, quantas interfaces já estão criadas. Note que aqui usamos o nome da interface e nao seu identificador, por isso é importante cruzar estes dados.
```
cat /sys/class/net/<nome da interface>/device/sriov_totalvfs
cat /sys/class/net/<nome da interface>/device/sriov_numvfs
```
A criação de interfaces virtuais na placa de rede envolve a manipulação do mesmo arquivo consultado acima. Mas vale lembrar que para adicionar novas interfaces, o valor do arqivo deve ser modificado para **0** revertendo qualquer configuração anterior em interfaces virtuais caso elas já existam. Os comandos abaixo executam respectivamente o reset da configuração e a criação de 8 SR-IOV identificadas com o nome da interface principal um numero identificador e sinalizador **v**. por exemplo **intv0**.
```
echo "0" > /sys/class/net/<nome da interface>/device/sriov_numvfs
echo "8" > /sys/class/net/<nome da interface>/device/sriov_numvfs
```
## Segmentando VLANs nas interfaces
Para maior desempenho e segurança cada uma das interfaces criadas podem ser dedicadas a cada uma das funções do core que se comunicam externamente. Através de tags nas interfaces virtuais que serão posteriormente dedicadas ao core é possível segmentar o tráfego do UPF, AMF, OAM e Iternet. Neste cenário especifíco serão usadas o indicador **int** como nome da interface principal das vlans e **br101** como interface bridge que vai prover acesso a internet, no qual será dedicado as 4 primeiras interfaces criadas no ambiente. Sendo assim a tabela fica da seguinte forma:
|AMF|UPF|OAM|Internet|
|---|---|---|--------|
|intv0|intv1|intv2|br101|
|VLAN60|VLAN70|VLAN90|BRGD|

A inclusão de tags, de uma forma que possam ser dedicada a VM sem perder sua configuração dever ser realizada por meio de VFs com os seguintes comandos:
```
sudo ip link set int vf 0 vlan 60 spoof off trust off
sudo ip link set int vf 1 vlan 70 spoof off trust off
sudo ip link set int vf 2 vlan 90 spoof off trust off
```
Quando as interfaces forem alocadas para a VM criada no passo seguinte, elas perderão todas a informações, para facilitar a interpretação é possível realizar a alteração no MAC address de cada uma destas interfaces. Neste exemplo modificaremos os dois ultimos digitos de cada uma subtituindo por um valor relativo a sua respectiva vlan com os comandos.
```
sudo ip link set intvX down
sudo ip link set dev intvX address 00:11:22:33:44:55:XX
sudo ip link set intvX up
```
O comando para verificar as modificações do MAC address bem como das TAGs criadas nas interfaces virtuais é o seguinte.
```
ip link show dev int
```
Neste contexto a interface bridge que receberá um ip na mesma rede do servidor não será criada com pass through. Esta deve ser configurada no arquivo .yaml que fica na pasta **/etc/netplan/**. A adição de uma interface no modo brifge que pode ser usada pelas VMs demanada a inclusão das seguiintes informações:
```
bridges:
    br101:
        dhcp: false
        interfaces:
            -eno1
        address: [10.7.101.101/24]
        routes:
            - to: default
              via 10.7.101.1
        nameservers:
            address:
                - 8.8.8.8
version: 2
```
## Criando a Máquina Virtual
Antes de voltar ao VirtManager e iniciar as configurações do servidor podemos aproveitar o acesso ssh para realizar o download da imagem do Ubuntu Server 22.04 LTS. Para facilitar o acesso do utilitário e evitar problemas de permissão de acesso o comando a baixo pode ser rodado a partir do diretório de armazenamento de imagens na path **"/var/lib/libvirt/images/"**.
```
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
```
Com a imagem baixada no servidor podemos voltar ao ambiente de administração de VM da máquina de usuário. Caso tenha se desconectado, e se a configurações estiverem corretas e o servidor acessível, a conexão deverá ser fechada assim que selecionadoa a opção QEMU/KVM com o IP do servidor. 
Se houver máquinas instaladas no ambiente, elas aparecerão no display gráfico automáticamente. Para adicionar uma nova máquina, com o servidor selecionado, deve-se:
- iniciar pela opção **Create a new virtual machine**;
- selecionar o opção **Local install media**;
- escolher o arquivo .iso baixado na pasta **"/var/lib/libvirt/images/"**;
- indicar a distribuição **Ubuntu 22.04**;
- configurar o campo memory com **8196**;
- configurar o campo CPU com o calor **4**;
- manter a opção **Create a disk image for the virtual machine**
- com no mínimo **20GiB**;
- alterar o nome caso seja necessário;
- abrir o opção **network**, alterar para **bridge** e especificar no nome da interface **br101**
- selecionar a caixa **Customize configuration before install**.
Como a última opção sugere, depois de selecionar a tecla **Finish**, uma nova área de configuração será apresentada. Neste momento serão dedicadas as interfaces PCI virtualizadas anteriormente. 
- selcionar a opção **Add Hardware**;
- selecionar no canto esquerdo **PCI Host Device**;
- selecionar as interfaces virtuais criada **3B:02:X**;
- selecionar o opção **finish**.
A interface virtual será então adicionada no painel de administração da VM e a partir de agora vai pertencer à mesma.
## Instalando o Ubuntu Server 22.04 LTS
Depois de finalizar o processo de configuração e pass through do hardware virtualizada para a VM e selecionado, a VM vai inicializar com o setup incial da .ISO do Ubuntu. Seu processo de intalação se assemelha ao de qualquer outra distribuição Linux Based, na qual não será aborada aqui. Com execessão da configuração Netplan.

Para facilitar o trabalho o Ubuntu oferece a opção de configurar manualmente seu gerenciador de redes, o **Netplan**. Apontado quais interfaces serão utilizadas definimos o IP de cada uma das interfaces virtuais dedicadas à VM. Levando em conta o IP terá sempre o final 186 a configuração incial o arquivo final de configuração das interfaces, que fica localizado na path **/etc/netplan/** com a extenção em .yaml, ficará parecida com a disposição das informações abaixo:
```
network:
    ethernets:
        enp1s0:
            dhcp: true
        enp5s0:
            address:
            - 10.7.60.186/24
            nameservers:
                address: []
                search: []
           
        enp6s0:
            address:
            - 10.7.70.186/24
            nameservers:
                address: []
                search: []
           
        enp7s0:
            address:
            - 10.7.90.186/24
            nameservers:
                address:[]
                search: []
           
```
A configuração acima define que a interface enp1s0 criada como NAT por padrão pelo virtmanager seja desativada. As interfaces das VLAN 60,70 e 90 recebam IPs estáticos e a interface intv3, receba IP dinâmico para acesso a internet.
# Instalando e configurando o Open5Gs
A instalação do Open5Gs no Ubuntu é simples e rápida. dentro de VM criada e com privilégio suficiente, basta rodar os seguintes comandos:
```
apt update
apt install software-properties-common
add-apt-repository ppa:open5gs/latest
apt update
apt install open5gs
```
A seequencia de comandos acima adiciona o repositório do open5Gs ao apt, atualiza a máquina e instala o utilitário com todas a dependencias necessárias. Todos os arquivos de configuração podem ser encontrado no diretório **/etc/open5gs/**. Para inicializar a configuração da rede precisamos definir os seus parâmetros.

**PLMN:** 99970
**tac:** 1
**sst:** 1

Essa informações mais os endereções de ip de cada função de rede, atribuídos na interface anteriomente são suficientes para iniciar a configuração no arquivo **/etc/open5gs/amf.yaml** que deve dicar com as seguintes informações

```
     ngap:
       server:
         - address: 10.7.60.186
     metrics:
       server:
         - address: 127.0.0.5
           port: 9090
     guami:
       - plmn_id:
           mcc: 999
           mnc: 70
         amf_id:
           region: 2
           set: 1
     tai:
       - plmn_id:
           mcc: 999
           mnc: 70
         tac: 1
     plmn_support:
       - plmn_id:
           mcc: 999
           mnc: 70
         s_nssai:
           - sst: 1
     security
```
Em seguida o próximo arquivo a ser configurado será o correspondente ao UPF que fica no mesmo diretório do arquivo anterior e pode ser modificado com a path **/etc/open5gs/upf.yaml**. Depois de modificado as informações deverão ser as seguintes:
```
     gtpu:
       server:
         - address: 10.7.70.186
     session:
       - subnet: 10.45.0.1/16
       - subnet: 2001:db8:cafe::1/48
```
Por fim pode-se alterar o arquivo NRF, caso seja necessário. Sua path é **/etc/open5gs/nrf.yaml** e as informações para atender as nossas configurações deverão ser. 
```
 nrf:
   serving:  # 5G roaming requires PLMN in NRF
     - plmn_id:
         mcc: 999
         mnc: 70
   sbi:
     server:
       - address: 127.0.0.10
```
Com os arquivos de configuração modificados é necessário reiniciar os serviços com os seguintes comandos:
```
systemctl restart open5gs-nrfd
systemctl restart open5gs-amfd
systemctl restart open5gs-upfd
```
## Configurando o ambiente gráfico de Administração
O ambiente gráfico de configuração do Open5Gs permite o cadastro e administração de UEs de forma simplificada. Uma vez instalado o Open5Gs, o mesmo se torna acessível na máquina local, através da url **http://localhost:9999**. Ele deve ser intalado separadamente seguindos os passo abaixo.
```
apt install -y ca-certificates curl gnupg
mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
sudo apt install nodejs -y
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```
Para alterar sua forma de acesso uma variável no arquivo **/lib/systemd/system/open5gs-webui.service** deve ser acrescentada. Como o OAM receberá acesso via vlan 90, indicaremos o ip estático desta rede para ser acessível de forma remota. A variavél acrescentada no ambiente em questão será a seguinte: 
```
Environment=HOSTNAME=10.7.90.186
```
Assim como nas funções de rede, para aplicar a modificações é necessário reiniciar o serviço.
```
systemctl reload-daemon
systemctl restart open5gs-webui
```
## Configuração de roteamento para a internet
O open5Gs promove o roteamento para internet por padrão, mas para que as UEs na rede expecificada no UPF possam passar por esse roteamento e alcançar a internet é necessário habilitar o forward no firewall do sistema operacional. Como o ubuntu possui iptables por padrão o primeiro comando habilita o forward e o segundo estabelece uma regra no Iptables.
```
sudo sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```
# Validação de funcionamento com UERANSIM























