# 🛡️ Home SOC Lab — SIEM & EDR com Wazuh

Lab prático de Blue Team simulando um ambiente real de monitoramento de segurança, com SIEM e EDR usando Wazuh. Projeto desenvolvido como parte da minha transição de carreira para Cybersecurity, com foco em SOC Analyst.

---

## 🎯 Objetivo

Montar um ambiente funcional de detecção e resposta a ameaças, cobrindo:
- Centralização e análise de logs em tempo real (SIEM)
- Monitoramento de integridade de arquivos (FIM)
- Detecção de comportamento suspeito em endpoints Windows (EDR)
- Simulação de ataques reais e verificação de detecção via MITRE ATT&CK

---

## 🖥️ Ambiente do Lab

| Componente | Sistema | Função |
|---|---|---|
| Wazuh Manager | Ubuntu Server (VMware) | Servidor central — coleta, correlaciona e exibe alertas |
| Wazuh Agent | Windows 11 Pro (host) | Endpoint monitorado — envia logs e eventos ao Manager |
| Wazuh Dashboard | Navegador via `https://192.168.100.140` | Interface gráfica de análise e visualização |

**Configuração de rede:**
- Adaptador configurado em modo **NAT** no VMware
- IP do Manager: `192.168.100.140` (atribuído via DHCP pelo VMnet8)
- IP do Agent Windows: `192.168.100.1`

**Instalação:**
- Manager instalado via OVA oficial do Wazuh (`wazuh-4.14.5.ova`)
- Agent instalado via PowerShell no Windows com comando gerado pelo dashboard

---

## 🔧 O que foi configurado

### 1. Importação e configuração da VM (Ubuntu)
- Importação do arquivo `.ova` no VMware
- Configuração do adaptador de rede para NAT (VMnet8) para obter IP via DHCP
- Acesso ao dashboard via navegador: `https://192.168.100.140`
- Credenciais padrão: `admin / admin`

![Dashboard Wazuh](prints/01-dashboard-wazuh.jpeg)

### 2. File Integrity Monitoring (FIM)
- FIM habilitado por padrão no `ossec.conf` (`<disabled>no</disabled>`)
- Frequência de varredura ajustada de 12 horas para **5 minutos** para fins de lab
- Diretórios monitorados: `/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, `/boot`

### 3. Instalação do Wazuh Agent (Windows 11)
- Deploy gerado diretamente pelo dashboard em **Endpoints → Deploy new agent**
- Sistema selecionado: Windows MSI 32/64 bits
- Comando executado via **PowerShell como Administrador:**

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.100.140' WAZUH_AGENT_NAME='hackeratk'

NET START Wazuh
```

- Agent registrado com nome: `hackeratk` — Status: **Active**

![Agent Windows Ativo](prints/03-agent-windows-ativo.jpeg)

---

## 🔴 Simulações de Ameaça e Detecção

### Cenário 1 — File Integrity Monitoring (FIM)

**Objetivo:** verificar se o Wazuh detecta criação e modificação de arquivos em diretórios críticos do sistema.

**Ações simuladas:**
```bash
sudo touch /etc/arquivo-teste-fim.txt
sudo tee -a /etc/arquivo-teste-fim.txt <<< "alterado"
sudo touch /etc/teste-intrusao.txt
```

**Alertas gerados no Wazuh:**
| Campo | Valor |
|---|---|
| Regra | File added to the system |
| Ação | added |
| Agente | wazuh-server (000) |
| Usuário | root |
| Severidade | 5 |

![Alerta FIM](prints/02-fim-alerta-arquivo.jpeg)

**Análise SOC:** criação de arquivo em `/etc` por root fora de janela de manutenção programada é sinal de alerta. Em ambiente real, o analista verificaria se a alteração foi autorizada e rastrearia o processo responsável.

---

### Cenário 2 — Brute Force SSH

**Objetivo:** simular um ataque de força bruta SSH e verificar se o Wazuh detecta e classifica o padrão de ataque automaticamente.

**Técnica MITRE ATT&CK:** T1110 — Brute Force / Password Guessing

**Ataque simulado** (movimento lateral interno — Ubuntu atacando ele mesmo):
```bash
for i in {1..15}; do ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 wronguser@127.0.0.1; done
```

