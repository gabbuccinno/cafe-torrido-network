# cafe-torrido-network # Café Torrido -- Infraestrutura de Rede Corporativa

Projeto completo de design, simulação e documentação de rede corporativa para o Café Torrido, Belo Horizonte - MG.
Simulado integralmente no **Cisco Packet Tracer**.

## Visão Geral

Rede segmentada em **7 VLANs** com firewall de perímetro dual WAN, roteamento inter-VLAN, servidor NAS centralizado, câmeras IP, Access Points WiFi e cabeamento estruturado Cat6 com PoE.

### Topologia

        ISP-1 (198.41.128.x)     ISP-2 (198.41.130.x)
               ↓                         ↓
            Modem-1                   Modem-2
               ↓                         ↓
        ┌──────────────────────────────────┐
        │            ASA 5506-X            │
        │  Firewall / Dual WAN / NAT       │
        │  outside-1 / outside-2 / inside  │
        └──────────────────┬───────────────┘
                           ↓ 10.0.80.x
                    ┌──────────────┐
                    │   Cisco 2911 │
                    │    subnet    │
                    │    10-70     │
                    └──────┬───────┘
                           ↓ native 40 trunk
                    ┌──────────────┐
                    │  Catalyst    │
                    │    2960      │
                    │ SW Principal │
                    └──┬───┬───┬───┘
                       ↓   ↓   ↓
               MONITOR-SW NAS AP-SW
               (câmeras) (serv.) (WiFi)

# Arquitetura

### Dispositivos

| Dispositivo | Modelo | IP | Função |
|-------------|------------------|-------------|-----------------------------|
| bh-fw01     | Cisco ASA 5506-X | 10.0.80.250 | Firewall, dual WAN, NAT     |
| bh-rtr1     | Cisco 2911       | 10.0.80.251 | Roteamento inter-VLAN       |
| bh-sw01     | Catalyst 2960    | 10.0.40.2   | Switch principal, DHCP      |
| MONITOR-SW  | Cisco 2950T-24   | 10.0.40.4   | Switch câmeras IP           |
| AP-SW       | Cisco 2950T-24   | 10.0.40.3   | Switch Access Points        |
| BH-NAS      | TrueNAS SCALE    | 10.0.40.5   | NAS, DNS, SYSLOG, TFTP, IoT |
| Notebook TI | Dell Latitude    | 10.0.40.30  | Gerência exclusiva          |

### VLANs

| VLAN | Nome       | Rede         | DHCP Pool  | Função                   |
|------|------------|--------------|------------|--------------------------|
|  10  | ADM        | 10.0.10.0/24 | .11 – .254 | Administração / Sócios   |
|  20  | PROD       | 10.0.20.0/24 | .11 – .254 | Produção / Depósito      |
|  30  | VENDAS     | 10.0.30.0/24 | .11 – .254 | Vendas / Atendimento     |
|  40  | MGMT       | 10.0.40.0/24 | .31 – .254 | Gerência de dispositivos |
|  50  | WIFI-CORP  | 10.0.50.0/24 | .11 – .254 | WiFi corporativo         |
|  60  | WIFI-GUEST | 10.0.60.0/24 | .11 – .254 | WiFi visitantes          |
|  70  | WIFI-OPEN  | 10.0.70.0/24 | .11 – .254 | WiFi café aberto         |
|  80  | RTR-ASA    | 10.0.80.0/24 |   —        | Link roteador ↔ firewall |


## Políticas de Segurança

### Acesso inter-VLAN

| Origem            | ADM | PROD | VENDAS | NAS | Câmeras | MGMT | WiFi 60/70 | Internet |
|-------------------|-----|------|--------|-----|---------|------|------------|----------|
| VLAN10 ADM        | ✅ |  ✅  |   ✅  | ✅  |   ✅   |  ❌  |    ❌     |    ✅    |
| VLAN20 PROD       | ✅ |  ✅  |   ✅  | ✅  |   ❌   |  ❌  |    ❌     |    ✅    |
| VLAN30 VENDAS     | ✅ |  ✅  |   ✅  | ✅  |   ❌   |  ❌  |    ❌     |    ✅    |
| VLAN40 MGMT       | ✅ |  ✅  |   ✅  | ✅  |   ✅   |  ✅  |    ✅     |    ✅    |
| VLAN50 WiFi Corp  | ✅ |  ✅  |   ✅  | ✅  |   ❌   |  ❌  |    ❌     |    ✅    |
| VLAN60 Guest      | ❌ |  ❌  |   ❌  | ❌  |   ❌   |  ❌  |    ❌     |    ✅    |
| VLAN70 Café Open  | ❌ |  ❌  |   ❌  | ❌  |   ❌   |  ❌  |    ❌     |    ✅    |

### Firewall (ASA 5506-X)

- Dual WAN com failover por métrica — ISP-1 preferencial, ISP-2 backup
- NAT/PAT — IPs internos ocultos atrás do IP público
- ACL de perímetro — anti-spoofing, bloqueio de tráfego não solicitado
- Security-level 100 (inside) vs 0 (outside) — tráfego de fora bloqueado por padrão
- Acesso Telnet restrito ao notebook de gerência (10.0.40.30)

### Switch (bh-sw01)

- DHCP snooping ativo em todas as VLANs
- Port-security sticky nas portas de usuário e câmeras
- Autenticação local com níveis de privilégio (spec / admin)

## Serviços do Servidor (BH-NAS)

| Serviço       | Protocolo | Função                                          |
|---------------|-----------|-------------------------------------------------|
| Armazenamento | SMB       | Compartilhamento de arquivos por setor          |
| DNS           | UDP 53    | Resolução de nomes internos (cafetorrido.local) |
| SYSLOG        | UDP 514   | Centralização de logs de todos os dispositivos  |
| TFTP          | UDP 69    | Backup de configurações dos dispositivos        |
| IoT           | HTTP      | Gerência das câmeras IP CAM-1 e CAM-2           |

