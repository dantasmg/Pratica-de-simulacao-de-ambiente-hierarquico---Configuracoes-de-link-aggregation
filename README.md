# Prática de Simulação de Ambiente Hierárquico — Agregação de Links (EtherChannel LACP) e Trunk (Atividade 08)

**Instituição:** Instituto Federal de Educação, Ciência e Tecnologia de Rondônia (IFRO) — *Campus* Porto Velho Zona Norte  
**Curso:** Superior de Tecnologia em Redes de Computadores  
**Disciplina:** Comutação de Redes Locais (TCN.0639)  
**Docente:** Prof. Jhordano Malacarne Bravim  
**Autor(a):** Mariane Gonçalves Dantas

**Ferramenta:** Cisco Packet Tracer

## 1. Objetivo da Atividade
Configurar canais de agregação de links (**Link Aggregation / EtherChannel**) via protocolo **LACP (IEEE 802.3ad)** entre os switches de Distribuição e Núcleo (2 links agregados) e entre os dois switches de Núcleo (4 links agregados), configurando as interfaces lógicas (`Port-channels`) em modo `trunk` e validando a comunicação de ponta a ponta entre ramos opostos da topologia hierárquica.

<img width="847" height="626" alt="Print Packet Tracer" src="https://github.com/user-attachments/assets/e94d25a0-3947-4977-8bc2-9c025477295a" />


## 2. Pesquisa e Fundamentação sobre Agregação de Links (IEEE 802.3ad / LACP e PAgP)

* **Resolução do Bloqueio do STP:** Quando múltiplos cabos físicos paralelos interligam dois switches, o **STP (*Spanning Tree Protocol*)** bloqueia todas as portas redundantes (deixando apenas uma ativa em `FWD` e as demais em `Altn BLK`) para evitar loops de Camada 2. Com o **EtherChannel**, os links físicos são agrupados em uma única interface lógica (`Port-channel`), permitindo somar largura de banda, distribuir carga por fluxo e manter redundância automática.
* **LACP (*Link Aggregation Control Protocol* — IEEE 802.3ad / IEEE 802.1AX):** Protocolo **padrão aberto** compatível entre diferentes fabricantes. Opera nos modos `active` (inicia negociação enviando quadros LACP) e `passive` (responde à negociação).
* **PAgP (*Port Aggregation Protocol*):** Protocolo **proprietário da Cisco** (modos `desirable` e `auto`).

### Análise do STP nos Enlaces Paralelos (`NUCLEO01` / `Switch0`)
A execução do comando `show spanning-tree vlan 10` demonstra como o STP trata enlaces paralelos individuais (`Fa2/1` em `Root FWD` e `Fa3/1`, `Fa4/1` e `Fa5/1` bloqueadas em `Altn BLK` para evitar loop):

<img width="700" height="708" alt="CLI switch" src="https://github.com/user-attachments/assets/595f6dda-c7cf-4fe8-a7bb-7350b6c7bf00" />


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

<img width="702" height="711" alt="CLI switch III" src="https://github.com/user-attachments/assets/7d95b909-78ac-4b54-b362-c62a1d66a180" />

* **No `DIST01` (`Switch2`):** O comando `show interfaces trunk` comprova que a interface lógica **`Po2`** e os troncos de acesso (`Fa0/1` e `Fa1/1`) encaminham normalmente as **VLANs 10 e 20**:

<img width="700" height="709" alt="CLI switch II" src="https://github.com/user-attachments/assets/f4c8d4d6-c4a9-47b5-82d4-ab24bc119aaf" />

## 6. Testes Finais de Conectividade de Ponta a Ponta

### 6.1 Rastreamento de Rota (`tracert`) de `PC0` (`Switch4`) até `PC3` (`Switch7`)
O comando `tracert 192.168.10.14` disparado do `PC0` (`192.168.10.11`, VLAN 10 na borda esquerda) alcança o `PC3` (`192.168.10.14`, VLAN 10 na borda direita) em **1 salto (`1 ms / 0 ms / 0 ms`)**, pois todos os switches do caminho atuam em **Camada 2** e encaminham o quadro sem alterar o TTL do pacote IP:

<img width="699" height="284" alt="prompt de comando PC0" src="https://github.com/user-attachments/assets/74e701b9-a2f8-4d43-990a-f144796e8c5e" />


### 6.2 Rastreamento Visual Passo a Passo Cruzando Toda a Rede (Modo Simulation)

1. **Etapa 1 (`PC0` $\rightarrow$ `Switch4`):** O pacote ICMP sai do `PC0` e chega ao switch de borda esquerdo (`Switch4`).
<img width="870" height="587" alt="Simulação I" src="https://github.com/user-attachments/assets/52fa353d-ed09-4728-8c22-40effa607eab" />

2. **Etapa 2 (`Switch4` $\rightarrow$ `Switch2 / DIST01`):** O quadro sobe etiquetado na VLAN 10 (802.1Q) para o switch de distribuição esquerdo (`Switch2`).
<img width="914" height="595" alt="Simulação II" src="https://github.com/user-attachments/assets/cb1e8441-436a-43e1-9edd-d34b46aa25f5" />

3. **Etapa 3 (`Switch2 / DIST01` $\rightarrow$ `Switch0 / NUCLEO01`):** O quadro sobe pelo canal agregado `Po2` até o núcleo esquerdo (`Switch0`).
<img width="874" height="595" alt="Simulação III" src="https://github.com/user-attachments/assets/701746f3-5ad0-4256-b8dd-8046cd01b5e3" />

4. **Etapa 4 (`Switch0 / NUCLEO01` $\rightarrow$ `Switch1 / NUCLEO02`):** O quadro atravessa a ligação entre os dois switches de núcleo até chegar ao `Switch1` (`NUCLEO02`).
<img width="832" height="584" alt="Simulação V" src="https://github.com/user-attachments/assets/8f097a2c-6b90-4da4-ab49-1f6908344819" />

5. **Etapa 5 (`Switch1 / NUCLEO02` $\rightarrow$ `Switch3 / DIST02` $\rightarrow$ `Switch7`):** O quadro desce pelo canal agregado `Po5` na distribuição direita (`Switch3`) e alcança o switch de borda direito (`Switch7`).
<img width="813" height="574" alt="Simulação IV" src="https://github.com/user-attachments/assets/d850119d-aa36-47a0-9e08-de0e873164ea" />

6. **Etapa 6 (`Switch7` $\rightarrow$ `PC3`):** O `Switch7` retira a tag 802.1Q e entrega o pacote na porta de acesso da VLAN 10 ao destino final (`PC3`).
<img width="924" height="579" alt="Simulação VI" src="https://github.com/user-attachments/assets/3b560601-6d45-4ff9-bc23-4b5b5a399f2e" />


