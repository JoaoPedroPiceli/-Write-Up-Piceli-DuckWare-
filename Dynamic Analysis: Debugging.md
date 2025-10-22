# Dynamic Analysis — Debugging

> Autor: @Piceli{DuckWare}

---

## Introdução
Analisar malware é um ciclo de observação, controlo e validação. O objetivo é entender o comportamento de uma amostra em execução, identificar rotinas de evasão e extrair indicadores úteis para deteção e mitigação. Este documento reúne conceitos, práticas e ilustrações do uso de x32dbg/x64dbg para depuração e patching defensivo. Todo o conteúdo é conceitual e destinado a fins defensivos e educacionais.

---

## Etapa 1 — Resumo executivo
- Malware usa evasões estáticas (ofuscação, packing, strings cifradas, carregamento dinâmico) para esconder intenção antes da execução.  
- Malware usa evasões dinâmicas (detecção de VM/sandbox, checagens temporais, verificação de atividade do usuário, detecção de ferramentas) para alterar comportamento em execução.  
- Depuradores permitem pausar e inspecionar execução, ver registradores, pilha e memória, e manipular estado em memória para testar hipóteses.  
- Patching defensivo modifica o binário em ambiente isolado para impedir rotinas de evasão e permitir análises subsequentes.  
- Trabalhe sempre em VMs isoladas com snapshots e registre logs de rede e sistema.  
- Ferramentas úteis: x64dbg/x32dbg, WinDbg, GDB, IDA/Ghidra, Process Monitor, Wireshark.

---

## Etapa 2 — Evasão estática (detalhe)
Técnicas que dificultam a leitura do binário sem execução:

- **Alteração de hash:** pequenas alterações no binário mudam assinaturas baseadas em hash.  
- **Ofuscação de código e strings:** valores importantes são cifrados e só revelados em tempo de execução.  
- **Packing:** o conteúdo malicioso é encapsulado por um stub que o desembrulha em execução.  
- **Carregamento dinâmico de DLLs:** imports reais podem não aparecer nos cabeçalhos estáticos.

Entender essas técnicas orienta a escolha de ferramentas e hipóteses para análise dinâmica.

---

## Etapa 3 — Evasão dinâmica (detalhe)
Técnicas observadas durante execução:

- **Detecção de VM/Sandbox:** busca por artefactos de virtualização e recursos reduzidos.  
- **Ataques de tempo:** longos sleeps ou comparações de tempo para frustrar sandboxes de curta duração.  
- **Verificação de atividade do usuário:** procura por histórico de uso, eventos de I/O e outros sinais de atividade humana.  
- **Detecção de ferramentas:** enumeração de processos/janelas para identificar monitores e depuradores.

Identificar essas checagens ajuda a priorizar onde inspecionar e quando aplicar medidas defensivas.

---

## Etapa 4 — Depuração: conceitos e tipos
- **Nível de origem:** opera sobre código-fonte, mostra variáveis de alto nível.  
- **Nível de montagem:** trabalha com binários compilados, mostra instruções e registradores. Padrão em engenharia reversa.  
- **Nível de kernel:** depura código em modo kernel; normalmente requer máquina separada para não travar o sistema alvo.

A depuração fornece visibilidade instrução-a-instrução e permite validar hipóteses sobre fluxos e condições.

---

## Etapa 5 — x32dbg / x64dbg — visão funcional

