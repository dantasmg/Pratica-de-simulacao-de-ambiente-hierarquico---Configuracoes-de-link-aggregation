# Prática de Simulação de Ambiente Hierárquico — Agregação de Links (EtherChannel LACP) e Trunk (Atividade 08)

**Instituição:** Instituto Federal de Educação, Ciência e Tecnologia de Rondônia (IFRO) — *Campus* Porto Velho Zona Norte  
**Curso:** Superior de Tecnologia em Redes de Computadores  
**Disciplina:** Comutação de Redes Locais (TCN.0639)  
**Docente:** Prof. Jhordano Malacarne Bravim  
**Autor(a):** Mariane Gonçalves Dantas
**Ferramenta:** Cisco Packet Tracer

## 1. Objetivo da Atividade
Configurar canais de agregação de links (**Link Aggregation / EtherChannel**) via protocolo **LACP (IEEE 802.3ad)** entre os switches de Distribuição e Núcleo (2 links agregados) e entre os dois switches de Núcleo (4 links agregados), configurando as interfaces lógicas (`Port-channels`) em modo `trunk` e validando a comunicação de ponta a ponta entre ramos opostos da topologia hierárquica.

## 2. Pesquisa e Fundamentação sobre Agregação de Links (IEEE 802.3ad / LACP e PAgP)

* **Resolução do Bloqueio do STP:** Quando múltiplos cabos físicos paralelos interligam dois switches, o **STP (*Spanning Tree Protocol*)** bloqueia todas as portas redundantes (deixando apenas uma ativa em `FWD` e as demais em `Altn BLK`) para evitar loops de Camada 2. Com o **EtherChannel**, os links físicos são agrupados em uma única interface lógica (`Port-channel`), permitindo somar largura de banda, distribuir carga por fluxo e manter redundância automática.
* **LACP (*Link Aggregation Control Protocol* — IEEE 802.3ad / IEEE 802.1AX):** Protocolo **padrão aberto** compatível entre diferentes fabricantes. Opera nos modos `active` (inicia negociação enviando quadros LACP) e `passive` (responde à negociação).
* **PAgP (*Port Aggregation Protocol*):** Protocolo **proprietário da Cisco** (modos `desirable` e `auto`).

### Análise do STP nos Enlaces Paralelos (`NUCLEO01` / `Switch0`)
A execução do comando `show spanning-tree vlan 10` demonstra como o STP trata enlaces paralelos individuais (`Fa2/1` em `Root FWD` e `Fa3/1`, `Fa4/1` e `Fa5/1` bloqueadas em `Altn BLK` para evitar loop):

[CLI%20switch.png]

## 3. Documentação Técnica e Mapeamento dos Port-channels

| Canal Lógico | Switch A | Interfaces Físicas (A) | Switch B | Interfaces Físicas (B) | Protocolo | Modo Trunk | Enlaces Agrupados |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`Po1`** | `Switch0` (`NUCLEO01`) | `Fa2/1, Fa3/1, Fa4/1, Fa5/1` | `Switch1` (`NUCLEO02`) | `Fa2/1, Fa3/1, Fa4/1, Fa5/1` | LACP (`active`) | `trunk (VLANs 10,20)` | **4 enlaces** |
| **`Po2`** | `Switch2` (`DIST01`) | `Fa2/1, Fa3/1` | `Switch0` (`NUCLEO01`) | `Fa8/1, Fa9/1` | LACP (`active`) | `trunk (VLANs 10,20)` | **2 enlaces** |
| **`Po5`** | `Switch3` (`DIST02`) | `Fa4/1, Fa5/1` | `Switch1` (`NUCLEO02`) | `Fa8/1, Fa9/1` | LACP (`active`) | `trunk (VLANs 10,20)` | **2 enlaces** |

## 4. Relação Sequencial de Comandos Cisco IOS (EtherChannel LACP + Trunk)

