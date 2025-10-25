# Guia Completo: Configuração Dual GPU (Intel + NVIDIA) no Arch Linux com Hyprland

> **Sistema:** Arch Linux | **Compositor:** Hyprland | **Dotfiles:** omarchy  
> **Hardware:** MSI Notebook com Intel TigerLake Iris Xe + NVIDIA RTX 3060 Mobile (Muxless)  
> **Objetivo:** Usar Intel iGPU por padrão (economia de bateria) e NVIDIA dGPU apenas com `prime-run`

---

## Índice

1. [Informações do Sistema](#informações-do-sistema)
2. [Objetivo da Configuração](#objetivo-da-configuração)
3. [Pacotes Necessários](#pacotes-necessários)
4. [Configuração Passo a Passo](#configuração-passo-a-passo)
5. [Configuração do Hyprland](#configuração-do-hyprland)
6. [Verificações e Testes](#verificações-e-testes)
7. [Troubleshooting](#troubleshooting)
8. [Referências](#referências)

---

## Informações do Sistema

### Hardware Detectado

#### Para identificar sua placa gráfica use:
```bash
lspci -k -d :03xx
```

```bash
# GPUs
00:02.0 VGA compatible controller: Intel Corporation TigerLake-LP GT2 [Iris Xe Graphics]
01:00.0 VGA compatible controller: NVIDIA Corporation GA106M [GeForce RTX 3060 Mobile / Max-Q]

# Mapeamento de dispositivos (lembrar que pode alterar em novas instalações)
/dev/dri/card2 → Intel (00:02.0)
/dev/dri/card1 → NVIDIA (01:00.0)
```

### Software

- **Bootloader:** Limine
- **Kernel:** Linux
- **Display Server:** Wayland
- **Compositor:** Hyprland (com dotfiles omarchy)
- **Session Manager:** uwsm (Universal Wayland Session Manager)

---

## Objetivo da Configuração

### O que queremos alcançar:

1. ✅ **Intel iGPU como padrão** para todas as aplicações (economia de bateria)
2. ✅ **NVIDIA dGPU desligada** quando não estiver em uso (RTD3 Power Management)
3. ✅ **NVIDIA dGPU ativa apenas com `prime-run`** para jogos/apps pesados
4. ✅ **Suspensão funcionando corretamente** ao fechar a tampa do notebook
5. ✅ **Preservação da VRAM** durante suspensão (sem crashes ao acordar)

---

## Pacotes Necessários

### Drivers Intel

```bash
sudo pacman -S mesa vulkan-intel lib32-vulkan-intel intel-media-driver libva-intel-driver mesa-utils
```

### Drivers NVIDIA

```bash
sudo pacman -S nvidia-open-dkms nvidia-utils lib32-nvidia-utils nvidia-prime
```

### Suporte adicional para PRIME

```bash
sudo pacman -S vulkan-mesa-layers lib32-vulkan-mesa-layers
```

---

## Configuração Passo a Passo

### 1. Configurar módulos do kernel

Edite `/etc/mkinitcpio.conf`:

```bash
sudo nano /etc/mkinitcpio.conf
```

**Modifique a linha MODULES:**

```conf
# IMPORTANTE: i915 deve vir ANTES dos módulos NVIDIA
MODULES=(i915 nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

### 2. Habilitar DRM kernel modesetting da NVIDIA

Crie `/etc/modprobe.d/nvidia.conf`:

```bash
sudo nano /etc/modprobe.d/nvidia.conf
```

**Adicione:**

```conf
options nvidia_drm modeset=1 fbdev=1
```

### 3. Configurar gerenciamento de energia RTD3

Crie `/etc/modprobe.d/nvidia-pm.conf`:

```bash
sudo nano /etc/modprobe.d/nvidia-pm.conf
```

**Adicione:**

```conf
# Para RTX 3060 Mobile (Ampere) usar 0x03
# Para placas Turing usar 0x02
options nvidia "NVreg_DynamicPowerManagement=0x03" "NVreg_PreserveVideoMemoryAllocations=1"
```

**Explicação dos parâmetros:**
- `NVreg_DynamicPowerManagement=0x03`: Habilita RTD3 para placas Ampere/Ada (0x02 para Turing)
- `NVreg_PreserveVideoMemoryAllocations=1`: Preserva VRAM durante suspensão (evita crashes ao acordar)

### 4. Configurar regras udev para power management

Crie `/etc/udev/rules.d/80-nvidia-pm.rules`:

```bash
sudo nano /etc/udev/rules.d/80-nvidia-pm.rules
```

**Adicione:**

```conf
# Enable runtime PM for NVIDIA VGA/3D controller devices on driver bind
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"

# Disable runtime PM for NVIDIA VGA/3D controller devices on driver unbind
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="on"
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="on"

# Enable runtime PM for NVIDIA VGA/3D controller devices on adding device
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"
```

### 5. Habilitar serviços systemd

```bash
# Habilitar persistência NVIDIA
sudo systemctl enable nvidia-persistenced.service

# Habilitar power management daemon (para Dynamic Boost)
sudo systemctl enable nvidia-powerd.service

# Habilitar ações de suspensão/hibernação
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
```

**Nota sobre nvidia-powerd:** Pode apresentar erros relacionados a Dynamic Boost no MSI, mas não afeta funcionalidade. Os erros são inofensivos.

### 6. Configurar suspensão ao fechar a tampa

Edite `/etc/systemd/logind.conf`:

```bash
sudo nano /etc/systemd/logind.conf
```

**Descomente e configure:**

```conf
HandleLidSwitch=suspend-then-hibernate
HandleLidSwitchExternalPower=suspend
HandleLidSwitchDocked=ignore
```

**Reinicie o serviço:**

```bash
sudo systemctl restart systemd-logind.service
```

### 7. Reconstruir initramfs

```bash
sudo mkinitcpio -P
```

---

## Configuração do Hyprland

### Identificar dispositivos DRI

```bash
ls -l /dev/dri/by-path/
```

**Output esperado:**

```
pci-0000:00:02.0-card -> ../card2  (Intel)
pci-0000:01:00.0-card -> ../card1  (NVIDIA)
```

### Configurar variáveis de ambiente

**Se usar dotfiles omarchy padrão:**

Edite `~/.config/hypr/hyprland.conf` ou o arquivo de env específico (pode ser `~/.config/hypr/env.conf`):

```conf
# Multi-GPU: usar Intel (card2) como primária
env = AQ_DRM_DEVICES,/dev/dri/card2:/dev/dri/card1

# Usar Intel/Mesa por padrão para economia de energia
env = __EGL_VENDOR_LIBRARY_FILENAMES,/usr/share/glvnd/egl_vendor.d/50_mesa.json
env = __GLX_VENDOR_LIBRARY_NAME,mesa
env = LIBVA_DRIVER_NAME,iHD  # Usar Intel para vídeo
## Força o carregador Vulkan a ver apenas o driver da Intel
env = VK_ICD_FILENAMES,/usr/share/vulkan/icd.d/intel_icd.x86_64.json
## Electron fix
env = ELECTRON_OZONE_PLATFORM_HINT,wayland
```

**Se usar uwsm (recomendado para omarchy):**

Edite `~/.config/uwsm/env-hyprland`:

```bash
export AQ_DRM_DEVICES="/dev/dri/card2:/dev/dri/card1"
export __EGL_VENDOR_LIBRARY_FILENAMES="/usr/share/glvnd/egl_vendor.d/50_mesa.json"
export __GLX_VENDOR_LIBRARY_NAME="mesa"
```

### ⚠️ IMPORTANTE: Remover variáveis NVIDIA antigas

**Remova ou comente estas linhas se existirem:**

```conf
# REMOVER/COMENTAR estas linhas:
# env = NVD_BACKEND,direct
# env = LIBVA_DRIVER_NAME,nvidia
# env = __GLX_VENDOR_LIBRARY_NAME,nvidia
# env = GBM_BACKEND,nvidia-drm  # Esta especialmente causa problemas em multi-GPU!
```

**Por quê?** Essas variáveis forçam todas as aplicações a usarem a NVIDIA, anulando a configuração de economia de energia.

---

## Verificações e Testes

### 1. Reiniciar o sistema

```bash
sudo reboot
```

### 2. Verificar status dos serviços

```bash
# nvidia-persistenced deve estar running
systemctl status nvidia-persistenced.service

# nvidia-powerd deve estar running (pode ter erros não-críticos)
systemctl status nvidia-powerd.service

# Verificar se GPU NVIDIA está suspensa
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# Deve mostrar: suspended
```

### 3. Verificar qual GPU está sendo usada por padrão

```bash
# Deve mostrar Intel
glxinfo | grep "OpenGL renderer"
# Output esperado: Mesa Intel(R) Xe Graphics (TGL GT2)

vulkaninfo | grep "deviceName"
# Output esperado: Intel(R) Xe Graphics (TGL GT2)
```

### 4. Testar NVIDIA com prime-run

```bash
# Deve mostrar NVIDIA
prime-run glxinfo | grep "OpenGL renderer"
# Output esperado: NVIDIA GeForce RTX 3060 Mobile

prime-run vulkaninfo | grep "deviceName"
# Output esperado: NVIDIA GeForce RTX 3060 Mobile

# Ver informações da GPU
prime-run nvidia-smi
```

### 5. Verificar RTD3 (Runtime D3)

```bash
cat /proc/driver/nvidia/gpus/0000:01:00.0/power
```

**Output esperado:**

```
Runtime D3 status:          Enabled (fine-grained)
Video Memory:               Off
GPU Hardware Support:
 Video Memory Self Refresh: Supported
 Video Memory Off:          Supported
```

### 6. Testar suspensão

```bash
# Suspender manualmente
systemctl suspend

# Aguardar alguns segundos e acordar (pressionar tecla)

# Verificar se GPU voltou a suspender
sleep 10
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# Deve mostrar: suspended
```

### 7. Testar fechamento da tampa

1. Feche a tampa do notebook
2. Aguarde alguns segundos
3. Abra a tampa
4. Sistema deve acordar normalmente
5. Aplicativos devem continuar funcionando

---

## Usando NVIDIA para jogos e aplicações pesadas

### Com aplicativos nativos

```bash
prime-run nome-do-aplicativo
```

### Com jogos Steam

Nas propriedades do jogo → Opções de lançamento:

```
prime-run %command%
```

### Com Wine/Proton

As variáveis já funcionam automaticamente com `prime-run`.

---

## Troubleshooting

### GPU NVIDIA não suspende

**Verificar:**

```bash
# Ver o que está usando a GPU
sudo lsof /dev/nvidia*

# Logs do sistema
journalctl -b 0 | grep -i nvidia
```

**Possíveis causas:**
- Algum processo usando a GPU (feche aplicativos)
- nvidia-persistenced não está rodando
- Regras udev não foram aplicadas (reinicie)

### Hyprland não inicia ou tela preta

**Verificar variáveis de ambiente:**

```bash
# Se usar uwsm
cat ~/.config/uwsm/env-hyprland

# Verificar se não tem conflito de variáveis
env | grep -i nvidia
env | grep -i glx
```

**Solução:** Remova todas as variáveis que forçam NVIDIA globalmente.

### Aplicativos crasham ao acordar da suspensão

**Causa:** `NVreg_PreserveVideoMemoryAllocations` não está configurado.

**Solução:** Verifique se está em `/etc/modprobe.d/nvidia-pm.conf` e reconstrua initramfs:

```bash
sudo mkinitcpio -P
sudo reboot
```

### Erros do nvidia-powerd sobre Dynamic Boost

```
ERROR! DC power limits table is not supported
ERROR! Failed to get SysPwrLimitGetInfo!!
ERROR! Client (presumably SBIOS) has requested to disable Dynamic Boost DC controller
```

**Isso é NORMAL em notebooks MSI!** O BIOS não expõe as tabelas necessárias.

**Impacto:** Nenhum. RTD3 e economia de energia funcionam normalmente.

**Opcional:** Se os erros te incomodam visualmente:

```bash
sudo systemctl disable nvidia-powerd.service
```

### Suspensão não funciona ao fechar a tampa

**Verificar:**

```bash
# Ver configuração atual
cat /etc/systemd/logind.conf | grep HandleLidSwitch

# Reiniciar serviço
sudo systemctl restart systemd-logind.service
```

---

## Referências

### Documentação Oficial

- [Arch Wiki - PRIME](https://wiki.archlinux.org/title/PRIME)
- [Hyprland Wiki - Multi-GPU](https://wiki.hypr.land/Configuring/Multi-GPU/)
- [Hyprland Wiki - NVIDIA](https://wiki.hypr.land/Nvidia/)
- [NVIDIA Linux Driver Documentation](https://download.nvidia.com/XFree86/Linux-x86_64/580.95.05/README/)

### Parâmetros NVIDIA

- `NVreg_DynamicPowerManagement`:
  - `0x00`: Desabilitado
  - `0x02`: RTD3 para Turing (RTX 16/20 series)
  - `0x03`: RTD3 para Ampere/Ada (RTX 30/40 series)

- `NVreg_PreserveVideoMemoryAllocations`:
  - `0`: Não preserva VRAM (padrão)
  - `1`: Preserva VRAM durante suspensão

### Variáveis de Ambiente

- `AQ_DRM_DEVICES`: Define prioridade de GPUs para Hyprland (substitui `WLR_DRM_DEVICES`)
- `__GLX_VENDOR_LIBRARY_NAME`: Define biblioteca GLX (mesa ou nvidia)
- `__EGL_VENDOR_LIBRARY_FILENAMES`: Define bibliotecas EGL
- `DRI_PRIME`: Usado por drivers open-source (não usado com NVIDIA)

---

## Notas Finais

### Consumo de energia esperado

- **Idle (Intel apenas):** ~5-8W (GPU NVIDIA suspensa)
- **Navegação/trabalho leve:** ~8-12W
- **Com NVIDIA ativa:** +15-30W adicional

### Benefícios desta configuração

Máxima duração de bateria no dia a dia  
Performance completa quando necessário  
Transição automática e transparente  
Sem necessidade de reiniciar para trocar de GPU  
Compatível com Wayland/Hyprland  

### Manutenção

Ao atualizar o kernel ou drivers NVIDIA:

```bash
# Reconstruir módulos DKMS automaticamente
sudo dkms autoinstall

# Reconstruir initramfs
sudo mkinitcpio -P
```

---

## 🙏 Créditos

Configuração baseada em:
- Arch Wiki
- Hyprland Wiki
- Comunidade Arch Linux
- NVIDIA Developer Documentation

---

**Data da última atualização:** Outubro 2025  
**Testado com:** NVIDIA Driver 580.95.05 (Open Kernel Module)