### Imagem 1 — Interface vazia
[![image.png](https://i.postimg.cc/j2ccBPyP/image.png)](https://postimg.cc/rRd5rdnw)  
Interface inicial do x32dbg, sem nenhum arquivo aberto. Mostra os painéis principais: CPU, Dump, Registradores, Breakpoints, Memory Map e Call Stack.

---

### Imagem 2 — Amostra aberta (CPU / desmontagem)
[![image.png](https://i.postimg.cc/8cpBdcxf/image.png)](https://postimg.cc/sQLhsjj3)  
Mostra uma amostra em execução pausada em breakpoint de sistema. Painel central exibe a desmontagem (instruções assembly), o direito mostra registradores e flags, e o inferior exibe dumps de memória e pilha.

---

### Imagem 3 — Logs do depurador
[![image.png](https://i.postimg.cc/RF9c9qSY/image.png)](https://postimg.cc/nsRsGFQK)  
Log do processo de depuração: inicialização, carregamento de DLLs e breakpoints. Permite acompanhar o progresso do programa e confirmar se o depurador está ativo.

---

### Imagem 4 — Lista de breakpoints
[![image.png](https://i.postimg.cc/05XJXVwm/image.png)](https://postimg.cc/7b76fMXY)  
Lista de breakpoints definidos. Mostra endereço, módulo e estado. Permite ativar, desativar e analisar pontos de interrupção críticos, como `EntryPoint` e `IsDebuggerPresent`.

---

### Imagem 5 — Memory Map
[![image.png](https://i.postimg.cc/sXDvwWDV/image.png)](https://postimg.cc/1g2mzfqT)  
Exibe o mapeamento de memória do processo. Mostra regiões reservadas e seções do binário (`.text`, `.data`, `.rdata`). Ajuda a localizar código executável, dados e memória alocada por threads.

---

### Imagem 6 — Call Stack / Threads
[![image.png](https://i.postimg.cc/Ghs97drt/image.png)](https://postimg.cc/1fs94x81)  
Mostra pilhas de chamadas e threads em execução. Ajuda a rastrear o fluxo de execução e entender as rotinas ativas no momento da depuração.

---

## Etapa 6 — Abrir amostra e pontos de parada automáticos (TLS)

### Imagem — arquivo aberto
[![image.png](https://i.postimg.cc/rp5PQdZ9/image.png)](https://postimg.cc/Y4qRSSH4)  
x32dbg ligado ao processo após abrir `crackme-arebel.exe`. O depurador liga ao processo e pausa antes da execução. Janela do programa pode aparecer em segundo plano. Painéis visíveis: desmontagem (CPU), registradores, dump e stack.

---

### Imagem — opções de pausa automáticas
[![image.png](https://i.postimg.cc/NFbhbyFw/image.png)](https://postimg.cc/m1c6gg7d)  
Opções em **Options > Preferences** que controlam pontos de parada automáticos. Callbacks TLS e System TLS podem estar marcados. Quando marcados, o depurador pausa automaticamente ao atingir esses callbacks.

---

### Pontos essenciais (resumido)
- Ao abrir, o depurador pausa no início. Use Run / Pause / Step in / Step over para controlar execução.  
- TLS callbacks são eventos comuns onde malware insere rotinas de inicialização ou anti-análise.  
- Dentro do callback, avance instrução a instrução para inspecionar registradores, flags, pilha e chamadas de API.  
- Se identificar rotinas de evasão, interrompa e volte ao snapshot da VM em vez de prosseguir em host não isolado.  

---

## Etapa 7 — Patching defensivo — edição e preenchimento com NOPs

### Imagem 1 — Interface de edição do binário
[![image.png](https://i.postimg.cc/XqWGdPGy/image.png)](https://postimg.cc/8JyzVtb1)  
Janela de edição onde se inspeciona/modifica a instrução alvo no depurador. Mostra o offset/endereços afetados e os bytes atuais.

---

### Imagem 2 — Visualização da instrução e dos opcodes
[![image.png](https://i.postimg.cc/FsRF8m4g/image.png)](https://postimg.cc/t7KHnK7Y)  
Detalhe dos opcodes que compõem a instrução original. Aqui é possível avaliar o tamanho da instrução e opções de substituição conceitual.

---

### Imagem 3 — Preenchimento com NOPs
[![image.png](https://i.postimg.cc/02Tgnq22/image.png)](https://postimg.cc/4mQF4q9r)  
Exemplo de instruções substituídas por NOPs para neutralizar uma chamada ou rotina de evasão. Cada NOP corresponde a um byte que não altera o fluxo útil do programa.

---

## Notas conceituais rápidas
- Editar bytes no depurador permite alterar o fluxo sem depender de mudanças em tempo de execução.  
- Substituir instruções por NOPs neutraliza efeitos sem reescrever lógica maior.  
- Alternativa: transformar salto condicional em salto incondicional para garantir o caminho desejado.  
- Depois de patch aplicado, exporte/salve o binário corrigido para análise posterior.

---

## Boas práticas operacionais
- Sempre usar VMs isoladas, com snapshots antes e depois da execução.  
- Registrar telemetria: chamadas de API, processos, arquivos, e tráfego de rede.  
- Combine análise estática e dinâmica para validar conclusões.  
- Trabalhe com amostras apenas em redes isoladas ou controladas.  
- Documente mudanças e salve cópias dos binários patchados com metadados (data, razão, hash).

---

## Conclusão
A depuração é a ferramenta que transforma observação passiva em controle ativo durante a investigação de malware. Compreender evasões estáticas e dinâmicas, saber usar depuradores como x32dbg/x64dbg e aplicar patching defensivo em ambiente controlado permite revelar comportamentos ocultos e gerar indicadores úteis. Toda a atividade deve priorizar segurança operacional e ética. O propósito é detecção, mitigação e investigação.


