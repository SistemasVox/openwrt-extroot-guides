# Guia público: extroot, swap e USB rápido em um OpenWrt novo

Este procedimento serve para configurar **outro roteador e outro pendrive do zero**. Ele não depende das chaves, UUIDs, IPs, SSIDs ou senhas de nenhuma instalação anterior. Foi escrito com base no extroot do OpenWrt e na experiência de um Archer C5 v4, mas nomes de módulos, arquitetura, kernel, partições e limites variam conforme o equipamento.

> Atenção: particionar/formatear apaga o disco selecionado. Leia e valide o dispositivo antes de executar qualquer comando de particionamento.

## O que o extroot faz

O OpenWrt normalmente inicia seu firmware somente leitura em `/rom` e usa uma pequena área gravável interna como `/overlay`. O extroot monta uma partição externa sobre `/overlay`, ampliando o espaço para configurações, arquivos e pacotes. O firmware principal continua na memória do roteador. A documentação oficial descreve a preparação, a cópia do overlay e a configuração do `fstab` [aqui](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration).

O pendrive passa a ser parte essencial do boot. Sem ele, muitos roteadores voltam ao overlay interno, mas podem mostrar uma configuração antiga ou mínima. Mantenha uma cópia dos backups **fora do pendrive e fora do roteador**.

## 1. Preparar o roteador e os backups

1. Conecte um computador por cabo a uma LAN funcional. Tenha uma forma local de recuperar o aparelho caso perca a rede depois do reboot.
2. Baixe a imagem OpenWrt correta para o **modelo e revisão de hardware exatos** na página oficial do aparelho. Em firmware de fábrica use a imagem `factory`; para atualizar um OpenWrt existente use `sysupgrade`. Não grave uma imagem de outro modelo.
3. Em LuCI, use **System → Backup / Flash Firmware → Generate archive** e baixe o backup; copie também para fora do roteador os arquivos e scripts necessários. Um backup guardado somente no extroot não protege contra a falha do próprio pendrive.
4. Não restaure cegamente o backup integral de outro roteador. Interfaces, rádios, VLANs, nomes de dispositivo e regras de firewall podem ser diferentes. Reaplique apenas o que foi revisado para o novo hardware.
5. Registre o estado inicial:

```sh
ubus call system board
cat /etc/openwrt_release 2>/dev/null
uname -r
opkg print-architecture 2>/dev/null || true
ip link show
df -h
```

Anote modelo/revisão, release, kernel, arquitetura, RAM/flash disponível e nomes de interfaces. Não publique uma exportação UCI integral sem remover segredos.

## 2. Confirmar USB e selecionar os drivers do hardware

Antes de comprar ou particionar, confirme que o roteador oferece host USB e que o kernel possui driver para o controlador. Em geral, módulos de armazenamento não bastam: o chipset pode precisar de EHCI (USB 2.0), xHCI (USB 3.x) ou de outro controlador específico. Use a página de hardware do roteador e os pacotes da mesma release.

Conecte o pendrive e confira:

```sh
dmesg | grep -iE 'usb|ehci|xhci|scsi|storage'
ls -l /sys/block
block info
for f in /sys/bus/usb/devices/*/speed; do
    [ -f "$f" ] && echo "$f: $(cat "$f") Mb/s"
done
```

O arquivo `speed` do **dispositivo do pendrive** indica a negociação efetiva: `12` normalmente é USB Full-Speed (USB 1.1), `480` é USB 2.0 High-Speed e `5000` é USB 3.x a 5 Gb/s. Os hubs/controladores também aparecem na lista e podem mostrar velocidades diferentes. Um conector USB 2.0 nominalmente suporta 480 Mb/s, mas isso não garante a mesma velocidade de gravação: pendrive, controlador e CPU costumam limitar o resultado.

No OpenWrt, os módulos `kmod-*` precisam corresponder exatamente ao kernel instalado. Não instale módulos copiados de outro roteador ou de outra versão do firmware. Confirme a versão antes:

```sh
uname -r
opkg list-installed 2>/dev/null | grep '^kmod-' | head
```

