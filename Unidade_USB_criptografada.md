# 🧠 Unidade USB criptografada

> Autor: @Piceli{DuckWare}

Este write-up documenta como resolvi o desafio **"BitLocker + XOR ransomware"**, recebido como cenário forense de recuperação de arquivos. A situação simulava um ataque real: um dispositivo USB havia sido bloqueado com BitLocker e, além disso, os arquivos estavam cifrados com um ransomware rudimentar. Foi necessário aplicar engenharia reversa, descriptografia manual, e desmontagem segura do ambiente.

---

## 🔍 O Desafio

* Arquivo entregue: `encrypted_usb.dd` (imagem bruta de disco)
* Outros arquivos:
  * `recovery_keys_dump.txt` — centenas de chaves de 48 dígitos
  * Arquivos `.xxx.crypt` — PNGs renomeados e cifrados
  * Binário `cryptor` — suposto utilitário do atacante
  * `ransom.txt` — instruções de pagamento e ameaça

---

## 🧪 Etapa 1 – Desbloqueando o Volume BitLocker

A imagem foi inspecionada com:

```bash
hexdump -C -n 512 encrypted_usb.dd | grep 'FVE-FS'
```

Resultado: `FVE-FS` no byte `0x00` → volume BitLocker "raw" sem tabela de partição.

Em seguida:

```bash
sudo losetup /dev/loop30 encrypted_usb.dd --read-only
```

E com a ferramenta `dislocker`, testei a chave:

```bash
sudo dislocker -r -V /dev/loop30 \
  -p334565-564641-129580-248655-292215-551991-326733-393679 \
  -- /tmp/bit_test
```

🟢 **RC=0** — a chave bateu, volume destravado.

Então, desbloqueei totalmente:

```bash
sudo dislocker -V /dev/loop30 -p... -- /tmp/bit
sudo mount -o ro,loop /tmp/bit/dislocker-file /mnt/bit_decrypted
```

Copiei os arquivos:

```bash
rsync -aP /mnt/bit_decrypted/ ./recuperado/
```

---

## 🧬 Etapa 2 – Engenharia Reversa do Ransomware

Analisando o conteúdo:

```bash
file cryptor
sha256sum cryptor
strings -n8 cryptor | head
```

Confirmado:

```
ELF 64-bit LSB PIE executable, stripped
```

Ou seja: binário Linux x86-64 sem símbolos.

Inspecionei os `.crypt` com:

```bash
hexdump -C -n 64 crypto_passphrase.png.xxx.crypt
```

Percebi que os arquivos começavam com:

```
EF 33 21 3E ...
```

Comparei com um PNG padrão (`89 50 4E 47 ...`) e apliquei XOR entre os dois — apareceu a chave:

🔑 **Chave: `"fcoy"`** (`0x66 0x63 0x6F 0x79`)  
Cifra de 4 bytes repetidos via XOR puro, extremamente fraco.

---

## 🛠️ Etapa 3 – Script de Descriptografia

Criei um script simples em Python:

```python
KEY = b"fcoy"
def decrypt(data): return bytes(b ^ KEY[i % 4] for i, b in enumerate(data))
```

Apliquei isso a todos os arquivos:

```bash
python3 decrypt_fcoy.py
```

Todos os `.crypt` viraram arquivos `.png` íntegros e visualizáveis.

---

## 📌 Observações Importantes

* O volume estava travado por BitLocker, não por um ransomware exótico
* A cifra XOR usada era fraca e sem sal ou IV — totalmente reversível
* O binário `cryptor` servia apenas como embaralhador local (não usava OpenSSL, mbedtls, etc.)
* A `ransom.txt` era apenas dissuasiva — não havia cifra assimétrica ou chave remota

---

## 🛠️ Ferramentas que usei

* `dislocker`, `losetup`, `mount`, `rsync` — para montar e extrair dados
* `hexdump`, `file`, `strings` — para inspeção rápida
* `Python` — para automatizar a descriptografia
* `lsof`, `fuser`, `kill` — para desmontar e liberar o loop preso
* `shred`, `rm` — para limpeza segura da imagem

---

## ✅ Conclusão

Esse desafio foi uma ótima junção de análise forense e engenharia reversa leve.  
Com poucos passos e foco nos detalhes, conseguimos:

✅ Destravar o BitLocker com brute-force seguro  
✅ Reverter uma cifra XOR fraca  
✅ Recuperar todos os arquivos com integridade total  
✅ Eliminar com segurança a imagem forense após análise

---

🟢 **Flag:** `247CTF{494f7cceb2baf33a0879543fe673b1ae}`
