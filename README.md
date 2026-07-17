# Homelab: Proxmox VE + Segurança + Monitoramento + Acesso Remoto

Documentação passo a passo da construção de um homelab de virtualização, com foco em um laboratório de segurança ofensiva/defensiva isolado, stack de monitoramento e SOAR, e acesso remoto seguro contornando CGNAT.

Este repositório serve como registro técnico do projeto e como base para uma série de posts no LinkedIn, publicados etapa por etapa.

---

## 📋 Visão geral do projeto

**Objetivo:** transformar um servidor físico em uma plataforma de virtualização completa, hospedando:

- Um laboratório isolado de segurança ofensiva/defensiva (Kali Linux ↔ Metasploitable2)
- Um SIEM (Wazuh) para detecção de ameaças
- Um stack de monitoramento de infraestrutura (Zabbix)
- Um SOAR (Shuffle) para automação de resposta a incidentes
- Acesso remoto seguro à infraestrutura, sem expor serviços diretamente à internet

**Hardware utilizado:**
- CPU: Intel Xeon E3-1200 v2 (Ivy Bridge), com suporte a VT-d
- RAM: 22GB nominal (21GiB reconhecidos pelo sistema)
- GPU: AMD Radeon R7 370 (Curaçao PRO) — configurada via passthrough, atualmente sem uso ativo no projeto
- Roteador de borda: MikroTik RB750GL (RouterOS 7.23.2, arquitetura MIPSBE)

---

## Etapa 1 — Instalação do Proxmox VE

- Instalação via ISO oficial, sistema de arquivos ext4
- Pós-instalação: remoção do repositório `pve-enterprise` e ativação do repositório `pve-no-subscription`, para permitir atualizações sem assinatura paga:

```bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt full-upgrade -y
```

---

## Etapa 2 — Identificação de hardware (GPU passthrough)

Antes de decidir a arquitetura das VMs, identifiquei a GPU dedicada disponível no servidor:

```bash
lspci -k | grep -A 3 -i vga
```

Resultado: **AMD Radeon R7 370 / R9 270/370 OEM** (Gigabyte), endereços PCI `01:00.0` (vídeo) e `01:00.1` (áudio HDMI).

### Confirmação de IOMMU

```bash
lscpu | grep -i vendor        # GenuineIntel
cat /etc/default/grub | grep GRUB_CMDLINE_LINUX_DEFAULT
# GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
dmesg | grep -e DMAR -e IOMMU  # IOMMU enabled
```

### Verificação de isolamento do grupo IOMMU

```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```

A GPU ficou isolada em um grupo IOMMU próprio (Group 1), junto apenas do root port PCIe — cenário ideal, sem necessidade de patch de ACS override.

### Binding via VFIO

