No laboratório 3, o objetivo foi criar e configurar um servidor DHCP. Para o desenvolvimento da atividade, foi recomendada a criação de três máquinas: Rn (roteador), S1n (servidor 1) e S2n (servidor 2), em que n corresponde ao valor do terceiro octeto da rede do aluno. Esse valor foi definido no laboratório 2, sendo, neste caso, igual a 57.

Entretanto, foram realizadas adaptações na nomenclatura das máquinas. A máquina Rn, responsável por atuar como servidor DHCP da rede 57, foi nomeada como R7-57, em que 7 representa a posição do computador na sala de laboratório de redes. Já as máquinas S1n e S2n foram nomeadas como S1-57 e S2-57, respectivamente, havendo apenas a adição de um travessão para melhorar a legibilidade.

Em relação à rede, houve o reaproveitamento de uma das redes criadas no laboratório 2, não sendo necessária a criação de uma nova rede no OpenStack. Além disso, conforme ilustrado na topologia, a máquina R7-57 possui duas interfaces de rede: a interface ens3, conectada à rede labredes, que permite a comunicação com as máquinas do laboratório por meio de um IP flutuante (Floating IP); e a interface ens4, conectada à rede 57.

O endereçamento da rede labredes não deve ser alterado, sendo modificada apenas a rede do aluno. Inicialmente, foi realizada uma pesquisa sobre como transformar a máquina R7-57 em um servidor DHCP. Foi então identificada a ferramenta de código aberto isc-dhcp-server, que foi instalada na máquina por meio do comando `sudo apt install isc-dhcp-server`. Essa ferramenta permite que a máquina R7-57 atue como roteador e forneça endereços IP para as máquinas S1-57 e S2-57, conectadas à rede 57.

Após a instalação, é necessário editar o arquivo de configuração dhcpd.conf, utilizando o comando `sudo nano /etc/dhcp/dhcpd.conf`. Nesse arquivo, é possível definir parâmetros como a sub-rede e sua máscara, o intervalo de endereços IP disponíveis (range) e o gateway que, neste caso, corresponde ao IP da máquina R7-57. Essas são as configurações principais, mas há diversas outras opções disponíveis, como a definição de servidores DNS e endereço de broadcast, etc.

No presente caso, foram adicionadas as seguintes configurações ao arquivo dhcpd.conf:

```
subnet 192.168.57.0 netmask 255.255.255.192
{range 192.168.57.2 192.168.57.11;
 range 192.168.57.13 192.168.57.62;
 option routers 192.168.57.12;
 option broadcast-address 192.168.57.63;
 option subnet-mask 255.255.255.192;
 option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

Uma observação importante sobre essa etapa é a utilização de dois intervalos de endereços IP (ranges) e a exclusão do endereço 192.168.57.1 da distribuição automática. Inicialmente, essa decisão foi tomada devido a erros observados nas máquinas, que não estavam conseguindo obter endereços IP corretamente.

Posteriormente, identificou-se que o problema estava relacionado ao security port da rede 57, que estava habilitado e impedia que as máquinas servidoras recebessem endereços IP via DHCP. Após a desativação dessa configuração, as máquinas passaram a obter IPs normalmente.

Ainda assim, optou-se por manter a configuração com dois ranges como forma de demonstração e estudo, evidenciando que é possível segmentar a distribuição de endereços dentro de uma mesma sub-rede e reservar faixas específicas conforme a necessidade.
O próximo passo consistiu em verificar os endereços IP atribuídos às máquinas. Nessa etapa, observou-se que ambas receberam o mesmo IP, o que caracteriza um comportamento inesperado.

Não foi possível determinar com precisão a causa do problema, uma vez que, em condições normais, o servidor DHCP utiliza o endereço MAC de cada máquina para associar um IP único a cada cliente. No entanto, por algum motivo, o servidor não realizou essa distinção adequadamente.

Uma possível explicação é o fato de as máquinas serem virtuais, o que pode levar o servidor DHCP a utilizar outro tipo de identificador em vez do endereço MAC. Dessa forma, levanta-se a hipótese de que ambas as máquinas estejam sendo reconhecidas como o mesmo cliente devido a identificadores idênticos ou mal configurados.

Como continuidade do estudo, seria interessante investigar quais identificadores estão sendo utilizados pelo servidor DHCP nesse ambiente e verificar se há alguma duplicidade entre as máquinas, o que poderia explicar o conflito na atribuição de endereços IP.

Para solucionar esse problema, foi criado um arquivo de configuração com maior prioridade nas máquinas servidoras, com o objetivo de forçar a utilização do endereço MAC pelo servidor DHCP. Isso foi realizado por meio da adição da linha dhcp-identifier: mac no respectivo arquivo de configuração.

Com essa modificação, cada máquina passou a ser identificada de forma única pelo seu endereço MAC, evitando a atribuição de endereços IP duplicados.

Como sugestão para trabalhos futuros, destaca-se a possibilidade de investigar se essa configuração pode ser realizada diretamente na máquina roteadora (servidor DHCP), em vez de ser aplicada individualmente nas máquinas clientes, o que poderia tornar a solução mais centralizada e eficiente.
Uma das etapas propostas no roteiro consiste em configurar o roteador da rede 57 de modo a permitir o roteamento entre essa rede e as redes dos demais colegas, sem a utilização de NAT. Essa etapa é bastante semelhante à atividade realizada no laboratório 2.

Para isso, é necessário configurar corretamente a tabela de rotas em ambas as máquinas roteadoras. Na máquina R7-57, deve-se executar o comando:
```
ip route add <rede_do_colega> via <ip_da_maquina_R_do_colega_na_rede_labredes>
```
De forma análoga, na máquina roteadora do colega (Rn), deve-se executar:
```
ip route add <rede_do_57> via <ip_da_maquina_R7-57_na_rede_labredes>
```
Além disso, é fundamental habilitar o roteamento IP nas duas máquinas, garantindo que os pacotes possam ser encaminhados entre as redes.

A última etapa do roteiro consiste em permitir que uma máquina da rede externa (10.10.0.0/16) seja capaz de acessar o serviço SSH em todas as máquinas da rede interna, incluindo a máquina Rn.

Nessa etapa, foi utilizada a funcionalidade de tunelamento do SSH, e o procedimento foi realizado em duas partes. Inicialmente, cria-se um túnel associando uma porta local (preferencialmente uma porta alta) à porta do serviço SSH de uma das máquinas da rede interna (S1-57 ou S2-57).

O comando utilizado é:

ssh -L [porta_local]:[IP_destino]:[porta_destino] [usuario]@[servidor_ssh]

Onde:

-L: indica o uso de tunelamento local;
[porta_local]: porta da máquina local que será utilizada para o acesso;
[IP_destino]: endereço IP da máquina da rede interna que se deseja acessar;
[porta_destino]: porta do serviço SSH na máquina de destino (geralmente 22);
[usuario]: usuário da máquina R7-57;
[servidor_ssh]: endereço IP da máquina R7-57.

Dessa forma, ao acessar a porta local definida, o tráfego é redirecionado de maneira segura para o serviço SSH da máquina interna, passando pela máquina R7-57.

Após a criação do túnel, a conexão com a máquina de destino pode ser realizada por meio do comando:

ssh -p [porta_local] [usuario_do_destino]@localhost

Assim, o acesso é feito como se o serviço estivesse disponível localmente, embora a conexão esteja sendo encaminhada para a máquina interna via túnel SSH.


OBS: 

ssh:
 -N : não executa um comando remoto (somente port forward)
 -g : permite que hosts remotos se conectem em uma porta local 
 -L : não abre o prompt
 -R : especifica que conexões para uma porta remota no host remoto serão redirecionadas para o host local
 -D : especifica uma aplicação dinâmica de redirecionamento de portas