**Log do sistema confirmando o ataque:**

![Log Brute Force](prints/04-brute-force-log.jpeg)

**Alertas gerados no Wazuh:**
| Rule ID | Descrição | Severidade |
|---|---|---|
| 5712 | sshd: brute force trying to get access to the system. Non existent user | **10** |
| 2502 | syslog: User missed the password more than one time | **10** |
| 5710 | sshd: Attempt to login using a non-existent user | 5 |
| 5503 | PAM: User login failed | 5 |

**Total de eventos gerados:** 78 hits

![Eventos Brute Force Detectados](prints/08-brute-force-eventos-detectados.jpeg)

**Mapeamento MITRE ATT&CK automático pelo Wazuh:**
- Password Guessing
- SSH
- Brute Force
- Valid Accounts
- Remote Services

![Dashboard MITRE Brute Force](prints/06-dashboard-mitre-brute-force.jpeg)

![MITRE ATT&CK](prints/07-mitre-attack-deteccao.jpeg)

**Análise SOC:** regra 5712 com severidade 10 indica padrão de brute force confirmado — não apenas falhas isoladas. Em ambiente real, o analista bloquearia o IP de origem, verificaria outros sistemas afetados e abriria um ticket de incidente.

---

### Bônus — Monitoramento Passivo do Endpoint Windows

Além das simulações ativas, o Wazuh coletou automaticamente **506 eventos** do agent Windows, incluindo:
- Windows Logon Success/Failure
- Service startup changes
- Configuration Assessment contra benchmark **CIS Microsoft Windows 11**
- Policy changes

![Eventos Windows Threat Hunting](prints/05-eventos-windows-threat-hunting.jpeg)

---

## 📸 Evidências

| Arquivo | Descrição |
|---|---|
| `prints/01-dashboard-wazuh.jpeg` | Dashboard do Wazuh com alertas ativos |
| `prints/02-fim-alerta-arquivo.jpeg` | Alerta de FIM — arquivo criado em `/etc` |
| `prints/03-agent-windows-ativo.jpeg` | Agent `hackeratk` (Windows 11) conectado e ativo |
| `prints/04-brute-force-log.jpeg` | Log do sistema confirmando tentativas de brute force |
| `prints/05-eventos-windows-threat-hunting.jpeg` | 506 eventos do Windows monitorados em tempo real |
| `prints/06-dashboard-mitre-brute-force.jpeg` | Dashboard com 55 authentication failures e MITRE ATT&CK |
| `prints/07-mitre-attack-deteccao.jpeg` | Mapeamento MITRE ATT&CK automático |
| `prints/08-brute-force-eventos-detectados.jpeg` | Eventos de brute force detectados com rule IDs |

---

## 📚 O que aprendi

- Como funciona a arquitetura Manager/Agent de um SIEM/EDR
- Como configurar File Integrity Monitoring para detectar mudanças em arquivos críticos
- Como fazer o deploy de um agente Windows e integrá-lo ao servidor central
- Como um ataque de brute force SSH é detectado e classificado automaticamente
- Como o Wazuh mapeia ataques para o framework MITRE ATT&CK
- Como um SOC Analyst investiga alertas de severidade alta no dashboard
- Configuração de rede em ambiente virtualizado (VMware NAT)

---

## 🔗 Referências

- [Documentação oficial do Wazuh](https://documentation.wazuh.com)
- [MITRE ATT&CK — T1110 Brute Force](https://attack.mitre.org/techniques/T1110/)

---

## 🚀 Próximos passos

- [ ] Configurar FIM no Agent Windows para monitorar arquivos críticos do sistema
- [ ] Integrar Sysmon para monitoramento avançado de processos no Windows
- [ ] Adicionar IDS com Snort/Suricata para monitoramento de rede
- [ ] Simular ataques com Kali Linux e verificar detecção no Wazuh
- [ ] Criar regras customizadas de alerta no Wazuh

---

*Projeto parte do meu Home SOC Lab — construindo base prática para atuar como SOC Analyst / Blue Team.*  
*Transição de carreira: Engenharia de Computação → Cybersecurity | Estudando para CompTIA Security+*
