# Sistema Embarcado Nios II - Contador Binário em LEDs

Este repositório contém a implementação completa (hardware e firmware) de um sistema embarcado baseado no processador soft-core Nios II da Intel. O sistema foi projetado para operar na placa FPGA DE1-SoC, configurado para atuar como um contador binário de 10 bits utilizando temporização por interrupção de hardware.

## Estrutura do Diretório

A estrutura principal do projeto é composta por:
* `niios_bo_bc.qpf` / `.qsf`: Arquivos de projeto e configurações de pinagem do Quartus Prime.
* `nio.qsys` / `nio.sopcinfo`: Definição da arquitetura do sistema no Platform Designer e mapa de memória exportado.
* `output_files/`: Contém o arquivo binário compilado de hardware (`.sof`).
* `software/app_leds/`: Contém o código-fonte em C (`main.c`), o Makefile e o executável final compilado (`.elf`).
* `software/bsp_leds/`: Board Support Package (BSP) gerado a partir do hardware.

## Pré-requisitos

* **Hardware:** Placa de desenvolvimento FPGA DE1-SoC, Fonte de alimentação, Cabo USB-Blaster.
* **Software:** Intel Quartus Prime Lite Edition (versão 23.1std) com suporte a dispositivos Cyclone V e Nios II EDS (Eclipse Build Tools) instalados.

---

## Verificação do Projeto no Quartus Prime

Como o projeto já está estruturado, as etapas abaixo servem para inspeção, auditoria de conexões ou recompilação em caso de alterações.

### 1. Abrir o Projeto
1. Inicie o Quartus Prime.
2. Acesse `File` > `Open Project...` e selecione o arquivo `niios_bo_bc.qpf`.

### 2. Verificar o Hardware (Platform Designer)
1. No Quartus, acesse `Tools` > `Platform Designer`.
2. Abra o arquivo `nio.qsys`.
3. Na aba *System Contents*, é possível verificar as conexões físicas (barramentos Avalon, linhas de Clock/Reset e interrupções IRQ) entre os módulos: `nios2_gen2_0`, `onchip_memory2_0`, `jtag_uart_0`, `spi_0`, `timer_0` e `pio_0`.
4. Caso faça alguma alteração, clique em `Generate HDL...` para atualizar a descrição do hardware.

### 3. Verificar o Mapeamento de Pinos
1. No Quartus, acesse `Assignments` > `Assignment Editor` ou `Pin Planner`.
2. Verifique se as saídas lógicas do módulo PIO estão corretamente mapeadas para os pinos físicos dos LEDs (LEDR0 a LEDR9) e se os pinos de Clock (50MHz) estão designados corretamente para a placa DE1-SoC.

### 4. Compilação do Hardware
Caso o hardware seja alterado no Platform Designer ou na pinagem, é necessário recompilar:
1. No Quartus, clique no ícone **Compile Design** (ou acesse `Processing` > `Start Compilation`).
2. Este processo executa as etapas de *Analysis & Synthesis*, *Fitter* (roteamento físico) e *Assembler*, resultando na geração de um novo arquivo `niios_bo_bc.sof` na pasta `output_files/`.

---

## Procedimento de Execução (Gravação na Placa)

O envio do projeto para a placa física é dividido em duas fases rigorosas: configuração do hardware e injeção do firmware.

### Fase 1: Gravação do Hardware (.sof)
Esta etapa reconfigura os transistores do FPGA para instanciar o processador Nios II e seus periféricos.

1. Conecte a placa DE1-SoC na energia e conecte o cabo USB na porta **USB-Blaster**. Ligue a placa.
2. No Quartus Prime, acesse `Tools` > `Programmer`.
3. No canto superior esquerdo, clique em **Hardware Setup...** e selecione `USB-Blaster`.
   * *Nota para Linux:* Se o cabo não aparecer, consulte a seção de *Troubleshooting* abaixo.
4. Clique em **Add File...**, navegue até a pasta `output_files/` e selecione `niios_bo_bc.sof`.
5. Marque a caixa **Program/Configure** na linha do arquivo e clique em **Start**. A barra de progresso indicará 100% (Sucesso).

### Fase 2: Injeção do Software (.elf)
Com o hardware ativo, o código executável deve ser carregado na memória RAM (On-Chip Memory) do processador.

**No Linux (Fedora/Ubuntu):**
1. Abra um terminal de linha de comando.
2. Carregue as variáveis de ambiente do Nios II:
   ```bash
   ~/intelFPGA_lite/23.1std/nios2eds/nios2_command_shell.sh
   ```
3. Navegue até o diretório do projeto de software:
   ```bash
   cd /caminho/para/o/projeto/software/app_leds/
   ```
4. Execute o download do `.elf` para a placa via JTAG:
   ```bash
   nios2-download -g app_leds.elf
   ```
   * A flag `-g` instrui o processador a iniciar a execução imediatamente após o download.

**No Windows:**
1. Abra o **Nios II Command Shell** pelo menu Iniciar (instalado junto ao Quartus/Nios II EDS).
2. Navegue até o diretório `software/app_leds/` e execute o mesmo comando `nios2-download -g app_leds.elf`.

> **Resultado esperado:** Após a injeção do firmware, os LEDs LEDR0–LEDR9 da placa DE1-SoC devem começar a contar em binário de forma sequencial, incrementando a cada intervalo de tempo definido pelo `timer_0`.

---

## Troubleshooting

### USB-Blaster não reconhecido no Linux
Caso o cabo USB-Blaster não apareça no Programmer do Quartus:

1. Verifique se o serviço `jtagd` está em execução:
   ```bash
   ps aux | grep jtagd
   ```
2. Se não estiver rodando, inicie-o manualmente:
   ```bash
   ~/intelFPGA_lite/23.1std/quartus/bin/jtagd
   ```
3. Adicione regras udev para o dispositivo (pode requerer `sudo`):
   ```bash
   sudo nano /etc/udev/rules.d/51-usbblaster.rules
   ```
   Insira o seguinte conteúdo:
   ```
   SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0666"
   SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0666"
   SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0666"
   ```
4. Recarregue as regras udev e reconecte o cabo:
   ```bash
   sudo udevadm control --reload-rules
   ```

### Erro ao executar `nios2-download`
* Certifique-se de estar dentro do **Nios II Command Shell** (Linux) ou do shell equivalente no Windows — o comando não está disponível em terminais comuns.
* Confirme que a **Fase 1** (gravação do `.sof`) foi concluída com sucesso antes de tentar a injeção do firmware.
* Verifique se o arquivo `app_leds.elf` existe no diretório `software/app_leds/`. Caso não exista, recompile o software com `make` dentro do Nios II Command Shell.

---

## Recompilando o Software

Para recompilar o firmware a partir do código-fonte `main.c`:

1. Abra o **Nios II Command Shell**.
2. Navegue até `software/app_leds/`:
   ```bash
   cd /caminho/para/o/projeto/software/app_leds/
   ```
3. Execute:
   ```bash
   make clean && make
   ```
4. O novo arquivo `app_leds.elf` será gerado no mesmo diretório.
