# Guia Completo: Configuração Dual GPU (Intel + NVIDIA) no Arch Linux com Hyprland

> **Sistema:** Arch Linux | **Compositor:** Hyprland | **Dotfiles:** omarchy  
> **Hardware:** MSI Notebook com Intel TigerLake Iris Xe + NVIDIA RTX 3060 Mobile (Muxless)  
> **Objetivo:** Usar Intel iGPU por padrão (economia de bateria) e NVIDIA dGPU apenas com `prime-run`

---

##  Índice

1. [Informações do Sistema](#informações-do-sistema)
2. [Objetivo da Configuração](#objetivo-da-configuração)
3. [Pacotes Necessários](#pacotes-necessários)
4. [Configuração Passo a Passo](#configuração-passo-a-passo)
5. [Symlinks Permanentes (CRÍTICO)](#symlinks-permanentes-crítico)
6. [Configuração do Hyprland](#configuração-do-hyprland)
7. [Otimizar prime-run](#otimizar-prime-run)
8. [Verificações e Testes](#verificações-e-testes)
9. [Troubleshooting](#troubleshooting)
10. [Referências](#referências)

---

## Informações do Sistema

### Hardware Detectado

#### Para identificar sua placa gráfica use:
```bash
lspci -k -d :03xx
ls -l /dev/dri/by-path/
```


```bash
# GPUs
00:02.0 VGA compatible controller: Intel Corporation TigerLake-LP GT2 [Iris Xe Graphics]
01:00.0 VGA compatible controller: NVIDIA Corporation GA106M [GeForce RTX 3060 Mobile / Max-Q]

# Mapeamento de dispositivos (MUDA A CADA REBOOT - por isso precisamos de symlinks!)
/dev/dri/card0 ou card1 → Pode ser Intel ou NVIDIA
/dev/dri/card1 ou card2 → Pode ser Intel ou NVIDIA

# Paths estáveis
pci-0000:00:02.0 → Intel (sempre)
pci-0000:01:00.0 → NVIDIA (sempre)
```

### Software

- **Bootloader:** Limine
- **Kernel:** Linux
- **Display Server:** Wayland
- **Compositor:** Hyprland (com dotfiles omarchy)
- **Session Manager:** uwsm (Universal Wayland Session Manager)

---

##  Objetivo da Configuração

### O que queremos alcançar:

1. **Intel iGPU como padrão** para todas as aplicações (economia de bateria)
2. **NVIDIA dGPU desligada** quando não estiver em uso (RTD3 Power Management)
3. **NVIDIA dGPU ativa apenas com `prime-run`** para jogos/apps pesados
4. **Suspensão funcionando corretamente** ao fechar a tampa do notebook
5. **Preservação da VRAM** durante suspensão (sem crashes ao acordar)
6. **XWayland usando Intel** por padrão (evitar que apps X11 ativem a NVIDIA)
7. **Symlinks permanentes** para evitar problemas com card0/card1 mudando

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

**Nota sobre nvidia-powerd:** Pode apresentar erros relacionados a Dynamic Boost no MSI, mas não afeta funcionalidade. Os erros são inofensivos:

```
ERROR! DC power limits table is not supported
ERROR! Failed to get SysPwrLimitGetInfo!!
ERROR! Client (presumably SBIOS) has requested to disable Dynamic Boost DC controller
```

Isso acontece porque o BIOS do MSI não expõe as tabelas de gerenciamento necessárias. O RTD3 continua funcionando normalmente.

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

## Symlinks Permanentes (CRÍTICO)

### Por que isso é necessário?

Os dispositivos `/dev/dri/card0`, `/dev/dri/card1`, etc. **MUDAM A CADA REBOOT**! 

Em um boot:
- Intel pode ser `card1`, NVIDIA `card0`

No próximo boot:
- Intel pode ser `card2`, NVIDIA `card1`

**Solução:** Criar symlinks permanentes baseados nos endereços PCI (que nunca mudam).

### 1. Criar regra udev para Intel

```bash
sudo nano /etc/udev/rules.d/50-intel-igpu-dev-path.rules
```

**Adicione:**

```bash
KERNEL=="card*", KERNELS=="0000:00:02.0", SUBSYSTEM=="drm", SUBSYSTEMS=="pci", SYMLINK+="dri/intel-igpu"
```

### 2. Criar regra udev para NVIDIA

```bash
sudo nano /etc/udev/rules.d/51-nvidia-dgpu-dev-path.rules
```

**Adicione:**

```bash
KERNEL=="card*", KERNELS=="0000:01:00.0", SUBSYSTEM=="drm", SUBSYSTEMS=="pci", SYMLINK+="dri/nvidia-dgpu"
```

### 3. Recarregar regras udev

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 4. Verificar os symlinks

```bash
ls -l /dev/dri/
```

**Você deve ver:**

```
lrwxrwxrwx intel-igpu -> card1 (ou card0, card2, etc)
lrwxrwxrwx nvidia-dgpu -> card0 (ou card1, card2, etc)
```

Os symlinks `intel-igpu` e `nvidia-dgpu` agora são **permanentes** e sempre apontarão para os dispositivos corretos, independente de qual `cardX` seja atribuído.

---

## Configuração do Hyprland

### Configurar variáveis de ambiente

**Edite `~/.config/hypr/envs.conf` (ou arquivo de env do omarchy):**

```conf
# Extra env variables
# env = MY_GLOBAL_ENV,setting

# NVIDIA environment variables
## NVIDIA - parâmetros são setados temporariamente ao executar prime-run
#env = NVD_BACKEND,direct
#env = LIBVA_DRIVER_NAME,nvidia
#env = __GLX_VENDOR_LIBRARY_NAME,nvidia

# Multi-GPU: Intel como primária (USAR SYMLINKS!)
env = AQ_DRM_DEVICES,/dev/dri/intel-igpu:/dev/dri/nvidia-dgpu
env = WLR_DRM_DEVICES,/dev/dri/intel-igpu:/dev/dri/nvidia-dgpu

# Intel drivers como padrão
env = LIBVA_DRIVER_NAME,iHD  # Usar Intel para vídeo (VA-API)
env = __GLX_VENDOR_LIBRARY_NAME,mesa
env = __EGL_VENDOR_LIBRARY_FILENAMES,/usr/share/glvnd/egl_vendor.d/50_mesa.json

# Força o carregador Vulkan a ver apenas o driver da Intel
env = VK_ICD_FILENAMES,/usr/share/vulkan/icd.d/intel_icd.x86_64.json

# Prevenir XWayland de ativar NVIDIA automaticamente
env = __NV_PRIME_RENDER_OFFLOAD,0  # Desabilita offload NVIDIA por padrão
env = __VK_LAYER_NV_optimus,NVIDIA_only  # Força uso explícito da NVIDIA

## Electron fix
env = ELECTRON_OZONE_PLATFORM_HINT,wayland
```

**Se usar uwsm, edite também `~/.config/uwsm/env`:**

```bash
export AQ_DRM_DEVICES="/dev/dri/intel-igpu:/dev/dri/nvidia-dgpu"
export __EGL_VENDOR_LIBRARY_FILENAMES="/usr/share/glvnd/egl_vendor.d/50_mesa.json"
export __GLX_VENDOR_LIBRARY_NAME="mesa"
```

### IMPORTANTE: Remover variáveis NVIDIA antigas

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

## Otimizar prime-run

O script `prime-run` padrão pode precisar de ajustes para funcionar perfeitamente em Wayland/Hyprland.

### Verificar e modificar prime-run

```bash
sudo nano /usr/bin/prime-run
```

**Conteúdo recomendado:**

```bash
#!/bin/bash
__NV_PRIME_RENDER_OFFLOAD=1 \
__VK_LAYER_NV_optimus=NVIDIA_only \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
GBM_BACKEND=nvidia-drm \
LIBVA_DRIVER_NAME=nvidia \
"$@"
```

**Explicação das variáveis adicionadas:**
- `GBM_BACKEND=nvidia-drm`: Força GBM a usar NVIDIA (importante para Wayland)
- `LIBVA_DRIVER_NAME=nvidia`: Usa aceleração de vídeo da NVIDIA (VA-API)
- `__VK_LAYER_NV_optimus=NVIDIA_only`: Força Vulkan a usar apenas NVIDIA
- `__NV_PRIME_RENDER_OFFLOAD=1`: Habilita offload PRIME da NVIDIA
- `__GLX_VENDOR_LIBRARY_NAME=nvidia`: Usa biblioteca OpenGL da NVIDIA

---

## Verificações e Testes

### 1. Reiniciar o sistema

```bash
sudo reboot
```

### 2. Verificar symlinks permanentes

```bash
ls -l /dev/dri/
# Deve mostrar: intel-igpu e nvidia-dgpu
```

### 3. Verificar status dos serviços

```bash
# nvidia-persistenced deve estar running
systemctl status nvidia-persistenced.service

# nvidia-powerd deve estar running (pode ter erros não-críticos)
systemctl status nvidia-powerd.service

# Verificar se GPU NVIDIA está suspensa
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# Deve mostrar: suspended
```

### 4. Verificar qual GPU está sendo usada por padrão

```bash
# Deve mostrar Intel
glxinfo | grep "OpenGL renderer"
# Output esperado: Mesa Intel(R) Xe Graphics (TGL GT2)

vulkaninfo | grep "deviceName"
# Output esperado: Intel(R) Xe Graphics (TGL GT2)
```

### 5. Verificar que NVIDIA não tem processos rodando

```bash
nvidia-smi
# Deve mostrar: "No running processes found"
```

**Se aparecer `/usr/lib/Xorg` (XWayland) na lista, significa que alguma aplicação X11 está forçando NVIDIA. As variáveis de ambiente que configuramos devem prevenir isso.**

### 6. Testar NVIDIA com prime-run

```bash
# Deve mostrar NVIDIA
prime-run glxinfo | grep "OpenGL renderer"
# Output esperado: NVIDIA GeForce RTX 3060 Mobile

prime-run vulkaninfo | grep "deviceName"
# Output esperado: NVIDIA GeForce RTX 3060 Mobile

# Ver informações da GPU
prime-run nvidia-smi
```

### 7. Verificar RTD3 (Runtime D3)

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

### 8. Testar que GPU volta a suspender após uso

```bash
# Usar NVIDIA
prime-run glxinfo | grep "OpenGL renderer"

# Aguardar 10 segundos
sleep 10

# Verificar se voltou a suspender
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# Deve mostrar: suspended
```

### 9. Testar suspensão

```bash
# Suspender manualmente
systemctl suspend

# Aguardar alguns segundos e acordar (pressionar tecla)

# Verificar se GPU voltou a suspender
sleep 10
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# Deve mostrar: suspended
```

### 10. Testar fechamento da tampa

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

### Exemplos práticos

```bash
# Jogos
prime-run steam
prime-run lutris

# Blender com NVIDIA
prime-run blender

# DaVinci Resolve
prime-run davinci-resolve

# OBS Studio com NVENC
prime-run obs
```

---

## Troubleshooting

### GPU NVIDIA não suspende

**Verificar:**

```bash
# Ver o que está usando a GPU
sudo lsof /dev/nvidia*

# Ver processos no nvidia-smi
nvidia-smi

# Logs do sistema
journalctl -b 0 | grep -i nvidia
```

**Possíveis causas:**
- Algum processo usando a GPU (feche aplicativos)
- XWayland está usando NVIDIA (verifique variáveis de ambiente)
- nvidia-persistenced não está rodando
- Regras udev não foram aplicadas (reinicie)

### XWayland continua usando NVIDIA

**Verificar variáveis:**

```bash
# Ver variáveis de ambiente do Hyprland
env | grep -E "GLX|EGL|VK_ICD|NV_PRIME|VK_LAYER"
```

**Deve mostrar:**
```
__GLX_VENDOR_LIBRARY_NAME=mesa
__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json
__NV_PRIME_RENDER_OFFLOAD=0
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/intel_icd.x86_64.json
```

**Se estiver diferente:** Verifique os arquivos de configuração do Hyprland e uwsm.

### Hyprland não inicia ou tela preta

**Verificar symlinks:**

```bash
ls -l /dev/dri/
# Deve ter: intel-igpu e nvidia-dgpu
```

**Verificar logs:**

```bash
journalctl -b 0 | grep -i hyprland
```

**Solução temporária:** Se symlinks não foram criados, identifique manualmente:

```bash
ls -l /dev/dri/by-path/
# Use os paths corretos temporariamente na configuração
```

### Aplicativos crasham ao acordar da suspensão

**Causa:** `NVreg_PreserveVideoMemoryAllocations` não está configurado.

**Solução:** Verifique se está em `/etc/modprobe.d/nvidia-pm.conf` e reconstrua initramfs:

```bash
cat /etc/modprobe.d/nvidia-pm.conf
# Deve conter: NVreg_PreserveVideoMemoryAllocations=1

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

Você perderá apenas o Dynamic Boost (que já não funciona no MSI), mas RTD3 continuará funcionando.

### Suspensão não funciona ao fechar a tampa

**Verificar:**

```bash
# Ver configuração atual
cat /etc/systemd/logind.conf | grep HandleLidSwitch

# Deve mostrar:
# HandleLidSwitch=suspend

# Reiniciar serviço
sudo systemctl restart systemd-logind.service
```

### Symlinks não são criados após reboot

**Verificar regras udev:**

```bash
cat /etc/udev/rules.d/50-intel-igpu-dev-path.rules
cat /etc/udev/rules.d/51-nvidia-dgpu-dev-path.rules
```

**Testar regras manualmente:**

```bash
sudo udevadm test /sys/class/drm/card0
sudo udevadm test /sys/class/drm/card1
```

**Forçar trigger:**

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=drm
```

### Performance baixa com prime-run

**Verificar que está realmente usando NVIDIA:**

```bash
prime-run nvidia-smi
# Deve mostrar o processo rodando

prime-run glxgears -info | grep GL_RENDERER
# Deve mostrar: NVIDIA GeForce RTX 3060
```

**Se ainda usar Intel:** Verifique o script `/usr/bin/prime-run` e certifique-se que contém todas as variáveis.

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

#### Para usar Intel por padrão:
- `AQ_DRM_DEVICES`: Define prioridade de GPUs para Hyprland (substitui `WLR_DRM_DEVICES`)
- `__GLX_VENDOR_LIBRARY_NAME=mesa`: Define biblioteca GLX como Mesa (Intel)
- `__EGL_VENDOR_LIBRARY_FILENAMES`: Define bibliotecas EGL como Mesa
- `VK_ICD_FILENAMES`: Define driver Vulkan como Intel
- `LIBVA_DRIVER_NAME=iHD`: Usa driver VA-API da Intel
- `__NV_PRIME_RENDER_OFFLOAD=0`: Desabilita offload NVIDIA globalmente

#### Para usar NVIDIA com prime-run:
- `__NV_PRIME_RENDER_OFFLOAD=1`: Habilita offload NVIDIA
- `__GLX_VENDOR_LIBRARY_NAME=nvidia`: Usa OpenGL da NVIDIA
- `__VK_LAYER_NV_optimus=NVIDIA_only`: Força Vulkan NVIDIA
- `GBM_BACKEND=nvidia-drm`: Usa GBM da NVIDIA (Wayland)
- `LIBVA_DRIVER_NAME=nvidia`: Usa VA-API da NVIDIA

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
Symlinks permanentes (sem problemas com card0/card1 mudando)  
XWayland usando Intel (evita ativação acidental da NVIDIA)  

### Manutenção

Ao atualizar o kernel ou drivers NVIDIA:

```bash
# Reconstruir módulos DKMS automaticamente
sudo dkms autoinstall

# Reconstruir initramfs
sudo mkinitcpio -P
```

### Checklist de verificação pós-atualização

```bash
# 1. Symlinks ainda existem?
ls -l /dev/dri/ | grep -E "intel-igpu|nvidia-dgpu"

# 2. Serviços rodando?
systemctl status nvidia-persistenced.service

# 3. NVIDIA suspensa?
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status

# 4. Intel por padrão?
glxinfo | grep "OpenGL renderer"

# 5. prime-run funciona?
prime-run glxinfo | grep "OpenGL renderer"
```

---

## Créditos

Configuração baseada em:
- Arch Wiki
- Hyprland Wiki
- Comunidade Arch Linux
- NVIDIA Developer Documentation
- Testado e refinado em MSI Notebook com RTX 3060 Mobile

---
**Testado com:** 
- NVIDIA Driver 580.95.05 (Open Kernel Module)
- Hyprland com omarchy dotfiles
- Arch Linux (atualizado em 27/10/2025)
