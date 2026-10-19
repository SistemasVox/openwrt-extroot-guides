# Guias de extroot e armazenamento USB no OpenWrt

Este repositório reúne procedimentos para ampliar o armazenamento gravável de roteadores OpenWrt com um pendrive, configurar swap e diagnosticar problemas de USB/extroot após a reinicialização.

## Guias

- [Instalação em outro roteador e pendrive](GUIA_EXTROOT_NOVO_ROTEADOR_PENDRIVE.md) — preparação, drivers, particionamento, extroot, swap, testes e recuperação.
- [Recuperação do extroot e diagnóstico USB](GUIA_RECUPERACAO_EXTROOT_USB2.md) — verificações de boot, módulos USB/kernel, Wi-Fi e WireGuard, montagem e desempenho.

## Antes de usar

Os comandos e nomes de pacotes variam por modelo, revisão de hardware e versão do OpenWrt. Confirme esses dados e identifique cuidadosamente o dispositivo do pendrive antes de particionar ou formatar: essas operações apagam dados. Mantenha backups fora do roteador e do próprio pendrive.

Os resultados de desempenho citados são referências observadas em um equipamento específico, não garantias para outros roteadores ou unidades USB. Consulte também a [documentação oficial de extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration), a [documentação de armazenamento USB](https://openwrt.org/docs/guide-user/storage/usb-drives) e as [releases do OpenWrt](https://openwrt.org/releases/start).
