# Tutorial bem explicado: extroot e USB 2.0 no OpenWrt

Este tutorial mostra como usar um pendrive como armazenamento gravável do OpenWrt (**extroot**) e como conferir se ele está conectado em **USB 2.0 High-Speed**. Também inclui a configuração de 512 MiB de swap.

Foi preparado com base no que funcionou em um **TP-Link Archer C5 v4 / MT7620A**, com **OpenWrt 18.06.7**, kernel **4.14.167** e pendrive de 64 GB. O OpenWrt 18.06 está fora de suporte; use a seção de pacotes antiga somente para reproduzir essa instalação. Em outro roteador, primeiro confirme modelo, revisão, versão e drivers. Não instale módulos de kernel de outro modelo ou versão.

Os resultados e valores do Archer são reais; o histórico não preservou o comando exato usado para particionar o pendrive. Por isso, a preparação com GParted é apresentada como um método reproduzível, e não como uma alegação de que foi a ferramenta usada naquela ocasião. A formatação ext4 no roteador é recomendada aqui para manter compatibilidade com o firmware antigo.

> **Cuidado:** particionar ou formatar apaga os dados do disco escolhido. Antes de continuar, faça backup da configuração do roteador e de tudo que estiver no pendrive. Se um comando mostrar um disco diferente do esperado, pare: não tente adivinhar.

## Em palavras simples

- `/rom` é a parte do firmware que normalmente não pode ser alterada.
- `/overlay` é onde ficam configurações, pacotes e arquivos graváveis.
- Com extroot, o OpenWrt monta a partição ext4 do pendrive em `/overlay` durante o boot.
- O pendrive passa a ser importante para o funcionamento normal. Se ele falhar ou for retirado, o roteador poderá iniciar usando o pequeno armazenamento interno, com uma configuração antiga ou incompleta.
- **USB 2.0 a 480 Mb/s é a velocidade de conexão do barramento, não a velocidade garantida de gravação.** No Archer C5 de referência, a escrita medida ficou em média perto de 8,7 MiB/s.

## 1. Anote o modelo e faça backup

Conecte um computador ao roteador por cabo. Baixe um backup pelo LuCI em **System → Backup / Flash Firmware → Generate archive** e guarde-o no computador, não apenas no pendrive.

No SSH do roteador, rode:

```sh
ubus call system board
uname -r
df -h
```

Guarde a saída. Ela ajuda a escolher os pacotes corretos. O pacote `kmod-*` precisa corresponder exatamente ao kernel e à versão do OpenWrt instalados.

## 2. Instale o suporte necessário

### Para reproduzir o Archer C5 v4 antigo

No equipamento de referência, estes pacotes foram usados para armazenamento USB, USB 2.0, ext4 e montagem do extroot:

```sh
opkg update
opkg install block-mount kmod-fs-ext4 e2fsprogs kmod-usb-core kmod-usb-ehci kmod-usb2 kmod-usb-storage kmod-usb-ohci kmod-usb-uhci
```

Essa lista é **específica da instalação antiga**. Não a copie para um modelo diferente só porque o comando está aqui. Em versões atuais, nomes e dependências de pacotes podem mudar; use a página do seu aparelho e os pacotes da mesma versão do firmware. OpenWrt 25.12 e posteriores usam `apk`; versões anteriores usam `opkg`.

Depois da instalação, confira o kernel e os módulos. Arquivos de módulo com tamanho zero, ou mensagens `failed to mmap`, indicam instalação inválida ou incompatível — não tente resolver copiando `.ko` de outro aparelho:

```sh
uname -r
opkg list-installed | grep -E '^(block-mount|kmod-fs-ext4|kmod-usb|e2fsprogs)'
ls -l /lib/modules/$(uname -r)/ehci-hcd.ko /lib/modules/$(uname -r)/ehci-platform.ko 2>/dev/null
```

Se os módulos corretos acabaram de ser instalados, reinicie uma vez e confirme a velocidade USB antes de configurar o extroot. Mantenha o computador conectado por cabo.

## 3. Confira se o pendrive está em USB 2.0

Conecte o pendrive e execute:

```sh
dmesg | grep -iE 'usb|ehci|ohci|xhci|storage'
for f in /sys/bus/usb/devices/*/speed; do
    [ -f "$f" ] && echo "$f: $(cat "$f") Mb/s"
done
```

