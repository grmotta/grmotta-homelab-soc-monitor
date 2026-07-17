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
