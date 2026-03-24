
O primeiro passo do laboratório foi montar a rede lógica composta por quatro maquinas virtuais, C1, G1, G2 e C2, conforme especificado no roteiro. Além disso, foi definido que deveriam ser criadas três redes diferentes, sendo:

	Rede 1 entre as máquinas C1 e G1 
	Rede 2 entre G1 e G2
	Rede 3 entre G2 e C2 

A faixa de endereçamento utilizada para a configuração das redes foi definida com base na posição dos alunos na sala. COmo minha posição é a mesa 7, a faixa disponibilizada foi 192.168.57.0/24. A partir dessa faixa de endereço, foram criadas sub-redes utilizando a máscara 255.255.255.192 (/26). Cada sub-rede possui 64 endereços IP, sendo 62 endereços válidos para hosts. Apesar de apenas três reedes serem necessárias para o laboratório, optou-se por dividir a rede em quatro sub-redes, deixando uma disponível para utilizar no futuro.

Buscando manter um padrão de nomenclatura, as redes foram nomeadas da seguinte forma:

	rede57_N-ter

Onde, 57 corresponde ao terceiro octeto da rede atribuída ao aluno, N representa o número da sub-rede e ter indica que o laboratório ocorre na terça-feira. As sub-resdes foram definidas da seguinte forma:


	Rede rede57_1-ter:
		subnet: 192.168.57.0
		host address: 192.168.57.1 – 192.168.57.62
		broadcast: 192.168.57.63

	Rede rede57_2-ter:
		subnet: 192.168.57.64
		host address: 192.168.57.65 – 192.168.57.126
		broadcast: 192.168.57.127

	Rede rede57_3-ter:
		subnet: 192.168.57.128
		host address: 192.168.57.129 – 192.168.57.190
		broadcast: 192.168.57.191

	Rede rede57_4-ter:
		subnet: 192.168.57.192
		host address: 192.168.57.193 – 192.168.57.254
		broadcast: 192.168.57.255

Após a criação das máquinas virtuais e das redes no ambiente OpenStack, utilizando as configurações exigidas pelo professor, foi iniciado o proccesso de execução do roteiro.

Inicialmente, a máquina C1 recebeu um Floaing IP, que permite que as máquinas físicas do laboratório acessem a máquina virtual. Isso é possível porque uma das interfaces da máquina C1 está conectada à rede externa labredes1.

Os testes de conectividade foram realizados utilizando o comando ping, inicialmente entre máquinas conectadas diretamente:

	C1 	<-->  G1
	G1  <-->  G2
	G2  <-->  C2

Posteriormente foram realizados testes entre máquinas que não estavam diretamente conectadas:

	C1 	<-->  G2
	C1 	<-->  C2
	C2 	<-->  G1

Para o primeiro conjunto de testes, a máquina C1 não precisou de configuração adicional. Entretanto, as demais máquinas precisaram ser configuradas com endereços IP fixos e rotas.

Inicialmente, essas configurações foram feitas utilizando comandos diretamente no terminal, como:

	ip addr para configurar os endereços IP
	ip route add para adicionar rotas na tabela de roteamento

Entretanto, essas alterações eram temporárias e eram perdidas após a reinicialização da máquina. Por esse motivo, foi necessário pesquisar uma forma de tornar essas configurações permanentes.

A solução encontrada foi modificar os arquivos de configuração do Netplan, localizados no diretório:

	/etc/netplan/

Inicialmente tentou-se modificar o arquivo:

	/etc/netplan/50-cloud-init.yaml

Porém esse arquivo é sobrescrito automaticamente pelo OpenStack sempre que a máquina é reiniciada, pois ele é gerado pelo sistema de cloud-init.

Como o Netplan aplica os arquivos de configuração de forma hierárquica, baseada no número presente no nome do arquivo, foi criada uma nova configuração com prioridade maior:

	/etc/netplan/99-netcfg.yaml