### Registros DNS

| Nome                         | IP          |
|------------------------------|-------------|
| server.cafetorrido.local     | 10.0.40.5   |
| fw.cafetorrido.local         | 10.0.80.250 |
| rtr.cafetorrido.local        | 10.0.40.1   |
| sw-main.cafetorrido.local    | 10.0.40.2   |
| sw-monitor.cafetorrido.local | 10.0.40.4   |
| sw-ap.cafetorrido.local      | 10.0.40.3   |
| cam1.cafetorrido.local       | 10.0.40.6   |
| cam2.cafetorrido.local       | 10.0.40.7   |


## WiFi

| SSID          | VLAN | Segurança | Acesso                       |
|---------------|------|-----------|------------------------------|
| Torrido-ADM   | 50   | WPA2      | Dispositivos corporativos    |
| Torrido-Guest | 60   | WPA2      | Visitantes — apenas internet |
| Torrido-Open  | 70   | Aberta    | Clientes do café             |


## Cabeamento Estruturado

- **Padrão:** Cat6 UTP em toda a instalação
- **PoE:** Injetores PoE 802.3at para câmeras e Access Points
- **Patch Panels:** 2x Furukawa Cat6 24 portas
- **Rack:** Fechado 30U 600×800mm com chave
- **Percurso:** Calhas no forro acompanhando as paredes

### IPs fixos reservados (DHCP excluded)

| Range           | VLAN | Reservado para                 |
|-----------------|------|--------------------------------|
| 10.0.10.0 – .10 | 10   | Gateways e SVIs                |
| 10.0.20.0 – .10 | 20   | Gateways e SVIs                |
| 10.0.30.0 – .10 | 30   | Gateways e SVIs                |
| 10.0.40.0 – .30 | 40   | Dispositivos de infraestrutura |
| 10.0.50.0 – .10 | 50   | Gateways e SVIs                |
| 10.0.60.0 – .10 | 60   | Gateways e SVIs                |
| 10.0.70.0 – .10 | 70   | Gateways e SVIs                |


## Estrutura do Repositório

cafe-torrido-network/

├── simulation/

│   └── cafe-torrido.pkt           # Arquivo Cisco Packet Tracer

├── docs/

│   ├── guia-configuracao.pdf      # Comandos completos por dispositivo

│   ├── guia-debug.pdf             # Problemas, causas raiz e soluções

│   ├── plano-etiquetagem.pdf      # Etiquetas de dispositivos e cabos

│   ├── topologia.png              # Visão geral da topologia

│   ├── vlans.png                  # Segmentação de VLANs

│   ├── firewall-block.png         # ASA bloqueando tráfego não autorizado

│   └── nat-translations.png       # Tabela de traduções NAT

└── README.md

## Testes de Validação

### Conectividade

| Teste               | Origem      | Destino      | Resultado    |
|---------------------|-------------|--------------|--------------|
| Inter-VLAN ADM→PROD | PC-ADM      | 10.0.20.x    |      ✅      |
| Inter-VLAN ADM→NAS  | PC-ADM      | 10.0.40.5    |      ✅      |
| Inter-VLAN ADM→CAM  | PC-ADM      | 10.0.40.6    |      ✅      |
| Inter-VLAN PROD→CAM | PC-PROD     | 10.0.40.6    | ❌ Bloqueado |
| WiFi Guest→rede     | PHONE-GUEST | 10.0.10.x    | ❌ Bloqueado |
| Interno→internet    | PC-ADM      | 8.8.8.8      |      ✅      |
| Externo→IP privado  | RTR-ISP1    | 10.0.40.5    | ❌ Bloqueado |
| Externo→IP público  | RTR-ISP1    | 198.41.128.2 |      ✅      |

### Segurança

| Teste                                      | Resultado    |
|--------------------------------------------|--------------|
| Anti-spoofing (IPs privados vindo de fora) | ❌ Bloqueado |
| Telnet ao ASA de VLAN não autorizada       | ❌ Bloqueado |
| Telnet ao ASA do notebook gerência         | ✅ Permitido |
| NAT — IPs internos ocultos                 | ✅ Traduzido |
| Failover ISP-1 → ISP-2 (manual no PT)      | ✅ Funcional |

## Simulador vs Hardware Real

| Recurso                | Packet Tracer                    | Hardware Real            |
|------------------------|----------------------------------|--------------------------|
| NAT no ASA             | Feito no N-Object (limitação PT) | ASA global nativo        |
|  established  nas ACLs | Não suportado                    | Suportado - ASA stateful |
| SLA tracking failover  | Manual                           | Automático               |
| ASDM (GUI do ASA)      | Não suportado                    | Totalmente funcional     |
| Logging no ASA         | Não suportado                    | Suportado                |

## Tecnologias e Conceitos

`Cisco IOS` `ASA OS 9.6` `VLANs 802.1Q` `Router-on-a-stick`
`NAT/PAT` `ACL` `DHCP` `DNS` `SYSLOG` `TFTP` `PoE 802.3at`
`Port-security` `DHCP snooping` `Spanning-tree PVST`
`Cabeamento estruturado Cat6` `TrueNAS SCALE` `Cisco Packet Tracer`

## Autor

**Gabriel Anacleto**
[LinkedIn](https://linkedin.com/in/gabriel-anacleto-network/) · [Email](Gabrielanacletocontpage@gmail.com)

*Projeto desenvolvido para fins educacionais e profissionais - simulação no Cisco Packet Tracer.*