```bash
# /etc/modprobe.d/vfio.conf
options vfio-pci ids=1002:6811,1002:aab0

# /etc/modprobe.d/blacklist.conf
blacklist amdgpu
blacklist radeon

# /etc/modules
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

```bash
update-initramfs -u -k all
reboot
```

Confirmação pós-reboot:
```bash
lspci -k -s 01:00.0   # Kernel driver in use: vfio-pci
lspci -k -s 01:00.1   # Kernel driver in use: vfio-pci
```

> **Nota:** o plano inicial incluía um servidor de streaming de jogos (Windows + Sunshine/Moonlight) usando essa GPU. O escopo do projeto foi posteriormente ajustado para focar no laboratório de segurança e monitoramento, então a GPU permanece configurada para passthrough mas sem uso ativo no momento.

---

## Etapa 3 — Arquitetura de rede

O roteamento de borda já era feito por um MikroTik RB750GL existente, então não foi necessário criar um firewall virtualizado no Proxmox.

### Bridge isolada para o laboratório de segurança

Para garantir que o tráfego de ataque/defesa não vazasse para a rede doméstica ou internet, foi criada uma bridge dedicada, sem IP e sem interface física associada:

- **Datacenter → Node → System → Network → Create → Linux Bridge**
- Nome: `vmbr1`
- Sem IPv4/CIDR, sem gateway, sem bridge ports

Essa rede fica completamente isolada — VMs conectadas nela se comunicam entre si, mas não têm rota para a LAN nem para a internet.

---

## Etapa 4 — Laboratório de segurança (ataque × defesa)

### VM de ataque: Kali Linux
- Rede: `vmbr1` (isolada)
- Disco: 40GB | CPU: 4 núcleos | RAM: 4096 MB *(confirmado via `qm config`)*

### VM alvo: Metasploitable2
Diferente do Kali, essa VM não usa ISO — é uma imagem de disco pronta (`.vmdk`), então o processo de criação foi diferente:

1. Download do pacote oficial (SourceForge) e extração do `.vmdk`
2. Upload via WinSCP para `/var/lib/vz/template/iso/`
3. Criação de uma VM vazia (sem mídia, disco padrão removido)
4. Importação do disco:
```bash
qm importdisk <VMID> /var/lib/vz/template/iso/Metasploitable.vmdk local-lvm
```
5. Anexação do disco importado em **Hardware → Unused Disk → Edit → Add**
6. Ajuste da ordem de boot para esse disco

- Rede: `vmbr1` (isolada, mesma rede do Kali)
- CPU: 1 núcleo | RAM: 1024 MB *(confirmado via `qm config`)*

---

## Etapa 5 — Stack de monitoramento e resposta a incidentes

Após avaliar opções (Wazuh, Security Onion, OSSIM, Graylog, Elastic Security), a arquitetura escolhida foi:

| Ferramenta | Função | Motivo da escolha |
|---|---|---|
| **Wazuh** | SIEM | Open-source completo (GPLv2), sem restrições de licença, mais leve que Security Onion (que recomenda 24-32GB só para si) |
| **Zabbix** | Monitoramento de infraestrutura | Serviço único e consolidado, menor pegada de RAM comparado a Grafana+Prometheus (que exigem exporters por VM) |
| **Shuffle** | SOAR | Open-source, self-hosted, sem necessidade de licenciamento (ao contrário do TheHive, que passou a exigir cadastro em licença Community a partir da v5) |

### VMs criadas

- `wazuh-siem`: Ubuntu Server, 4 vCPU, 6114 MB RAM, 80GB disco *(confirmado via `qm config`)*
- `monitor-soar`: Ubuntu Server, 2 vCPU, 4096 MB RAM, 40GB disco (Zabbix + Shuffle + Tailscale) *(confirmado via `qm config`)*

> A VM do Wazuh recebeu uma segunda interface de rede (`vmbr1`), permitindo que ela colete logs de agentes instalados no laboratório isolado, sem quebrar o isolamento das demais VMs.

> A VM `monitor-soar` foi posteriormente escolhida também para hospedar o cliente Tailscale (ver Etapa 7), por já permanecer sempre ligada.

---

## Etapa 6 — Hardening do firewall MikroTik

Levantamento inicial revelou que o roteador estava rodando **sem nenhuma regra de firewall**, exceto uma criada durante testes de VPN. Aplicado um conjunto básico de proteção:

### Chain INPUT
```
/ip firewall filter add chain=input connection-state=established,related action=accept
/ip firewall filter add chain=input connection-state=invalid action=drop
/ip firewall filter add chain=input src-address=192.168.0.0/24 action=accept
/ip firewall filter add chain=input in-interface=ether1 action=drop
```

### Chain FORWARD
```
/ip firewall filter add chain=forward connection-state=established,related action=accept
/ip firewall filter add chain=forward connection-state=invalid action=drop
/ip firewall filter add chain=forward in-interface=bridge1 out-interface=ether1 action=accept
```

Cada bloco foi testado antes de aplicar a regra de *drop* final, para evitar bloqueio acidental do próprio acesso administrativo.

---

## Etapa 7 — Acesso remoto seguro (contornando CGNAT)

### Diagnóstico

Tentativa inicial: WireGuard nativo no MikroTik + DDNS. Ao comparar o IP da interface WAN (`100.70.5.67`, faixa `100.64.0.0/10`) com o IP público real (`187.180.164.182`, verificado via whatismyip.com), foi confirmado que a rede está atrás de **CGNAT** — inviabilizando port-forward tradicional.

### Tentativa de Tailscale direto no MikroTik

Verificação de hardware:
```
/system resource print
# architecture-name: mipsbe
# total-memory: 64.0MiB
```

Arquitetura MIPSBE e apenas 64MB de RAM — hardware incompatível com o pacote oficial de Tailscale (que requer RouterOS 7.11+ em ARM/ARM64/x86) e com a função de containers do RouterOS.

### Solução adotada: Tailscale como subnet router em VM Linux

Instalado na VM `monitor-soar` (já sempre ligada):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=192.168.0.0/24 --accept-routes
```

