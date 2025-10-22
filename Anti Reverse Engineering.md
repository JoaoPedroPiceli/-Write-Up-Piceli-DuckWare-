# Anti-Reverse Engineering — White-Up

**Desafio:** Anti-Reverse Engineering  
**Autor:** @Piceli{DuckWare}  
**Ambiente:** VM FLARE (isolada). Use snapshot.

---

## Objetivo
Identificar e documentar técnicas de engenharia anti-reversa comuns (detecção de VM, anti-debug, packers) e aplicar contramedidas defensivas: análise estática, análise dinâmica controlada, manipulação em memória, patching e desempacotamento.

---

## Pré-requisitos
- Familiaridade com depuração (x32dbg / x64dbg).  
- Noções básicas de assembly (registradores) e C (fluxo condicional).  
- Executar todas as ações em VM isolada com snapshot.

---

## Ferramentas citadas
x32dbg / x64dbg, DetectItEasy (DIE), PEStudio, Scylla (plugin), Process Monitor, Wireshark.

---

## Etapa 2 — Depuração e anti-depuração
Depuração é o processo de inspecionar a execução de um programa para entender seu comportamento. Autores de malware empregam técnicas para dificultar essa análise:

- **Verificação de depurador:** chamada como `IsDebuggerPresent`.  
- **Adulteração de logs de depuração.**  
- **Código auto-modificante.**  
- **Enumeração de janelas/processos** para identificar ferramentas de análise.

### Interface e amostra carregada
Depurador com amostra aberta e painel CPU/desmontagem em foco.  
[![amostra aberta](https://i.postimg.cc/fRPtWCsv/image.png)](https://postimg.cc/ZWrKVrC9)

### Logs do depurador
Registros que indicam inicialização, DLLs mapeadas e eventos do depurador.  
[![logs](https://i.postimg.cc/gJW0rBTq/image.png)](https://postimg.cc/VSKwV41d)

---

## Etapa 4 — Anti-debug: SuspendThread (exemplo prático)
Descrição: rotina que enumera janelas/processos e chama `SuspendThread` em threads relacionadas a depuradores, fazendo o depurador travar.

**Fluxo defensivo resumido**
1. Localizar a chamada a `SuspendThread`.  
2. Neutralizar a chamada substituindo por NOPs (`0x90`) ou editar o salto condicional que leva à chamada.  
3. Exportar patches para reaplicação automática em execuções subsequentes.

**Respostas rápidas**
- API que enumera janelas: `EnumWindows`  
- Opcode NOP (hex): `0x90`  
- Instrução em `0x004011CB`: `call` para `SuspendThread`

### Identificação da técnica
Captura relacionada à identificação da chamada `SuspendThread`.  
[![suspend-thread](https://i.postimg.cc/mr605cQ1/image.png)](https://postimg.cc/McVPcp3z)

### Breakpoints / Pontos de parada
Painel de breakpoints para interceptar chamadas críticas (ex.: TLS, EntryPoint).  
[![breakpoints](https://i.postimg.cc/dtLJQYV6/image.png)](https://postimg.cc/gnFC41RL)

### Patching / Preenchimento com NOPs
Exemplo de edição de instruções para neutralizar a chamada de suspensão.  
[![patching](https://i.postimg.cc/PJHK3hFJ/image.png)](https://postimg.cc/62HdTsKx)

---

## Etapa 5 — Detecção de VM via WMI (Win32_TemperatureProbe)
Técnica: consultar `Win32_TemperatureProbe` via WMI. Em muitas VMs essa classe retorna `Not Supported`, sinalizando virtualização.

**Contramedidas demonstradas**
- **Manipulação de memória:** alterar o valor de retorno (`uReturn`) na memória para influenciar a checagem.  
- **Alteração de fluxo (EIP):** ajustar o registrador EIP para pular diretamente ao bloco desejado.

**Respostas rápidas**
- Consulta WQL usada: `SELECT * FROM Win32_TemperatureProbe`  
- Registrador que indica próxima instrução: **EIP** (x86)  
- Antes da comparação `uReturn == 0`, `[ebp-4]` contém o ponteiro **pclsObj** (objeto WMI)

### Memory Map / Seções
Mapa de memória para localizar buffers e estruturas relevantes.  
[![memory map](https://i.postimg.cc/63jFF8ny/image.png)](https://postimg.cc/gXhMZkMW)

### Call Stack / Threads
Pilhas de chamadas e threads para rastrear o fluxo da checagem WMI.  
[![call stack](https://i.postimg.cc/bY2j4RCC/image.png)](https://postimg.cc/jCK97yrP)

---

## Etapa 6 — Ofuscação e Packers
Ofuscação abrange codificação (XOR, Base64), criptografia e reorganização do código. Packers encapsulam e ofuscam executáveis, tornando a análise estática limitada e forçando a execução para obter a versão desembrulhada em memória.

*(Nenhuma imagem específica nesta etapa; use DetectItEasy / PEStudio para identificação.)*

---

## Etapa 7 — Identificar e desempacotar
**Identificação:** use DetectItEasy e PEStudio. Nomes de seções (ex.: `UPX0/UPX1/UPX2`) e entropia alta são pistas de packers.

**Desempacotamento (fluxo genérico para UPX-like)**
1. Abrir `packed.exe` no depurador.  
2. Localizar o endereço onde o payload já foi desempacotado em memória e definir breakpoint (ex.: `0x004172D4`).  
3. Avançar até o EntryPoint do payload (ex.: `0x00401262`).  
4. Usar **Scylla** (Plugins > Scylla): `Dump` → `IAT Autosearch` → `Get Imports` → `Fix Dump`.  
5. Salvar o arquivo `_SCY` e validar com DetectItEasy / PEStudio.

**Observação:** UPX é simples. Outros packers podem exigir scripts, ferramentas específicas ou processos manuais. Serviços como `unpac.me` podem ajudar.

### Despejo / Desempacotamento (Scylla / IAT)
Tela exemplo do processo de dump e reparo de IAT.  
[![dump iat](https://i.postimg.cc/nzGqLm0Z/image.png)](https://postimg.cc/TLhK7pzH)

---

## Coleta e entregáveis sugeridos
- Binários: `original.exe`, `patched.exe`, `original_SCY.exe`.  
- Hashes (MD5 / SHA256) e metadados (data, motivo do patch).  
- Logs do depurador (breakpoints, timestamps).  
- Captura de tráfego (pcap) se aplicável.  
- IOCs: APIs observadas (`IsDebuggerPresent`, `EnumWindows`, `SuspendThread`, WMI queries), nomes de seções suspeitas, entropia elevada.

---

## Boas práticas operacionais
- Execute todas as análises em VMs isoladas com snapshots.  
- Exporte/import patches para reaplicação.  
- Documente cada alteração e salve artefatos reprodutíveis.  
- Trabalhe apenas em ambientes autorizados e controlados.

---

## Créditos e referências
- Imagens fornecidas pelo autor do desafio.  
- Material de referência: https://tryhackme.com/room/antireverseengineering

---

**Aviso:** conteúdo com finalidade educativa e defensiva. Não contém instruções para criação, distribuição ou ocultação de malware. Trabalhe sempre em ambiente autorizado.