Identifique na saída qual entrada corresponde ao pendrive; o roteador e eventuais hubs também aparecem. Em geral, `12` significa USB Full-Speed (USB 1.1), `480` significa USB 2.0 High-Speed e `5000` significa USB 3.x a 5 Gb/s. Se o pendrive aparecer a `12`, verifique se o roteador tem controlador USB 2.0 e se o driver EHCI/USB2 correto foi instalado. Nem todo conector ou equipamento suporta USB 2.0.

No caso de referência, o pendrive aparecia a **12 Mb/s** porque os arquivos do driver EHCI estavam inválidos (tamanho zero e erro `failed to mmap`). Após corrigir e carregar os módulos USB/EHCI correspondentes ao firmware, o pendrive passou a aparecer a **480 Mb/s**. Não use `insmod` manual como solução permanente: o módulo precisa ser instalado corretamente e carregar sozinho no boot.

## 4. Prepare as partições do pendrive

O método mais simples para iniciantes é criar as partições em um computador com **GParted**. Selecione o pendrive com muita atenção — a operação apaga o disco selecionado — e crie:

1. Uma partição para extroot, ocupando quase todo o pendrive.
2. Uma partição **Linux swap** de **512 MiB**.

No computador, aplique as operações e ejete o pendrive com segurança. Não formate nem particione um disco de computador por engano. A partição grande pode ser criada como ext4; ela será formatada novamente no roteador, usando as ferramentas compatíveis com o firmware. Isso evita que uma ferramenta recente ative recursos ext4 que um kernel antigo não reconheça.

> A criação das partições pelo GParted é a recomendação para este tutorial. Os comandos abaixo são para identificar e usar partições já preparadas; não executam particionamento destrutivo.

Conecte novamente o pendrive ao roteador e confira:

```sh
block info
cat /proc/partitions
```

Na instalação de referência, a partição grande era `/dev/sda1` e a swap `/dev/sda2`. **Isso pode ser diferente no seu aparelho.** A saída de `block info` deve mostrar as partições do pendrive. Confira seus nomes antes de formatar.

Somente depois de confirmar com `block info` e `cat /proc/partitions` que `/dev/sda1` é a partição grande do pendrive — e não outro disco — formate-a no roteador:

```sh
mkfs.ext4 -L extroot /dev/sda1
block info
```

> **Este comando apaga tudo em `/dev/sda1`.** Se o dispositivo no seu roteador tiver outro nome, substitua-o apenas depois de confirmar. `mkfs.ext4` precisa estar instalado (`e2fsprogs`). Ele cria o filesystem; não cria nem redimensiona partições.

Confirme que `block info` mostra `TYPE="ext4"` para a partição grande e `TYPE="swap"` para a partição de swap. Se não estiverem assim, pare e corrija antes de prosseguir. Anote o UUID ext4.

## 5. Configure o extroot e a swap

Antes de mexer no `fstab`, confira o que já existe e salve uma cópia:

```sh
if [ ! -s /etc/config/fstab ]; then
    block detect > /etc/config/fstab
fi
/etc/init.d/fstab enable
uci show fstab
cp /etc/config/fstab /root/fstab-antes-extroot
```

Se já houver uma seção montando outro disco em `/overlay`, não cole os comandos seguintes sem revisar: entradas antigas duplicadas podem causar falha no boot.

No exemplo abaixo, substitua `/dev/sda1` e `/dev/sda2` pelos dispositivos que você identificou em `block info`. O primeiro bloco encontra o UUID da partição ext4 e para com erro se não conseguir encontrá-lo:

```sh
EXTROOT_DEV=/dev/sda1
SWAP_DEV=/dev/sda2
EXTROOT_UUID="$(block info "$EXTROOT_DEV" | sed -n 's/.*UUID="\([^"]*\)".*/\1/p')"
if [ -z "$EXTROOT_UUID" ]; then
    echo "ERRO: UUID ext4 não encontrado. Confira block info e EXTROOT_DEV. Nada foi configurado."
    exit 1
fi
echo "UUID do extroot: $EXTROOT_UUID"
```

Confira visualmente o UUID mostrado. Depois configure o ponto de montagem `/overlay`, a swap e as opções globais usadas na instalação de referência:

```sh
uci -q delete fstab.extroot
uci set fstab.extroot='mount'
uci set fstab.extroot.target='/overlay'
uci set fstab.extroot.uuid="$EXTROOT_UUID"
uci set fstab.extroot.fstype='ext4'
uci set fstab.extroot.options='rw,noatime'
uci set fstab.extroot.enabled='1'

uci -q delete fstab.usb_swap
uci set fstab.usb_swap='swap'
uci set fstab.usb_swap.device="$SWAP_DEV"
uci set fstab.usb_swap.enabled='1'

uci set fstab.@global[0].anon_swap='0'
uci set fstab.@global[0].anon_mount='0'
uci set fstab.@global[0].auto_swap='1'
uci set fstab.@global[0].auto_mount='1'
uci set fstab.@global[0].delay_root='5'
uci set fstab.@global[0].check_fs='0'
uci commit fstab
uci show fstab
```