O gerenciador de pacotes depende da release: OpenWrt 24.10 e anteriores usam `opkg`; 25.12 e posteriores usam `apk` ([documentação oficial](https://openwrt.org/docs/guide-user/additional-software/apk)). Atualize listas e instale os pacotes correspondentes à release e ao controlador identificados:

```sh
# OpenWrt 24.10 ou anterior: exemplo; acrescente o driver do controlador USB
opkg update
opkg install block-mount kmod-fs-ext4 e2fsprogs kmod-usb-storage

# OpenWrt 25.12 ou posterior: exemplo; acrescente o driver do controlador USB
apk update
apk add block-mount kmod-fs-ext4 e2fsprogs kmod-usb-storage
```

Instale **somente uma** das alternativas acima. Acrescente os pacotes do controlador que a página do hardware exigir (por exemplo, EHCI/USB2 ou xHCI/USB3). Se precisar formatar pelo próprio roteador, instale também `parted` ou use `fdisk`; alternativamente, particione no computador Linux. Instale estes componentes antes de configurar extroot, para que o roteador consiga detectar o dispositivo durante o boot inicial.

Se for usar `parted`/`fdisk` no roteador, instale o utilitário disponível no repositório compatível com esse firmware (`parted` e/ou `fdisk`). Se não estiver disponível, faça a preparação de partições em um computador Linux.

### Exemplo histórico: Archer C5 v4 / MT7620A

Na instalação de referência com OpenWrt 18.06.7/kernel 4.14.167 foram necessários `kmod-usb-core`, `kmod-usb-ehci`, `kmod-usb2`, `kmod-usb-storage`, `kmod-usb-ohci`, `kmod-usb-uhci` e `kmod-fs-ext4`, além de `block-mount`. Os módulos USB/ext4/SCSI tinham links em `/etc/modules-boot.d` para carregar cedo. Não copie essa lista para outro modelo sem validar o controlador e a release.

## 3. Identificar o disco e decidir o particionamento

O dispositivo pode ser `/dev/sda`, `/dev/sdb` ou outro nome. Identifique-o imediatamente antes de qualquer alteração:

```sh
block info
ls -l /sys/block
fdisk -l
```

Confira capacidade, modelo e partições pelo sistema disponível. Defina o nome correto apenas depois dessa verificação. O exemplo a seguir supõe um pendrive novo de 64 GB reconhecido como `/dev/sda`; **substitua o nome se for diferente**.

Layout sugerido:

- partição 1: maior parte do disco, ext4, rótulo `extroot`;
- partição 2: 512 MiB de swap, rótulo `swap`.

No computador Linux ou com `parted` compatível no roteador, o exemplo abaixo cria GPT, extroot grande e swap de 512 MiB no fim do disco:

```sh
parted -s /dev/sda -- mklabel gpt
parted -s /dev/sda -- mkpart extroot ext4 1MiB -513MiB
parted -s /dev/sda -- mkpart swap linux-swap -512MiB 100%
```

Se `parted` não aceitar esses offsets, pare e use seu gerenciador de partições para criar o mesmo layout. Não improvise sobre um disco que contenha dados importantes. Depois confirme novamente as partições:

```sh
fdisk -l /dev/sda
```

Somente depois de confirmar que `/dev/sda1` e `/dev/sda2` são as novas partições do pendrive:

```sh
mkfs.ext4 -L extroot /dev/sda1
mkswap -L swap /dev/sda2
block info
```

Anote o UUID de `/dev/sda1`. UUIDs são específicos de cada formato e não devem ser copiados deste guia.

## 4. Configurar extroot e copiar o overlay

Confirme que o pacote `block-mount` está instalado, que o disco aparece no `block info`, e que o overlay atual é o interno (não monte o pendrive de destino sobre a própria origem). Monte temporariamente a partição nova:

```sh
mkdir -p /mnt/extroot
mount /dev/sda1 /mnt/extroot
```

Copie o conteúdo do overlay gravável atual, incluindo arquivos ocultos e permissões:

```sh
tar -C /overlay -cf - . | tar -C /mnt/extroot -xf -
sync
umount /mnt/extroot
```

Use `uci show fstab` para inspecionar a configuração existente. Preserve outras montagens que o roteador já use. O exemplo seguinte cria seções nomeadas para extroot e swap; antes, remova **somente** seções antigas que montem `/overlay` ou habilitem o swap da unidade substituída, para não deixar configurações duplicadas:

```sh
# Troque UUID_NOVO pelo UUID que block info mostrou para a partição ext4.
uci -q delete fstab.extroot
uci set fstab.extroot='mount'
uci set fstab.extroot.target='/overlay'
uci set fstab.extroot.uuid='UUID_NOVO'
uci set fstab.extroot.fstype='ext4'
uci set fstab.extroot.options='rw,noatime'
uci set fstab.extroot.enabled='1'

uci -q delete fstab.usb_swap
uci set fstab.usb_swap='swap'
uci set fstab.usb_swap.device='/dev/sda2'
uci set fstab.usb_swap.enabled='1'

uci commit fstab
uci show fstab
```

Se `fstab` ainda não tiver uma seção `global`, crie-a e configure `auto_mount=1`, `auto_swap=1`, `anon_mount=0`, `anon_swap=0`. Em roteadores com USB lento para enumerar, pode ser necessário aumentar `delay_root` (por exemplo, para 5 segundos); ajuste apenas se os logs mostrarem que o dispositivo chega tarde.

Com UCI, a seção global pode ser criada/configurada assim:

```sh
uci -q get fstab.global >/dev/null || uci set fstab.global='global'
uci set fstab.global.anon_swap='0'
uci set fstab.global.anon_mount='0'
uci set fstab.global.auto_swap='1'
uci set fstab.global.auto_mount='1'
uci set fstab.global.delay_root='5'
uci set fstab.global.check_fs='0'
uci commit fstab
```

Antes de reiniciar, confira o UUID, a montagem `/overlay`, os módulos e os links de carregamento antecipado:

```sh
block info
uci show fstab
ls -l /etc/modules-boot.d 2>/dev/null
ls -l /lib/modules/$(uname -r)
```

## 5. Reiniciar e provar que o extroot entrou

Reinicie com acesso local disponível:

```sh
reboot
```

Após reconectar, confirme todos estes pontos:

```sh
mount | grep -E 'overlay|/rom'
df -h /overlay
block info
cat /proc/swaps
dmesg | grep -iE 'mount_root|extroot|overlay|usb|ehci|xhci|error|fail'
```

Esperado: a partição ext4 do pendrive montada em `/overlay`, `overlayfs` montado em `/`, e a partição de swap ativa em `/proc/swaps`. Se `/overlay` continuar no armazenamento interno, extroot não entrou. Não continue instalando pacotes: revise UUID, `fstab`, driver USB/ext4 e logs. Remover o pendrive e reiniciar geralmente permite que dispositivos com overlay interno iniciem pelo armazenamento interno; use isso como recuperação e faça as correções localmente.

Confirme também a velocidade do dispositivo de armazenamento e os módulos do controlador:

```sh
for f in /sys/bus/usb/devices/*/speed; do
    [ -f "$f" ] && echo "$f: $(cat "$f") Mb/s"
done
lsmod | grep -E 'ehci|xhci|usb_storage|ext4'
```

Investigue arquivos `.ko` de tamanho zero ou erros `failed to mmap`. Reinstale os pacotes de kernel correspondentes **à release e ao kernel daquele roteador** antes de reiniciar de novo; não use `insmod` manual como correção permanente.

## 6. Medir a escrita real

Faça o teste somente depois de confirmar que `/overlay` é o pendrive. Ele escreve 64 MiB, força `fsync` e apaga o arquivo temporário:

```sh
TEST="/overlay/.writebench.$$"
trap 'rm -f "$TEST"' EXIT HUP INT TERM
time dd if=/dev/zero of="$TEST" bs=1M count=64 conv=fsync
rm -f "$TEST"
trap - EXIT HUP INT TERM
df -h /overlay
```

Na unidade de referência (Archer C5 v4, pendrive de 64 GB), foram medidos 8,0–9,9 MiB/s, média aproximada de 8,7 MiB/s. Outro pendrive/roteador pode ser mais lento ou mais rápido. A taxa de transferência por rede também depende do Wi-Fi/Ethernet e da CPU. Evite executar benchmarks repetidamente: pendrives têm vida útil limitada por ciclos de escrita.

## 7. Recuperação e restauração

### Se o pendrive falhar ou o roteador não montar o extroot

1. Não formate o pendrive defeituoso antes de tentar copiar os dados.
2. Desligue o roteador, retire o pendrive e ligue novamente. Se o hardware conservar um overlay interno funcional, o OpenWrt deve iniciar em modo reduzido. Conecte o computador por cabo e descubra o IP de gerenciamento.
3. Confirme `mount`, `df -h`, `block info` e `logread`. Identifique se o problema é USB, partição, UUID, filesystem ou módulo.
4. Prepare o pendrive substituto e repita as seções 2–6. O UUID será novo e precisa ser atualizado no `fstab` antes do reboot.
5. Reinstale/restaure somente o que for compatível com o hardware e firmware novos. Recrie manualmente interfaces Wi-Fi, VLAN e firewall quando a placa for de outro modelo.
6. Não conte com arquivos de backup guardados no próprio pendrive que falhou. Recupere-os da cópia externa.

### Arquivos de backup e Git

Para backup privado, guarde a exportação de configurações, a lista de pacotes e os scripts em local externo e protegido. Para Git público, publique apenas este guia (e scripts realmente genéricos revisados). **Não** publique `sysupgrade`/UCI exports, `/etc/config/network`, `/etc/config/wireless`, configurações de WireGuard, chaves SSH, arquivos `.env`, números de telefone, logs com IPs pessoais nem arquivos com credenciais.

## 8. Exemplo observado e limitações

No roteador de referência, antes da correção o dispositivo de armazenamento apareceu a 12 Mb/s e a escrita havia sido reportada perto de 1 MB/s. Depois que os drivers EHCI/USB2 e arquivos de módulo válidos passaram a carregar no boot, o dispositivo apareceu a 480 Mb/s; três gravações sincronizadas deram média de cerca de 8,7 MiB/s. Isso mostra ganho real naquele conjunto específico, mas não garante o mesmo resultado para outro modelo ou pendrive.

OpenWrt 18.06 é uma série encerrada e sem atualizações de segurança. Para um roteador novo, use uma versão ainda suportada e imagem própria para o hardware; as instruções e nomes de pacotes antigos podem mudar. Consulte o [estado oficial das releases](https://openwrt.org/releases/start), a [página de hardware](https://openwrt.org/toh/start) e a [documentação de armazenamento USB](https://openwrt.org/docs/guide-user/storage/usb-drives).
