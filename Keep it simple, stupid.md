# 🧠 Keep it simple, stupid — Análise de Bootloader com Flag Visual

> Autor: @Piceli{DuckWare}

Este write-up documenta como resolvi o desafio **"Keep it simple, stupid"** da H2HC 2024. A flag não estava visível diretamente — ela era exibida graficamente por meio de chamadas da BIOS. Tivemos que explorar o binário, entender o que ele fazia, e por fim **alterar manualmente o código para que a flag aparecesse corretamente**.

---

## 🔍 O Desafio

* Arquivo entregue: `somerandomfile.bin` (512 bytes)
* Indícios: formato MBR boot sector, final `0x55AA`
* Ao executar no QEMU: **nada aparecia**
* A flag está lá... mas não é exibida diretamente via `puts`, `printf`, ou nada do tipo

---

## 🧪 Etapa 1 – Primeiros Testes

Testei com o QEMU direto:

```bash
qemu-system-x86_64 -fda boot.img
```

Resultado: tela preta com nada.

Fui investigar o conteúdo do binário:

```bash
xxd boot.img | less
```

Depois disso usei `ndisasm`:

```bash
ndisasm -b 16 boot.img
```

E vi várias chamadas à `int 0x10`, que são típicas para **serviços de vídeo da BIOS**.

Isso me mostrou que o binário estava tentando desenhar algo na tela — provavelmente a flag — mas **não estava conseguindo por erro de ponteiro** ou de configuração de registradores.

---

## 🧬 Etapa 2 – Engenharia Reversa

O binário usava algo como:

```asm
mov ah, 0x13       ; escrever string
mov al, 0x01       ; modo 1 (atualiza cursor)
mov bh, 0x00       ; page 0
mov bl, 0x0A       ; cor verde
mov cx, 0x08       ; tamanho da string
mov dh, 0x0C       ; linha Y
mov dl, 0x1E       ; coluna X
lea bp, [some_offset] ; ponteiro para string
int 0x10           ; chamada BIOS
```

A instrução crucial é o `int 0x10` com `AH=0x13`, que exibe uma string em modo texto.
Mas o `bp` não estava apontando pro lugar certo da string. Resultado: nada era exibido.

---

## 🛠️ Etapa 3 – Corrigindo o Binário Manualmente

Abrimos o arquivo em modo binário com:

```bash
vim -b boot.img
```

Fomos até o offset onde a instrução `lea bp, [offset]` aparecia. Lá trocamos os bytes pra que o BP apontasse exatamente pro endereço onde estava a string da flag `H2HC4FUN`.

### Exemplo real:

* Offset `0x0022` (hex): estava `8D BE XX XX` (lea bp, \[si-XX])
* Corrigimos manualmente pra que `bp` apontasse para a posição onde sabíamos que a string da flag começava (por exemplo, `0x01F4`)

Também conferimos valores como `cx`, `dh`, `dl` (tamanho da string e posição na tela).

Com isso, salvamos o binário e testamos novamente:

```bash
qemu-system-x86_64 -fda boot.img
```

Resultado:

```
H2HC4FUN
```

Renderizado no centro da tela com cor verde (definido por `bl=0x0A`).

---

## 📌 Observações Importantes

* O desafio se baseia em execução real-mode de 16 bits
* A BIOS interpreta os valores dos registradores diretamente
* A flag estava presente desde o começo, só não era exibida corretamente
* Só foi possível visualizar após redirecionarmos `bp` manualmente

---

## 🛠️ Ferramentas que usei

* `qemu-system-x86_64` — para rodar o binário como disquete
* `xxd`, `vim -b` — para editar bytes
* `ndisasm`, `objdump` — para analisar os opcodes
* `gdb` — para testar dinamicamente os registradores
* `ghidra` — para inspeção mais visual do código e dos dados

---

## ✅ Conclusão

Esse desafio ensina muito sobre arquitetura x86 real-mode e como é possível esconder uma flag de forma criativa. O nome do desafio é literal — a estrutura é simples, mas **exige manipulação manual** e atenção a detalhes.

✅ **Flag:** `H2HC4FUN`