Habilitação de IP forwarding:
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo systemctl restart tailscaled
```

Rota `192.168.0.0/24` aprovada manualmente no painel administrativo do Tailscale (`Machines → Edit route settings`).

**Resultado:** acesso completo e criptografado a toda a rede local (incluindo interface web do Proxmox) a partir de qualquer rede externa, sem necessidade de porta aberta, IP público ou DDNS — contornando completamente a limitação do CGNAT.

Regras de firewall específicas do WireGuard (abandonado nessa etapa) foram removidas do MikroTik, mantendo apenas o hardening geral (Etapa 6).

---

## Etapa 8 — Medição real de recursos (RAM e armazenamento)

Após o provisionamento das 4 VMs principais, foi feito um levantamento real de consumo no host, em vez de assumir os valores configurados no momento da criação.

### RAM alocada por VM (confirmado via `qm config`)

```bash
for vm in $(qm list | awk 'NR>1{print $1}'); do
  echo "--- VM $vm ---"
  qm config $vm | grep -E "^(memory|cores|name)"
done
```

| VM | vCPU | RAM alocada |
|---|---|---|
| `wazuh-siem` | 4 | 6114 MB |
| `monitor-soar` | 2 | 4096 MB |
| `kali-attack` | 4 | 4096 MB |
| `target-defense` | 1 | 1024 MB |
| **Total alocado** | **11** | **15330 MB (~15GB)** |

### RAM real em uso no host (`free -h`)

```
              total    used    free   shared  buff/cache  available