### 4.1 Switches de Núcleo (`NUCLEO01` e `NUCLEO02`)
```text
enable
configure terminal
hostname NUCLEO01

! Agrupamento dos 4 enlaces inter-núcleo no Port-channel 1 (LACP active)
interface FastEthernet2/1
 channel-group 1 mode active
exit
interface FastEthernet3/1
 channel-group 1 mode active
exit
interface FastEthernet4/1
 channel-group 1 mode active
exit
interface FastEthernet5/1
 channel-group 1 mode active
exit

interface Port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

! Agrupamento dos 2 enlaces para a Distribuição (Po2 no NUCLEO01 / Po5 no NUCLEO02)
interface FastEthernet8/1
 channel-group 2 mode active
exit
interface FastEthernet9/1
 channel-group 2 mode active
exit

interface Port-channel 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

end
copy running-config startup-config
```

### 4.2 Switches de Distribuição (`DIST01` — `Po2` e `DIST02` — `Po5`)
```text
enable
configure terminal
hostname DIST02

interface FastEthernet4/1
 channel-group 5 mode active
exit
interface FastEthernet5/1
 channel-group 5 mode active
exit

interface Port-channel 5
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

end
copy running-config startup-config
```

## 5. Validação dos Canais Agregados (`show etherchannel summary` e `show interfaces trunk`)

* **No `DIST02` (`Switch3`):** O comando `show etherchannel summary` comprova que o grupo **`Po5(SU)`** está ativo em Camada 2 (`S = Layer2`, `U = in use`) via protocolo **LACP**, com as interfaces **`Fa4/1(P)` e `Fa5/1(P)`** associadas ao bundle:

[CLI%20switch%20III.png]

* **No `DIST01` (`Switch2`):** O comando `show interfaces trunk` comprova que a interface lógica **`Po2`** e os troncos de acesso (`Fa0/1` e `Fa1/1`) encaminham normalmente as **VLANs 10 e 20**:

[CLI%20switch%20II.png]

## 6. Testes Finais de Conectividade de Ponta a Ponta

### 6.1 Rastreamento de Rota (`tracert`) de `PC0` (`Switch4`) até `PC3` (`Switch7`)
O comando `tracert 192.168.10.14` disparado do `PC0` (`192.168.10.11`, VLAN 10 na borda esquerda) alcança o `PC3` (`192.168.10.14`, VLAN 10 na borda direita) em **1 salto (`1 ms / 0 ms / 0 ms`)**, pois todos os switches do caminho atuam em **Camada 2** e encaminham o quadro sem alterar o TTL do pacote IP:

[prompt%20de%20comando%20PC0.png]

### 6.2 Rastreamento Visual Passo a Passo Cruzando Toda a Rede (Modo Simulation)

1. **Etapa 1 (`PC0` $\rightarrow$ `Switch4`):** O pacote ICMP sai do `PC0` e chega ao switch de borda esquerdo (`Switch4`).
[Simula%C3%A7%C3%A3o%20I.png]

2. **Etapa 2 (`Switch4` $\rightarrow$ `Switch2 / DIST01`):** O quadro sobe etiquetado na VLAN 10 (802.1Q) para o switch de distribuição esquerdo (`Switch2`).
[Simula%C3%A7%C3%A3o%20II.png]

3. **Etapa 3 (`Switch2 / DIST01` $\rightarrow$ `Switch0 / NUCLEO01`):** O quadro sobe pelo canal agregado `Po2` até o núcleo esquerdo (`Switch0`).
[Simula%C3%A7%C3%A3o%20III.png]

4. **Etapa 4 (`Switch0 / NUCLEO01` $\rightarrow$ `Switch1 / NUCLEO02`):** O quadro atravessa a ligação entre os dois switches de núcleo até chegar ao `Switch1` (`NUCLEO02`).
[Simula%C3%A7%C3%A3o%20V.png]

5. **Etapa 5 (`Switch1 / NUCLEO02` $\rightarrow$ `Switch3 / DIST02` $\rightarrow$ `Switch7`):** O quadro desce pelo canal agregado `Po5` na distribuição direita (`Switch3`) e alcança o switch de borda direito (`Switch7`).
[Simula%C3%A7%C3%A3o%20IV.png]

6. **Etapa 6 (`Switch7` $\rightarrow$ `PC3`):** O `Switch7` retira a tag 802.1Q e entrega o pacote na porta de acesso da VLAN 10 ao destino final (`PC3`).
[Simula%C3%A7%C3%A3o%20VI.png]