Se `uci show fstab` não exibir uma seção global (`fstab.@global[0]` ou equivalente), pare e peça ajuda antes de continuar. As opções globais variam conforme a versão e a configuração inicial.

## 6. Copie o overlay atual para o pendrive

Ainda **antes de reiniciar**, monte a partição ext4 em uma pasta temporária e copie o conteúdo de `/overlay`. Confira novamente que `EXTROOT_DEV` é a partição ext4 certa:

```sh
mkdir -p /mnt/extroot
mount -t ext4 "$EXTROOT_DEV" /mnt/extroot
df -h /overlay /mnt/extroot
tar -C /overlay -cf - . | tar -C /mnt/extroot -xf -
sync
ls -la /mnt/extroot | head
umount /mnt/extroot
```

O comando `tar` inclui arquivos ocultos e mantém a estrutura de diretórios. Se `mount`, `tar` ou `umount` der erro, não reinicie: corrija primeiro. Confirme também que o UUID salvo no `fstab` é o da mesma partição ext4.

## 7. Reinicie e confirme que funcionou

Reinicie com o computador ainda conectado por cabo:

```sh
reboot
```

Após o roteador voltar, conecte-se e rode:

```sh
mount | grep -E 'overlay|/rom'
df -h /overlay /
block info
cat /proc/swaps
dmesg | grep -iE 'mount_root|extroot|overlay|usb|ehci|error|fail'
```

O esperado é:

- `/dev/sda1` (ou a partição que você identificou) montada em `/overlay` como ext4;
- `overlayfs:/overlay` montada em `/`;
- partição de swap de 512 MiB listada em `/proc/swaps`.

Se `/overlay` continuar na memória interna, não instale mais pacotes ainda. Confira o UUID, o `fstab`, o suporte a ext4 e os erros de USB no `dmesg`.

Confirme novamente a negociação USB e o carregamento dos módulos:

```sh
for f in /sys/bus/usb/devices/*/speed; do
    [ -f "$f" ] && echo "$f: $(cat "$f") Mb/s"
done
lsmod | grep -E 'ehci|xhci|usb_storage|ext4'
```

Faça ao menos mais um reinício de teste, ainda com o pendrive conectado, para confirmar que extroot, USB e swap voltam automaticamente depois de um boot a frio.

## 8. Teste de velocidade de gravação

Faça este teste **somente depois de confirmar que `/overlay` está no pendrive**. Ele grava um arquivo temporário de 64 MiB e o apaga ao final:

```sh
TEST_FILE="/overlay/.writebench.$$"
time dd if=/dev/zero of="$TEST_FILE" bs=1M count=64 conv=fsync
rm -f "$TEST_FILE"
df -h /overlay
```

No Archer C5 v4 de referência, três medições deram aproximadamente **8,0, 8,2 e 9,9 MiB/s**, média de **8,7 MiB/s**. O resultado depende do pendrive, do controlador USB e da CPU. Não espere 480 Mb/s de gravação: esse número descreve a conexão USB, não a taxa real do armazenamento. Evite repetir o teste muitas vezes para não desgastar o pendrive.

## Se algo der errado

- Se o roteador não montar o extroot, não formate o pendrive. Guarde as saídas de `block info`, `uci show fstab`, `mount`, `df -h` e os erros relevantes de `dmesg`.
- Se for seguro para o seu aparelho, desligue-o, retire o pendrive e ligue novamente. O OpenWrt pode iniciar pelo overlay interno, que talvez não contenha as alterações feitas depois de ativar extroot.
- Corrija UUID, drivers ou `fstab` antes de tentar outro reboot. Não instale `kmod-*` de outro kernel e não use `insmod` como solução de boot.
- Backups importantes devem ficar também em outro computador ou armazenamento, nunca somente no pendrive que sustenta o extroot.

## Documentação oficial

- [Configuração de extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)
- [Guia rápido para unidades USB](https://openwrt.org/docs/guide-user/storage/usb-drives-quickstart)
- [Armazenamento USB](https://openwrt.org/docs/guide-user/storage/usb-drives)
- [Gerenciamento de pacotes](https://openwrt.org/docs/guide-user/additional-software/managing_packages)
- [Aparelhos suportados pelo OpenWrt](https://openwrt.org/toh/start)