Mem:           21Gi    14Gi    877Mi    54Mi        6.9Gi       7.3Gi
Swap:         8.0Gi    8.0Ki   8.0Gi
```

De 21GB físicos, 14GB estão em uso (host + VMs ligadas), restando 7,3GB efetivamente disponíveis (considerando cache reclamável).

### Uso real de armazenamento (`pvesm status`)

```
Name          Type     Status    Total (KiB)   Used (KiB)   Available (KiB)   %
local          dir     active      98497780     17923440         75524792   18.20%
local-lvm  lvmthin     active     832888832     40228530        792660301    4.83%
```

| Storage | Total | Usado | % |
|---|---|---|---|
| `local-lvm` (discos das VMs) | 794,3 GB | 38,4 GB | 4,83% |
| `local` (ISOs/templates) | 93,9 GB | 17,1 GB | 18,20% |

> **Achado interessante:** apesar de ~168GB alocados nominalmente em discos virtuais (somando os discos configurados de cada VM), o uso real em disco físico é de apenas **38,4GB** — efeito do thin provisioning do LVM, que só ocupa espaço conforme os dados são de fato escritos, não conforme o tamanho "declarado" do disco virtual.

### Arquivos de instalação (ISOs e imagens)

```bash
ls -la /var/lib/vz/template/iso/
```

| Arquivo | Tamanho |
|---|---|
| `kali-linux-2026.2-installer-amd64.iso` | 4,47 GB |
| `ubuntu-26.04-live-server-amd64.iso` | 2,72 GB |
| `Metasploitable.vmdk` | 1,79 GB |
| `metasploitable-linux-2.0.0.zip` | 0,81 GB |
| **Total** | **9,79 GB** |

---

## ✅ Status atual

- [x] Proxmox VE instalado e atualizado
- [x] GPU configurada via passthrough (VFIO)
- [x] Rede isolada para laboratório de segurança
- [x] VM Kali Linux (ataque) — rede configurada
- [x] VM Metasploitable2 (alvo) — disco importado
- [x] VM Wazuh (SIEM) — sistema operacional instalado
- [x] VM Zabbix + Shuffle — sistema operacional instalado
- [x] Firewall MikroTik com hardening básico
- [x] Acesso remoto via Tailscale funcionando
- [x] Levantamento real de RAM e armazenamento
- [ ] Instalação do Wazuh (script assistido `wazuh-install.sh -a`) — **em andamento**

## 🔜 Próximas etapas

- [ ] Concluir instalação e configuração do Wazuh (Manager + Indexer + Dashboard)
- [ ] Instalação do Zabbix Server + Frontend
- [ ] Deploy do Shuffle via Docker Compose
- [ ] Finalização da instalação do Kali Linux
- [ ] Testes de exploração no Metasploitable2 com detecção via Wazuh
- [ ] Integração Wazuh → Shuffle para automação de resposta

---

## 🛠️ Stack utilizada

`Proxmox VE` `Kali Linux` `Metasploitable2` `Wazuh` `Zabbix` `Shuffle` `MikroTik RouterOS` `Tailscale` `VFIO/IOMMU`

<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Homelab — Status Geral</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@600;700&family=JetBrains+Mono:wght@400;500;600;700&display=swap');

  :root{
    --bg: #F4F6F9;
    --panel: #FFFFFF;
    --panel-border: #DDE4EC;
    --line: #1C2733;
    --muted: #6B7A8C;
    --green: #0FA968;
    --orange: #E0722A;
    --blue: #1279C6;
    --purple: #7B4FD1;
    --red: #D6453D;
    --bar-bg: #EAEFF4;
  }

  *{ box-sizing:border-box; margin:0; padding:0; }

  body{
    background:
      radial-gradient(circle at 10% 0%, rgba(18,121,198,0.05), transparent 45%),
      radial-gradient(circle at 90% 100%, rgba(15,169,104,0.05), transparent 45%),
      var(--bg);
    font-family: 'JetBrains Mono', monospace;
    color: var(--line);
    padding: 28px;
  }

  .board{
    max-width: 1360px;
    margin: 0 auto;
    display: grid;
    grid-template-columns: 300px 1fr 300px;
    gap: 16px;
  }

  .panel{
    background: var(--panel);
    border: 1px solid var(--panel-border);
    border-radius: 6px;
    padding: 18px 20px;
    box-shadow: 0 1px 3px rgba(28,39,51,0.06);
  }

  .panel-title{
    font-size: 11px;
    letter-spacing: 1.5px;
    color: var(--blue);
    text-transform: uppercase;
    margin-bottom: 14px;
    font-weight: 600;
  }

  .header{
    grid-column: 1 / -1;
    display:flex;
    justify-content: space-between;
    align-items:flex-start;
    padding-bottom: 20px;
    border-bottom: 1px solid var(--panel-border);
    margin-bottom: 4px;
  }
  .header h1{
    font-family:'Space Grotesk', sans-serif;
    font-size: 30px;
    letter-spacing: 0.5px;
  }
  .header .status-line{
    font-family:'Space Grotesk', sans-serif;
    font-size: 20px;
    color: var(--green);
    margin-top: 2px;
  }
  .header p{
    max-width: 600px;
    color: var(--muted);
    font-size: 12.5px;
    margin-top: 10px;
    line-height: 1.6;
  }
  .header .tag{
    font-family:'Space Grotesk',sans-serif;
    font-size: 11px;
    color: #fff;
    background: var(--orange);
    padding: 5px 12px;
    border-radius: 3px;
    font-weight: 700;
    letter-spacing: 1px;
    height: fit-content;
  }

  /* LEFT: fases */
  .task{
    display:flex;
    gap: 10px;
    margin-bottom: 15px;
    align-items:flex-start;
  }
  .task .ico{
    width: 24px; height:24px;
    border-radius: 5px;
    background: rgba(18,121,198,0.1);
    border: 1px solid rgba(18,121,198,0.3);
    flex-shrink:0;
    display:flex; align-items:center; justify-content:center;
    font-size: 12px;
    color: var(--blue);
  }
  .task strong{
    font-family:'Space Grotesk', sans-serif;
    font-size: 12px;
    display:block;
    margin-bottom: 3px;
  }
  .task span{
    font-size: 10.3px;
    color: var(--muted);
    line-height: 1.5;
  }

  /* CENTER: topologia */
  .arch{
    display:flex;
    flex-direction:column;
    align-items:center;
    padding: 20px 6px;
  }
  .arch-title{
    font-family:'Space Grotesk', sans-serif;
    font-size: 12px;
    color: var(--orange);
    letter-spacing: 1.5px;
    text-transform: uppercase;
    margin-bottom: 22px;
  }
  .box{
    border: 1.5px solid var(--line);
    border-radius: 6px;
    padding: 9px 18px;
    font-family:'Space Grotesk', sans-serif;
    font-size: 12.5px;
    font-weight: 700;
    background: rgba(0,0,0,0.015);
    text-align:center;
  }
  .box.blue{ border-color: var(--blue); color:var(--blue); }
  .box.green{ border-color: var(--green); color:var(--green); }
  .box.orange{ border-color: var(--orange); color: var(--orange); }
  .box.red{ border-color: var(--red); color: var(--red); }
  .box.purple{ border-color: var(--purple); color: var(--purple); }
  .caption{
    font-size: 9.5px;
    color: var(--muted);
    margin-top: 5px;
    text-align:center;
  }
  .arrow{ color: var(--muted); font-size: 15px; margin: 7px 0; }

  .row{
    display:flex;
    gap: 30px;
    margin: 2px 0;
    flex-wrap: wrap;
    justify-content:center;
  }
  .node{ display:flex; flex-direction:column; align-items:center; }

  .isolation-frame{
    border: 1.5px dashed var(--red);
    border-radius: 8px;
    padding: 14px 24px;
    margin-top: 6px;
  }
  .isolation-label{
    font-size: 9px;
    color: var(--red);
    text-align:center;
    letter-spacing: 1px;
    margin-bottom: 10px;
    text-transform:uppercase;
  }

  /* budget table */
  .budget{
    margin-top: 26px;
    width: 100%;
  }
  .budget-row{
    margin-bottom: 12px;
  }
  .budget-head{
    display:flex;
    justify-content: space-between;
    font-size: 10.5px;
    margin-bottom: 4px;
  }
  .budget-head .name{ font-weight:600; }
  .budget-head .val{ color: var(--muted); }
  .bar-bg{
    height: 8px;
    background: var(--bar-bg);
    border-radius: 4px;
    overflow:hidden;
  }
  .bar-fill{
    height: 100%;
    border-radius: 4px;
  }

  /* RIGHT: status */
  .status-row{
    display:flex;
    justify-content: space-between;
    align-items:center;
    font-size: 11px;
    padding: 6px 0;
    border-bottom: 1px dashed rgba(107,122,140,0.25);
  }
  .status-row:last-child{ border-bottom:none; }
  .pill{
    font-size: 9px;
    font-weight: 700;
    padding: 3px 8px;
    border-radius: 20px;
    letter-spacing: 0.4px;
    white-space:nowrap;
  }
  .pill.ok{ background: rgba(15,169,104,0.13); color: var(--green); }
  .pill.warn{ background: rgba(224,114,42,0.13); color: var(--orange); }
  .pill.pend{ background: rgba(123,79,209,0.13); color: var(--purple); }

  /* BOTTOM */
  .bottom{
    grid-column: 1 / -1;
    display:grid;
    grid-template-columns: 1.3fr 1fr;
    gap: 16px;
    margin-top: 4px;
  }

  table.resources{
    width: 100%;
    border-collapse: collapse;
    font-size: 11px;
  }
  table.resources th{
    text-align:left;
    color: var(--muted);
    font-weight: 500;
    font-size: 9.5px;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    padding-bottom: 8px;
    border-bottom: 1px solid var(--panel-border);
  }
  table.resources td{
    padding: 8px 0;
    border-bottom: 1px dashed rgba(107,122,140,0.2);
  }
  table.resources tr:last-child td{ border-bottom:none; font-weight:700; }
  table.resources .dot{
    display:inline-block;
    width:7px;height:7px;
    border-radius:50%;
    margin-right:6px;
  }

  .totals-row{
    margin-top: 14px;
    padding-top: 12px;
    border-top: 1px solid var(--panel-border);
    font-size: 11px;
    display:flex;
    justify-content: space-between;
  }
  .totals-row b{ color: var(--line); }

  .note{
    margin-top: 12px;
    font-size: 9.5px;
    color: var(--muted);
    line-height: 1.5;
    font-style: italic;
  }

  .next-list{ display:flex; flex-direction:column; gap: 9px; }
  .next-item{ display:flex; gap: 10px; align-items:center; font-size: 11px; }
  .next-item .n{
    width: 18px; height:18px;
    border-radius:50%;
    border: 1px solid var(--muted);
    display:flex; align-items:center; justify-content:center;
    font-size: 9.5px;
    color: var(--muted);
    flex-shrink:0;
  }

  .quote{
    grid-column: 1 / -1;
    text-align:center;
    font-size: 11.5px;
    color: var(--muted);
    padding-top: 16px;
    margin-top: 4px;
    border-top: 1px solid var(--panel-border);
    font-style: italic;
  }
  .quote b{ color: var(--line); font-style: normal; }
</style>
</head>
<body>

<div class="board">

  <div class="header">
    <div>
      <h1>HOMELAB PESSOAL</h1>
      <div class="status-line">7 ETAPAS EM ANDAMENTO · 6 CONCLUÍDAS</div>
      <p>Servidor de virtualização Proxmox VE com laboratório isolado de segurança ofensiva/defensiva, stack de SIEM/monitoramento/SOAR, e acesso remoto seguro via VPN mesh — contornando limitação de CGNAT do provedor.</p>
    </div>
    <div class="tag">STATUS GERAL</div>
  </div>

  <!-- LEFT: fases -->
  <div class="panel">
    <div class="panel-title">Fases implementadas</div>

    <div class="task">
      <div class="ico">1</div>
      <div><strong>Proxmox VE + GPU passthrough</strong><span>Instalação, repo no-subscription, IOMMU/VFIO configurados.</span></div>
    </div>
    <div class="task">
      <div class="ico">2</div>
      <div><strong>Rede isolada (vmbr1)</strong><span>Bridge sem gateway, dedicada ao laboratório de segurança.</span></div>
    </div>
    <div class="task">
      <div class="ico">3</div>
      <div><strong>Lab ataque × defesa</strong><span>Kali Linux + Metasploitable2 provisionados na rede isolada.</span></div>
    </div>
    <div class="task">
      <div class="ico">4</div>
      <div><strong>Stack SIEM/SOAR</strong><span>Wazuh, Zabbix e Shuffle — VMs provisionadas, instalação em curso.</span></div>
    </div>
    <div class="task">
      <div class="ico">5</div>
      <div><strong>Hardening MikroTik</strong><span>Firewall input/forward configurado com established/related + drop.</span></div>
    </div>
    <div class="task">
      <div class="ico">6</div>
      <div><strong>Acesso remoto (Tailscale)</strong><span>Subnet router contornando CGNAT, sem porta exposta.</span></div>
    </div>
    <div class="task">
      <div class="ico">7</div>
      <div><strong>Documentação técnica</strong><span>Registro completo em README + diagramas de arquitetura.</span></div>
    </div>
  </div>

  <!-- CENTER: topologia -->
  <div class="panel arch">
    <div class="arch-title">Topologia geral do ambiente</div>

    <div class="node"><div class="box">INTERNET (CGNAT)</div></div>
    <div class="arrow">↓</div>
    <div class="node">
      <div class="box red">MIKROTIK RB750</div>
      <div class="caption">Firewall hardening · WAN/LAN</div>
    </div>
    <div class="arrow">↓</div>
    <div class="node">
      <div class="box blue">PROXMOX VE</div>
      <div class="caption">vmbr0 · 192.168.0.0/24</div>
    </div>
    <div class="arrow">↓</div>

    <div class="row">
      <div class="node">
        <div class="box green">WAZUH-SIEM</div>
        <div class="caption">SIEM · 2ª NIC p/ vmbr1</div>
      </div>
      <div class="node">
        <div class="box purple">MONITOR-SOAR</div>
        <div class="caption">Zabbix · Shuffle · Tailscale</div>
      </div>
    </div>

    <div class="isolation-frame">
      <div class="isolation-label">⛊ rede isolada · vmbr1 · sem rota externa</div>
      <div class="row">
        <div class="node">
          <div class="box orange">KALI-ATTACK</div>
          <div class="caption">Ataque</div>
        </div>
        <div class="node">
          <div class="box orange">TARGET-DEFENSE</div>
          <div class="caption">Metasploitable2</div>
        </div>
      </div>
    </div>

    <!-- Orçamento de recursos -->
    <div class="budget">
      <div class="panel-title" style="margin-bottom:12px;">RAM alocada por VM (confirmado via qm config · 21GB físicos)</div>

      <div class="budget-row">
        <div class="budget-head"><span class="name">wazuh-siem</span><span class="val">6114 MB</span></div>
        <div class="bar-bg"><div class="bar-fill" style="width:29%; background:var(--green);"></div></div>
      </div>
      <div class="budget-row">
        <div class="budget-head"><span class="name">monitor-soar (Zabbix+Shuffle+Tailscale)</span><span class="val">4096 MB</span></div>
        <div class="bar-bg"><div class="bar-fill" style="width:19%; background:var(--purple);"></div></div>
      </div>
      <div class="budget-row">
        <div class="budget-head"><span class="name">kali-attack</span><span class="val">4096 MB</span></div>
        <div class="bar-bg"><div class="bar-fill" style="width:19%; background:var(--orange);"></div></div>
      </div>
      <div class="budget-row">
        <div class="budget-head"><span class="name">target-defense (Metasploitable2)</span><span class="val">1024 MB</span></div>
        <div class="bar-bg"><div class="bar-fill" style="width:5%; background:var(--red);"></div></div>
      </div>
      <div class="budget-row">
        <div class="budget-head"><span class="name" style="color:var(--muted);">Host + margem não alocada</span><span class="val">~6 GB</span></div>
        <div class="bar-bg"><div class="bar-fill" style="width:28%; background:var(--bar-bg); border:1px dashed var(--muted);"></div></div>
      </div>

      <div class="note" style="margin-top:10px;">📊 Medição real (<code>free -h</code>): 14GB em uso · 6,9GB em buff/cache · 877MB livre imediato · 7,3GB disponível (cache reclamável) de 21GB físicos totais.</div>
    </div>
  </div>

  <!-- RIGHT: status -->
  <div class="panel">
    <div class="panel-title">Status dos serviços</div>
    <div class="status-row"><span>Proxmox VE</span><span class="pill ok">ATIVO</span></div>
    <div class="status-row"><span>GPU passthrough (VFIO)</span><span class="pill ok">CONFIGURADO</span></div>
    <div class="status-row"><span>Bridge isolada (vmbr1)</span><span class="pill ok">ATIVA</span></div>
    <div class="status-row"><span>Kali Linux</span><span class="pill warn">INSTALAÇÃO</span></div>
    <div class="status-row"><span>Metasploitable2</span><span class="pill ok">DISCO IMPORTADO</span></div>
    <div class="status-row"><span>Wazuh (SIEM)</span><span class="pill warn">EM INSTALAÇÃO</span></div>
    <div class="status-row"><span>Zabbix</span><span class="pill pend">PENDENTE</span></div>
    <div class="status-row"><span>Shuffle (SOAR)</span><span class="pill pend">PENDENTE</span></div>
    <div class="status-row"><span>Firewall MikroTik</span><span class="pill ok">HARDENED</span></div>
    <div class="status-row"><span>Tailscale (acesso remoto)</span><span class="pill ok">FUNCIONANDO</span></div>

    <div class="panel-title" style="margin-top:20px;">Rede</div>
    <div class="status-row"><span>LAN (bridge1)</span><span class="pill ok">192.168.0.0/24</span></div>
    <div class="status-row"><span>Lab isolado (vmbr1)</span><span class="pill ok">sem gateway</span></div>
    <div class="status-row"><span>Tailscale subnet</span><span class="pill ok">100.112.162.26</span></div>
    <div class="status-row"><span>WAN (CGNAT)</span><span class="pill warn">100.70.5.67</span></div>
  </div>

  <!-- BOTTOM -->
  <div class="bottom">
    <div class="panel">
      <div class="panel-title">Alocação real por VM (qm config)</div>
      <table class="resources">
        <tr><th>VM</th><th>vCPU</th><th>RAM</th><th>Disco virtual</th><th>Rede</th></tr>
        <tr>
          <td><span class="dot" style="background:var(--green);"></span>wazuh-siem</td>
          <td>4</td><td>6114 MB</td><td>80 GB</td><td>vmbr0 + vmbr1</td>
        </tr>
        <tr>
          <td><span class="dot" style="background:var(--purple);"></span>monitor-soar</td>
          <td>2</td><td>4096 MB</td><td>40 GB</td><td>vmbr0</td>
        </tr>
        <tr>
          <td><span class="dot" style="background:var(--orange);"></span>kali-attack</td>
          <td>4</td><td>4096 MB</td><td>40 GB</td><td>vmbr1 (isolada)</td>
        </tr>
        <tr>
          <td><span class="dot" style="background:var(--red);"></span>target-defense</td>
          <td>1</td><td>1024 MB</td><td>~8 GB*</td><td>vmbr1 (isolada)</td>
        </tr>
        <tr>
          <td>TOTAL ALOCADO</td>
          <td>11</td><td><b>15330 MB</b></td><td><b>~168 GB</b></td><td>—</td>
        </tr>
      </table>
      <div class="totals-row"><span>RAM física total (host)</span><b>21 GiB</b></div>
      <div class="totals-row"><span>RAM em uso agora (live)</span><b>14 GiB</b></div>
      <div class="totals-row"><span>RAM disponível (live)</span><b>7,3 GiB</b></div>
      <div class="note">Dados confirmados via <code>qm config &lt;id&gt;</code> e <code>free -h</code> em 17/07/2026. *Tamanho do disco virtual do Metasploitable2 conforme imagem oficial; demais discos conforme configurados na criação da VM (thin-provisioned — uso real de disco é bem menor, ver tabela ao lado).</div>
    </div>

    <div class="panel">
      <div class="panel-title">Uso real de armazenamento (pvesm status)</div>
      <table class="resources">
        <tr><th>Storage</th><th>Total</th><th>Usado</th><th>%</th></tr>
        <tr><td><span class="dot" style="background:var(--blue);"></span>local-lvm (discos das VMs)</td><td>794,3 GB</td><td>38,4 GB</td><td>4,83%</td></tr>
        <tr><td><span class="dot" style="background:var(--orange);"></span>local (ISOs/templates)</td><td>93,9 GB</td><td>17,1 GB</td><td>18,20%</td></tr>
      </table>
      <div class="note" style="margin-top:8px;">💡 Apesar de ~168GB alocados em discos virtuais, o uso real em disco físico é de apenas <b style="color:var(--line);">38,4GB</b> — efeito do thin provisioning do LVM (só ocupa espaço conforme os dados são de fato escritos).</div>

      <div class="panel-title" style="margin-top:18px;">Arquivos de instalação (/var/lib/vz/template/iso/)</div>
      <table class="resources">
        <tr><th>Arquivo</th><th>Tamanho</th></tr>
        <tr><td>kali-linux-2026.2-installer.iso</td><td>4,47 GB</td></tr>
        <tr><td>ubuntu-26.04-live-server.iso</td><td>2,72 GB</td></tr>
        <tr><td>Metasploitable.vmdk</td><td>1,79 GB</td></tr>
        <tr><td>metasploitable-linux-2.0.0.zip</td><td>0,81 GB</td></tr>
        <tr><td>TOTAL (arquivos brutos)</td><td><b>9,79 GB</b></td></tr>
      </table>

      <div class="panel-title" style="margin-top:18px;">Próximas etapas</div>
      <div class="next-list">
        <div class="next-item"><div class="n">1</div> Finalizar instalação do Wazuh (Indexer+Manager+Dashboard)</div>
        <div class="next-item"><div class="n">2</div> Instalar Zabbix Server + Frontend</div>
        <div class="next-item"><div class="n">3</div> Deploy do Shuffle via Docker Compose</div>
        <div class="next-item"><div class="n">4</div> Finalizar instalação do Kali Linux</div>
        <div class="next-item"><div class="n">5</div> Testes de exploração com detecção via Wazuh</div>
      </div>
    </div>
  </div>

  <div class="quote">
    "Documentar cada etapa transforma um projeto pessoal em um portfólio técnico." <b>— Homelab, status atual</b>
  </div>

</div>

</body>
</html>

---

*Documentação em construção — atualizada conforme o projeto avança.*