Dessa forma, como 99 possui prioridade maior que 50, as configurações definidas nesse arquivo sobrescrevem as configurações geradas automaticamente pelo cloud-init.

Assim, foi criado um arquivo 99-netcfg.yaml em cada máquina, contendo as configurações de IP fixo e rotas necessárias para permitir a comunicação entre todas as redes do laboratório 2. 

	# maquina C1-57
	network:
	version: 2
	ethernets:
		ens3:
		match:
			macaddress: "fa:16:3e:a3:ae:f3"
		dhcp4: true
		set-name: "ens3"
		mtu: 1450
		ens4:
		match:
			macaddress: "fa:16:3e:75:e9:6c"
		addresses:
		- "192.168.57.50/26"
		routes:
		- to: 192.168.57.64/26
			via: 192.168.57.46
		- to: 192.168.57.128/26
			via: 192.168.57.46
		nameservers:
			addresses:
			- 8.8.8.8
			search: []
		set-name: "ens4"
		mtu: 1450

	# 99-netcfg.yaml -- maquina G1-57
	network:
	version: 2
	ethernets:
		ens3:
		match:
			macaddress: "fa:16:3e:e4:dc:eb"
		addresses:
		- "192.168.57.46/26"
		dhcp4: false
		dhcp6: false
		set-name: "ens3"
		ens4:
		match:
			macaddress: "fa:16:3e:2b:37:d8"
		addresses:
		- "192.168.57.92/26"
		routes:
		- to: 192.168.57.128/26
			via: 192.168.57.121
		dhcp4: false
		dhcp6: false
		set-name: "ens4"


	# 99-netcfg.yaml -- maquina G2-57
	network:
	version: 2
	ethernets:
		ens3:
		match:
			macaddress: "fa:16:3e:de:ed:b7"
		addresses:
		- "192.168.57.121/26"
		routes:
		- to: 192.168.57.0/26
			via: 192.168.57.92
		dhcp4: false
		dhcp6: false
		set-name: "ens3"
		ens4:
		match:
			macaddress: "fa:16:3e:55:94:52"
		addresses:
		- "192.168.57.169/26"
		dhcp4: false
		dhcp6: false
		set-name: "ens4"

	# 99-netcfg.yaml -- maquina C2-57
	network:
	version: 2
	ethernets:
		ens3:
		match:
			macaddress: "fa:16:3e:3d:b3:4c"
		addresses:
		- "192.168.57.188/26"
		routes:
		- to: 192.168.57.0/26
			via: 192.168.57.169
		- to: 192.168.57.64/26
			via: 192.168.57.169
		dhcp4: false
		dhcp6: false
		set-name: "ens3"

Após a criação dos arquivos de configuração no diretório /etc/netplan/, foi necessário aplicar as novas configurações de rede utilizando o comando:

	sudo netplan apply

Esse comando faz com que o Netplan processe os arquivos de configuração YAML e aplique as alterações nas interfaces de rede do sistema, permitindo que os endereços IP, rotas e demais parâmetros definidos entrem em funcionamento.

Além disso, para que as máquinas G1 e G2 pudessem atuar como roteadores, foi necessário habilitar o encaminhamento de pacotes (IP Forwarding) no sistema. Inicialmente isso pode ser feito temporariamente com o comando:

	echo 1 > /proc/sys/net/ipv4/ip_forward

Entretanto, para que essa configuração permanecesse ativa mesmo após a reinicialização da máquina, foi necessário modificar o arquivo de configuração do sistema:

	sudo nano /etc/sysctl.conf

Nesse arquivo foi descomentadaa seguinte linha:

	net.ipv4.ip_forward = 1

Essa configuração habilita permanentemente o roteamento de pacotes IPv4 no sistema operacional, permitindo que as máquinas G1 e G2 encaminhem pacotes entre diferentes redes, possibilitando a comunicação entre todas as máquinas virtuais do laboratório 2.
