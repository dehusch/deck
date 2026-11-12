# Validação — Deck Lite 0.1.0

Executada em 9 de novembro de 2026, em hospedeiro amd64, VirtualBox 7.2.22. Os resultados abaixo referem-se ao protótipo e à configuração BIOS/IDE documentada; não comprovam compatibilidade Unix geral nem hardware físico.

## Artefatos finais

- Bootloader: 512 bytes, assinatura BIOS `55 aa`.
- Kernel binário: 34464 bytes; ELF64 AMD64 estático com símbolos de depuração em `build/kernel.elf`.
- ISO: 6258688 bytes, El Torito i386/BIOS com emulação de disquete e carga inicial de 1 setor.
- Imagem raw: 16777216 bytes. Gerada, sem teste em USB físico.
- Checksums: `dist/SHA256SUMS`.

## Testes em C

Comando: `sh scripts/test.sh`.

Testa diretamente o parser e as funções de filesystem do mesmo `kernel.c`, com saída e disco simulados apenas no executável de teste hospedeiro. O kernel compilado para a ISO não contém essa simulação.

Passaram:

- Normalização de caminhos relativos, `.` e `..`, inclusive acima da raiz e caminhos longos.
- Aspas e redirecionamento `>`/`>>` sem exigir espaços.
- Criação, cópia, movimentação, conteúdo e modos de arquivos.
- Rejeição de modos inválidos, aspas abertas, diretórios não vazios e pais inexistentes.
- Rejeição de cópia sobre o próprio arquivo e de append além de 4096 bytes, mantendo o conteúdo anterior.
- Limite de 96 entradas e recusa de novas criações ao esgotar o filesystem.
- Vetor conhecido de CRC32.
- Rejeição de caminhos sem terminador, entradas duplicadas e tipos inválidos.
- Alternância de snapshots, seleção de geração mais recente, falha de escrita e recuperação de snapshot anterior após corrupção do payload.
- Recusa de header inválido.
- Sintaxe dos scripts shell.

Resultado: `logs/unit-tests.log`.

## Teste de integração da ISO final

Comando executado: `sh scripts/smoke-test.sh Deck-Lite`.

VM criada exclusivamente para o teste: 128 MiB RAM, 1 CPU, BIOS, IDE PIIX4, disco VDI novo de 32 MiB como primário mestre, ISO como secundário mestre, COM1 gravando log, sem rede e sem discos físicos.

Passaram:

1. Boot pela ISO final até `deck login:`.
2. Login `root`, `uname -a`, listagem de diretórios e comandos básicos.
3. Criação de `/home/test/proof`, append, cópia/movimentação e modos.
4. Erros esperados: diretório não vazio, arquivo ausente, pai ausente, aspas abertas e destino de instalação inválido.
5. `pkg install vim` informa claramente indisponibilidade e não simula sucesso.
6. Cancelamento da instalação ao responder `CANCEL`.
7. Instalação após `ERASE ada0`, gravação de kernel e snapshot.
8. Desligamento, retirada da ISO e novo boot somente pelo disco.
9. Recuperação do snapshot e leitura do conteúdo `Deck persistence works` e `second-line`.
10. Criação de segundo snapshot com `sync` e desligamento.

Evidências:

- `logs/Deck-Lite-serial.log`: boot live e instalação.
- `logs/Deck-Lite-installed.log`: boot pelo disco e leitura dos arquivos.
- `logs/Deck-Lite-persistence.png`: captura do sistema instalado.
- `logs/integration-tests.log`: resultado do teste automatizado.
- `vm/Deck-Lite/Deck-Lite.vbox` e `vm/Deck-Lite/disk.vdi`: VM instalada, desligada e registrada no VirtualBox.

A rodada exploratória anterior permanece em `vm/Deck-Lite-0.1-test/`, com disco em `vm/deck-test.vdi` e logs `serial-live.log` / `serial-installed.log`. Para uso, prefira a VM final **Deck-Lite**.

## Não testado / não implementado

Hardware físico, USB/UEFI/Secure Boot, SATA/NVMe, CPU apenas 32 bits, memória menor que a configuração recomendada, interrupções concorrentes, processos, usuários comuns, autenticação, POSIX completo, rede, pacotes externos ou robustez de filesystem para produção. A injeção de falhas em snapshots foi feita em testes hospedeiros, não por corte de energia real durante escrita na VM.
