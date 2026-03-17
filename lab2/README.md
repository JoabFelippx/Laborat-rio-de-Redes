# Laboratório 2 — Configuração de uma rede local

## Descrição

Este laboratório consiste na configuração de uma rede lógica com quatro máquinas virtuais (**C1**, **G1**, **G2** e **C2**) no ambiente OpenStack, divididas em sub-redes distintas. O objetivo é habilitar a comunicação entre todas as máquinas, inclusive aquelas que não estão diretamente conectadas.

---

## Topologia

<img src="topologia.png" alt="Descrição" width="500">

| Máquina | Interface | Endereço IP       | Rede            |
|---------|-----------|-------------------|-----------------|
| C1      | ens4      | 192.168.57.50/26  | rede57_1-ter    |
| G1      | ens3      | 192.168.57.46/26  | rede57_1-ter    |
| G1      | ens4      | 192.168.57.92/26  | rede57_2-ter    |
| G2      | ens3      | 192.168.57.121/26 | rede57_2-ter    |
| G2      | ens4      | 192.168.57.169/26 | rede57_3-ter    |
| C2      | ens3      | 192.168.57.188/26 | rede57_3-ter    |

> C1 também possui uma interface conectada à rede externa `labredes1`, com um **Floating IP** para acesso externo.

---

## Endereçamento — Sub-redes

Faixa base atribuída: `192.168.57.0/24` (mesa 7)  
Máscara utilizada: `255.255.255.192` (/26) — 62 hosts por sub-rede

| Sub-rede        | Network           | Hosts válidos                       | Broadcast        |
|-----------------|-------------------|-------------------------------------|------------------|
| rede57_1-ter    | 192.168.57.0      | 192.168.57.1 – 192.168.57.62        | 192.168.57.63    |
| rede57_2-ter    | 192.168.57.64     | 192.168.57.65 – 192.168.57.126      | 192.168.57.127   |
| rede57_3-ter    | 192.168.57.128    | 192.168.57.129 – 192.168.57.190     | 192.168.57.191   |
| rede57_4-ter    | 192.168.57.192    | 192.168.57.193 – 192.168.57.254     | 192.168.57.255   |

> A sub-rede `rede57_4-ter` foi criada como reserva para uso futuro.

---

## Configuração Permanente de Rede (Netplan)

As configurações temporárias via `ip addr` e `ip route add` são perdidas após reinicialização. A solução adotada foi criar o arquivo `/etc/netplan/99-netcfg.yaml` em cada máquina, com prioridade maior que o arquivo `50-cloud-init.yaml` gerado automaticamente pelo OpenStack.

Após editar o arquivo, aplicar com:

```bash
sudo netplan apply
```

### C1

```yaml
# /etc/netplan/99-netcfg.yaml — C1-57
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
      set-name: "ens4"
      mtu: 1450
```

### G1

```yaml
# /etc/netplan/99-netcfg.yaml — G1-57
network:
  version: 2
  ethernets:
    ens3:
      match:
        macaddress: "fa:16:3e:e4:dc:eb"
      addresses:
        - "192.168.57.46/26"
      dhcp4: false
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
      set-name: "ens4"
```

### G2

```yaml
# /etc/netplan/99-netcfg.yaml — G2-57
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
      set-name: "ens3"
    ens4:
      match:
        macaddress: "fa:16:3e:55:94:52"
      addresses:
        - "192.168.57.169/26"
      dhcp4: false
      set-name: "ens4"
```

### C2

```yaml
# /etc/netplan/99-netcfg.yaml — C2-57
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
      set-name: "ens3"
```

---

## IP Forwarding (G1 e G2)

Para que G1 e G2 atuem como roteadores, é necessário habilitar o encaminhamento de pacotes IPv4.

**Temporariamente:**
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

**Permanentemente** — editar `/etc/sysctl.conf` e descomentar:
```
net.ipv4.ip_forward = 1
```

---

## Testes de Conectividade

Utilize `ping` para verificar a comunicação entre as máquinas.

**Conexões diretas:**
```
C1 <--> G1
G1 <--> G2
G2 <--> C2
```

**Conexões indiretas (roteamento):**
```
C1 <--> G2
C1 <--> C2
C2 <--> G1
```

---

## Observações

- O arquivo `50-cloud-init.yaml` é **sobrescrito pelo OpenStack** a cada reinicialização.
- O Netplan aplica configurações em **ordem numérica crescente**, portanto `99-netcfg.yaml` tem prioridade sobre `50-cloud-init.yaml`.
- O Floating IP em C1 é necessário para acesso SSH a partir das máquinas físicas do laboratório.
