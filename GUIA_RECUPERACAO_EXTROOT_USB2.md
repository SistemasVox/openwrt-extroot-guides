# Guia de recuperação: extroot, USB 2.0 e serviços após a troca do pendrive

Este documento complementa o [guia de instalação para um roteador novo](GUIA_EXTROOT_NOVO_ROTEADOR_PENDRIVE.md). Resume verificações e falhas observadas durante a recuperação de um OpenWrt com armazenamento externo. Foi sanitizado para publicação: não contém UUIDs, endereços, chaves, credenciais, nomes de SSID ou caminhos privados de backup.

## Regras de segurança

- Guarde cópias de configuração e dos arquivos importantes fora do roteador e fora do pendrive de extroot. Um backup no próprio extroot não ajuda se o pendrive falhar.
- Não restaure uma configuração completa de outro modelo sem revisar rede, VLANs, Wi-Fi, firewall e nomes dos dispositivos.
- Confirme modelo, revisão de hardware, release, kernel e arquitetura antes de instalar firmware ou módulos `kmod-*`.
- Não execute `mkfs`, `mkswap` ou `parted mklabel` até confirmar com `block info` e `fdisk -l` qual dispositivo é realmente o pendrive. Esses comandos apagam dados.
- Mantenha credenciais, arquivos de configuração completos, chaves WireGuard/SSH e backups do roteador privados. Publique apenas documentação genérica e sanitizada.

## Diagnóstico inicial, sem alterar o sistema

Execute e salve a saída localmente, removendo dados identificáveis antes de compartilhar:

```sh
ubus call system board
cat /etc/openwrt_release 2>/dev/null
uname -r
mount | grep -E 'overlay|/rom|/dev/sd'
df -h
block info
cat /proc/swaps
ls -l /sys/block
dmesg | grep -iE 'usb|ehci|xhci|extroot|overlay|scsi|error|fail'
logread | grep -iE 'usb|ehci|xhci|extroot|overlay|error|fail' | tail -100
```

Antes de instalar pacotes, verifique se `/overlay` é a partição ext4 externa. `block info` mostrar o pendrive, por si só, não prova que o extroot foi montado.

## Falha observada: controlador USB carregando em modo lento

Na instalação de referência, o armazenamento foi inicialmente negociado a **12 Mb/s** e as gravações ficaram perto de **1 MB/s**. Depois de corrigir os módulos do controlador USB 2.0 e garantir que carregassem cedo no boot, o dispositivo de armazenamento negociou a **480 Mb/s**. Três gravações sincronizadas de 64 MiB ficaram na faixa de **8,0 a 9,9 MiB/s**, com média aproximada de **8,7 MiB/s**.

Esses valores são uma medição daquele roteador e daquele pendrive, não uma garantia para outro equipamento. O valor `12` pode pertencer a um hub/controlador USB 1.1 separado. Confira o arquivo `speed` do próprio dispositivo de armazenamento:

```sh
for f in /sys/bus/usb/devices/*/speed; do
    [ -f "$f" ] && echo "$f: $(cat "$f") Mb/s"
done
```

Em geral, `480` indica USB 2.0 High-Speed; `12` indica USB Full-Speed; `5000` indica USB 3.x a 5 Gb/s. A taxa de escrita real também depende da memória flash, controlador USB, CPU e filesystem.

## Verificar módulos USB e extroot

Os módulos do kernel precisam corresponder exatamente à release e ao kernel instalados. Confira:

```sh
uname -r
opkg list-installed 2>/dev/null | grep -E '^(kmod-usb|kmod-fs-ext4|block-mount)'
ls -l /lib/modules/$(uname -r)/*ehci* /lib/modules/$(uname -r)/*xhci* 2>/dev/null
ls -l /etc/modules-boot.d 2>/dev/null
lsmod | grep -E 'ehci|xhci|usb_storage|ext4|scsi'
```

Arquivos `.ko` com tamanho zero, mensagens `failed to mmap`, ou ausência dos módulos do controlador exigem corrigir a instalação antes de reiniciar. Instale os pacotes correspondentes ao firmware e kernel exatos; não copie módulos de outro aparelho nem use `insmod` manual como correção permanente.

O controlador também precisa carregar cedo o bastante para que o dispositivo USB esteja disponível ao montar `/overlay`. Se os logs mostrarem que o armazenamento aparece tarde, confirme os módulos em `/etc/modules-boot.d` e revise a opção `delay_root` do `fstab`. Não aumente o atraso sem evidência nos logs.

## WireGuard e Wi-Fi após recuperação

WireGuard e Wi-Fi dependem de módulos específicos do kernel e do hardware. Se serviços de rede desaparecerem após um reboot, verifique estado e logs antes de alterar rotas ou recriar configurações:

```sh
opkg list-installed | grep -E '^(kmod-wireguard|wireguard-tools|wpad|hostapd|kmod-mt76|kmod-cfg80211|kmod-mac80211)'
lsmod | grep -E 'wireguard|udp_tunnel|cfg80211|mac80211|mt76'
ls -l /lib/modules/$(uname -r)/*wireguard* 2>/dev/null
wg show
wifi status
logread | grep -iE 'wireguard|wg0|hostapd|wpad|mt76|cfg80211|protocol not supported|cannot find device' | tail -100
```

Erros como `Protocol not supported` ou `Cannot find device` sugerem que o módulo compatível não está carregado. Primeiro confirme release, kernel e integridade dos pacotes. Restaure as configurações de VPN somente a partir de um backup confiável e privado; nunca publique chaves.

## Confirmar montagem e swap depois do boot

Após corrigir os módulos e reiniciar com acesso local disponível:

```sh
mount | grep -E 'overlay|/rom'
df -h /overlay
block info
cat /proc/swaps
dmesg | grep -iE 'mount_root|extroot|overlay|usb|ehci|xhci|error|fail'
```

O extroot deve aparecer montado em `/overlay`; a partição de swap deve aparecer em `/proc/swaps`. Se o overlay permanecer no armazenamento interno, não continue instalando pacotes nem assuma que os arquivos do extroot estão ativos. Revise UUID no `fstab`, drivers USB/ext4, atraso de inicialização do USB e mensagens do boot.

## Teste controlado de escrita

Faça o teste somente depois de confirmar que `/overlay` é a partição externa. Ele grava 64 MiB temporários, força a sincronização e remove o arquivo:

```sh
TEST="/overlay/.writebench.$$"
trap 'rm -f "$TEST"' EXIT HUP INT TERM
time dd if=/dev/zero of="$TEST" bs=1M count=64 conv=fsync
rm -f "$TEST"
trap - EXIT HUP INT TERM
df -h /overlay
```

Evite repetir benchmarks sem necessidade: a memória flash tem ciclos de escrita limitados. A velocidade percebida por navegador também depende da rede e da CPU, não apenas do pendrive.

## Leituras oficiais

- [OpenWrt — configuração de extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)
- [OpenWrt — dispositivos e drivers de armazenamento USB](https://openwrt.org/docs/guide-user/storage/usb-drives)
- [OpenWrt — estado das releases](https://openwrt.org/releases/start)
