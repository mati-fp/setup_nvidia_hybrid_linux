# Guia de Configuração NVIDIA(Turing+) em notebook Híbrido no Linux

Este guia aborda a instalação e configuração do driver NVIDIA para placas gráficas RTX 3060 Mobile em sistemas com gráficos híbridos (Intel + NVIDIA) no Arch Linux. O foco é garantir o suporte ao Wayland.

## 1. Identificação da Placa Gráfica

Para identificar sua placa gráfica, use:

```bash
lspci -k -d :03xx
```

Você deve ver uma saída semelhante a:

```
00:02.0 VGA compatible controller: Intel Corporation TigerLake-LP GT2 [Iris Xe Graphics] (rev 01)
    Kernel driver in use: i915
    Kernel modules: i915, xe

01:00.0 VGA compatible controller: NVIDIA Corporation GA106M [GeForce RTX 3060 Mobile / Max-Q] (rev a1)
    Kernel driver in use: nouveau (ou nvidia se já estiver instalado)
    Kernel modules: nouveau, nvidia_drm, nvidia
```

## 2. Escolhendo o Driver Correto

Para placas da série RTX 30xx (arquitetura Ampere) como a RTX 3060, você tem três opções:

1. **Driver NVIDIA Open Kernel (Recomendado)**
   - Pacote: `nvidia-open`
   - Melhor compatibilidade com Wayland
   - Quase o mesmo desempenho do driver proprietário
   - Suporte a funcionalidades modernas

2. **Driver NVIDIA Proprietário**
   - Pacote: `nvidia`
   - Desempenho levemente superior em alguns casos
   - Pode ter problemas com Wayland
   - Suporte completo a recursos NVIDIA

3. **Driver Nouveau (Open-Source)**
   - Pré-instalado no Arch Linux
   - Desempenho significativamente menor para jogos e aplicações 3D
   - Melhor compatibilidade com sistema, pior desempenho

## 3. Instalação do Driver NVIDIA Open Kernel

### 3.1 Bloqueando o Driver Nouveau

Primeiro, bloqueie o driver nouveau para evitar conflitos:

```bash
sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

Adicione as seguintes linhas:

```
blacklist nouveau
options nouveau modeset=0
```

### 3.2 Instalando o Pacote nvidia-open

```bash
sudo pacman -S nvidia-open
```

Se você já tiver o pacote `nvidia-open-dkms` instalado, o sistema irá perguntar se deseja removê-lo. Responda "y" para continuar.

## 4. Configurando o Suporte ao Wayland

### 4.1 Habilitando o Modeset

Para habilitar o suporte ao Wayland com o driver NVIDIA, você precisa ativar o parâmetro `modeset` do kernel:

```bash
echo "options nvidia_drm modeset=1" | sudo tee /etc/modprobe.d/nvidia.conf
```

### 4.2 Regenerando o Initramfs

Para aplicar as mudanças, regenere o initramfs:

```bash
sudo mkinitcpio -P
```

### 4.3 Configurando Hook do Pacman (Opcional)

Para garantir que o initramfs seja regenerado automaticamente quando o driver NVIDIA for atualizado:

```bash
sudo mkdir -p /etc/pacman.d/hooks/
sudo nano /etc/pacman.d/hooks/nvidia.hook
```

Adicione o seguinte conteúdo:

```
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-open
Target=linux

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P
```

## 5. Verificando a Instalação

Reinicie o sistema:

```bash
reboot
```

Após a reinicialização, verifique se o driver NVIDIA está sendo usado:

```bash
lspci -k -d :03xx
```

Você deve ver `Kernel driver in use: nvidia` para sua placa NVIDIA.

Verifique se o modeset está ativado:

```bash
cat /sys/module/nvidia_drm/parameters/modeset
```

Deve retornar `Y`.

Para verificar se o driver está funcionando corretamente, execute:

```bash
nvidia-smi
```

Isso deve mostrar informações sobre a sua placa gráfica.

## 6. Solução de Problemas Comuns

### 6.1 Driver não carrega após atualização do kernel

Se o driver não carregar após uma atualização do kernel, regenere o initramfs:

```bash
sudo mkinitcpio -P
```

### 6.2 "modprobe: ERROR: could not insert 'nvidia_drm': No such device"

Este erro geralmente indica que o módulo principal `nvidia` não está carregado. Verifique se os pacotes foram instalados corretamente:

```bash
pacman -Qs nvidia
```

Se faltar algum pacote essencial como `nvidia-utils`, reinstale o pacote principal:

```bash
sudo pacman -S nvidia-open
```

### 6.3 Tela preta ou problemas de login no Wayland

Verifique se o `modeset=1` está configurado corretamente:

```bash
cat /sys/module/nvidia_drm/parameters/modeset
```

Se não retornar `Y`, verifique seu arquivo `/etc/modprobe.d/nvidia.conf` e regenere o initramfs.

### 6.4 Problemas com suspensão ou hibernação

O driver NVIDIA pode ter problemas com suspensão ou hibernação. Verifique se os serviços necessários estão ativos:

```bash
systemctl status nvidia-suspend nvidia-hibernate nvidia-resume
```

### 6.5 Switching entre GPUs

Para laptops com gráficos híbridos, você pode usar ferramentas como:

- `nvidia-prime` (instale com `sudo pacman -S nvidia-prime`)
- `optimus-manager` (AUR)
- `supergfxctl` (para laptops mais novos)

Após instalar o `nvidia-prime`, você pode executar aplicações usando sua GPU NVIDIA assim:

```bash
prime-run nome_do_aplicativo
```

Para verificar se a GPU NVIDIA está sendo usada corretamente:

```bash
prime-run glxinfo | grep "OpenGL renderer"
```

Isso deve mostrar sua placa NVIDIA como o renderizador OpenGL.

## 7. Verificando o Funcionamento 3D com Wayland

Para verificar se o aceleramento 3D está funcionando com Wayland:

```bash
glxinfo | grep "OpenGL renderer"
```

Deverá mostrar sua placa gráfica se estiver funcionando corretamente.

Para testar o desempenho 3D:

```bash
glxgears
```

---

Este guia foi criado especificamente para o Arch Linux com placas NVIDIA RTX 3060 Mobile usando o driver nvidia-open. Os comandos e procedimentos podem variar levemente para outras distribuições ou versões de hardware.
